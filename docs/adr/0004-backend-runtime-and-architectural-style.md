---
status: accepted
date: 2026-11-11
decision-makers: Gustav Gybäck
---

# Backend runtime and architectural style — .NET 10 modular monolith with vertical slices

<!-- Related issue: #3 -->

## Context and Problem Statement

The backend for the [TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) competition platform will be built on .NET and ASP.NET Core. We need to decide the target .NET runtime version, the macro architectural style, the code organisation inside each module, and the supporting conventions so that:

- the **MVP** (user management, club and team management, competition management, competition applications, and rotation-based scheduling across the three TeamGym disciplines **Floor**, **Tumble** and **Trampet** and sections **Women**, **Men**, **Mixed** and **Open**) can be delivered quickly by a solo maintainer working with AI coding agents, and
- **long-term capabilities** (discipline/tariff sheet declaration for **Floor**, **Tumble** and **Trampet**, music submission, judging and score calculation across all three disciplines, a live on-the-day competition runner, and hierarchical organisation management) can be added as new modules without rewiring the existing system.

## Decision Drivers

- **Long-term supported runtime (LTS):** Predictable security and servicing window without forced mid-cycle upgrades.
- **Simple to deploy and operate for a solo maintainer:** Avoid premature distribution, network hops, and multi-service operational overhead while keeping cloud hosting costs minimal ([ADR-0002](0002-repository-strategy.md)).
- **Clear, mechanically enforced module boundaries:** Future TeamGym capabilities must plug in cleanly, and AI coding agents must be able to work inside one bounded context without loading or accidentally coupling to the rest of the codebase ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)).
- **High testability of TeamGym domain rules:** Complex domain invariants — such as rotation-based scheduling across **Floor**, **Tumble** and **Trampet**, and team/section composition rules (8–10 gymnasts on **Floor**, 6 per round on **Tumble** and **Trampet**, gender composition for **Women**, **Men**, **Mixed** [50%/50%] and **Open**) — must be unit-testable in milliseconds without a database or HTTP host.
- **Conventional .NET structure:** Standard idioms that .NET developers, IDEs (Rider, Visual Studio, VS Code) and AI agents recognise immediately.

## Considered Options

### 1. .NET Runtime Version

- **Option R1:** **.NET 10 (`net10.0`, LTS)** — Long Term Support release (supported until November 2028), upgrading from LTS to LTS (`.NET 10` → `.NET 12`).
- **Option R2:** **.NET 11 (`net11.0`, STS)** — Standard Term Support release (18-month support window).

### 2. Macro Architectural Style

- **Option A1:** **Modular Monolith** — Single deployable ASP.NET Core host (`TeamGym.Arena.Api`) composed of isolated per-bounded-context modules (`Users`, `Organisations`, `Competitions`, `Applications`, `Scheduling`), with strict compile-time and architecture-test boundaries.
- **Option A2:** **Traditional Layered / Clean Architecture Monolith** — Single solution-wide `Domain`, `Application`, `Infrastructure`, and `Api` project quadruplet shared across all business areas.
- **Option A3:** **Microservices** — Separate deployable services and databases per bounded context communicating over HTTP/gRPC and an external message broker from day one.

### 3. Code Organisation Within Modules

- **Option M1:** **Vertical Slices with a Rich Domain Core** — Feature folders per use case (`Features/<UseCase>/` containing the Minimal API endpoint, request/response contract, validator, and command/query handler) paired with a persistence-ignorant `Domain/` folder for aggregates, entities, value objects, and domain services when business rules are non-trivial.
- **Option M2:** **Horizontal Layers per Module** — Separate `Controllers/`, `Services/`, `Repositories/`, `DTOs/`, and `Models/` folders (or sub-projects) inside every module regardless of complexity.

### 4. Supporting Backend Choices (API Surface, Dispatch, Orchestration & Boundary Enforcement)

- **Option S1:** **ASP.NET Core Minimal APIs + explicit plain handlers (no third-party mediator library) + `.NET Aspire` local orchestration + `ArchUnitNET` architecture tests.**
- **Option S2:** **ASP.NET Core MVC Controllers + third-party mediator library (e.g. MediatR) + manual Docker Compose only.**

## Decision Outcome

Chosen options:

1. **Runtime — Option R1:** Target **.NET 10 (`net10.0`, LTS)** across all backend and test projects, following an LTS-to-LTS upgrade cadence.
2. **Macro architecture — Option A1:** Build the backend as a **Modular Monolith** deployed as a single container image, with one module assembly per bounded context and explicit `.Contracts` assemblies for cross-module communication.
3. **Intra-module structure — Option M1:** Organise each module into **Vertical Slices (`Features/`) backed by a pure `Domain/` model** for TeamGym business invariants.
4. **Supporting choices — Option S1:** Use **ASP.NET Core Minimal APIs**, **plain strongly-typed handlers** (with lightweight in-process domain/integration event dispatch), **`.NET Aspire`** for one-command local orchestration and OpenTelemetry defaults, and **`ArchUnitNET`** in `tests/ArchitectureTests` to enforce module boundaries in CI.

### 1. Runtime Version and Upgrade Path

- All `.NET` projects target **`net10.0`** (LTS, supported until November 2028), configured centrally in the root `Directory.Build.props` alongside `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`, and `<AnalysisLevel>latest-all</AnalysisLevel>`.
- A root `global.json` pins the `.NET 10` SDK band (`10.0.100` with `rollForward: latestFeature`) so local machines and GitHub Actions CI use deterministic compiler and analyser versions.
- **Upgrade policy:** We track **LTS-to-LTS** releases by default (planning to upgrade to `.NET 12 LTS` after its release in November 2027). `.NET 11 (STS)` is skipped unless a specific runtime feature becomes a hard blocker, avoiding an unnecessary mandatory upgrade inside an 18-month STS window.

### 2. Modular Monolith Bounded Contexts

The system runs as a single ASP.NET Core process (`TeamGym.Arena.Api`) that registers independent modules.

#### Initial MVP Modules

| Module | Bounded Context & Responsibilities |
| --- | --- |
| **`Users`** | User profiles, onboarding, identity claims mapping, and platform/resource role assignments (in coordination with the authentication ADR, #6). |
| **`Organisations`** | Clubs (and future regional/national federations), coaches, gymnasts, and **Teams** — enforcing TeamGym team and **Section** (**Women**, **Men**, **Mixed**, **Open**) composition invariants. |
| **`Competitions`** | Competition lifecycle, dates, venues, configured **Sections** (**Women**, **Men**, **Mixed**, **Open**), age/level **Classes**, active **Disciplines** (**Floor**, **Tumble**, **Trampet**), application windows, and competition rules. |
| **`Applications`** | Club applications entering teams into a competition's section/class, deadline validation, organiser review (accept/reject/waitlist), and withdrawal handling. |
| **`Scheduling`** | Building, validating, and republishing the competition running order from accepted applications — sessions, subgroups, warm-up slots, and rotation schedules across **Floor**, **Tumble** and **Trampet**. |

#### Planned Long-Term Modules (Added Without Rewiring MVP)

- **`Declarations`** — Pre-competition tariff/difficulty sheet declarations per discipline (**Floor**, **Tumble**, **Trampet**).
- **`Music`** — Upload, metadata verification, and cue management for **Floor** (and **Tumble** / **Trampet**) music tracks.
- **`Scoring`** — Judge panels, tariff/execution/artistry score entry across **Floor**, **Tumble** and **Trampet**, tie-break rules, and official result publication.
- **`Runner`** — Live competition-day orchestration, floor/apparatus calls, scoreboard feeds, and real-time updates.

#### Shared Kernel (`Common`)

- `TeamGym.Arena.Common` is kept deliberately small and free of business workflows. It contains only foundational building blocks shared by all modules:
  - Ubiquitous TeamGym primitive types (`Discipline`: `Floor`, `Tumble`, `Trampet`; `Section`: `Women`, `Men`, `Mixed`, `Open`).
  - Strongly-typed entity ID primitives, `Result<T>` / `Error` types for explicit domain failure handling without exceptions for control flow, `TimeProvider` helpers, and in-process domain/integration event abstractions.

### 3. Concrete Solution and Project Layout

Expanding on [ADR-0002](0002-repository-strategy.md), the `.NET` projects under `src/` and `tests/` follow a predictable layout:

```text
/
├── src/
│   ├── AppHost/                                      # .NET Aspire local orchestrator (API + PostgreSQL + Web)
│   ├── ServiceDefaults/                              # OpenTelemetry, health checks, resilience & discovery defaults
│   ├── Api/                                          # ASP.NET Core host, middleware, auth, OpenAPI, module composition
│   ├── Common/                                       # Shared kernel (Result, DomainEvent, Discipline, Section, etc.)
│   ├── Modules/
│   │   ├── Users/
│   │   │   ├── TeamGym.Arena.Modules.Users/
│   │   │   └── TeamGym.Arena.Modules.Users.Contracts/
│   │   ├── Organisations/
│   │   │   ├── TeamGym.Arena.Modules.Organisations/
│   │   │   └── TeamGym.Arena.Modules.Organisations.Contracts/
│   │   ├── Competitions/
│   │   │   ├── TeamGym.Arena.Modules.Competitions/
│   │   │   └── TeamGym.Arena.Modules.Competitions.Contracts/
│   │   ├── Applications/
│   │   │   ├── TeamGym.Arena.Modules.Applications/
│   │   │   └── TeamGym.Arena.Modules.Applications.Contracts/
│   │   └── Scheduling/
│   │       ├── TeamGym.Arena.Modules.Scheduling/
│   │       └── TeamGym.Arena.Modules.Scheduling.Contracts/
│   └── Web/                                          # Frontend web application (see ADR #4)
└── tests/
    ├── ArchitectureTests/                            # ArchUnitNET rules enforcing module & layer boundaries
    ├── Api.IntegrationTests/                         # End-to-end HTTP + Testcontainers integration tests
    └── Modules/
        ├── Users.UnitTests/
        ├── Organisations.UnitTests/
        ├── Competitions.UnitTests/
        ├── Applications.UnitTests/
        └── Scheduling.UnitTests/
```

- **One implementation assembly + one optional `.Contracts` assembly per module:** Unlike Clean Architecture templates that multiply every module into 4 separate `.csproj` files (which would create 20+ projects for the MVP alone), each module is **one implementation project** (`TeamGym.Arena.Modules.<Module>`) with internal visibility by default, plus a lightweight **`<Module>.Contracts`** project containing only public DTOs, integration events, and query/facade interfaces needed by other modules.

### 4. Code Organisation Within Each Module (Vertical Slices + Rich Domain Model)

Inside each module project (`TeamGym.Arena.Modules.<Module>`), code is organised around **domain concepts** and **vertical feature slices**:

```text
TeamGym.Arena.Modules.Scheduling/
├── Domain/                           # Pure C# domain model (zero EF Core, HTTP, or infrastructure references)
│   ├── Schedules/
│   │   ├── CompetitionSchedule.cs    # Aggregate root & rotation invariants
│   │   ├── RotationSlot.cs           # Floor / Tumble / Trampet rotation assignment
│   │   └── RotationGenerator.cs      # Pure domain service for TeamGym rotation scheduling
├── Features/                         # Vertical slices — one folder per use case (command or query)
│   ├── GenerateSchedule/
│   │   ├── GenerateScheduleEndpoint.cs
│   │   ├── GenerateScheduleCommand.cs
│   │   ├── GenerateScheduleHandler.cs
│   │   └── GenerateScheduleValidator.cs
│   └── GetScheduleByCompetition/
│       ├── GetScheduleByCompetitionEndpoint.cs
│       ├── GetScheduleByCompetitionQuery.cs
│       └── GetScheduleByCompetitionHandler.cs
├── Persistence/                      # Module-scoped EF Core DbContext, entity configurations & migrations
│   ├── SchedulingDbContext.cs
│   └── Configurations/
└── SchedulingModule.cs               # DI registration (AddSchedulingModule) & endpoint mapping (MapSchedulingEndpoints)
```

- **Why vertical slices + rich domain model:**
  - Simple CRUD or query features (`GetScheduleByCompetition`) can read directly from the module's `DbContext` using projections in a single self-contained folder without ceremonial repository layers.
  - Complex TeamGym domain rules (`RotationGenerator`, team size and gender composition checks for **Women**, **Men**, **Mixed** and **Open**, application deadline and capacity rules) live in pure, persistence-ignorant classes under `Domain/` where they are exhaustively unit-tested with zero mocks or database spin-up.
  - AI coding agents working on a feature only need to read `Domain/` and the single `Features/<UseCase>/` folder, keeping context small and eliminating merge conflicts across features.
- **Internal by default:** All types in a module implementation assembly (`Domain`, `Features`, `Persistence`) are declared `internal` (with `InternalsVisibleTo` granted only to that module's unit/integration test project). Only `<Module>Module` registration extensions are `public`.

### 5. Supporting Architectural Decisions

- **Minimal APIs over MVC Controllers:**
  - Endpoints are defined with ASP.NET Core Minimal APIs using typed results (`TypedResults` / `Results<Ok<TResponse>, ProblemHttpResult, ValidationProblem>`) and grouped per module (`/api/v1/competitions`, `/api/v1/schedules`, etc.).
  - Typed results feed directly into first-class `.NET 10` OpenAPI document generation (`Microsoft.AspNetCore.OpenApi`) and RFC 9457 `ProblemDetails` error responses, which the frontend uses to generate a type-safe API client (#4).
- **Plain handlers over third-party mediator libraries:**
  - Each vertical slice defines an explicit handler class (implementing a lightweight `ICommandHandler<TCommand, TResult>` or `IQueryHandler<TQuery, TResult>` interface in `Common`) injected directly into the Minimal API endpoint via ASP.NET Core DI.
  - We intentionally avoid third-party mediator libraries such as `MediatR` (which moved to a commercial license) — direct handler injection gives compile-time navigation ("Go to Implementation" works directly in IDEs), clearer stack traces, and zero external licensing risk in an MIT open-source repository.
- **Inter-module communication and data isolation:**
  - Each module owns its own logical database schema (e.g. `users`, `organisations`, `competitions`, `applications`, `scheduling` inside the shared database — detailed in the data store ADR, #5) and its own `DbContext`.
  - A module may **never** reference another module's implementation project or query another module's tables directly.
  - **Synchronous cross-module reads** (e.g. `Scheduling` fetching accepted team applications from `Applications` and competition discipline/section config from `Competitions`) go through explicit public interfaces in `<Module>.Contracts` implemented by the owning module.
  - **Cross-module side effects** (e.g. notifying `Scheduling` when an accepted team withdraws in `Applications`) are published via an in-process integration event dispatcher defined in `Common`.
- **Local orchestration with `.NET Aspire`:**
  - `src/AppHost` (.NET Aspire) orchestrates local development: a single `dotnet run --project src/AppHost` starts the PostgreSQL container, runs database migrations, boots `TeamGym.Arena.Api`, starts the frontend dev server, and provides the local Aspire Dashboard for structured logs, distributed traces, and metrics.
  - `src/ServiceDefaults` wires standard OpenTelemetry, health check endpoints (`/health`, `/alive`), and HTTP resilience policies into `TeamGym.Arena.Api` while keeping the production container image completely standard and portable ([ADR-0002](0002-repository-strategy.md)).
- **Real-time communication (SignalR):**
  - **Not required for the MVP:** User, team, competition, application, and schedule authoring workflows are standard HTTP request/response interactions.
  - **Ready for the future Competition Runner:** When the live **Competition Runner** (`Runner`) and **Score Management** (`Scoring`) modules are built, ASP.NET Core **SignalR** hubs will be mapped inside those modules in the same `TeamGym.Arena.Api` host, broadcasting live rotation progressions and scores without architectural rework.

### Consequences

- Good, because `.NET 10 (LTS)` provides a stable, supported runtime through November 2028 with native OpenAPI, C# 14, and ASP.NET Core Minimal API improvements.
- Good, because deploying a single modular monolith container keeps Azure hosting simple and inexpensive for a solo maintainer while preserving strict domain boundaries inside the codebase.
- Good, because vertical slices + `internal` module types allow AI coding agents and contributors to implement or review a feature in one small folder without unintended side effects in other modules.
- Good, because pure `Domain/` models make TeamGym scheduling (**Floor**, **Tumble**, **Trampet**) and team composition rules (**Women**, **Men**, **Mixed**, **Open**) fast and deterministic to unit test.
- Good, because avoiding commercial third-party mediator libraries keeps the dependency tree permissive (MIT/Apache-2.0) and IDE navigation direct.
- Bad, because synchronous cross-module reads require mapping through `<Module>.Contracts` rather than writing ad-hoc cross-module EF Core joins (an intentional trade-off that prevents spaghetti coupling).

### Confirmation

- `tests/ArchitectureTests` runs on every pull request in CI ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)) and fails the build if:
  - any `TeamGym.Arena.Modules.<Module>` assembly references another module's implementation assembly (only `<OtherModule>.Contracts` and `TeamGym.Arena.Common` are permitted),
  - any class in a `.Domain` namespace references `Microsoft.EntityFrameworkCore`, `Microsoft.AspNetCore.*`, or infrastructure packages,
  - any module assembly references `TeamGym.Arena.Api`, or
  - domain, feature handler, or `DbContext` types inside a module implementation assembly are declared `public`.
- `dotnet run --project src/AppHost` boots the complete local environment from a fresh clone with Docker running.

## Pros and Cons of the Options

### Runtime Version

#### Option R1: .NET 10 (`net10.0`, LTS)

- Good, because Long Term Support guarantees patches and security updates until November 2028.
- Good, because Azure managed services, GitHub Actions runners, and container base images have first-class LTS support.
- Good, because upgrading from LTS to LTS (`.NET 10` → `.NET 12`) halves upgrade churn for a solo maintainer.

#### Option R2: .NET 11 (`net11.0`, STS)

- Good, because it provides early access to incremental runtime features once released.
- Bad, because Standard Term Support expires 18 months after release, forcing a mandatory runtime upgrade shortly after launch.

### Macro Architectural Style

#### Option A1: Modular Monolith

- Good, because it combines the operational simplicity of a single deployable container with the domain isolation of separate bounded contexts.
- Good, because cross-module calls and transactions happen in-process without network latency, distributed tracing complexity, or eventual-consistency failure modes during the MVP.
- Good, because if any future module (such as the live **Competition Runner**) ever needs independent scaling, its `.Contracts` boundary and isolated database schema make extraction straightforward.
- Neutral, because boundaries must be guarded by compiler visibility (`internal`) and automated architecture tests so developers do not take shortcuts.

#### Option A2: Traditional Layered / Clean Architecture Monolith

- Good, because a 4-project solution (`Domain`, `Application`, `Infrastructure`, `Api`) is familiar from generic .NET templates.
- Bad, because all bounded contexts share a single `Domain` and `Infrastructure` project, allowing `Applications`, `Scheduling`, `Users`, and future `Scoring` code to become tightly coupled over time.
- Bad, because modifying one feature routinely requires touching files across 4 separate projects and deep folder trees, increasing context size for AI coding agents.

#### Option A3: Microservices

- Good, because each service can be deployed and scaled independently by separate teams.
- Bad, because operating 5+ services, service-to-service authentication, distributed sagas, and message brokers is heavy, costly overkill for a solo maintainer and a TeamGym workload.
- Bad, because local development and end-to-end integration testing become significantly slower and more brittle.

### Code Organisation Within Modules

#### Option M1: Vertical Slices (`Features/`) with a Rich Domain Core (`Domain/`)

- Good, because everything needed to understand or change one use case (endpoint, request/response, validation, handler) sits in a single feature folder.
- Good, because complex TeamGym invariants (rotation scheduling across **Floor**, **Tumble**, **Trampet**; section rules for **Women**, **Men**, **Mixed**, **Open**) still live in a rich, persistence-ignorant `Domain/` model rather than being duplicated across procedural scripts.
- Good, because simple read queries bypass unnecessary repository/mapper layers and project directly from `DbContext`.

#### Option M2: Horizontal Layers Within Each Module

- Good, because files are grouped by technical kind (`Controllers`, `Services`, `Repositories`).
- Bad, because adding or reviewing a single endpoint requires jumping across multiple technical folders and Pass-Through service/repository abstractions.
- Bad, because "fat service" classes (e.g. `CompetitionService` with 30 methods) become merge bottlenecks when multiple contributors or agents work in parallel.

### Supporting Choices

#### Option S1: Minimal APIs + Plain Handlers + .NET Aspire + ArchUnitNET

- Good, because Minimal APIs with `TypedResults` are concise, fast, colocate naturally with vertical slice folders, and produce accurate OpenAPI schemas out of the box in `.NET 10`.
- Good, because plain strongly-typed handlers (`ICommandHandler` / `IQueryHandler`) have zero external licensing issues, zero reflection magic, and direct IDE code navigation.
- Good, because `.NET Aspire` gives one-command local orchestration (`AppHost`) and production-ready OpenTelemetry defaults (`ServiceDefaults`) while still publishing standard container images.
- Good, because `ArchUnitNET` mechanically enforces architectural rules in CI.

#### Option S2: MVC Controllers + Third-Party Mediator Library + Docker Compose Only

- Good, because MVC Controllers and `MediatR` appear frequently in older .NET codebases and training data.
- Bad, because `MediatR` transitioned to a commercial license, creating friction for an MIT-licensed open-source project, and indirection via `ISender.Send` obscures call hierarchies.
- Bad, because MVC Controllers encourage grouping multiple unrelated use cases into single controller classes rather than self-contained vertical slices.

## More Information

- Related issue: [#3](https://github.com/gyback/teamgym-arena/issues/3)
- Parent epic: [#9](https://github.com/gyback/teamgym-arena/issues/9)
- Related ADRs and decisions:
  - [ADR-0001 Record architecture decisions using MADR](0001-record-architecture-decisions.md)
  - [ADR-0002 Repository strategy — public app monorepo, private IaC repo and artefact hand-off](0002-repository-strategy.md)
  - [ADR-0003 Engineering workflow and quality gates for agent-assisted development](0003-engineering-workflow-and-quality-gates.md)
  - [#4 Frontend framework and API contract](https://github.com/gyback/teamgym-arena/issues/4) (consumes the OpenAPI specification generated by `TeamGym.Arena.Api`)
  - [#5 Primary data store and data access](https://github.com/gyback/teamgym-arena/issues/5) (PostgreSQL, EF Core per-module schemas and migrations)
  - [#6 Identity, authentication and authorisation](https://github.com/gyback/teamgym-arena/issues/6) (authentication and resource-scoped authorisation policies in `Api` and `Users`)
  - [#7 Tenancy and organisation model](https://github.com/gyback/teamgym-arena/issues/7) (domain modelling for organisations, clubs, and multi-club roles)
  - [#8 Cloud hosting platform and Infrastructure as Code](https://github.com/gyback/teamgym-arena/issues/8) (hosting the modular monolith container image on Azure)
- Follow-up implementation issues:
  - [#11 Scaffold monorepo folder structure and root .NET configuration](https://github.com/gyback/teamgym-arena/issues/11)
  - [#18 Scaffold .NET 10 modular monolith host, Aspire orchestration and MVP module projects](https://github.com/gyback/teamgym-arena/issues/18)
  - [#19 Add architecture test suite (`tests/ArchitectureTests`) to enforce module boundaries](https://github.com/gyback/teamgym-arena/issues/19)
