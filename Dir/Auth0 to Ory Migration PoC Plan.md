# Auth0 to Ory Migration PoC Plan

Nov 25, 2026 · @Mohamed Ahmed

## Overview

The PoC runs on **self-hosted Ory open source** (Kratos, Hydra, Polis) on a dev Kubernetes cluster. It proves three migrations from Auth0 and checks whether parts 1 and 2 can be done with zero downtime.

**Scope**

1. Auth0 database (username/password) users to Ory Kratos. Must validate zero downtime.
2. SAML (enterprise connection) users to Ory Polis + Kratos. Must validate zero downtime.
3. Machine-to-machine (client credentials) apps to Ory Hydra.

**Why self-hosted open source, not the Ory Network free plan**

| Capability | Ory Network Developer (free) | Self-hosted open source |
| --- | --- | --- |
| Part 1: password hash import + password migration hook | Yes | Yes (Kratos v1.3.0 or later) |
| Part 2: SAML | No | Yes, through Ory Polis |
| Part 3: OAuth2 client credentials | Yes | Yes (Hydra) |
| Members / environments | 1 member, 2 dev environments | Unlimited |
| Close to the production target (Ory Enterprise License, self-hosted) | No | Yes: OEL is a drop-in replacement for the open-source builds |

One gap to close: in production, OEL Kratos supports SAML natively, while open source uses Polis as a SAML-to-OIDC bridge. Ask Ory for an **OEL trial license** so the SAML path you test is the one you'll run in production.

## PoC environment

Everything runs on one dev cluster with Helm, next to an Auth0 test tenant and a Keycloak test IdP.

&#91;embedded content: PoC architecture · Ory components, test apps, Auth0 and test IdP\]

Hydra's login and consent flow is connected to Kratos, so the sample web app gets OIDC tokens from Ory, the same way it gets them from Auth0 today.

| Component | Role in the PoC | Chart / image |
| --- | --- | --- |
| Ory Kratos (v1.3.0 or later) | Identities, login, password migration hook | `ory/kratos` Helm chart |
| Ory Hydra | OAuth2/OIDC server for the web app and M2M | `ory/hydra` Helm chart |
| Ory Polis | SAML service provider, exposed to Kratos as OIDC | `ory/polis` image |
| Login UI | Login, registration, recovery screens | Ory Elements or `kratos-selfservice-ui-node` |
| Postgres | Database for Kratos, Hydra, Polis | Bitnami chart or Crossplane RDS |
| Migration hook service | Checks passwords against Auth0 on first login | Sample repo |
| Sample web app, caller and target backends | Test clients | Sample repo |
| Keycloak | Test SAML IdP standing in for a customer IdP | `quay.io/keycloak/keycloak` |
| Auth0 test tenant | Source of users, clients and SAML connection | Existing Terraform |

## Part 1: Auth0 database users

Zero downtime is achievable: import every user before cutover and let passwords move on first login through the password migration hook. Users sign in once more at cutover because sessions can't be migrated.

**Two ways to move passwords**

|  | Option A: bulk hash import | Option B: password migration hook |
| --- | --- | --- |
| How | Import bcrypt hashes from Auth0 (`$2b$10$…`) into `credentials.password.config.hashed_password` | Import with `use_password_migration_hook: true` and an empty `hashed_password`. On first login Kratos calls your hook, which checks the password against Auth0 |
| Needs | An Auth0 support ticket for the hash export; paid Auth0 plan only; no committed turnaround | Password grant enabled on one Auth0 app; Auth0 stays up during the transition |
| Password changed in Auth0 after export | Stale hash, login fails | Checked live, works |
| Auth0 can be shut down | Right after import | After the transition window, plus a reset for users who never signed in |
| PoC use | Test with a small sample if a hash export is available | Main path |

The hook's contract: Kratos sends `identifier` and `password`; the hook answers `200 {"status":"password_match"}` (Kratos stores the hash and logs the user in, and never calls the hook again for that identity) or `403` for a wrong password. Config lives at `selfservice.methods.password.config.migrate_hook`. It was added in Kratos v1.3.0; v1.2 rejects that key.

**PoC steps**

1. Create 20 to 50 test users in the Auth0 test tenant's database connection, including some with MFA and some unverified.
2. Export them with the Management API (users export job).
3. Write the import script: map Auth0 fields to the Kratos identity schema, keep the Auth0 `user_id` in `metadata_admin`, set `verifiable_addresses` from `email_verified`.
4. Import in batches with `PATCH /admin/identities`, using Option B for most users and Option A for a sample if hashes are available.
5. Build the migration hook service: call Auth0 `/oauth/token` with `grant_type=password` (realm = the database connection), return `password_match` or `403`.
6. Point the sample web app's login at Hydra + Kratos and test: migrated login, wrong password, password changed in Auth0 after import, user created in Auth0 after the export (delta import), recovery flow for a user who never migrated.
7. Run a cutover rehearsal: switch the web app's OIDC issuer from Auth0 to Hydra with a rolling deploy, and note what users see.

**Zero-downtime conditions to check**

- Sessions: Ory can't import active sessions, so every user signs in once at cutover. Service stays up.
- Delta users: signups in Auth0 after the export need a second import run, or Auth0 signups are frozen at cutover.
- APIs that check user tokens must trust both issuers (Auth0 and Hydra) during the switch.
- MFA enrollments don't move. Users re-enroll in Ory.
- Identity IDs change. Downstream systems map users through the Auth0 `user_id` stored in `metadata_admin`.

## Part 2: SAML users

Zero downtime is achievable per connection, but each customer's IdP admin must register Ory as a new SAML service provider. No passwords move, because these users authenticate at their own IdP.

**How it works in open source**

Kratos doesn't speak SAML. Ory Polis acts as the SAML service provider and exposes each connection to Kratos as an OAuth2/OIDC provider: customer IdP → Polis (SAML) → Kratos (generic OIDC provider) → Hydra → app.

**Biggest risk, test first:** Polis expects `tenant` and `product` context on its authorize call, and Kratos's generic OIDC provider can't add custom query parameters. One Ory community answer says the direct hookup isn't supported out of the box; an Ory team member says the generic OIDC provider works. Validate this on day one, e.g. using the per-connection client ID Polis issues. If it fails, the fallbacks are a thin proxy in front of Polis, or testing OEL Kratos's native SAML with a trial license.

**PoC steps**

1. Deploy Polis with its own Postgres database and admin API key.
2. Run Keycloak as the test IdP: create a realm, a few users, and a SAML client for Polis.
3. Create a Polis SSO connection from Keycloak's SAML metadata; note the ACS URL and entity ID Polis gives you.
4. Add Polis to Kratos as a generic OIDC provider, with a Jsonnet mapper for email, name and groups claims.
5. Pre-create the SAML users in Kratos with OIDC credentials (provider id + the subject Polis returns) so first login lands on the existing identity, not a duplicate.
6. Mirror today's setup: put the same Keycloak users behind an Auth0 SAML connection, so you can switch one connection from Auth0 to Ory and back.
7. Test: first login (no duplicate identity), claims and groups in the app's token, IdP-initiated login if your customers use it, logout.

**Zero-downtime conditions to check**

- The new SP (ACS URL, entity ID, signing certificate) is added in the customer IdP **next to** Auth0, not replacing it. Each connection switches when its customer is ready.
- Users sign in once more at cutover (no session import).
- The subject Polis returns must match what you pre-imported, or account linking by email must be configured; otherwise users get duplicate accounts.
- IdP-initiated SSO needs a separate test, because the IdP posts straight to the new ACS URL.

## Part 3: Machine-to-machine apps

Zero downtime comes from a dual-trust period: target backends accept tokens from both Auth0 and Hydra while callers move over one at a time.

**Hydra settings that must match Auth0**

| Setting | Why |
| --- | --- |
| `strategies.access_token: jwt` | Hydra issues opaque tokens by default; backends validate JWTs locally today |
| `strategies.jwt.scope_claim` | Auth0 sends scopes as a `scope` string; Hydra defaults to an `scp` array |
| Token hook | Recreates custom claims added today by Auth0 Actions |
| Client `audience` and `scope` | Same values as the Auth0 API identifier and permissions |
| `token_endpoint_auth_method: client_secret_post` | Matches apps that send credentials in the body; Hydra defaults to `client_secret_basic` |
| Same `client_id` / `client_secret` | Hydra's admin API lets you set both, so no secret rotation is needed |

Note: the pricing page says open source lacks "stateless JWT access tokens". That's an OEL extra that skips storing JWTs in the database; the JWT strategy itself works in open source.

**PoC steps**

1. Deploy Hydra with the settings above.
2. Create the caller's client in Hydra, either with Hydra Maester `OAuth2Client` CRDs (fits the Kyverno setup) or with a script calling the admin API.
3. Configure the target backend to trust two issuers (two issuer/JWKS pairs) and accept both scope formats.
4. Decode one Auth0 token and one Hydra token side by side; compare `iss`, `aud` (string vs array), `scope`/`scp`, `exp`, custom claims.
5. Switch the caller's token URL to Hydra with a rolling deploy; confirm calls keep working mid-rollout.
6. Roll back (restore the Auth0 config) to prove it's a config-only change.
7. Remove Auth0 trust from the target after one token lifetime (24 h by default in Auth0).

## Success criteria, risks and open questions

The PoC passes when all three parts migrate test users or clients with no failed requests and, at most, one extra sign-in per user.

**Success criteria**

- [ ] Part 1: users imported with the hook flag sign in with their Auth0 password, migrate silently, and the hook isn't called again
- [ ] Part 1: users with an imported hash sign in directly; users who never migrated can reset their password
- [ ] Part 2: a Keycloak user signs in through Polis and Kratos, lands on the pre-created identity, and the app gets the expected claims
- [ ] Part 3: the caller switches to Hydra mid-traffic with zero failed calls, and rollback is a config change
- [ ] Cutover rehearsal documented: user impact per part (expected: one re-login for parts 1 and 2, none for part 3)

**Risks**

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Polis can't be used as a Kratos generic OIDC provider | Part 2 blocked | Test first; fallback proxy or OEL native SAML trial |
| Auth0 hash export slow or unavailable | Option A blocked | Option B (hook) needs no export |
| Token claim differences (`scope` vs `scp`, `aud` format) | Backends reject Hydra tokens | Side-by-side token comparison; config Hydra to match |
| Customer IdP admins slow to add the new SP | Part 2 cutover drags | Start outreach early; switch per connection |
| Open source upgrades need downtime (OEL doesn't) | Not representative of prod ops | Note it; test upgrades on OEL later |

**Open questions**

- Can Ory provide an OEL trial license for the PoC?
- Is the Auth0 tenant on a paid plan, so a hash export can be requested?
- Do any customers use IdP-initiated SAML login?
- Which Auth0 Actions add custom claims that Hydra and Kratos must reproduce?
- How many users have MFA enrolled today?

## Jira stories

One story per part, plus an environment story the other three depend on.

### Story 0: PoC environment

**Title:** \[Ory PoC\] Set up self-hosted Ory environment and sample apps

**Description:** As the platform team, we need a dev environment running Ory Kratos, Hydra and Polis, so we can test migration from Auth0. Deploy with Helm on the PoC cluster with Postgres, a login UI, a Keycloak test IdP, and the sample apps repo (web app, caller backend, target backend, migration hook service). Connect Hydra's login and consent flow to Kratos. Use an Auth0 test tenant as the source.

**Acceptance criteria**

- Kratos (v1.3.0 or later), Hydra, Polis and Postgres run on the PoC cluster from Helm values in Git
- Sample web app can register and sign in a new user through Hydra + Kratos
- Keycloak and the Auth0 test tenant are reachable from the cluster

### Story 1: Database users

**Title:** \[Ory PoC\] Migrate Auth0 database users to Ory Kratos with zero downtime

**Description:** Validate moving Auth0 database (email/password) users to Kratos without users resetting passwords or losing access. Export users with the Auth0 Management API and import them with `PATCH /admin/identities`. Main path: the Kratos password migration hook, which checks each password against Auth0 on first login. Also test bulk bcrypt hash import on a sample if an Auth0 hash export is available. Keep the Auth0 `user_id` in `metadata_admin`. Rehearse the cutover and document user impact.

**Acceptance criteria**

- Hook-flagged users sign in with their Auth0 password and migrate; the hook isn't called again
- Users created or with changed passwords in Auth0 after the export are handled (delta import, live check)
- Users who never migrate can recover their account
- Cutover rehearsal shows no failed logins beyond one extra sign-in; zero-downtime verdict recorded

### Story 2: SAML users

**Title:** \[Ory PoC\] Migrate SAML enterprise connection users to Ory Polis + Kratos with zero downtime

**Description:** Validate moving SAML-federated users from an Auth0 enterprise connection to Ory, using Ory Polis as SAML service provider and Kratos as identity store. First task: confirm Polis works as a Kratos generic OIDC provider (tenant/product routing). Use Keycloak as the customer IdP, pre-create users in Kratos to avoid duplicates, and switch one connection from Auth0 to Ory and back. Document what customer IdP admins must change.

**Acceptance criteria**

- Polis to Kratos integration works, or the blocker and fallback are documented
- A Keycloak user signs in through Ory and lands on the pre-created identity with expected claims
- Auth0 and Ory work side by side for the same IdP during switch-over
- Customer-facing change list (ACS URL, entity ID, certificate) and zero-downtime verdict recorded

### Story 3: M2M apps

**Title:** \[Ory PoC\] Migrate Auth0 M2M clients to Ory Hydra with zero downtime

**Description:** Validate moving machine-to-machine apps from Auth0 client credentials to Ory Hydra. Configure Hydra for JWT access tokens with Auth0-compatible claims, recreate the client with the same credentials (Hydra Maester CRD or admin API), make the target backend trust both issuers, and switch the caller with a rolling deploy.

**Acceptance criteria**

- Hydra tokens pass the target backend's validation (issuer, audience, scopes)
- Caller switches from Auth0 to Hydra mid-traffic with zero failed calls
- Rollback to Auth0 is a config-only change
- Client management approach (Maester CRD vs admin API) recommended

## Sample apps repo

Build the runnable code (Dockerfiles, Helm values, apps) with Claude Code in a Git repo, not in this doc. It can run `docker compose` and `kind` locally, fix what fails, and commit. This section gives the layout and a prompt to hand it.

**Repo layout**

```
ory-poc/
  docker-compose.yml          # local loop: everything below + postgres + keycloak
  helm/                        # values for kratos, hydra, polis, postgres, keycloak
  ory/kratos/                  # kratos.yml, identity.schema.json, oidc mapper (jsonnet)
  ory/hydra/                   # hydra.yml (jwt strategy, scope_claim, token hook)
  apps/web-app/                # Node/Express OIDC client, Dockerfile
  apps/caller-backend/         # client credentials, token caching, Dockerfile
  apps/target-backend/         # JWT validation, two issuers, scope per route, Dockerfile
  apps/migration-hook/         # Kratos password migration hook -> Auth0 password grant, Dockerfile
  scripts/auth0-export.sh      # Management API users export job
  scripts/import-to-kratos.py  # Auth0 export -> PATCH /admin/identities batches
  keycloak/realm-export.json   # test realm, users, SAML client for Polis
  k8s/                         # Deployments, Services, Hydra Maester OAuth2Client CRDs
  README.md                    # how to run each part's tests
```

**Prompt for Claude Code**

```markdown
Create a repo `ory-poc` for a PoC migrating from Auth0 to self-hosted Ory open source.
Target: docker compose for local runs, plus Helm values and manifests for a kind cluster.

Components:
- Ory Kratos v1.3+ (Postgres), identity schema with email + name, metadata_admin.auth0_user_id.
  Enable selfservice.methods.password.config.migrate_hook pointing at apps/migration-hook.
  Add Ory Polis as a generic OIDC provider with a Jsonnet mapper (email, name, groups).
- Ory Hydra (Postgres): strategies.access_token=jwt, strategies.jwt.scope_claim matching Auth0's
  `scope` string, login/consent connected to Kratos via a login UI (Ory Elements or kratos-selfservice-ui-node).
- Ory Polis (Postgres) with an admin API key.
- Keycloak with a realm export: 3 users and a SAML client for Polis.

Apps (Node 20, Express, each with a small multi-stage Dockerfile, non-root user):
1. web-app: OIDC login (openid-client). Issuer configurable so it can point at Auth0 or Hydra.
2. caller-backend: client credentials grant, caches the token until near expiry, calls target-backend
   every 2 seconds and logs failures. Token URL, client id/secret, audience from env.
3. target-backend: validates JWTs from TWO issuers (Auth0 and Hydra) via each JWKS, checks aud,
   requires scope read:orders on GET /orders; accepts scopes as `scope` string or `scp` array.
4. migration-hook: implements the Kratos password migration hook. Receives identifier + password,
   calls Auth0 /oauth/token with grant_type=password (realm from env), returns
   200 {"status":"password_match"} or 403. Protect with an API key header.

Scripts:
- scripts/auth0-export.sh: start a Management API users export job, download and gunzip it.
- scripts/import-to-kratos.py: map the export to Kratos identities, batch PATCH /admin/identities,
  default use_password_migration_hook=true, optional hashed_password from an Auth0 hash export file.

Also: Hydra Maester OAuth2Client CRD for the caller, a README with test steps for each part
(database users, SAML users, M2M), and a .env.example. Never commit real secrets.
Run docker compose up and fix errors until all services are healthy.
```

## References

- [Ory pricing](https://www.ory.com/pricing): plan features (SAML, client credentials, environments)
- [Import identities](https://www.ory.com/docs/kratos/manage-identities/import-user-accounts-identities): bulk import, hash formats, password migration hook
- [Migrate from Auth0 to Ory](https://www.ory.com/docs/migrate-to-ory/migrate-from-auth0): Auth0 export and hash export ticket
- [Kratos v1.3.0 release notes](https://newreleases.io/project/github/ory/kratos/release/v1.3.0-pre.0): password migration hook added to open source
- [Ory Polis](https://www.ory.com/docs/polis) and [Generic SAML](https://www.ory.com/docs/integrates-with/generic-protocols/generic-saml): SAML through Polis
- [Kratos + Polis discussion](https://archive.ory.sh/t/30216757/we-re-trying-integrating-ory-kratos-and-ory-polis-our-plan-w): tenant/product routing caveat
- [Ory Enterprise License](https://www.ory.com/docs/self-hosted/oel): OEL vs open source
- [JWT access tokens](https://www.ory.com/docs/oauth2-oidc/jwt-access-token) and [Hydra Maester chart](https://k8s.ory.sh/helm/hydra-maester.html)
