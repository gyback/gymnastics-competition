# Contributing to TeamGym Arena

Thank you for your interest in TeamGym Arena. Contributions are welcome from developers and especially
from people who run, judge, coach or compete in TeamGym.

By taking part you agree to follow our [Code of Conduct](CODE_OF_CONDUCT.md).

> **Project status: pre-alpha, decision phase.** No application code exists yet. Right now the most
> valuable contributions are comments on the open
> [ADR issues](https://github.com/gyback/teamgym-arena/issues?q=is%3Aissue+is%3Aopen+label%3Aadr) and
> descriptions of how TeamGym competitions are run in your club, region or country.

## Ways to contribute

- **Share domain knowledge.** Comment on issues with how competitions, sections, classes, schedules and
  scoring work where you are.
- **Discuss decisions.** Comment on `adr` issues with requirements, experience or alternatives.
- **Report a bug or suggest a feature.** Open an issue. Search existing issues first.
- **Contribute code or docs.** Pick an open issue, comment that you are working on it, and open a pull
  request.

Do not report security vulnerabilities in a public issue. E-mail the maintainer at
[gustav@gyback.se](mailto:gustav@gyback.se) instead.

## Local setup

The development environment is defined by
[ADR-0004](docs/adr/0004-backend-runtime-and-architectural-style.md) and
[ADR-0005](docs/adr/0005-frontend-framework-and-api-contract.md). You will need:

- [.NET 10 SDK](https://dotnet.microsoft.com/download) (the exact band is pinned in `global.json`)
- [Node.js](https://nodejs.org/) LTS with npm (for `src/Web`)
- A container runtime such as [Docker Desktop](https://www.docker.com/products/docker-desktop/) or
  [Podman](https://podman.io/) (for .NET Aspire and Testcontainers)

Then run the whole application, including PostgreSQL, Keycloak and Mailpit, with:

```shell
dotnet run --project src/AppHost
```

No cloud account is needed. Local sign-in uses seeded test users in Keycloak
([ADR-0008](docs/adr/0008-identity-authentication-and-authorisation.md)).

These instructions become usable once the scaffolding issues (#11, #18, #21) are merged, and those pull
requests update this section.

## Workflow

We use trunk-based development with short-lived branches
([ADR-0003](docs/adr/0003-engineering-workflow-and-quality-gates.md)).

1. **Start from an issue.** Every pull request references an open issue. If none exists, open one first so
   the change can be discussed.
2. **Branch from `main`** using a prefix that matches the change: `feat/`, `fix/`, `docs/`, `chore/`,
   `refactor/`, `test/`, `build/` or `ci/` (for example `feat/scheduling-rotation-generator`).
3. **Make small, focused commits** using [Conventional Commits](#commit-messages).
4. **Run the quality checks locally** (see [below](#quality-checks)).
5. **Open a pull request** against `main`, fill in the
   [pull request template](.github/PULL_REQUEST_TEMPLATE.md), and link the issue (`Closes #N` or
   `Refs #N`).
6. **Review.** A maintainer reviews every pull request. All conversations must be resolved and all
   required checks must pass.
7. **Merge.** Pull requests are **squash-merged**, so the pull request title becomes the commit on `main`.
   Branches are deleted after merging.

Keep pull requests small enough to review in one sitting. Split large changes into several pull requests.

### Commit messages

Commit messages and pull request titles follow
[Conventional Commits v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/):

```text
<type>(<optional scope>): <description>
```

| Type | Use for |
| --- | --- |
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation only, including ADRs (`docs(adr): …`) |
| `refactor` | A code change that neither fixes a bug nor adds a feature |
| `test` | Adding or correcting tests |
| `build` | The build system, dependencies or container images |
| `ci` | CI workflows |
| `chore` | Other maintenance |

Use a module or area as the scope where it helps, for example `feat(scheduling): …` or
`fix(applications): …`. Mark breaking changes with `!` (`feat(api)!: …`) and a `BREAKING CHANGE:` footer.
These messages drive automated versioning and changelogs
([ADR-0002](docs/adr/0002-repository-strategy.md)).

## Quality checks

CI runs these on every pull request and blocks merging on any failure. Run them locally first:

```shell
dotnet build                         # warnings are errors
dotnet format --verify-no-changes    # .NET formatting
dotnet test                          # unit, integration and architecture tests (needs a container runtime)
```

In `src/Web`:

```shell
npm ci
npx tsc --noEmit                     # strict type check
npx eslint . --max-warnings 0        # lint with zero warnings
npx prettier --check .               # formatting
npm run build
```

The scaffolding pull requests replace these with the project's `npm run` scripts.

### Tests

- Add **unit tests** for new domain rules and handlers, and **integration tests** for new endpoints,
  persistence changes and authorisation rules.
- Keep the **architecture tests** passing. They enforce the module boundaries described in
  [ADR-0004](docs/adr/0004-backend-runtime-and-architectural-style.md).
- Add or update **Playwright E2E tests** when you change a critical user journey.
- There is no coverage percentage target. Reviewers look for meaningful tests of behaviour and edge cases.

### Test data

Use synthetic data only. Never commit real names, national identifiers, e-mail addresses or other personal
data, especially of gymnasts, who are often minors.

## Architecture decisions

Significant decisions are recorded as [Architecture Decision Records](docs/adr) in MADR format
([ADR-0001](docs/adr/0001-record-architecture-decisions.md)).

Before you change something architecturally significant (a new framework or foundational dependency, a
change to module boundaries, data storage, security or hosting), check whether an ADR covers it. If not:

1. Open an issue using the **ADR proposal** form. It is labelled `adr`.
2. Discuss requirements, options and trade-offs on the issue.
3. Open a pull request that copies [`docs/adr/template.md`](docs/adr/template.md) to the next number
   (`docs/adr/NNNN-title-with-dashes.md`) and adds it to the index in
   [`docs/adr/README.md`](docs/adr/README.md).
4. Merging the pull request accepts the decision.

Accepted ADRs are not rewritten. A changed decision gets a new ADR that supersedes the old one.

## Working with AI coding agents

Much of this project is written with AI coding agents. The same rules apply to agent-written and
human-written changes:

- Agents follow [`AGENTS.md`](AGENTS.md), the single source of truth for agent instructions. Tool-specific
  files such as [`CLAUDE.md`](CLAUDE.md) only point to it. Change agent instructions in `AGENTS.md`.
- Agents work from an issue, stay within its scope, and raise an issue rather than making an unrecorded
  architectural decision.
- **You are responsible for what you submit.** If an agent wrote your change, you must have read and
  understood it, and be able to explain it in review.
- Disclose AI assistance in the pull request template. Keep `Co-authored-by` trailers that your tool adds.
- A human reviews every pull request line by line before it is merged. While the project has a single
  maintainer, the maintainer's own pull requests are merged with an administrator bypass of the approval
  rule only; all status checks still apply
  ([ADR-0003](docs/adr/0003-engineering-workflow-and-quality-gates.md)).

## Licence

TeamGym Arena is licensed under the [MIT License](LICENSE). By contributing, you agree that your
contributions are licensed under the same licence (inbound = outbound). There is no Contributor Licence
Agreement and no `Signed-off-by` requirement.
