---
status: accepted
date: 2026-11-10
decision-makers: Gustav Gybäck
---

# Repository strategy — public app monorepo, private IaC repo and artefact hand-off

<!-- Related issue: #1 -->

## Context and Problem Statement

The [TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) competition application is open source (this repository), while the deployment infrastructure for the hosted service must live in a **separate private repository** as Infrastructure as Code (IaC). Before writing any application or infrastructure code, we need to decide:

1. How the public application repository hands off deployable versions to the private infrastructure repository.
2. How this public repository is structured internally to house the .NET backend, the web frontend, tests, documentation and build configuration.

## Decision Drivers

- Anyone must be able to clone, build, test and run the application locally from this public repository alone — with zero dependency on the private repository.
- No secrets, tenant IDs, subscription IDs, domain names or environment-specific configuration may appear in the public repository.
- A clear, reproducible, auditable hand-off from "built in public" to "deployed in private" using immutable, versioned artefacts.
- A conventional directory layout for .NET + JavaScript/TypeScript projects so both humans and AI coding agents find files predictably.
- Easy for contributors: one repository and one pull request workflow for full-stack application changes.

## Considered Options

### Hand-off between repositories

- **Option H1:** Public repo builds and publishes versioned OCI container images (to GitHub Container Registry, `ghcr.io`) and release artefacts; private repo deploys a pinned version by tag/digest.
- **Option H2:** Private repo checks out the public repo at a git tag and builds the application itself during deployment.
- **Option H3:** Private repo references the public repo as a git submodule and builds/deploys from it.

### Internal layout of the public repository

- **Option L1:** Structured monorepo with `src/` (containing both .NET backend projects and the frontend web project), `tests/`, `docs/` and `build/`, coordinated by a single root `.slnx` / `.sln` solution.
- **Option L2:** Separate top-level directories per deployable (`backend/`, `frontend/`) with completely independent tooling roots.

## Decision Outcome

Chosen options:

1. **Hand-off — Option H1: Publish versioned container images and release artefacts to GHCR; deploy pinned versions from the private IaC repo.**
2. **Internal layout — Option L1: Structured monorepo (`src/`, `tests/`, `docs/`, `build/`) with a root .NET solution.**

### 1. Hand-off and Versioning Mechanics

- **Public artefact publishing:** GitHub Actions in this repository builds, tests and packages the application into OCI container images (and any companion artefacts such as database migration bundles) and pushes them to GitHub Container Registry (`ghcr.io/gyback/gymnastics-competition/...`).
- **Versioning scheme:** Semantic Versioning (`vMAJOR.MINOR.PATCH`) driven by [Conventional Commits](https://www.conventionalcommits.org/) (to be detailed further in the engineering workflow ADR, #2). Every release tag produces an immutable image tagged with both the SemVer version and the git commit SHA, plus an attested image digest (`@sha256:...`) and auto-generated release notes.
- **Deployment trigger:** Publishing a release in this public repository does **not** directly mutate production infrastructure. Instead, deployments are triggered by updating the pinned image version/digest in the private infrastructure repository via a pull request (either opened automatically by a release bot / repository dispatch workflow or via Renovate/Dependabot, and promoted across environments in the private repo). This keeps environment promotion, infrastructure changes and rollbacks audited in one place without granting the public repo credentials to the cloud environment.

### 2. Internal Repository Layout

The public repository uses a single, predictable monorepo layout:

```text
/
├── .github/                  # GitHub Actions workflows, issue forms and PR templates
├── build/                    # Shared build scripts, containerfiles or CI helpers (when needed)
├── docs/
│   └── adr/                  # Architecture Decision Records (MADR)
├── src/                      # Production source code (.NET modules, API host, AppHost, web frontend)
│   └── Web/                  # Frontend application (React / TypeScript)
├── tests/                    # Automated test projects (unit, integration, architecture, end-to-end)
├── Directory.Build.props     # Shared MSBuild properties (target framework, nullable, analysers)
├── Directory.Packages.props  # Central Package Management (CPM) for NuGet dependencies
└── GymnasticsCompetition.slnx # Root .NET solution
```

- **Local development:** Local orchestration (such as .NET Aspire AppHost or Docker Compose for local backing services like PostgreSQL) lives inside this repository under `src/` so that a fresh clone can be started with a single command. The exact local orchestrator and project names will be finalised in the backend architecture ADR (#3) and data store ADR (#5).

### Consequences

- Good, because any contributor or self-hoster can run the full stack locally or deploy the published container images to their own infrastructure without access to the private repo.
- Good, because the public repository never needs cloud credentials, Azure tenant/subscription IDs or access to the private repository.
- Good, because build happens once in public CI; the private repo deploys the exact tested binary artefact by version and digest.
- Good, because `src/` + `tests/` with `Directory.Build.props` and `Directory.Packages.props` at the root is the standard .NET convention expected by IDEs (Rider, Visual Studio, VS Code) and AI coding agents.
- Bad, because rolling out an application release to the hosted service requires a version bump commit/PR in the private repo (mitigated by automating the PR creation on release).

### Confirmation

- CI workflows in `.github/workflows/` build and test from this repository alone with no secrets required for PR validation.
- Secret scanning and review ensure no cloud environment identifiers or credentials are committed.
- Architecture and project structure checks verify that all production code resides under `src/` and all test projects reside under `tests/`.

## Pros and Cons of the Options

### Hand-off Option H1: Publish versioned container images to GHCR; private repo deploys pinned version

- Good, because "build once, deploy anywhere" guarantees that what was tested in public CI is the exact image deployed to staging and production.
- Good, because external community members can also pull and run the official container images.
- Good, because the public repository requires zero secrets or permissions for the private repo or Azure tenant.
- Neutral, because container image tags/digests must be bumped in the private repo to trigger a deployment.

### Hand-off Option H2: Private repo checks out public repo at a tag and builds it

- Good, because the private repo controls the entire build-and-deploy pipeline in one workflow run.
- Bad, because it duplicates build logic and CI time across both repositories.
- Bad, because it violates "build once" — a rebuild in the private repo could produce slightly different output or fail due to transient tooling differences.
- Bad, because open-source users do not get pre-built container images from the public project.

### Hand-off Option H3: Private repo references public repo as a git submodule

- Good, because git records the exact commit SHA of the public repo inside the private repo.
- Bad, because it still requires rebuilding the application inside the private repo (same drawbacks as H2).
- Bad, because git submodules add friction for both humans and automation tools.

### Layout Option L1: Monorepo with `src/`, `tests/`, `docs/`, `build/` and single root solution

- Good, because it matches standard .NET repository conventions (`src/` + `tests/` + root `Directory.Build.props` / `Directory.Packages.props`).
- Good, because full-stack features (API + frontend + tests) can be implemented and reviewed in a single atomic pull request.
- Good, because local orchestration (e.g. .NET Aspire AppHost) naturally sits under `src/` and references both backend projects and the frontend app.
- Neutral, because frontend tooling (`package.json`, Node configuration) lives in `src/Web/` (or a workspace root) alongside `.NET` files.

### Layout Option L2: Separate top-level `backend/` and `frontend/` directories

- Good, because `.NET` and Node.js tooling are strictly separated at the repository root.
- Bad, because `.NET` tooling (`dotnet test`, root solution, `Directory.Build.props`, .NET Aspire orchestration) either has to be pushed down into `backend/` (making root commands awkward) or reach across sibling top-level folders.
- Bad, because cross-cutting integration/end-to-end tests and shared API contract generation do not fit cleanly into either folder.

## More Information

- Related issue: [#1](https://github.com/gyback/gymnastics-competition/issues/1)
- Parent epic: [#9](https://github.com/gyback/gymnastics-competition/issues/9)
- Related decisions:
  - [#2 Engineering workflow and quality gates](https://github.com/gyback/gymnastics-competition/issues/2) (Conventional Commits, CI checks, release automation)
  - [#3 Backend runtime and architectural style](https://github.com/gyback/gymnastics-competition/issues/3) (concrete `.NET` projects under `src/` and `tests/`, local orchestration)
  - [#4 Frontend framework and API contract](https://github.com/gyback/gymnastics-competition/issues/4) (frontend project structure under `src/Web/`)
  - [#8 Cloud hosting platform and Infrastructure as Code](https://github.com/gyback/gymnastics-competition/issues/8) (deployment contract between the container image and Azure hosting)
- Follow-up implementation issues:
  - [#11 Scaffold monorepo folder structure and root .NET configuration](https://github.com/gyback/gymnastics-competition/issues/11)
  - [#12 Add release workflow to publish versioned container images to GHCR](https://github.com/gyback/gymnastics-competition/issues/12)
