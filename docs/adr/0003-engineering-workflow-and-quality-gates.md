---
status: accepted
date: 2026-11-10
decision-makers: Gustav Gybäck
---

# Engineering workflow and quality gates for agent-assisted development

<!-- Related issue: #2 -->

## Context and Problem Statement

The [TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) competition platform is built primarily with AI coding agents while being maintained by humans as an open-source project. Every change — whether authored by an AI agent or a human contributor — must be verifiable by humans, follow industry standards, and be traceable to an issue or Architecture Decision Record ([ADR-0001](0001-record-architecture-decisions.md)).

Before writing application code, we need to agree on our engineering workflow and automated quality gates across:

1. **Branching, commits and pull request workflow** (including how a solo maintainer satisfies human review).
2. **Continuous integration (CI), testing strategy and coverage policy.**
3. **Supply chain and repository security.**
4. **AI agent instructions and contributor documentation.**

## Decision Drivers

- Every change is reviewed by a human before it reaches `main`.
- Automated checks catch what reviewers shouldn't have to (formatting, linting, compilation warnings, tests, vulnerable dependencies, leaked secrets).
- AI coding agents get explicit, versioned instructions: what to read first, how to verify their work, and what they may not decide on their own.
- Low ceremony for a solo maintainer today; scales cleanly to multiple contributors later.
- Widely recognised industry conventions over custom rules.

## Considered Options

### 1. Branching, Merging and Commit Convention

- **Option B1:** Trunk-based development (GitHub Flow) — short-lived feature branches off `main`, pull requests, squash merge to `main`, and [Conventional Commits](https://www.conventionalcommits.org/).
- **Option B2:** GitFlow — long-lived `develop` and `release/*` branches alongside `main` with merge commits.
- **Option B3:** Trunk-based development with free-form commit messages.

### 2. Branch Protection and Review Policy (Solo Maintainer vs Multi-Contributor)

- **Option R1:** Protect `main` with required status checks, linear history, no force-pushes, and required pull requests; require 1 approving review (satisfied when the maintainer reviews PRs opened by a dedicated bot/app identity or external contributor, with explicit admin bypass permitted only for maintainer-authored PRs while solo).
- **Option R2:** Require 1 non-author review with no admin bypass even while solo (requiring a second dummy account to approve every maintainer PR).
- **Option R3:** No branch protection on `main` until additional maintainers join.

### 3. CI Quality Gates, Testing Pyramid and Coverage Policy

- **Option Q1:** GitHub Actions CI enforcing zero-warning builds, formatting/linting (`dotnet format --verify-no-changes`, ESLint, Prettier), unit + integration tests (using Testcontainers for real backing services) + architecture tests on every PR, E2E tests (Playwright) for critical user journeys, and informational code coverage reported on PRs without a hard percentage gate initially.
- **Option Q2:** Same CI checks as Q1, plus a strict global minimum line-coverage threshold (e.g. 80%) blocking PR merge from day one.
- **Option Q3:** Unit tests only in PR CI; run integration and E2E tests manually or nightly.

### 4. Supply Chain and Security Scanning

- **Option S1:** GitHub-native security suite (Dependabot version & security updates, Dependency Review Action on PRs, CodeQL static analysis for C# and JavaScript/TypeScript, and GitHub Secret Scanning with push protection).
- **Option S2:** Renovate Bot + third-party SAST/secret scanners requiring external service tokens.

### 5. Agent Governance and Contributor Documentation

- **Option A1:** Tool-agnostic [`AGENTS.md`](https://agents.md/) at the repository root as the single source of truth (with thin pointers such as `CLAUDE.md` where tool-specific discovery requires it), paired with `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` (Contributor Covenant), a PR template requiring issue linkage and AI-assistance disclosure, and standard MIT inbound=outbound licensing (no CLA).
- **Option A2:** Separate, duplicated instruction files per AI tool without a canonical root specification.
- **Option A3:** Require a Contributor License Agreement (CLA) or formal `Signed-off-by` DCO check on every commit.

## Decision Outcome

Chosen options:

1. **Branching & commits — Option B1:** Trunk-based development on `main` with short-lived branches, squash-merged pull requests, and Conventional Commits.
2. **Branch protection & review — Option R1:** Protected `main` with required CI status checks, linear history, and human PR review (allowing documented admin bypass only when the solo maintainer is also the PR opener).
3. **CI, testing & coverage — Option Q1:** Strict formatting, linting, warnings-as-errors, unit + integration + architecture tests on every PR, Playwright E2E tests for critical paths, and PR coverage reporting without a brittle percentage gate.
4. **Security & supply chain — Option S1:** GitHub-native security (Dependabot, Dependency Review Action, CodeQL, Secret Scanning with push protection).
5. **Agent governance & contributor docs — Option A1:** Canonical root `AGENTS.md` (with `CLAUDE.md` pointer), `CONTRIBUTING.md`, Contributor Covenant `CODE_OF_CONDUCT.md`, structured PR template with AI disclosure, and MIT inbound=outbound licensing.

### 1. Branching, Commits and Pull Request Workflow

- **Single trunk (`main`):** `main` is always releasable. All changes — documentation, ADRs, features, fixes and chores — land via short-lived branches (`feat/...`, `fix/...`, `docs/...`, `chore/...`) and pull requests targeting `main`.
- **Squash merge & linear history:** Pull requests are merged using **Squash and merge** so that each PR produces one atomic, revertible commit on `main`. Branches are automatically deleted on merge.
- **Conventional Commits:** PR titles (which become the squash commit subject on `main`) and commit messages follow [Conventional Commits v1.0.0](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `build:`, `ci:`, `chore:`, with optional scope such as `docs(adr):` or `feat(scheduling):`). This directly feeds the automated SemVer tagging and changelog generation decided in [ADR-0002](0002-repository-strategy.md).
- **Issue linkage:** Every pull request must reference an open issue (`Closes #N` or `Refs #N`). Neither humans nor agents merge unrequested scope.

### 2. Branch Protection and the Solo-Maintainer Review Model

- **Repository ruleset on `main`:**
  - Disallow direct pushes, force pushes and branch deletion.
  - Require pull requests before merging.
  - Require linear commit history (squash merge only).
  - Require conversation resolution before merging.
  - Require status checks to pass before merging (build, lint/format, tests, dependency review, CodeQL).
- **How solo review works in practice:**
  - Every agent-authored change is inspected line-by-line by the human maintainer in the GitHub PR diff before merge.
  - When an agent opens a PR under a bot/app identity (e.g. GitHub Copilot coding agent or a GitHub App), the maintainer submits a standard approving review.
  - When an agent runs locally under the maintainer's own GitHub credentials (where GitHub blocks self-approval), the maintainer still performs the human review on the PR, verifies all required status checks are green, and uses the repository administrator bypass exclusively for the self-approval constraint. As soon as a second maintainer joins, admin bypass for reviews is disabled.
- **Commit signing:** Encouraged (via SSH/GPG or GitHub web/API automated signing on squash merges) rather than hard-blocking local commits during pre-alpha, since squash merging via GitHub automatically signs the resulting commit on `main`.

### 3. CI Quality Gates, Testing Strategy and Coverage

Every pull request triggers `.github/workflows/ci.yml` on GitHub Actions. No PR may merge unless all gates pass:

- **Static analysis & formatting:**
  - **.NET:** `TreatWarningsAsErrors=true`, nullable reference types enabled, and .NET Roslyn analyzers enforced via root `Directory.Build.props`; formatting verified via `dotnet format --verify-no-changes`.
  - **Frontend:** TypeScript strict mode (`tsc --noEmit`), ESLint (zero warnings) and Prettier (`prettier --check`).
  - **PR title check:** Validate that the PR title conforms to Conventional Commits.
- **Testing pyramid (mandatory per PR):**
  - **Unit tests (`tests/*.UnitTests` & frontend component/unit tests):** Fast, isolated tests for domain logic (e.g. TeamGym team composition rules, rotation scheduling algorithms), application handlers and UI components. Mandatory for all new domain/application behaviour.
  - **Integration tests (`tests/*.IntegrationTests`):** Verify API endpoints, data access, migrations and authorisation against real ephemeral backing services (such as PostgreSQL via Testcontainers) inside GitHub Actions. Mandatory for any persistence or HTTP endpoint change.
  - **Architecture tests (`tests/ArchitectureTests`):** Enforce module boundaries, dependency direction and naming conventions in code so agents cannot accidentally introduce circular or cross-layer dependencies.
  - **End-to-end tests (`tests/E2E` with Playwright):** Verify critical full-stack user journeys (sign-in, competition creation, team application, schedule generation) in a real browser. Run in CI once the web and API scaffold lands.
- **Code coverage policy:**
  - Coverage is collected on every CI run (`dotnet test --collect:"XPlat Code Coverage"` and frontend coverage) and posted as a summary comment/report on the pull request so reviewers can immediately see untested paths.
  - We intentionally choose **no hard global coverage percentage threshold** initially: arbitrary percentage gates incentivize AI agents to write brittle, low-value assertion-free tests for boilerplate while still missing domain edge cases. Instead, reviewers (and agent instructions) enforce that all domain rules, state transitions and API endpoints have meaningful behavioral tests.

### 4. Supply Chain and Security Gates

- **Secret scanning & push protection:** GitHub Secret Scanning and Push Protection enabled on the repository to block accidental commits of credentials, keys or tokens.
- **Dependency vulnerability scanning:**
  - **Dependabot alerts & security updates** enabled for NuGet, npm and GitHub Actions (`github-actions` ecosystem), grouped sensibly in `.github/dependabot.yml` to avoid PR noise.
  - **Dependency Review Action** runs on every pull request to block introducing packages with known vulnerabilities or incompatible licenses.
- **Static Application Security Testing (SAST):** **GitHub CodeQL** runs on pull requests affecting C# or JavaScript/TypeScript code and on a weekly schedule on `main`.
- **Least-privilege workflows:** All GitHub Actions workflows declare explicit top-level `permissions: contents: read` (elevating per job only when necessary) and pin third-party actions appropriately.

### 5. AI Agent Governance and Contributor Documentation

- **Single source of truth (`AGENTS.md`):**
  - A root `AGENTS.md` defines repository conventions, build/test commands, architecture boundaries, ubiquitous TeamGym terminology (Floor, Tumble, Trampet; Women, Men, Mixed, Open), and strict guardrails for AI coding agents:
    1. Read `README.md`, `AGENTS.md` and `docs/adr/` before planning or writing code.
    2. Never make an unrecorded architectural decision or add a new foundational dependency silently — open or comment on an ADR issue instead.
    3. Keep pull requests small, focused on a single issue, and covered by tests.
    4. Run formatting, linting and tests locally before opening or updating a PR.
  - Tool-specific entrypoints (such as `CLAUDE.md`) are kept as minimal pointers to `AGENTS.md` so instructions never drift.
- **AI disclosure and attribution in PRs:**
  - The pull request template (`.github/PULL_REQUEST_TEMPLATE.md`) includes a checklist covering issue linkage, ADR compliance, test coverage and AI-assistance disclosure, plus `Co-authored-by` trailers where supported.
- **Contributor onboarding (`CONTRIBUTING.md` & `CODE_OF_CONDUCT.md`):**
  - `CONTRIBUTING.md` documents local setup, branching, Conventional Commits, running quality checks locally, the ADR workflow, and how humans and agents collaborate.
  - `CODE_OF_CONDUCT.md` adopts the standard [Contributor Covenant](https://www.contributor-covenant.org/).
  - Contributions are accepted under the repository's MIT License (**inbound=outbound** per GitHub Terms of Service) without requiring a CLA bot or mandatory `Signed-off-by` DCO check on every commit.

### Consequences

- Good, because every change on `main` is guaranteed to pass automated build, formatting, test and security gates and to be reviewed by a human.
- Good, because Conventional Commits + squash merging keep `git log` on `main` clean, readable and directly usable for automated SemVer releases ([ADR-0002](0002-repository-strategy.md)).
- Good, because `AGENTS.md` plus architecture tests give AI coding agents unambiguous rules and fast automated feedback when they violate conventions.
- Good, because GitHub-native security tools (Dependabot, CodeQL, Dependency Review, Secret Scanning) require zero external accounts or secrets in a public repository.
- Bad, because running integration tests (Testcontainers) and CodeQL on PRs adds a few minutes to CI run times (mitigated by path filtering and caching).
- Bad, because while the project has a single maintainer, self-authored PRs (including local agent sessions using the maintainer's credentials) require an admin bypass for the approval requirement even though status checks remain strictly enforced.

### Confirmation

- Repository ruleset on `main` enforces pull requests, linear history and required status checks.
- `.github/workflows/ci.yml` runs on every PR targeting `main` and blocks merge on build, lint, format or test failure.
- `AGENTS.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` and `.github/PULL_REQUEST_TEMPLATE.md` are present in the repository and referenced from `README.md`.

## Pros and Cons of the Options

### Branching & Commits

#### Option B1: Trunk-based development (GitHub Flow) with Squash Merge and Conventional Commits

- Good, because short-lived branches keep PRs small and easy for humans to verify.
- Good, because squash merging turns noisy agent/WIP commit histories on a branch into a single clean commit on `main`.
- Good, because Conventional Commits enable deterministic SemVer version bumps and changelog generation for container releases ([ADR-0002](0002-repository-strategy.md)).
- Neutral, because PR titles must follow a strict prefix format (enforced automatically by CI).

#### Option B2: GitFlow (`develop`, `release/*`, `main`)

- Good, because it separates ongoing integration from production releases when releases are infrequent and manually coordinated.
- Bad, because maintaining multiple long-lived branches adds merge overhead, confusion for AI agents, and ceremony that slows down a solo maintainer.

#### Option B3: Trunk-based development with free-form commit messages

- Good, because contributors do not need to learn Conventional Commits syntax.
- Bad, because release notes and SemVer version bumps cannot be reliably automated without manual tagging decisions.

### Branch Protection & Review Policy

#### Option R1: Protected `main` with required checks, PRs, and solo-maintainer admin bypass only for self-approval

- Good, because direct pushes to `main` are prevented and every change goes through a PR and full CI validation.
- Good, because external contributors and bot-authored PRs require explicit maintainer approval, while the solo maintainer is not forced to maintain a sockpuppet GitHub account.
- Neutral, because disabling admin bypass waits until a second maintainer joins the project.

#### Option R2: Strict non-author approval with no admin bypass while solo

- Good, because the ruleset has zero exceptions at the platform level.
- Bad, because a solo maintainer must log into a second dummy GitHub account just to click "Approve" on their own PRs — pure ceremony with no real review benefit.

#### Option R3: Unprotected `main` while solo

- Good, because it has zero friction.
- Bad, because an AI agent or human command with push permissions could accidentally push directly to `main` or bypass CI checks.

### CI, Testing & Coverage

#### Option Q1: Full testing pyramid (unit, Testcontainers integration, architecture, Playwright E2E) + informational coverage on PRs

- Good, because integration tests against real database containers catch SQL, EF Core mapping and concurrency bugs that mocks hide.
- Good, because architecture tests mechanically prevent structural drift in an agent-heavy codebase.
- Good, because informational coverage highlights untested code in PR review without encouraging gaming of a coverage number.
- Neutral, because reviewers must verify that new domain behaviour includes tests.

#### Option Q2: Strict minimum coverage threshold (e.g. 80%) blocking PR merge

- Good, because coverage can never drop below the configured number.
- Bad, because AI agents respond to hard coverage gates by generating superficial tests for DTOs and boilerplate rather than meaningful domain assertions.

#### Option Q3: Unit tests only on PRs

- Good, because CI runs in seconds.
- Bad, because integration and database regressions reach `main` undetected.

### Supply Chain & Security

#### Option S1: GitHub-native security suite (Dependabot, Dependency Review, CodeQL, Secret Scanning)

- Good, because all tools are built into GitHub, free for public open-source repositories, and require no third-party tokens or external dashboards.
- Good, because Dependency Review and Secret Scanning Push Protection prevent issues before they reach `main`.
- Neutral, because Dependabot is slightly less configurable than Renovate for complex monorepo grouping, though `.github/dependabot.yml` version groups cover `.NET`, `npm` and `github-actions` well.

#### Option S2: Renovate + external SAST/security scanners

- Good, because Renovate offers powerful custom grouping and automerge rules.
- Bad, because installing external GitHub Apps and managing third-party security dashboards adds operational overhead for a solo maintainer.

### Agent Governance & Contributor Docs

#### Option A1: Root `AGENTS.md` (with thin `CLAUDE.md` pointer), `CONTRIBUTING.md`, Contributor Covenant, PR template with AI disclosure, MIT inbound=outbound

- Good, because `AGENTS.md` is an open, tool-agnostic standard supported across modern coding agents, avoiding duplicate instructions.
- Good, because explicit guardrails ("open an issue instead of guessing on architecture") align agent behaviour with [ADR-0001](0001-record-architecture-decisions.md).
- Good, because MIT inbound=outbound keeps contribution friction low without CLA bots.

#### Option A2: Separate full instruction files per AI tool

- Good, because each tool reads its own native file.
- Bad, because instructions inevitably drift out of sync across files.

#### Option A3: Require CLA or mandatory DCO sign-off on all commits

- Good, because every commit carries an explicit legal attestation trailer.
- Bad, because DCO bots frequently fail on web-authored commits, dependency bots, and squash merges, adding unnecessary friction to an MIT-licensed project.

## More Information

- Related issue: [#2](https://github.com/gyback/gymnastics-competition/issues/2)
- Parent epic: [#9](https://github.com/gyback/gymnastics-competition/issues/9)
- Related ADRs and decisions:
  - [ADR-0001 Record architecture decisions using MADR](0001-record-architecture-decisions.md)
  - [ADR-0002 Repository strategy — public app monorepo, private IaC repo and artefact hand-off](0002-repository-strategy.md)
  - [#3 Backend runtime and architectural style](https://github.com/gyback/gymnastics-competition/issues/3)
  - [#4 Frontend framework and API contract](https://github.com/gyback/gymnastics-competition/issues/4)
- Follow-up implementation issues:
  - [#14 Add `AGENTS.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` and PR template](https://github.com/gyback/gymnastics-competition/issues/14)
  - [#15 Configure `main` branch protection ruleset, Dependabot and CodeQL security scanning](https://github.com/gyback/gymnastics-competition/issues/15)
  - [#16 Add GitHub Actions CI workflow for PR validation, quality gates and tests](https://github.com/gyback/gymnastics-competition/issues/16)
