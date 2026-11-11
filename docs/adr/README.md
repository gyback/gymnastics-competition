# Architecture Decision Records

This directory holds the project's Architecture Decision Records (ADRs) in [MADR](https://adr.github.io/madr/)
format. The process is described in [ADR-0001](0001-record-architecture-decisions.md).

To propose a new decision, open an issue using the **ADR proposal** form, then copy [`template.md`](template.md)
to `NNNN-title-with-dashes.md` in a pull request.

| ADR | Title | Status |
| --- | --- | --- |
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions using MADR | Accepted |
| [0002](0002-repository-strategy.md) | Repository strategy — public app monorepo, private IaC repo and artefact hand-off | Accepted |
| [0003](0003-engineering-workflow-and-quality-gates.md) | Engineering workflow and quality gates for agent-assisted development | Accepted |
| [0004](0004-backend-runtime-and-architectural-style.md) | Backend runtime and architectural style — .NET 10 modular monolith with vertical slices | Accepted |
| [0005](0005-frontend-framework-and-api-contract.md) | Frontend framework and API contract — React 19 SPA with Vite, ASP.NET Core BFF and OpenAPI TypeScript client | Accepted |
| [0006](0006-tenancy-and-organisation-model.md) | Tenancy and organisation model — shared database with explicit organisation ownership, hierarchy-ready organisations and configurable rule sets | Accepted |
| [0007](0007-primary-data-store-and-data-access.md) | Primary data store and data access — PostgreSQL with EF Core 10, per-module schemas, migration bundles and GDPR retention policies | Accepted |

Open decisions are tracked as [issues labelled `adr`](https://github.com/gyback/gymnastics-competition/issues?q=is%3Aissue+label%3Aadr).
