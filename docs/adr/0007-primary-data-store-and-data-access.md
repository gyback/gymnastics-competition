---
status: accepted
date: 2026-11-11
decision-makers: Gustav Gybäck
---

# Primary data store and data access — PostgreSQL with EF Core 10, per-module schemas, migration bundles and GDPR retention policies

<!-- Related issue: #5 -->

## Context and Problem Statement

The [TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) competition platform domain is strongly relational and transactional:

- **Organisations** (`Club`, `Region`, `Federation` — [ADR-0006](0006-tenancy-and-organisation-model.md)) own **Gymnasts**, **Coaches** (`OrganisationMembership`), and **Teams**.
- **Competitions** configure **RuleSets**, **Sections** (**Women**, **Men**, **Mixed** and **Open**), age/level **Classes**, and active **Disciplines** (**Floor**, **Tumble** and **Trampet**).
- **Competition Applications** link an applying club's teams and competing gymnast rosters to a competition's sections and classes under strict capacity and deadline constraints.
- **Competition Schedules** assign accepted teams to sessions, warm-ups, and synchronized three-discipline rotation slots across **Floor**, **Tumble** and **Trampet** where no team or apparatus may ever be double-booked.

To support the `.NET 10` modular monolith ([ADR-0004](0004-backend-runtime-and-architectural-style.md)) and our shared-database tenancy model ([ADR-0006](0006-tenancy-and-organisation-model.md)), we need to decide:

1. **The primary database engine and managed Azure service.**
2. **The .NET data access strategy** (ORM, read/write patterns, concurrency, and domain persistence ignorance).
3. **How modules isolate their data** (one shared database with schema-per-module vs database-per-module vs a single shared schema).
4. **How schema migrations are authored, tested, and applied** across local development, CI, and production.
5. **The GDPR personal-data retention and anonymisation policy** for gymnasts (who are frequently minors) after a competition season, plus a forward-looking note on binary storage for future **Floor**, **Tumble** and **Trampet** music uploads.

## Decision Drivers

- **Relational integrity and ACID transactions:** Team applications, capacity limits, and rotation schedules across **Floor**, **Tumble** and **Trampet** require strong consistency, foreign-key integrity, unique constraints, and optimistic concurrency control.
- **Mature .NET 10 support and ergonomic local development:** First-class Entity Framework Core 10 provider, seamless `.NET Aspire` orchestration (`src/AppHost`), and fast ephemeral integration testing with `Testcontainers`.
- **Managed offering on Azure in an EU region:** Automated backups, point-in-time restore (PITR), high availability options, and EU data residency (e.g. `swedencentral`) with low operational overhead for a solo maintainer.
- **Low cost at small scale:** Predictable, non-profit-friendly monthly cost with a small starting SKU (`Burstable`) and the ability to stop/scale down non-production instances.
- **Open-source friendly:** 100% open-source database engine (`PostgreSQL` license) so contributors and self-hosting federations/clubs can run the exact same database engine locally and in production at zero licensing cost.
- **GDPR compliance by design:** EU data residency, strict personal-data minimisation for minor gymnasts, immediate right-to-erasure / anonymisation workflows, and an explicit retention policy after a competition season ends.

## Considered Options

### 1. Primary Database Engine and Managed Cloud Service

- **Option D1:** **PostgreSQL 17 (`Azure Database for PostgreSQL — Flexible Server` in an EU region)** — Open-source relational database running in an official container locally/in CI and on Azure Flexible Server (Burstable SKU initially) in production.
- **Option D2:** **Azure SQL Database (Serverless tier)** — Microsoft SQL Server in the cloud paired with the SQL Server Linux container image for local development and CI.
- **Option D3:** **Document Database (`Azure Cosmos DB`)** — Multi-model / NoSQL document store using the EF Core Cosmos provider or Azure Cosmos SDK.

### 2. Data Access Strategy in `.NET 10`

- **Option A1:** **EF Core 10 (`Npgsql.EntityFrameworkCore.PostgreSQL`) for writes, migrations, and projected `.AsNoTracking()` reads** — Explicit `IEntityTypeConfiguration<T>` mappings keeping `Domain/` persistence-ignorant, direct `Select` projections in query slices, and targeted raw SQL (`SqlQuery<T>`) only if a future reporting query needs it.
- **Option A2:** **Micro-ORM (`Dapper`) + hand-written SQL migrations (`DbUp` / `Flyway`)** — Writing all SQL queries, inserts, updates, and schema migration scripts by hand.
- **Option A3:** **EF Core wrapped behind generic `IRepository<T>` and `IUnitOfWork` interfaces in every module** — Abstracting `DbContext` behind custom repository layers for both commands and queries.

### 3. Module Database Topology

- **Option M1:** **One shared PostgreSQL database with one schema per module (`users`, `organisations`, `competitions`, `applications`, `scheduling`) and one `DbContext` per module** — Logical isolation at the schema and `DbContext` level inside a single database instance.
- **Option M2:** **Separate PostgreSQL database per module** — Five separate databases (`users_db`, `organisations_db`, `competitions_db`, `applications_db`, `scheduling_db`) from day one.
- **Option M3:** **Single shared `DbContext` and single `public` schema across all modules** — One monolithic `ApplicationDbContext` mapping every module's entities into `public`.

### 4. Schema Migration Strategy

- **Option S1:** **EF Core migrations applied via a dedicated migration runner (`src/MigrationService`) in `.NET Aspire` locally, verified in CI against `Testcontainers`, and executed in staging/production via a self-contained EF Core migration bundle (`efbundle`) before deploying the new app revision.**
- **Option S2:** **Apply `dbContext.Database.MigrateAsync()` automatically inside `GymnasticsCompetition.Api` (`Program.cs`) on application startup in all environments.**
- **Option S3:** **Generate raw SQL diff scripts manually and apply them out-of-band by a human administrator before releases.**

## Decision Outcome

Chosen options:

1. **Primary database — Option D1:** Use **PostgreSQL 17** as the primary relational data store, hosted on **Azure Database for PostgreSQL — Flexible Server** in an EU region (default `swedencentral`) for the managed service and the official `postgres:17-alpine` container image for local development and CI.
2. **Data access — Option A1:** Use **Entity Framework Core 10 (`Npgsql.EntityFrameworkCore.PostgreSQL`)** with `EFCore.NamingConventions` (`UseSnakeCaseNamingConvention()`), persistence-ignorant `Domain/` aggregates mapped via `IEntityTypeConfiguration<T>`, and direct `.AsNoTracking()` query projections inside vertical slices.
3. **Module isolation — Option M1:** Use **one shared PostgreSQL database with a dedicated PostgreSQL schema and dedicated `DbContext` per module** (`users`, `organisations`, `competitions`, `applications`, `scheduling`), each maintaining its own `__EFMigrationsHistory` table inside its schema.
4. **Schema migrations — Option S1:** Manage schema evolution with **per-module EF Core migrations**, executed automatically by a dedicated **`src/MigrationService`** worker in `.NET Aspire` during local development, verified from scratch on every PR with `Testcontainers.PostgreSql`, and published as **EF Core migration bundles** ([ADR-0002](0002-repository-strategy.md)) executed as a pre-deployment job in CI/CD before rolling out new application containers.

### 1. Primary Database Engine (`PostgreSQL 17`) and Azure Flexible Server

- **Why PostgreSQL fits the TeamGym domain:**
  - Enforces strict relational constraints (foreign keys, composite unique constraints, check constraints, and exclusion constraints) needed so a TeamGym **Schedule** can never assign two teams to the same **Floor**, **Tumble** or **Trampet** rotation slot simultaneously.
  - Native support for `uuid` (`UUIDv7`), `timestamptz` (UTC timestamps), rich indexing, and `jsonb` (ideal for storing immutable competition `RuleSet` snapshots — [ADR-0006](0006-tenancy-and-organisation-model.md) — alongside relational columns without giving up ACID transactions).
  - Lightweight, fast-booting official container images (`postgres:17-alpine`) start in ~1–2 seconds in `.NET Aspire` and `Testcontainers` on developer laptops and GitHub Actions Linux runners.
- **Managed Azure configuration (aligning with #8):**
  - Hosted on **Azure Database for PostgreSQL — Flexible Server** in an EU region (**Sweden Central**, `swedencentral`), ensuring strict EU/EEA data residency under GDPR.
  - Starts on a cost-effective **Burstable SKU** (e.g. `Standard_B1ms` / `B2s` with 32 GiB storage), with automated daily backups and **Point-in-Time Restore (PITR)** (7–35 days retention) enabled from day one, and zero-downtime vertical scaling available for major championship weekends.
  - Authenticates from Azure compute using **Microsoft Entra Managed Identity** (passwordless token authentication in production) and TLS 1.3 encryption in transit + Azure storage encryption at rest.

### 2. Per-Module Schema Isolation in a Single Shared Database

In accordance with our modular monolith architecture ([ADR-0004](0004-backend-runtime-and-architectural-style.md)) and shared-database tenancy model ([ADR-0006](0006-tenancy-and-organisation-model.md)), all modules connect to the **same physical PostgreSQL database** (`gymnastics_competition`), but each module owns an **isolated PostgreSQL schema and its own `DbContext`**:

| Module | `DbContext` (internal to module) | PostgreSQL Schema | Migration History Table | Key Tables Owned |
| --- | --- | --- | --- | --- |
| **`Users`** | `UsersDbContext` | `users` | `users."__EFMigrationsHistory"` | `users`, `external_identities`, `platform_role_assignments` |
| **`Organisations`** | `OrganisationsDbContext` | `organisations` | `organisations."__EFMigrationsHistory"` | `organisations`, `organisation_memberships`, `gymnasts`, `teams`, `team_members` |
| **`Competitions`** | `CompetitionsDbContext` | `competitions` | `competitions."__EFMigrationsHistory"` | `rule_sets`, `competitions`, `competition_classes`, `competition_disciplines` |
| **`Applications`** | `ApplicationsDbContext` | `applications` | `applications."__EFMigrationsHistory"` | `competition_applications`, `application_roster_entries` |
| **`Scheduling`** | `SchedulingDbContext` | `scheduling` | `scheduling."__EFMigrationsHistory"` | `competition_schedules`, `schedule_sessions`, `rotation_slots` |

- **Strict cross-module database boundary rules:**
  1. **No cross-schema foreign keys or SQL joins:** A table in `applications` (e.g. `competition_applications`) stores `competition_id`, `applying_organisation_id`, and `team_id` as typed `uuid` values, **not** as database-level cross-schema `FOREIGN KEY` references into `competitions.competitions` or `organisations.teams`.
  2. **Referential integrity across modules is enforced in application handlers via `.Contracts`:** Before creating or accepting an application, `Applications` validates competition state and team ownership through `ICompetitionsContracts` / `IOrganisationsContracts` ([ADR-0004](0004-backend-runtime-and-architectural-style.md)), while reacts to lifecycle changes (such as team withdrawal) via in-process integration events.
  3. **Within a module's own schema, full relational constraints apply:** Foreign keys, `UNIQUE` constraints (e.g. preventing the same `TeamId` from applying twice to the same `CompetitionClassId`, or preventing duplicate rotation slot assignments on **Floor**, **Tumble** or **Trampet**), and `CHECK` constraints are enforced directly by PostgreSQL.

### 3. EF Core 10 Data Access Conventions

- **Persistence-ignorant `Domain/` model:**
  - Entities, aggregates, and value objects in `GymnasticsCompetition.Modules.<Module>.Domain` contain pure C# business logic and have **zero dependencies on `Microsoft.EntityFrameworkCore`** ([ADR-0004](0004-backend-runtime-and-architectural-style.md)).
  - All table, column, key, value-converter, and index mappings live in `Persistence/Configurations/*Configuration.cs` implementing `IEntityTypeConfiguration<T>`.
- **Write path (Command slices) vs Read path (Query slices):**
  - **Command handlers** load the aggregate root through the module's `DbContext`, invoke domain methods (e.g. `schedule.AssignRotationSlot(...)` or `team.AddGymnast(...)`), and call `SaveChangesAsync(cancellationToken)`. Domain events raised by aggregates are dispatched via a shared EF Core `SaveChangesInterceptor`.
  - **Query handlers** set `.AsNoTracking()` (configured by default on read queries) and project directly into response DTO records using LINQ `.Select(...)` against the module's `DbContext`. No generic `IRepository<T>` wrapper is used.
- **Standard database mapping conventions (shared via `GymnasticsCompetition.Common` persistence helpers):**
  - **Naming:** `UseSnakeCaseNamingConvention()` (`EFCore.NamingConventions`) maps PascalCase C# types and properties to idiomatic PostgreSQL `snake_case` tables, columns, primary keys, foreign keys, and indexes.
  - **Primary keys:** Strongly-typed `readonly record struct` IDs (`OrganisationId`, `TeamId`, `GymnastId`, `CompetitionId`, `CompetitionApplicationId`, `CompetitionScheduleId`) backed by **`UUIDv7` (`Guid.CreateVersion7()`)** and stored as native PostgreSQL `uuid`. `UUIDv7` is time-ordered, giving sequential B-tree index locality comparable to `bigint` without central sequence coordination.
  - **Timestamps:** All timestamps use `DateTimeOffset` normalized to UTC and mapped to PostgreSQL `timestamp with time zone` (`timestamptz`), obtained via `.NET`'s `TimeProvider` abstraction for deterministic testing.
  - **Optimistic concurrency:** Aggregate roots map PostgreSQL's system transaction column **`xmin`** (`.UseXminAsConcurrencyToken()`) as an automatic row version token. Concurrent conflicting edits (e.g. two organisers editing the same **Floor** / **Tumble** / **Trampet** schedule or reviewing the same application simultaneously) throw `DbUpdateConcurrencyException`, which is mapped to an HTTP `409 Conflict` `ProblemDetails` response ([ADR-0005](0005-frontend-framework-and-api-contract.md)).
  - **Structured snapshots (`jsonb`):** Self-contained, immutable value snapshots owned by a single aggregate — such as the snapshotted `RuleSet` parameters on a `Competition` ([ADR-0006](0006-tenancy-and-organisation-model.md)) — use EF Core 10 complex-type / `ToJson()` mapping onto PostgreSQL `jsonb`.

### 4. Schema Migration Lifecycle

- **Authoring migrations per module:**
  - Each module's migrations live in `src/Modules/<Module>/GymnasticsCompetition.Modules.<Module>/Persistence/Migrations/`.
  - Each module registers `.MigrationsHistoryTable("__EFMigrationsHistory", "<module_schema>")` so modules can evolve their schemas independently without colliding in a shared migration table.
- **Local development (`.NET Aspire` + `src/MigrationService`):**
  - A lightweight worker project `src/MigrationService` references the module persistence registrations and runs `MigrateAsync()` sequentially across each module's `DbContext` on startup, then exits cleanly (`WaitForCompletion` in `src/AppHost`).
  - Running `dotnet run --project src/AppHost` starts the PostgreSQL container, runs `MigrationService` to apply all module migrations (and optional development seed data such as default European Gymnastics and Swedish TeamGym `RuleSet` presets), and only then starts `GymnasticsCompetition.Api`.
- **Continuous Integration (`Testcontainers.PostgreSql` + `Respawn`):**
  - In `tests/Api.IntegrationTests` ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)), a shared xUnit fixture boots a real `postgres:17-alpine` container via `Testcontainers.PostgreSql`, applies all module EF Core migrations from an empty database, and uses `Respawn` to reset table data between tests in milliseconds.
  - CI also verifies that no pending model changes exist (`HasPendingModelChanges()`) so a developer or AI agent cannot modify an entity configuration without generating the matching migration.
- **Staging and Production (EF Core Migration Bundles in CI/CD):**
  - We **never** run `Database.MigrateAsync()` inside `GymnasticsCompetition.Api` (`Program.cs`) in staging or production.
  - Instead, the public release workflow ([ADR-0002](0002-repository-strategy.md)) builds a self-contained **EF Core migration bundle** (or publishes the `MigrationService` container image alongside the API image).
  - During deployment from the private infrastructure repository (#8), the migration job runs to completion **before** the new `GymnasticsCompetition.Api` container revision is rolled out.
  - **Backward-compatible (expand-and-contract) migrations:** Schema changes follow an expand-and-contract discipline (additive column/table changes first; destructive drops in a subsequent release) so rolling container updates and rollbacks remain safe.

### 5. GDPR, Personal Data Minimisation and Retention Policy for Gymnasts

Because TeamGym gymnasts (particularly in Youth and Junior classes) are frequently **minors**, personal data handling is designed around **GDPR Data Minimisation (Art. 5(1)(c))**, **Storage Limitation (Art. 5(1)(e))**, and **Right to Erasure (Art. 17)**:

| Data Category | What Is Stored | What Is **Never** Stored | Retention & Erasure Policy |
| --- | --- | --- | --- |
| **Club `Gymnast` record** (`organisations.gymnasts`) | Given name, family name, birth year (or date of birth where required for age-class eligibility), competitive gender category (for **Women**, **Men**, **Mixed** [50%/50%] and **Open** section validation), owning `ClubOrganisationId`, `LastActiveAtUtc` | National identity numbers (e.g. Swedish *personnummer*), home address, personal phone/e-mail, medical data, or photos | Retained while active in the club across seasons so coaches do not re-enter squads each year. **Immediate erasure/anonymisation** available at any time on request; **automated inactivity retention limit of 24 months** (two competition seasons) after last team/application activity. |
| **Competition application roster snapshot** (`applications.application_roster_entries`) | Snapshot of competing gymnast eligibility data required by the organiser during the competition (`GymnastId`, display name / birth year / gender for rule-set verification on **Floor**, **Tumble**, **Trampet**) | Full club-internal notes or unrelated club roster members | Retained during the competition and post-event verification window; **personal identifiers automatically scrubbed/anonymised 12 months after `Competition.EndDate`** (leaving only non-personal aggregate counts and team-level records). |
| **Published schedules & results** (`scheduling.*`, future `scoring.*`) | Club name, **Team** name, **Section** (**Women**, **Men**, **Mixed**, **Open**), **Class**, and **Floor** / **Tumble** / **Trampet** rotation times and team scores | Individual minor gymnasts' personal data on public schedules (TeamGym is a team sport where the **Team** is the competing unit) | Retained indefinitely as historical sporting records since they contain team/club names rather than individual minor PII. |

- **How deletion vs anonymisation works in practice (`Right to Erasure`):**
  1. **Gymnast with no historical competition applications:** Hard-deleted (`DELETE FROM organisations.gymnasts`) immediately when removed by a club administrator.
  2. **Gymnast referenced by a past or completed competition application:** Immediately **anonymised in place** (personal name replaced with `"Anonymised Gymnast"`, exact birth date reduced to birth year or cleared after competition verification, and any external link removed) while preserving the row's non-personal gender/age-eligibility counters so historical team composition audits and competition records never suffer broken references.
  3. **Automated post-season retention cleanup:** A scheduled background job flags `Gymnast` records with `LastActiveAtUtc` older than **24 months** (configurable) for club admin renewal and automatically anonymises/purges unrenewed records after a 30-day notice window, and scrubs personal names from `application_roster_entries` **12 months** after a competition finishes.

### 6. Note on Future Binary / Media Storage (Music Uploads)

- Future modules such as **Music Submission (`Music`)** — where teams upload and validate audio tracks for **Floor** (and where applicable **Tumble** and **Trampet**) — will **not** store audio files as `bytea` blobs inside PostgreSQL.
- Instead, relational tables in PostgreSQL will store only **metadata** (track title, duration, format validation status, SHA-256 checksum, owning `TeamId` / `CompetitionId`, and object storage key), while the binary audio files will be stored in **Azure Blob Storage** in production and **Azurite** (Azure Storage emulator orchestrated via `.NET Aspire`) in local development and integration tests. A dedicated ADR / module design will finalise media upload limits, pre-signed URLs, and post-competition audio purge rules when the `Music` module is scheduled.

### Consequences

- Good, because PostgreSQL 17 + EF Core 10 (`Npgsql`) provides rock-solid ACID transactions, relational constraints, and `xmin` optimistic concurrency for TeamGym applications and **Floor** / **Tumble** / **Trampet** rotation schedules.
- Good, because one shared PostgreSQL database with per-module schemas (`users`, `organisations`, `competitions`, `applications`, `scheduling`) gives strong logical module boundaries while costing only a single small Azure Flexible Server instance (~$15–25/month on Burstable SKU).
- Good, because contributors and AI coding agents can boot the full database and apply all module migrations locally in seconds via `.NET Aspire` (`src/AppHost`) and `Testcontainers.PostgreSql` with zero cloud accounts or paid licenses.
- Good, because running schema migrations via a dedicated migration runner / bundle before container rollout prevents multi-replica startup race conditions and allows the runtime API database user to run with least-privilege DML permissions (no `DROP`/`ALTER` needed at runtime).
- Good, because the GDPR data-minimisation and 12/24-month retention/anonymisation design protects minor gymnasts' privacy while keeping historical team schedules and competition records intact.
- Bad, because disallowing cross-schema foreign keys requires validating cross-module IDs (`OrganisationId`, `CompetitionId`, `TeamId`) in application handlers via `<Module>.Contracts` and handling cross-module lifecycle events explicitly (an intentional modular monolith trade-off from [ADR-0004](0004-backend-runtime-and-architectural-style.md)).

### Confirmation

- **Architecture tests (`tests/ArchitectureTests`):**
  - Verify that `Domain` namespaces never reference `Microsoft.EntityFrameworkCore` or `Npgsql`.
  - Verify that each module's `DbContext` and entity configurations are `internal` and never referenced by another module.
- **Integration tests (`tests/Api.IntegrationTests`):**
  - Boot `postgres:17-alpine` via `Testcontainers.PostgreSql` on every pull request in GitHub Actions CI ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)).
  - Assert that all module `DbContext` migrations apply cleanly from an empty database and that `HasPendingModelChanges()` returns `false` for every module `DbContext`.
  - Verify optimistic concurrency (`409 Conflict` on stale `xmin`) and GDPR gymnast erasure/anonymisation behaviour against real PostgreSQL constraints.

## Pros and Cons of the Options

### Primary Database Engine and Managed Cloud Service

#### Option D1: PostgreSQL 17 (`Azure Database for PostgreSQL — Flexible Server`) + EF Core (`Npgsql`)

- Good, because PostgreSQL is the industry-standard open-source relational database with world-class `.NET` / EF Core support via `Npgsql`.
- Good, because the official `postgres:17-alpine` container image is tiny (~80 MB), starts in 1–2 seconds on both x86_64 and ARM64 machines, and uses minimal RAM in `.NET Aspire` and GitHub Actions CI.
- Good, because Azure Database for PostgreSQL Flexible Server is available in `swedencentral` (and `northeurope`/`westeurope`), supports Microsoft Entra Managed Identity, automated backups, PITR, and low-cost Burstable SKUs (`B1ms`).
- Good, because self-hosting clubs or federations can run PostgreSQL anywhere with zero proprietary licensing costs.

#### Option D2: Azure SQL Database (Serverless Tier) + EF Core

- Good, because Azure SQL Serverless supports auto-pause after an inactivity period and has deep .NET integration.
- Bad, because when an auto-paused Azure SQL Serverless database wakes up on the first request, cold-start resume takes ~30–60 seconds (often causing initial HTTP timeouts unless retry delays are long), whereas active days quickly accumulate vCore-second charges.
- Bad, because the local/CI container story (`mcr.microsoft.com/mssql/server`) requires a ~1.5 GB image, at least 2 GB of RAM, slower container startup in `Testcontainers`, and a proprietary EULA (with historical ARM64/Apple Silicon friction for open-source contributors).

#### Option D3: Document Database (`Azure Cosmos DB`)

- Good, because it offers turn-key global distribution and schema-less JSON document storage.
- Bad, because TeamGym data is fundamentally relational: cross-referencing organisations, multi-club memberships, teams, competition sections/classes, applications, and conflict-free **Floor** / **Tumble** / **Trampet** rotation schedules requires joins, uniqueness constraints, and multi-entity ACID transactions that are awkward and expensive in a document store.
- Bad, because the Cosmos DB local emulator is slow, heavy, and frequently brittle in Linux CI containers compared to PostgreSQL.

### Data Access Strategy in `.NET 10`

#### Option A1: EF Core 10 (`Npgsql`) for Commands, Migrations, and Projected `.AsNoTracking()` Queries

- Good, because EF Core 10 generates optimal SQL for LINQ `.Select(...)` projections while tracking aggregate changes and concurrency tokens (`xmin`) automatically on command paths.
- Good, because compile-time type checking catches schema/query mismatches immediately when domain or DTO properties change.
- Good, because EF Core 10 allows dropping down to parameterized `Database.SqlQuery<T>($"...")` for any specialised query without introducing a second data-access library.
- Neutral, because entity mappings are written explicitly in `IEntityTypeConfiguration<T>` classes to keep `Domain/` free of ORM attributes.

#### Option A2: Dapper + Manual SQL Scripts (`DbUp` / `Flyway`)

- Good, because every SQL statement is written explicitly by hand.
- Bad, because hand-maintaining CRUD SQL, change tracking, child-collection diffs (e.g. updating rotation slots or team rosters), and column-name refactoring across 5 modules adds substantial boilerplate and error surface.

#### Option A3: EF Core Wrapped in Generic `IRepository<T>` / `IUnitOfWork` Interfaces

- Good, because `DbContext` is hidden behind a custom interface.
- Bad, because EF Core's `DbContext` **already is** the Unit of Work and `DbSet<T>` **already is** the repository; wrapping it in generic interfaces either leaks `IQueryable<T>` anyway or forces query handlers to over-fetch full entity graphs instead of projecting slim DTOs in SQL.

### Module Database Topology

#### Option M1: Single Shared Database with One Schema per Module and One `DbContext` per Module

- Good, because each module (`users`, `organisations`, `competitions`, `applications`, `scheduling`) has an isolated namespace of tables and its own independent migration history (`<schema>."__EFMigrationsHistory"`).
- Good, because operating a single database instance keeps Azure costs, backups, point-in-time restore, and connection pooling simple.
- Good, because if any module ever needs to be extracted into a separate database in the future, its schema and `DbContext` are already cleanly separated.

#### Option M2: Separate Database per Module

- Good, because physical storage and compute can be scaled independently per module.
- Bad, because provisioning, backing up, migrating, and maintaining connection pools for 5 separate PostgreSQL databases from day one increases cloud cost and operational complexity for no MVP benefit.

#### Option M3: Single Shared `DbContext` and Single `public` Schema Across All Modules

- Good, because there is only one `DbContext` and one migration folder in the solution.
- Bad, because it breaks module autonomy ([ADR-0004](0004-backend-runtime-and-architectural-style.md)): a central `DbContext` must reference every module's domain types, making it trivial for developers or AI agents to write cross-module navigation properties and tight coupling.

### Schema Migration Strategy

#### Option S1: Dedicated Migration Runner (`MigrationService`) Locally + EF Core Migration Bundles in CI/CD

- Good, because local developers still get a one-command startup (`dotnet run --project src/AppHost` runs migrations automatically before the API boots).
- Good, because in staging and production, migrations run once in a controlled pre-deployment step rather than racing across multiple starting container replicas.
- Good, because the production `GymnasticsCompetition.Api` runtime identity only needs `SELECT`, `INSERT`, `UPDATE`, `DELETE` permissions rather than schema-altering DDL privileges.

#### Option S2: Run `Database.MigrateAsync()` Inside `Program.cs` on Production API Startup

- Good, because no separate migration step is needed in the deployment pipeline.
- Bad, because if two or more container replicas start concurrently, they compete to apply migrations; if a migration takes time or fails, container health probes fail and crash-loop the application.
- Bad, because the web application's runtime database identity must hold permanent DDL (`CREATE`/`ALTER`/`DROP`) permissions.

#### Option S3: Manual Out-of-Band SQL Execution by an Administrator

- Good, because a human inspects every SQL script at deployment time.
- Bad, because manual steps before deployments are error-prone and break automated continuous delivery ([ADR-0002](0002-repository-strategy.md), [ADR-0003](0003-engineering-workflow-and-quality-gates.md)).

## More Information

- Related issue: [#5](https://github.com/gyback/gymnastics-competition/issues/5)
- Parent epic: [#9](https://github.com/gyback/gymnastics-competition/issues/9)
- Related ADRs and decisions:
  - [ADR-0001 Record architecture decisions using MADR](0001-record-architecture-decisions.md)
  - [ADR-0002 Repository strategy — public app monorepo, private IaC repo and artefact hand-off](0002-repository-strategy.md)
  - [ADR-0003 Engineering workflow and quality gates for agent-assisted development](0003-engineering-workflow-and-quality-gates.md)
  - [ADR-0004 Backend runtime and architectural style — .NET 10 modular monolith with vertical slices](0004-backend-runtime-and-architectural-style.md)
  - [ADR-0005 Frontend framework and API contract — React 19 SPA with Vite, ASP.NET Core BFF and OpenAPI TypeScript client](0005-frontend-framework-and-api-contract.md)
  - [ADR-0006 Tenancy and organisation model — shared database with explicit organisation ownership, hierarchy-ready organisations and configurable rule sets](0006-tenancy-and-organisation-model.md)
  - [#6 Identity, authentication and authorisation](https://github.com/gyback/gymnastics-competition/issues/6) (user identity storage in `users` schema and privacy/data-protection overview)
  - [#8 Cloud hosting platform and Infrastructure as Code](https://github.com/gyback/gymnastics-competition/issues/8) (provisioning Azure Database for PostgreSQL Flexible Server in `swedencentral` and running the migration bundle job on deploy)
- Follow-up implementation issues:
  - [#27 Configure PostgreSQL + EF Core per-module persistence, migration runner and Testcontainers fixture](https://github.com/gyback/gymnastics-competition/issues/27)
  - [#28 Implement GDPR personal data erasure, anonymisation and retention workflows for gymnast records](https://github.com/gyback/gymnastics-competition/issues/28)
