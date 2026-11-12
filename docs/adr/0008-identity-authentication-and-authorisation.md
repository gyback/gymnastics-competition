---
status: accepted
date: 2026-11-11
decision-makers: Gustav Gybäck
---

# Identity, authentication and authorisation — Microsoft Entra External ID via OIDC, cookie-based ASP.NET Core BFF and resource-scoped permission policies

<!-- Related issue: #6 -->

## Context and Problem Statement

User management is part of the [TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) MVP. Club administrators, coaches and competition organisers sign in to manage **Teams** in the sections **Women**, **Men**, **Mixed** and **Open**, apply to competitions, and build rotation **Schedules** across **Floor**, **Tumble** and **Trampet**. TeamGym gymnasts, especially in Youth and Junior classes, are frequently **minors**. This raises the bar for privacy, data minimisation and access control.

Earlier ADRs have already fixed the surrounding constraints:

- [ADR-0004](0004-backend-runtime-and-architectural-style.md): one `.NET 10` modular monolith (`TeamGym.Arena.Api`) with a `Users` module responsible for identity claims mapping and role assignments.
- [ADR-0005](0005-frontend-framework-and-api-contract.md): a React SPA served same-origin by `TeamGym.Arena.Api` acting as the **Backend-for-Frontend (BFF)**. The browser never holds OAuth/OIDC tokens and authenticates with an `HttpOnly` session cookie via `/bff/login`, `/bff/logout` and `/bff/user`.
- [ADR-0006](0006-tenancy-and-organisation-model.md): a shared database where every resource has an explicit owning `OrganisationId`, plus many-to-many `OrganisationMembership` with organisation- and team-scoped roles (`Admin`, `Coach`, `CompetitionOrganiser`). Isolation comes from **explicit resource-scoped access rules**, not a global `TenantId` filter.
- [ADR-0007](0007-primary-data-store-and-data-access.md): PostgreSQL in `swedencentral` with a `users` schema (`users`, `external_identities`, `platform_role_assignments`) and GDPR retention policies.

We need to decide:

1. **The identity provider (IdP)** and the sign-in methods offered.
2. **How the BFF authenticates users** and maintains the browser session.
3. **How external identities map to platform users**, and which personal data we store about account holders.
4. **The authorisation model**: how "club admin for club A, coach for team T, organiser for competition X" is expressed and enforced.
5. **Who can invite whom**, and whether gymnasts have accounts in the MVP.
6. **How local development and automated tests authenticate** without cloud accounts.

## Decision Drivers

- **Never build password handling ourselves** when a standard, secure option exists (OpenID Connect / OAuth 2.x).
- **E-mail sign-up and common social logins now; Swedish BankID later** without re-architecting.
- **Resource-scoped authorisation** that matches the ownership and cross-organisation visibility rules in [ADR-0006](0006-tenancy-and-organisation-model.md).
- **GDPR:** minimise personal data, keep EU data residency, support consent and deletion. Protect minor gymnasts' data above all.
- **Low cost at small scale and low operational load** for a solo maintainer.
- **Works for local development and open-source contributors without cloud accounts**, and for fast, deterministic automated tests.
- **Standard OIDC only**, so the IdP can be swapped without rewriting the application.
- **Easy for humans and AI agents to apply correctly** ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)): secure by default, with one obvious way to protect an endpoint.

## Considered Options

### 1. Identity Provider

- **Option I1:** **Microsoft Entra External ID** (external tenant, Azure-native customer identity and access management).
- **Option I2:** **ASP.NET Core Identity (own user store) + OpenIddict or Duende IdentityServer.**
- **Option I3:** **Keycloak**, self-hosted on Azure.
- **Option I4:** **Third-party SaaS** (Auth0, Clerk, …).

### 2. Authorisation Model

- **Option A1:** **Role-based access with roles carried as token claims** (`[Authorize(Roles = "ClubAdmin")]`), with roles stored in the IdP.
- **Option A2:** **Resource-based ASP.NET Core authorisation policies over permissions**, derived from `OrganisationMembership` and competition staff assignments loaded from our database per request.
- **Option A3:** **Relationship-based access control (ReBAC)** with an external engine such as OpenFGA.

### 3. Local Development and Test Authentication

- **Option L1:** **Keycloak container orchestrated by `.NET Aspire`** with a seeded realm for local development, and a **test authentication handler** in integration tests.
- **Option L2:** **Test authentication handler only**: a fake "pick a dev user" sign-in locally and the same handler in tests.
- **Option L3:** **A shared cloud development tenant** in Entra External ID used by all contributors.

## Decision Outcome

Chosen options:

1. **Identity provider: Option I1.** Use **Microsoft Entra External ID** (an external tenant located in the Europe geography) as the production IdP. The application integrates with it **only via standard OIDC** (authorization code flow with PKCE).
2. **Authorisation: Option A2.** Use **resource-based ASP.NET Core authorisation policies over fine-grained permissions**. Permissions are derived per request from `OrganisationMembership` and competition staff assignments in **our database**. The IdP stores no application roles.
3. **Local development and tests: Option L1.** Use **Keycloak in `.NET Aspire`** with a seeded realm for real OIDC sign-in locally, and an **ASP.NET Core test authentication handler** for integration tests.

### 1. Identity Provider and Sign-In Methods

- **Microsoft Entra External ID** runs as a dedicated **external tenant** (separate from any workforce tenant), created in the **Europe** data location to keep account data inside the EU/EEA.
- **MVP sign-in methods:**
  - **E-mail with one-time passcode** (passwordless) and **e-mail + password**. Entra handles password storage, reset, breach protection and lockout, not us.
  - **Google** and **Microsoft** accounts as social identity providers. Apple can be added through configuration when needed.
- **Later: Swedish BankID.** BankID is added as a **custom OIDC identity provider federated into the Entra external tenant** through a Swedish e-ID broker. The application does not change, because it still only talks OIDC to Entra. This is needed only if a use case requires strong identification, for example federation officials verifying licences. It needs its own ADR because of the cost and the `personnummer` processing it implies ([ADR-0007](0007-primary-data-store-and-data-access.md)).
- **MFA:** Mandatory for platform administrators, enforced by a Conditional Access policy in the external tenant. All other users can opt in. Revisit making MFA mandatory for `OrganisationAdmin`s once real clubs are on the platform.
- **Cost:** Entra External ID's free tier covers the first 50,000 monthly active users at the time of writing, which is far beyond the expected MVP scale.
- **Provisioning:** The external tenant, app registration, user flows and Conditional Access policy are provisioned and documented in the **private IaC repository** ([ADR-0002](0002-repository-strategy.md), #8). This repository holds only configuration keys and no tenant-specific secrets.

### 2. BFF Authentication and Session

`TeamGym.Arena.Api` acts as a **confidential OIDC client** and uses only built-in ASP.NET Core handlers: `AddAuthentication().AddCookie().AddOpenIdConnect()`. We implement the three `/bff/*` endpoints ourselves instead of taking a dependency on a commercially licensed BFF library.

| Concern | Decision |
| --- | --- |
| **Flow** | Authorization code + PKCE, `response_type=code`, scopes `openid profile email`. The client authenticates to the IdP with a certificate or federated credential held in Azure Key Vault in production, and an Aspire-generated client secret locally. |
| **Tokens** | The ID token is validated and used only to establish the session. **No access or refresh tokens are stored** (`SaveTokens = false`), because the SPA calls the same process that holds the session and no downstream APIs are called on the user's behalf in the MVP. If a downstream API is added later, tokens are stored server-side, never in the browser. |
| **Session cookie** | `__Host-teamgym-session`, `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, no `Domain`. Sliding expiry of **8 hours** with an absolute maximum of **7 days**, after which the user re-authenticates (silently if the IdP session is still valid). The cookie holds only a minimal principal (`UserId`, authentication time and IdP session id), never memberships or roles. |
| **Cookie encryption keys** | ASP.NET Core Data Protection keys are persisted to Azure Blob Storage and protected with Azure Key Vault in production (#8), so every container replica can decrypt the session cookie. Locally, the default file-system key ring is used. |
| **CSRF** | `SameSite=Lax`, plus a mandatory custom request header (`X-CSRF: 1`) on all state-changing `/api/*` and `/bff/*` requests, enforced by middleware. The generated `openapi-fetch` client ([ADR-0005](0005-frontend-framework-and-api-contract.md)) adds the header automatically. Because no CORS policy is configured, a cross-site page cannot send the header. |
| **API behaviour** | `/api/*` endpoints return `401 Unauthorized` / `403 Forbidden` instead of redirecting to the IdP. This is the default for API endpoints with cookie authentication in ASP.NET Core 10, and is verified by tests. |
| **Endpoints** | `GET /bff/login?returnUrl=…` (only local, relative return URLs are accepted, to prevent open redirects), `POST /bff/logout` (clears the cookie and calls the IdP's end-session endpoint), `GET /bff/user` (returns `UserId`, display name, preferred language and the user's organisation memberships for the SPA's organisation switcher and route guards). |

### 3. User Identity Mapping and Stored Personal Data

Within the `Users` module (`users` schema, [ADR-0007](0007-primary-data-store-and-data-access.md)):

```text
User                                   # users.users
├── Id: UserId (UUIDv7)                # Our own stable id, used in every other module
├── DisplayName: string
├── Email: string                      # Verified e-mail from the IdP, used for invitations and notifications
├── PreferredLanguage: "sv" | "en"
├── CreatedAtUtc / LastSignInAtUtc
└── Status: Active | Deleted

ExternalIdentity                       # users.external_identities
├── UserId: UserId
├── Issuer: string                     # The IdP issuer (Entra tenant, or local Keycloak realm)
└── Subject: string                    # Stable subject per issuer: Entra `oid`, otherwise OIDC `sub`
    UNIQUE (Issuer, Subject)

PlatformRoleAssignment                 # users.platform_role_assignments
├── UserId: UserId
└── Role: PlatformAdmin
```

- **Just-in-time provisioning:** On the first successful sign-in (`OnTokenValidated`), the BFF looks up `(issuer, subject)` and creates a `User` if none exists. Every other module refers only to `UserId`, never to IdP identifiers. This keeps the IdP swappable: a migration only re-links `external_identities`.
- **E-mail is not an identity key.** Accounts are never merged automatically by e-mail address. An e-mail address is trusted only when the IdP asserts it as verified.
- **Data minimisation:** We store only the fields above about account holders. We store no IdP tokens, phone numbers, addresses, birth dates or profile photos.
- **Account holders must be at least 16 years old.** This is stated in the terms of use and confirmed at first sign-in, and no birth date is stored. Revisit if clubs need accounts for younger youth leaders.
- **Account deletion (GDPR Art. 17):** A user can delete their own account. This removes their `User` row, external identities and memberships, deletes the user object in the Entra external tenant through Microsoft Graph, and pseudonymises them (`Deleted user`) in audit and history records. Deletion is blocked only when the user is the **last admin** of an organisation, until they hand over or archive the organisation.

### 4. Authorisation Model

**Secure by default.** A **fallback authorisation policy** requires an authenticated user on every endpoint. Public endpoints, such as the published competition list and published schedules, must opt out explicitly with `.AllowAnonymous()`.

**Permissions, not roles, in code.** Endpoints and handlers check **permissions**. A single, unit-tested static map in the `Organisations` module translates roles into permissions:

| Role (scope) | Example permissions |
| --- | --- |
| `Admin` (organisation) | `Organisation.Manage`, `Organisation.ManageMembers`, `Team.Manage`, `Gymnast.Read`, `Gymnast.Manage`, `Application.Submit`, `Competition.Manage`, `Application.Review`, `Schedule.Manage` |
| `Coach` (organisation, optionally limited to `TeamIds`) | `Team.Read`, `Team.Manage`, `Gymnast.Read`, `Gymnast.Manage`, `Application.Submit` for the assigned teams only (all of the club's teams if `TeamIds` is empty) |
| `CompetitionOrganiser` (organisation) | `Competition.Manage`, `Application.Review`, `Schedule.Manage` for competitions hosted by the organisation |
| `CoOrganiser` (single competition, via `CompetitionStaffAssignment`) | `Competition.Manage`, `Application.Review`, `Schedule.Manage` for that competition only, including users from other organisations |
| `PlatformAdmin` (platform) | Manage organisations (seed federations and regions, archive abusive clubs) and platform role assignments. **No implicit access to club gymnast data.** Support access to club data is out of scope for the MVP. |

**Membership is read from our database, not from token claims.** The `Organisations` module exposes an `IOrganisationAccess` service in `TeamGym.Arena.Modules.Organisations.Contracts` ([ADR-0004](0004-backend-runtime-and-architectural-style.md)). The service loads the current user's memberships once per request (scoped cache) and answers questions such as `HasPermissionAsync(organisationId, permission, teamId?)`. Competition staff assignments live in the `Competitions` module and are exposed in the same way. As a result:

- Revoking a membership takes effect on the **next request**, not when a token expires.
- The session cookie and tokens stay small, and the IdP holds no application-specific roles.

**Two layers of enforcement**, both using standard ASP.NET Core authorisation:

1. **Endpoint policies for organisation-scoped routes.** Routes carry an explicit `{organisationId}` ([ADR-0006](0006-tenancy-and-organisation-model.md)), for example `/api/v1/organisations/{organisationId}/teams`. A reusable policy such as `RequireOrganisationPermission(Permission.TeamManage)` checks the route value before the handler runs.
2. **Resource-based checks in handlers.** For resources whose owner is known only after loading them (a `CompetitionApplication` shared between the applying club and the organiser, or a draft `Competition`), handlers call `IAuthorizationService.AuthorizeAsync(user, resource, requirement)`, which runs a module-local `AuthorizationHandler<TRequirement, TResource>`. Query handlers also include the ownership predicate in the SQL query itself (`WHERE club_organisation_id = @organisationId`), so lists can never contain another organisation's data.

**Response codes:** A resource that the user is not allowed to *see* returns **`404 Not Found`**, so that the existence of other clubs' gymnasts, teams or draft applications is not revealed. A resource that the user can see but not change returns **`403 Forbidden`**.

**ReBAC later.** OpenFGA or a similar engine is deferred. Revisit when delegated federation → region → club administration ([ADR-0006](0006-tenancy-and-organisation-model.md)) or judge panels make permission rules hierarchical. The `IOrganisationAccess` abstraction is the seam where such an engine would plug in.

### 5. Accounts, Invitations and Answers to Open Questions

1. **Do gymnasts need accounts in the MVP?**
   - **No.** Gymnasts, and their parents or guardians, are **records managed by their club** ([ADR-0006](0006-tenancy-and-organisation-model.md), [ADR-0007](0007-primary-data-store-and-data-access.md)), not platform users. This avoids collecting consent from, or creating accounts for, minors. Revisit if gymnast- or guardian-facing features (such as personal schedules or music uploads) are planned.
2. **Who can invite whom?**

   | Inviter | Can invite | Scope |
   | --- | --- | --- |
   | Any signed-in user | — | Can **create a new `Club`** organisation and becomes its first `Admin`. Platform admins can archive misuse. `Region` and `Federation` organisations are created by a `PlatformAdmin` in the MVP. |
   | `Admin` of an organisation | `Admin`, `Coach` (optionally limited to specific teams), `CompetitionOrganiser` | Own organisation |
   | `Admin` / `CompetitionOrganiser` of the hosting organisation | `CoOrganiser` for a single competition, including users from other organisations | One competition |
   | — (later, `Scoring` module) | Judges and competition-day officials | One competition, through the same competition staff mechanism |

   - Invitations are sent to an **e-mail address** and contain a **single-use, 7-day token**. Only a hash of the token is stored.
   - To accept, the user must be signed in with an account whose **verified e-mail matches the invited address** (case-insensitive). If the addresses differ, the inviter sends a new invitation to the correct address. This prevents a forwarded link from granting access to minors' data.
   - An organisation must always keep at least **one `Admin`**.
   - Membership, role and invitation changes are recorded as audit entries (actor, target, action, timestamp) in the owning module.
   - The e-mail delivery provider is decided together with hosting (#8). Locally, e-mails are captured by a Mailpit container in `.NET Aspire`.

### 6. Local Development and Automated Tests

- **Local development (`dotnet run --project src/AppHost`):**
  - `.NET Aspire` starts a **Keycloak** container (`Aspire.Hosting.Keycloak`) that imports a checked-in realm (`src/AppHost/Keycloak/teamgym-realm.json`).
  - The realm contains seeded test users such as `club-admin@example.test`, `coach@example.test`, `organiser@example.test` and `platform-admin@example.test`, each matched by seed data to memberships in the local database.
  - `TeamGym.Arena.Api` receives the Keycloak authority and a generated client secret through Aspire configuration. The same OIDC code path runs locally as in production, only the `Authority`, `ClientId` and credential differ. No cloud account is needed.
- **Integration tests (`tests/Api.IntegrationTests`, [ADR-0003](0003-engineering-workflow-and-quality-gates.md)):**
  - The `WebApplicationFactory` replaces the cookie and OIDC schemes with a **`TestAuthHandler`** that authenticates a seeded `UserId` chosen per request, for example `client.AsUser(TestUsers.ClubAAdmin)`.
  - Memberships are seeded in PostgreSQL (Testcontainers), so the **real `IOrganisationAccess` and authorisation handlers** are exercised. The cross-organisation test matrix from [ADR-0006](0006-tenancy-and-organisation-model.md) runs against them.
  - A small set of **BFF tests** covers cookie flags, CSRF header enforcement, `401` instead of redirects on `/api/*`, and `returnUrl` validation.
  - One **OIDC smoke test** runs the full login flow against a Keycloak Testcontainer.
- **Configuration keys** (`Authentication:Oidc:Authority`, `ClientId`, credential, `SubjectClaim`) come from Aspire locally and from environment variables and Key Vault in production. Nothing tenant-specific is committed.

### Consequences

- Good, because we never store or handle passwords. Password reset, breach detection, MFA and social logins are handled by a managed IdP at no cost at our scale.
- Good, because the application depends only on standard OIDC and our own `UserId`. Moving to Keycloak, another CIAM product, or adding BankID through federation requires configuration and identity re-linking, not code changes in the modules.
- Good, because the BFF keeps every token out of the browser ([ADR-0005](0005-frontend-framework-and-api-contract.md)), and the session cookie carries no authorisation data.
- Good, because permissions derived from database memberships enforce [ADR-0006](0006-tenancy-and-organisation-model.md)'s visibility rules precisely, and revoking access takes effect immediately.
- Good, because the fallback policy, one permission map and standard ASP.NET Core authorisation APIs give humans and AI agents one obvious, testable way to protect endpoints.
- Good, because contributors get real OIDC locally through Keycloak without any cloud account, while integration tests stay fast and deterministic.
- Bad, because local development (Keycloak) and production (Entra) use different IdPs, so IdP-specific behaviour (claim names, user flows, logout) is verified in production-like environments only. This is mitigated by the configurable `SubjectClaim`, the OIDC smoke test, and a staging environment against the real external tenant (#8).
- Bad, because loading memberships from the database adds one query per authenticated request. This is acceptable at MVP scale and can be cached briefly in memory with invalidation on membership changes if needed.
- Bad, because the Entra external tenant ties identity to Microsoft. This is accepted because the integration is OIDC-only and user data in our database is keyed by our own `UserId`.

### Confirmation

- **Architecture tests (`tests/ArchitectureTests`, #19):** Assert that the fallback authorisation policy is configured, that every endpoint marked `AllowAnonymous` is on an explicit allow-list, that modules access memberships only through `Organisations.Contracts`, and that no module references IdP-specific claim types outside `Api` and `Users`.
- **Unit tests:** The role → permission map, invitation rules (expiry, single use, e-mail match, last-admin protection) and the authorisation handlers' decision logic.
- **Integration tests:** Every endpoint that reads or changes organisation-owned data includes the cross-organisation matrix ([ADR-0006](0006-tenancy-and-organisation-model.md)) for allowed users, users from another club (`404`/`403`), and anonymous users (`401`). BFF tests verify cookie attributes, CSRF enforcement and redirect behaviour.
- **Code review:** The PR template's ADR compliance check ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)) covers new permissions and any new `AllowAnonymous` endpoint.

## Pros and Cons of the Options

### Identity Provider

#### Option I1: Microsoft Entra External ID

- Good, because it is a managed CIAM service with e-mail one-time passcode, e-mail + password, social logins, MFA, branding and Swedish/English user flows, without any servers for us to run or patch.
- Good, because the external tenant can be located in Europe, matching the EU data residency of our PostgreSQL database in `swedencentral` ([ADR-0007](0007-primary-data-store-and-data-access.md)).
- Good, because the free tier (first 50,000 monthly active users) makes it effectively free at our scale.
- Good, because custom OIDC identity provider federation leaves room for BankID through an e-ID broker later.
- Neutral, because it fits the Azure-centred hosting direction (#8) and the same Microsoft Entra platform used for managed identities.
- Bad, because local development cannot use it without a cloud tenant (mitigated by Keycloak locally, Option L1).
- Bad, because user flow customisation is more limited than with a self-hosted IdP.

#### Option I2: ASP.NET Core Identity + OpenIddict or Duende IdentityServer

- Good, because users are stored in our own `users` schema, with full control and no external dependency.
- Bad, because we would own password storage, reset flows, lockout, MFA and social login plumbing, which contradicts the first decision driver.
- Bad, because Duende IdentityServer requires a commercial licence above its community thresholds, and OpenIddict requires significant security-sensitive code that we would maintain ourselves.

#### Option I3: Keycloak, Self-Hosted

- Good, because it is open source, standards-compliant and feature-rich, and the same product would run locally and in production.
- Bad, because a solo maintainer would have to run, patch, scale, monitor and back up another stateful, security-critical service and its database on Azure, which adds operating cost and risk.

#### Option I4: Third-Party SaaS (Auth0, Clerk, …)

- Good, because they offer an excellent developer experience and ready-made UI components.
- Bad, because costs grow quickly beyond the free tiers and some features (custom domains, MFA, organisations) are paid.
- Bad, because the vendors are US-based. EU data residency is available only on some plans or regions, which adds GDPR transfer assessment work.

### Authorisation Model

#### Option A1: Role-Based Access with Roles as Token Claims

- Good, because `[Authorize(Roles = …)]` is simple and well known.
- Bad, because roles in TeamGym are **scoped to resources** (an admin of club A, a coach of team T, a co-organiser of competition X). Flat role claims cannot express this, and scoped claims bloat the token.
- Bad, because revoking a role takes effect only when the session or token is refreshed, and storing application roles in the IdP couples authorisation to the IdP.

#### Option A2: Resource-Based Policies over Permissions from Database Memberships

- Good, because it maps directly onto `OrganisationMembership`, team scoping and two-party `CompetitionApplication` visibility ([ADR-0006](0006-tenancy-and-organisation-model.md)).
- Good, because it uses built-in ASP.NET Core authorisation (`IAuthorizationService`, requirements, handlers) without extra infrastructure.
- Good, because revocation is immediate and the IdP stays free of application concerns.
- Bad, because it costs a membership lookup per request and requires discipline to call resource checks in handlers. This is mitigated by architecture and integration tests.

#### Option A3: Relationship-Based Access Control (OpenFGA)

- Good, because it expresses hierarchical relationships (federation → region → club → team) elegantly and scales to complex delegation.
- Bad, because it adds another service, data synchronisation between our database and the authorisation store, and a new modelling language. That is too much for the MVP's simple membership rules.

### Local Development and Test Authentication

#### Option L1: Keycloak in `.NET Aspire` + Test Authentication Handler

- Good, because contributors exercise the real OIDC and cookie BFF flow locally without any cloud account.
- Good, because integration tests stay fast and deterministic while still running the real authorisation handlers.
- Bad, because it adds a Keycloak container and realm file to maintain.

#### Option L2: Test Authentication Handler Only

- Good, because it is the simplest setup with no extra container.
- Bad, because the OIDC login, logout and cookie flow is never exercised before deployment, so BFF bugs surface only in cloud environments.

#### Option L3: Shared Cloud Development Tenant

- Good, because development behaves exactly like production.
- Bad, because every contributor would need credentials for a cloud tenant, which conflicts with an open-source project that should be runnable by anyone.

## More Information

- Related issue: [#6](https://github.com/gyback/teamgym-arena/issues/6)
- Parent epic: [#9](https://github.com/gyback/teamgym-arena/issues/9)
- Related ADRs and decisions:
  - [ADR-0002 Repository strategy — public app monorepo, private IaC repo and artefact hand-off](0002-repository-strategy.md)
  - [ADR-0003 Engineering workflow and quality gates for agent-assisted development](0003-engineering-workflow-and-quality-gates.md)
  - [ADR-0004 Backend runtime and architectural style — .NET 10 modular monolith with vertical slices](0004-backend-runtime-and-architectural-style.md)
  - [ADR-0005 Frontend framework and API contract — React 19 SPA with Vite, ASP.NET Core BFF and OpenAPI TypeScript client](0005-frontend-framework-and-api-contract.md)
  - [ADR-0006 Tenancy and organisation model](0006-tenancy-and-organisation-model.md)
  - [ADR-0007 Primary data store and data access](0007-primary-data-store-and-data-access.md)
  - [#8 Cloud hosting platform and Infrastructure as Code](https://github.com/gyback/teamgym-arena/issues/8) (Entra external tenant provisioning, Key Vault, Data Protection key storage, e-mail delivery)
- Follow-up implementation issues:
  - [#32 Implement BFF OIDC authentication (cookie session, `/bff` endpoints, CSRF), Keycloak in Aspire and test auth handler](https://github.com/gyback/teamgym-arena/issues/32)
  - [#33 Implement `Users` module identity mapping, just-in-time provisioning and account deletion](https://github.com/gyback/teamgym-arena/issues/33)
  - [#34 Implement resource-scoped permission authorisation (`IOrganisationAccess`, policies, handlers) and invitations](https://github.com/gyback/teamgym-arena/issues/34)
  - [#35 Write privacy and data-protection overview (`docs/privacy.md`)](https://github.com/gyback/teamgym-arena/issues/35)
- Revisit this decision when:
  - BankID or another strong e-ID is required (new ADR covering the broker choice and `personnummer` processing),
  - gymnasts or guardians need their own accounts,
  - delegated federation administration or judge panels make the permission model hierarchical (consider ReBAC, Option A3), or
  - Entra External ID pricing or data residency terms change materially.
