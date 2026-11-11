---
status: accepted
date: 2026-11-11
decision-makers: Gustav Gybäck
---

# Tenancy and organisation model — shared database with explicit organisation ownership, hierarchy-ready organisations and configurable rule sets

<!-- Related issue: #7 -->

## Context and Problem Statement

Full hierarchical organisation management (international federations such as [European Gymnastics](https://www.europeangymnastics.com/discipline/teamgym/intro), national federations such as the Swedish Gymnastics Federation [*Svenska Gymnastikförbundet*], regional associations, and local clubs) is a long-term goal, but the **TeamGym** MVP already revolves around organisations in practice:

- **Clubs** manage their own coaches, gymnasts, and **Teams** (entered into sections **Women**, **Men**, **Mixed** and **Open**).
- **Organisers** (typically a host club, regional association, or national federation) create and run **Competitions** across the three TeamGym disciplines (**Floor**, **Tumble** and **Trampet**).
- **Clubs apply** with their teams to competitions hosted by other organisations, and the organiser builds the rotation **Schedule** from all accepted applications across participating clubs.

Unlike classic B2B SaaS products where tenants are completely isolated silos that never interact, a TeamGym competition platform is inherently **collaborative across organisations**: clubs must discover competitions hosted by other clubs or federations, and a competition application is a shared business artefact between the **applying club** and the **hosting organiser**.

We need to decide:

1. **The tenancy and data-isolation model** — how data is isolated between clubs while enabling cross-club competition discovery, applications, and scheduling, and how the hosted service can grow from Sweden/the Nordics to multiple national federations.
2. **The organisation domain model** — whether to model a flat `Club` table now or a hierarchy-ready `Organisation` entity (`Club`, `Region`, `Federation`) from day one so future organisation management requires no destructive data migration.
3. **User-to-organisation membership** — whether a user can belong to and hold roles in multiple clubs/organisations simultaneously.
4. **TeamGym rule-set variability** — how differences between **European Gymnastics TeamGym** rules and national federation rules (sections, classes, squad sizes on **Floor**, **Tumble** and **Trampet**, gender composition for **Mixed** and **Open**) are represented.

## Decision Drivers

- **Avoid a costly schema and domain re-model later:** Adding regional and national federation hierarchies later must not require rewriting foreign keys across `Teams`, `Competitions`, `Applications`, and authorisation policies.
- **Strict privacy for club-internal data:** A club must only see and edit its own gymnasts (who are frequently minors), internal team rosters, and draft applications.
- **Cross-organisation collaboration for competitions:** Published competitions must be visible across clubs so they can apply, while each submitted application is private between **the applying club** and **the competition's organiser** (until accepted teams appear on a published start list / schedule).
- **Multi-federation & international readiness:** Support Swedish and Nordic TeamGym clubs and federations first while allowing a single hosted instance to serve multiple federations and rule sets in the future.
- **Keep the MVP simple:** Avoid multi-database operational overhead or building federation administration UIs before they are needed.

## Considered Options

### 1. Tenancy and Data Isolation Model

- **Option T1:** **Shared database with explicit `OrganisationId` ownership and resource-scoped access policies** — One shared database for the deployment; every owned resource carries an explicit owning `OrganisationId` (e.g. `ClubOrganisationId`, `OrganiserOrganisationId`, `ApplyingOrganisationId`), and query/command handlers enforce explicit resource-scoped access rules rather than blind single-tenant EF Core global query filters.
- **Option T2:** **Multi-tenant shared database with a single `TenantId` and EF Core global query filters on every table** — Treating each club (or each federation) as a hard-isolated silo filtered automatically by a single ambient `CurrentTenantId`.
- **Option T3:** **Single tenant per deployment** — Deploying a separate container and database instance per federation or per club.
- **Option T4:** **Multi-tenant with schema-per-tenant or database-per-tenant** — Provisioning a separate PostgreSQL schema or database per organisation.

### 2. Organisation Hierarchy Model

- **Option O1:** **Hierarchy-ready unified `Organisation` entity from day one (`OrganisationType`: `Club`, `Region`, `Federation` + nullable `ParentOrganisationId`)** — All ownership foreign keys reference `OrganisationId`; the MVP UI and workflows focus on `Club`s, while parent `Region` / `Federation` nodes can be linked at any time without a schema migration.
- **Option O2:** **Flat `Club` entity only for the MVP** — Model only a `Clubs` table now and migrate `ClubId` foreign keys to a polymorphic or hierarchical `Organisation` model when federation management is built later.
- **Option O3:** **Full multi-level hierarchy (`Federation` → `Region` → `Club`) with delegated administration UI in the MVP** — Build federation approval trees, regional roll-ups, and multi-tier admin portals before launching the MVP.

### 3. TeamGym Rule Set Configuration

- **Option R1:** **Configurable `RuleSet` model with built-in federation presets (`European Gymnastics TeamGym`, `Swedish Gymnastics Federation`, etc.) selectable per competition (with optional organisation default)** — Encapsulate section, class, and team composition parameters in a `RuleSet` referenced by each competition (and snapshotted when the competition opens).
- **Option R2:** **Hardcode European Gymnastics rules only in the MVP** — Hardcode 8–10 gymnasts on **Floor**, 6 per round on **Tumble** and **Trampet**, and fixed sections in C# constants, refactoring to configurable rules later.
- **Option R3:** **Tie rules strictly to a top-level federation tenant** — Force every competition inside a federation to use a single immutable federation-wide rule set.

### 4. User-to-Organisation Membership Model

- **Option M1:** **Many-to-many `OrganisationMembership` with organisation- and team-scoped roles** — A single user identity can belong to multiple organisations (e.g. coach in Club A, club admin in Club B, organiser for a regional competition) and switch active organisation context in the UI.
- **Option M2:** **Single organisation per user account** — Bind `User.OrganisationId` 1:1, requiring users who coach or administrate in two clubs to register multiple e-mail accounts.

## Decision Outcome

Chosen options:

1. **Tenancy & isolation — Option T1:** Use a **shared database with explicit `OrganisationId` ownership and resource-scoped access rules** across all modules ([ADR-0004](0004-backend-runtime-and-architectural-style.md)). Clubs are collaborating principals within the platform, not isolated single-tenant silos.
2. **Organisation hierarchy — Option O1:** Model a **unified, hierarchy-ready `Organisation` entity** (`OrganisationType`: `Club`, `Region`, `Federation` with nullable `ParentOrganisationId` and `CountryCode`) from day one, while keeping MVP user-facing workflows focused on **Clubs** and competition **Organisers**.
3. **Rule sets — Option R1:** Introduce a **configurable `RuleSet` model with built-in presets** (starting with **European Gymnastics TeamGym** and **Swedish Gymnastics Federation [*Svenska Gymnastikförbundet — Truppgymnastik*] / Nordic presets**), selectable per competition with an optional default on each `Organisation`.
4. **User membership — Option M1:** Support **many-to-many user-to-organisation memberships (`OrganisationMembership`)** with organisation-level and team-level roles so coaches, club admins, and organisers only ever need one account.

### 1. Tenancy, Ownership and Cross-Organisation Visibility Model

In TeamGym, data isolation is governed by **explicit resource ownership and relationship rules**, not by a single blind `TenantId` global filter:

| Domain Concept | Owning / Associated Organisation(s) | Who Can View | Who Can Create / Edit |
| --- | --- | --- | --- |
| **`Organisation`** (`Club`, `Region`, `Federation`) | Self (`OrganisationId`), optional `ParentOrganisationId` | Public summary (`Name`, `ShortName`, `CountryCode`, `Type`) visible to authenticated users; full settings visible to members | `OrganisationAdmin` of that organisation (and future parent federation admins) |
| **`OrganisationMembership`** | `OrganisationId` + `UserId` | Members of the same `OrganisationId` and the user themselves | `OrganisationAdmin` of that `OrganisationId` |
| **`Gymnast`** | `ClubOrganisationId` (`OrganisationId`) | **Strictly private to the owning club** (`OrganisationAdmin` and assigned `Coach`es of `ClubOrganisationId`). Only competing gymnast counts / required roster snapshots are shared on an application when required by competition rules | `OrganisationAdmin` and authorised `Coach`es of `ClubOrganisationId` |
| **`Team`** | `ClubOrganisationId` (`OrganisationId`) | Private to `ClubOrganisationId` by default; team name, club name, **Section** (**Women**, **Men**, **Mixed**, **Open**) and **Class** become visible to an organiser when entered in a competition, and publicly on published schedules | `OrganisationAdmin` and assigned `Coach`es of `ClubOrganisationId` |
| **`Competition`** | `OrganiserOrganisationId` (`OrganisationId`) | `Draft`: private to `OrganiserOrganisationId` (and explicitly invited co-organisers). `Published` / `OpenForApplications` / `Scheduled`: visible across all clubs (and publicly for published schedules) | `Organiser` / `OrganisationAdmin` of `OrganiserOrganisationId` |
| **`CompetitionApplication`** | `ApplyingOrganisationId` (club) **and** `OrganiserOrganisationId` (via `CompetitionId`) | **Shared between exactly two parties:** authorised members of `ApplyingOrganisationId` and authorised organisers of `OrganiserOrganisationId` | Created/edited/withdrawn by `ApplyingOrganisationId`; reviewed (accepted/rejected/waitlisted) by `OrganiserOrganisationId` |
| **`CompetitionSchedule`** | `OrganiserOrganisationId` (via `CompetitionId`) | `Draft`: private to `OrganiserOrganisationId`. `Published`: visible to all participating clubs and public spectators | `Organiser` of `OrganiserOrganisationId` |

- **Why EF Core global query filters on a single `CurrentTenantId` are rejected as the primary isolation mechanism:**
  - If every entity had a global query filter `WHERE TenantId == currentUser.TenantId`, a club logged into its own context could not browse competitions hosted by another club/federation, and an organiser generating a **Floor** / **Tumble** / **Trampet** rotation schedule could not query accepted applications and teams across 25 visiting clubs without constantly calling `.IgnoreQueryFilters()` (which is error-prone and defeats the safety guarantee).
  - Instead, each vertical-slice query and command handler in `Organisations`, `Competitions`, `Applications`, and `Scheduling` ([ADR-0004](0004-backend-runtime-and-architectural-style.md)) enforces explicit resource-scoped authorisation policies (to be detailed in the identity & authorisation ADR, #6) and explicit `OrganisationId` predicates verified by integration tests.

### 2. Hierarchy-Ready `Organisation` Domain Model

Inside `GymnasticsCompetition.Modules.Organisations` ([ADR-0004](0004-backend-runtime-and-architectural-style.md)), clubs, regions, and federations share a single aggregate root:

```text
Organisation
├── Id: OrganisationId (UUIDv7)
├── Name: string                          # e.g. "GF Fram", "Gymnastikförbundet Syd", "Svenska Gymnastikförbundet"
├── ShortName / Code: string?             # e.g. "GFF", "SWE"
├── Type: OrganisationType                # Club | Region | Federation
├── CountryCode: string                   # ISO 3166-1 alpha-2, e.g. "SE", "DK", "NO", "IS", "FI", "EU"
├── ParentOrganisationId: OrganisationId? # Nullable self-reference: Club -> Region -> Federation
├── DefaultRuleSetId: RuleSetId?          # Optional default RuleSet for competitions hosted by this organisation
└── Status: OrganisationStatus            # Active | Archived
```

- **How this keeps the MVP simple while preventing a future re-model:**
  - **Every foreign key is already `OrganisationId`:** `Team.OrganisationId`, `Gymnast.OrganisationId`, `Competition.OrganiserOrganisationId`, and `CompetitionApplication.ApplyingOrganisationId` all point to `OrganisationId`.
  - **Any organisation type can host a competition:** Whether a local club hosts an invitational cup, a regional association hosts a regional qualifier, or a national federation hosts a national championship (*SM i Truppgymnastik*) or the European TeamGym Championships, `Competition.OrganiserOrganisationId` works identically without polymorphic foreign keys or schema changes.
  - **MVP scope remains lean:** In the MVP UI, users create and manage `Club` organisations (and optionally link them to pre-seeded national/regional federations). Delegated parent-to-child administration (e.g. a `Federation` admin automatically governing all child `Region`s and `Club`s, sanctioning calendars, or managing national gymnast licences) can be layered onto `ParentOrganisationId` later with zero breaking changes to existing tables.

### 3. Multi-Organisation Membership (`OrganisationMembership`)

Real-world TeamGym participants frequently wear multiple hats across organisations — for example, a coach who coaches a Junior team in one club, competes or coaches a Senior team in another club, and serves as competition director for a regional federation event.

```text
OrganisationMembership
├── Id: OrganisationMembershipId
├── OrganisationId: OrganisationId
├── UserId: UserId
├── Role: OrganisationRole                # Admin | Coach | CompetitionOrganiser
├── TeamIds: IReadOnlyCollection<TeamId>  # Optional scoping for Coaches assigned to specific teams
└── JoinedAtUtc: DateTimeOffset
```

- A `User` (`GymnasticsCompetition.Modules.Users`) has a **many-to-many** relationship with `Organisation` via `OrganisationMembership`.
- In the React SPA (`src/Web`, [ADR-0005](0005-frontend-framework-and-api-contract.md)), users who belong to more than one organisation get a lightweight **organisation switcher** in the navigation header, while routes and API endpoints carry the explicit `organisationId` (e.g. `/api/v1/organisations/{organisationId}/teams`) so URLs are bookmarkable, unambiguous, and safe across multiple open browser tabs.

### 4. Configurable TeamGym `RuleSet` Model

While [European Gymnastics TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) defines the international standard across **Floor**, **Tumble** and **Trampet**, national federations adapt team sizes, age/level classes, and gender composition rules (especially at youth and grassroots levels).

Rather than hardcoding constants or locking an entire deployment to one federation's rules, `Competitions` (and `Organisations`) use an explicit **`RuleSet`** configuration:

| Rule Dimension | European Gymnastics TeamGym Preset (`EG-TEAMGYM`) | National / Regional Presets (e.g. Swedish Gymnastics Federation `SWE-TRUPP`) |
| --- | --- | --- |
| **Sections** | **Women**, **Men**, **Mixed** | **Women** (*Dam*), **Men** (*Herr*), **Mixed**, **Open** (*Öppen* — any gender ratio) |
| **Classes** | **Junior**, **Senior** | National age and level classes (e.g. Ungdom, Junior, Senior; Level/Steg 9 down to 1, Regional/Riks classes) |
| **Disciplines** | **Floor**, **Tumble**, **Trampet** (all 3 required) | **Floor**, **Tumble**, **Trampet** (or configurable subset for development/apparatus-only competitions) |
| **Squad size & Floor count** | **8–10 gymnasts** registered and competing on **Floor** (+ optional reserves per regulation) | Configurable min/max squad size on **Floor** per class (e.g. 6–16 gymnasts in youth/national classes, or 8–10 in championship classes) |
| **Tumble & Trampet round count** | **6 gymnasts** performing in each of the 3 rounds on **Tumble** and **Trampet** | **6 gymnasts** per round (or class-specific override where grassroots rules permit) |
| **Mixed gender composition** | Strictly **50% male / 50% female** competing gymnasts | 50%/50% for official **Mixed** classes; unrestricted gender composition in **Open** (*Öppen*) classes |

- **How `RuleSet` works in the MVP:**
  - The platform ships with built-in, versioned system presets (starting with **European Gymnastics TeamGym** and **Swedish Gymnastics Federation / Truppgymnastik**).
  - Each `Organisation` can set a `DefaultRuleSetId` (e.g. a Swedish club or federation defaults to the Swedish preset).
  - When an organiser creates a `Competition`, they pick a preset `RuleSet` and configure which **Sections** (**Women**, **Men**, **Mixed**, **Open**), **Classes**, and **Disciplines** (**Floor**, **Tumble**, **Trampet**) are offered in that competition, optionally tailoring team size bounds per class.
  - The competition stores an immutable snapshot of its active class/section rules so that subsequent updates to a federation preset never invalidate an ongoing competition or its submitted applications.

### 5. Answers to Open Questions

1. **Who are the realistic first users — one regional/national federation, or TeamGym clubs across Sweden or the Nordics?**
   - The realistic first adopters are **Swedish and Nordic TeamGym clubs and competition organisers** (where TeamGym / *truppgymnastik* has its highest participation density globally), hosting club invitationals, regional competitions, and national events, while keeping English and official **European Gymnastics** rules first-class from day one ([ADR-0005](0005-frontend-framework-and-api-contract.md)).
   - Because clubs from different regions (and neighbouring Nordic countries) regularly apply to the same invitational or national competitions, one shared hosted platform where clubs and organisers collaborate directly delivers immediate value without waiting for an entire national federation to mandate top-down adoption.
2. **Can a user belong to multiple clubs (e.g. a TeamGym coach in two clubs)?**
   - **Yes.** Modelled explicitly via `OrganisationMembership` (Option M1).
3. **Do rules, sections, classes, team composition rules and scoring differ per federation, so "rule sets" must be configurable?**
   - **Yes.** Modelled via configurable `RuleSet` presets and per-competition class rules (Option R1), avoiding hardcoded European Gymnastics-only assumptions while keeping domain validation pure and unit-testable ([ADR-0004](0004-backend-runtime-and-architectural-style.md)).

### Consequences

- Good, because clubs can seamlessly browse and apply to competitions hosted by any organiser on the platform without fighting single-tenant data silos.
- Good, because club-internal rosters and minor gymnasts' personal data remain strictly isolated to the owning club (`OrganisationId`), while competition applications are visible only to the applying club and the competition organiser.
- Good, because using a unified `Organisation` entity (`Club`, `Region`, `Federation` + `ParentOrganisationId`) means every foreign key in the database is future-proof for full hierarchical federation management.
- Good, because explicit `{organisationId}` route scopes + `OrganisationMembership` allow coaches and admins to belong to multiple clubs and work in multiple tabs without session-state race conditions.
- Good, because configurable `RuleSet` presets allow the same domain model to validate both strict European Gymnastics rules (8–10 on **Floor**, 6 per round on **Tumble** and **Trampet**, 50%/50% **Mixed**) and Swedish/Nordic national rules (including **Open** sections and larger youth squads).
- Bad, because developers and AI coding agents must write explicit ownership/authorisation checks in query and command handlers rather than relying on a single automatic `TenantId` EF Core global query filter (mitigated by shared authorisation helpers and mandatory integration tests per endpoint).

### Confirmation

- **Domain unit tests (`tests/Modules/Organisations.UnitTests` & `tests/Modules/Competitions.UnitTests`):** Verify `Organisation` hierarchy invariants, multi-club `OrganisationMembership` rules, and team composition validation against both European Gymnastics and national `RuleSet` presets (**Floor**, **Tumble**, **Trampet**; **Women**, **Men**, **Mixed**, **Open**).
- **Cross-organisation integration tests (`tests/Api.IntegrationTests`):** Every endpoint that reads or mutates club-owned or organiser-owned data must include integration tests verifying that:
  - Club A **cannot** read or modify Club B's gymnasts, teams, or draft applications (`403 Forbidden` / `404 Not Found`),
  - Club A **can** view published competitions hosted by Organiser C and submit an application, and
  - Club B **cannot** view Club A's application to Organiser C's competition, while both Club A and Organiser C can.

## Pros and Cons of the Options

### Tenancy and Data Isolation Model

#### Option T1: Shared Database with Explicit `OrganisationId` Ownership and Resource-Scoped Access

- Good, because it directly matches the domain reality of TeamGym: clubs own private teams and gymnasts, organisers host public/cross-club competitions, and applications are two-party records between the applying club and the organiser.
- Good, because a single PostgreSQL database and single container deployment keep Azure infrastructure costs minimal ([ADR-0002](0002-repository-strategy.md), #5, #8).
- Good, because multiple countries and federations can coexist in one hosted instance (filtered by country, federation, or rule set in the UI) while still allowing international invitational competitions.
- Neutral, because access control is enforced via explicit resource-scoped policies and query predicates rather than a single global tenant filter.

#### Option T2: Shared Database with a Single `TenantId` and EF Core Global Query Filters on Every Table

- Good, because in pure single-organisation B2B SaaS apps, global query filters prevent accidental cross-tenant reads automatically.
- Bad, because in a competition platform, clubs **must** interact across organisation boundaries (viewing another organiser's competition, submitting an application across organisations, building a rotation schedule across 20+ clubs).
- Bad, because if `TenantId` is set to a `ClubId`, cross-club competition workflows require bypassing the global filter everywhere; if `TenantId` is set to a top-level `FederationId`, it still fails to isolate Club A from Club B within the same federation and prevents cross-border Nordic/European competitions.

#### Option T3: Single Tenant per Deployment (One Federation or Club per Instance)

- Good, because physical deployment isolation is simple when a single national federation self-hosts its own dedicated instance (which remains supported by our open-source container images from [ADR-0002](0002-repository-strategy.md)).
- Bad, because if the code assumes only one organisation/federation exists per database, clubs still need intra-database isolation from each other, so it does not solve club-vs-club privacy at all.
- Bad, because operating separate cloud deployments per regional federation or club would multiply Azure hosting costs and maintenance work for a solo maintainer.

#### Option T4: Multi-Tenant with Schema-per-Tenant or Database-per-Tenant

- Good, because tenant data is separated at the database schema or instance level.
- Bad, because each module already uses its own PostgreSQL schema (`users`, `organisations`, `competitions`, `applications`, `scheduling` — [ADR-0004](0004-backend-runtime-and-architectural-style.md)), making schema-per-tenant incompatible with module-per-schema isolation.
- Bad, because running EF Core migrations across dozens or hundreds of tenant schemas/databases and performing cross-club competition queries across schemas is complex, slow, and unnecessary.

### Organisation Hierarchy Model

#### Option O1: Unified `Organisation` Entity (`Club`, `Region`, `Federation` + Nullable `ParentOrganisationId`) from Day One

- Good, because all foreign keys (`OrganisationId`) and role memberships (`OrganisationMembership`) are stable from the very first migration.
- Good, because clubs, regional associations, and national federations can all act as competition organisers (`Competition.OrganiserOrganisationId`) using the exact same model.
- Good, because MVP UI complexity stays identical to a flat club model (users only manage clubs initially), while hierarchical federation management can be unlocked later without data migration.

#### Option O2: Flat `Club` Entity Now, Migrate to Hierarchy Later

- Good, because the word `Club` is concrete for MVP team management.
- Bad, because competitions are often hosted by regional or national federations rather than a single club, immediately forcing awkward workarounds ("fake clubs" for federations).
- Bad, because renaming or migrating `ClubId` foreign keys across `Teams`, `Gymnasts`, `Competitions`, `Applications`, and API routes later causes avoidable breaking changes across the backend and frontend.

#### Option O3: Full Multi-Level Hierarchy and Delegated Administration UI in the MVP

- Good, because federation-level governance would be available immediately at launch.
- Bad, because building multi-tier approval workflows and federation admin dashboards before clubs can even apply to a competition delays the MVP significantly.

### TeamGym Rule Set Configuration

#### Option R1: Configurable `RuleSet` Presets Selectable per Competition (with Optional Organisation Default)

- Good, because it supports both strict **European Gymnastics TeamGym** rules (sections **Women**, **Men**, **Mixed**; 8–10 gymnasts on **Floor**, 6 per round on **Tumble** and **Trampet**, 50%/50% **Mixed**) and national rule sets such as **Swedish Gymnastics Federation (*Truppgymnastik*)** (including the **Open** section and national class squad sizes).
- Good, because snapshotting the effective rules onto each `Competition` guarantees that rule updates never corrupt historical or in-flight competitions.
- Good, because pure `Domain/` validators in `Organisations` and `Applications` can validate a `Team` against any `RuleSet` instance in memory during unit tests ([ADR-0004](0004-backend-runtime-and-architectural-style.md)).

#### Option R2: Hardcode European Gymnastics Rules Only in the MVP

- Good, because it requires no `RuleSet` configuration model initially.
- Bad, because many Swedish and Nordic domestic competitions (our primary first users) use national class rules (such as the **Open** section or different youth squad sizes on **Floor**) and would be blocked from using the MVP if European Championship numbers were hardcoded.

#### Option R3: Tie Rules Strictly to a Top-Level Federation Tenant

- Good, because every competition under a federation automatically inherits one rule set.
- Bad, because the same club or national federation often runs competitions under different rule sets (e.g. a Swedish club hosting both a national-rules youth cup and an international invitational under European Gymnastics rules).

### User-to-Organisation Membership

#### Option M1: Many-to-Many `OrganisationMembership`

- Good, because coaches, judges, and administrators in the TeamGym community frequently work with more than one club or regional committee.
- Good, because explicit `{organisationId}` routing paired with `OrganisationMembership` claims/checks fits resource-scoped authorisation (#6) cleanly.

#### Option M2: Single Club per User Account

- Good, because a user has only a single scalar `OrganisationId` property.
- Bad, because forcing a coach who helps two clubs (or a club coach who also organises a regional competition) to maintain multiple e-mail accounts is poor UX and breaks as soon as invitations and federated sign-in (#6) are introduced.

## More Information

- Related issue: [#7](https://github.com/gyback/gymnastics-competition/issues/7)
- Parent epic: [#9](https://github.com/gyback/gymnastics-competition/issues/9)
- Related ADRs and decisions:
  - [ADR-0001 Record architecture decisions using MADR](0001-record-architecture-decisions.md)
  - [ADR-0002 Repository strategy — public app monorepo, private IaC repo and artefact hand-off](0002-repository-strategy.md)
  - [ADR-0003 Engineering workflow and quality gates for agent-assisted development](0003-engineering-workflow-and-quality-gates.md)
  - [ADR-0004 Backend runtime and architectural style — .NET 10 modular monolith with vertical slices](0004-backend-runtime-and-architectural-style.md)
  - [ADR-0005 Frontend framework and API contract — React 19 SPA with Vite, ASP.NET Core BFF and OpenAPI TypeScript client](0005-frontend-framework-and-api-contract.md)
  - [#5 Primary data store and data access](https://github.com/gyback/gymnastics-competition/issues/5) (shared PostgreSQL database with per-module schemas)
  - [#6 Identity, authentication and authorisation](https://github.com/gyback/gymnastics-competition/issues/6) (ASP.NET Core resource-scoped authorisation policies evaluating `OrganisationMembership` and competition ownership)
- Follow-up implementation issues:
  - [#24 Implement initial `Organisations` domain model (`Organisation`, `OrganisationMembership`, `Gymnast`, `Team`)](https://github.com/gyback/gymnastics-competition/issues/24)
  - [#25 Implement configurable TeamGym `RuleSet` model, federation presets and cross-organisation visibility rules](https://github.com/gyback/gymnastics-competition/issues/25)
