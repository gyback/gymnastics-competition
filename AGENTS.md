# AGENTS.md

Instructions for AI coding agents (and humans) working in this repository. This file is the single source
of truth for agent instructions ([ADR-0003](docs/adr/0003-engineering-workflow-and-quality-gates.md)).
Tool-specific files such as [`CLAUDE.md`](CLAUDE.md) only point here. Do not duplicate these instructions
elsewhere.

TeamGym Arena is an open-source administration tool for
[TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) competitions. See
[`README.md`](README.md) for goals and scope, and [`CONTRIBUTING.md`](CONTRIBUTING.md) for the full
contribution workflow.

> **Project status: pre-alpha, decision phase.** No application code exists yet. The layout and commands
> below are those decided in the ADRs. They take effect as the scaffolding issues (#11, #18, #21) land, and
> each scaffolding PR updates this file with the real commands.

## Guardrails

These rules are not optional.

1. **Read first.** Before planning or writing code, read `README.md`, this file, and the ADRs in
   [`docs/adr/`](docs/adr) that touch the area you are changing. Also read the issue you are working on.
2. **Never make an unrecorded architectural decision.** Do not introduce a new framework, foundational
   library, service, data store, cross-module dependency, or pattern that contradicts or goes beyond an
   accepted ADR. Stop and open an issue with the **ADR proposal** form (or comment on the existing `adr`
   issue) instead. When in doubt, ask.
3. **Work from an issue, and stay in its scope.** Every pull request references one issue
   (`Closes #N` or `Refs #N`). Do not add unrequested features, refactors or "while I'm here" changes.
   Raise them as new issues.
4. **Keep pull requests small** enough for a human to review in one sitting.
5. **Test what you change.** New domain rules, state transitions and API endpoints need meaningful
   behavioural tests (see [Testing](#testing)). Do not write assertion-free or coverage-padding tests.
6. **Verify locally before opening or updating a pull request.** Run formatting, linting, build and tests
   (see [Commands](#commands)). Do not push code you have not built.
7. **Never commit secrets** (connection strings with passwords, keys, tokens, certificates) or real
   personal data. Use synthetic test data only. Do not commit the production domain name
   ([ADR-0009](docs/adr/0009-cloud-hosting-and-infrastructure-as-code.md)).
8. **Never weaken a quality gate** to make a check pass: no disabling analysers, suppressing warnings,
   skipping tests, lowering lint rules or editing generated files by hand without an explicit reason that a
   reviewer approves.
9. **Disclose AI assistance** in the pull request template, and keep `Co-authored-by` trailers where your
   tool adds them.

## Domain language

Use the TeamGym terms exactly as written, in code, UI copy, tests and docs. The glossary in
[`README.md`](README.md#glossary) is authoritative.

| Concept | Values | Notes |
| --- | --- | --- |
| **Discipline** | **Floor**, **Tumble**, **Trampet** | Never "floor exercise", "tumbling", "trampoline", or "apparatus" as a synonym. |
| **Section** | **Women**, **Men**, **Mixed**, **Open** | **Mixed** is 50 % male / 50 % female. **Open** allows any gender composition. |
| **Team** | | The competing unit, owned by a club. Not individual gymnasts. |
| **Class** | | A competition category (section + age group and/or level). |
| **Organisation** | `Club`, `Region`, `Federation` | British spelling: *organisation*, *authorisation*. |
| **Competition**, **Application**, **Schedule** | | See the glossary. |

- Rule details such as squad size and the **Mixed** ratio come from configurable `RuleSet` presets
  ([ADR-0006](docs/adr/0006-tenancy-and-organisation-model.md)). Do not hard-code European Gymnastics
  constants.
- Swedish terms (*Fristående*, *Tumbling*, *Trampett*; *Dam*, *Herr*, *Mixed*, *Öppen*) belong only in the
  `sv` locale files, never in code identifiers
  ([ADR-0005](docs/adr/0005-frontend-framework-and-api-contract.md)).

## Repository layout

```text
.github/                       Issue forms, PR template, workflows
docs/adr/                      Architecture Decision Records (MADR)
src/
  AppHost/                     .NET Aspire local orchestration
  ServiceDefaults/             OpenTelemetry, health checks, resilience
  Api/                         ASP.NET Core host and BFF (TeamGym.Arena.Api)
  Common/                      Small shared kernel (Result, IDs, Discipline, Section)
  MigrationService/            Applies EF Core migrations, then exits
  Modules/<Module>/            TeamGym.Arena.Modules.<Module> + .Contracts
  Web/                         React 19 + Vite SPA
tests/
  ArchitectureTests/           ArchUnitNET boundary rules
  Api.IntegrationTests/        HTTP + Testcontainers PostgreSQL
  Modules/<Module>.UnitTests/  Domain and handler unit tests
  E2E/                         Playwright critical journeys
```

MVP modules: `Users`, `Organisations`, `Competitions`, `Applications`, `Scheduling`
([ADR-0004](docs/adr/0004-backend-runtime-and-architectural-style.md)). Deployment infrastructure lives in
a separate private repository and nothing here may depend on it
([ADR-0002](docs/adr/0002-repository-strategy.md)).

## Architecture rules

The architecture tests enforce most of these. Breaking one is an ADR question, not a code-review question.

**Backend** ([ADR-0004](docs/adr/0004-backend-runtime-and-architectural-style.md),
[ADR-0007](docs/adr/0007-primary-data-store-and-data-access.md)):

- A module never references another module's implementation project, `DbContext` or tables. Cross-module
  reads go through `<Module>.Contracts` interfaces. Cross-module side effects go through in-process
  integration events.
- Each module has its own PostgreSQL schema and `DbContext`. No cross-schema foreign keys or joins.
- Inside a module, use `Domain/` (pure C#, no EF Core, HTTP or Azure references), `Features/<UseCase>/`
  (one vertical slice per command or query) and `Persistence/`. Types are `internal` by default.
- Minimal APIs with `TypedResults` and RFC 9457 `ProblemDetails`. Plain handler classes; no MediatR or other
  mediator library.
- Query handlers use `.AsNoTracking()` projections. No generic repositories.
- IDs are strongly typed `readonly record struct`s backed by UUIDv7. Timestamps are UTC `DateTimeOffset`
  from `TimeProvider`.
- Never call `Database.MigrateAsync()` from the API. Schema changes are per-module EF Core migrations and
  follow expand-and-contract.
- Azure SDKs and `DefaultAzureCredential` appear only in host infrastructure registrations, never in
  modules ([ADR-0009](docs/adr/0009-cloud-hosting-and-infrastructure-as-code.md)). Configuration comes
  from environment variables, and every new setting is documented in `docs/deployment.md` in the same PR.

**Security and privacy** ([ADR-0006](docs/adr/0006-tenancy-and-organisation-model.md),
[ADR-0008](docs/adr/0008-identity-authentication-and-authorisation.md)):

- Every endpoint requires an authenticated user by default. Public endpoints opt out explicitly with
  `.AllowAnonymous()`.
- Check **permissions**, never roles, in endpoints and handlers. Organisation-scoped routes carry
  `{organisationId}`, and query handlers include the ownership predicate in the query itself.
- Return `404` for resources the user may not see, and `403` for resources they may see but not change.
- Gymnasts are often minors. Store only the personal data listed in
  [ADR-0007](docs/adr/0007-primary-data-store-and-data-access.md), and never log personal data, e-mail
  addresses or tokens. Log identifiers only.

**Frontend** ([ADR-0005](docs/adr/0005-frontend-framework-and-api-contract.md)):

- TypeScript strict mode. Feature folders under `src/Web/src/features/` mirror the backend modules.
- Call the API only through the generated `openapi-fetch` client. Regenerate `schema.d.ts` whenever the API
  contract changes and commit it; never edit it by hand.
- No hard-coded user-facing strings. Use `i18next` keys and provide both `en` and `sv` translations.
- Use the `shadcn/ui` primitives in `src/Web/src/shared/components/ui/` and keep them accessible
  (WCAG 2.2 AA).
- The browser never stores tokens. Authentication goes through the `/bff/*` endpoints.

## Commands

Run all of these before opening or updating a pull request. CI runs the same checks
([ADR-0003](docs/adr/0003-engineering-workflow-and-quality-gates.md)).

| Purpose | Command |
| --- | --- |
| Run the whole app locally (PostgreSQL, Keycloak, Mailpit, API, Web) | `dotnet run --project src/AppHost` |
| Build the backend (warnings are errors) | `dotnet build` |
| Check .NET formatting | `dotnet format --verify-no-changes` |
| Run backend tests (needs Docker for Testcontainers) | `dotnet test` |
| Install frontend dependencies | `npm ci` (in `src/Web`) |
| Type-check, lint and format-check the frontend | `tsc --noEmit`, ESLint (zero warnings) and `prettier --check` (in `src/Web`) |
| Build the frontend | `npm run build` (in `src/Web`) |

The scaffolding PRs replace the frontend row with the exact `npm run` script names.

## Testing

- **Unit tests** for every domain rule and handler. Domain tests use no mocks and no database.
- **Integration tests** for every endpoint, persistence change, migration and authorisation rule, against
  real PostgreSQL through Testcontainers. Include the cross-organisation access cases.
- **Architecture tests** must keep passing. Do not change a rule to make your code pass it.
- **E2E tests** (Playwright) for the critical journeys only.
- There is no coverage percentage gate. Cover behaviour and edge cases, not lines.

## Git and pull requests

- Branch from `main` with a short-lived branch: `feat/…`, `fix/…`, `docs/…`, `chore/…`, `refactor/…`,
  `test/…`, `build/…` or `ci/…`.
- Commit messages and pull request titles follow
  [Conventional Commits](https://www.conventionalcommits.org/), for example `feat(scheduling): …` or
  `docs(adr): …`. The PR title becomes the squash commit on `main`.
- Fill in every section of the [pull request template](.github/PULL_REQUEST_TEMPLATE.md).
- Agents never push to `main`, force-push a shared branch, merge a pull request, or skip hooks or signing.
  A human reviews and merges every PR.

## ADRs

Follow [ADR-0001](docs/adr/0001-record-architecture-decisions.md): an `adr` issue first, then a pull
request that copies [`docs/adr/template.md`](docs/adr/template.md) to the next number and updates the
index in [`docs/adr/README.md`](docs/adr/README.md). Accepted ADRs are immutable apart from typo fixes and
status updates. A changed decision gets a new ADR that supersedes the old one.
