# Gymnastics Competition

An open-source administration tool for **troop gymnastics** competitions (e.g. TeamGym) — covering the
three apparatus disciplines **floor**, **tumbling** and **mini trampoline**.

> **Status: pre-alpha — decision phase.** No application code exists yet. We are currently recording the
> foundational architecture decisions (see [Architecture decisions](#architecture-decisions)). Watch the
> repository or check the [open issues](https://github.com/gyback/gymnastics-competition/issues) to follow along.

---

## Why

Running a troop gymnastics competition today usually means spreadsheets, e-mail threads and manual
copy-paste between the organiser, the clubs and the judges. Applications arrive in different formats,
schedules are rebuilt by hand every time a team withdraws, and nobody has a single source of truth.

This project aims to give organisers, clubs and officials **one shared, purpose-built tool** for the whole
competition lifecycle — starting with the parts that cause the most manual work: **applications** and
**scheduling**.

## Goals

- **Purpose-built for troop gymnastics.** Teams (not individuals) are the competing unit, and the three
  disciplines, age/level classes and rotation-based schedules are first-class concepts.
- **Open source and community-driven.** Built in the open so clubs, federations and developers can
  contribute and adapt it.
- **Verifiable by humans.** The project is developed largely with AI coding agents, but every change goes
  through reviewable pull requests, automated tests and documented decisions. Nothing is merged that a
  human cannot understand and verify.
- **Industry-standard practices.** Conventional tooling, explicit Architecture Decision Records,
  infrastructure as code, CI/CD and security-by-default — no clever one-offs.
- **Privacy-aware.** Gymnasts are frequently minors. Personal data is minimised and handled in line with
  GDPR.

## Scope

### MVP

| Area | What it covers |
| --- | --- |
| **User management** | Sign-up / sign-in, profiles, roles (e.g. organiser, club administrator, coach). |
| **Team management** | Clubs register teams, assign gymnasts and coaches, and pick class/level per team. |
| **Competition management** | Organisers create competitions: dates, venue, classes, disciplines, application window and rules. |
| **Competition application** | Clubs apply with teams to a competition; organisers review, accept or reject; deadlines are enforced. |
| **Competition scheduling** | Build the running order — sessions, rotations, apparatus and warm-up slots — from accepted applications, and republish when it changes. |

### Long term

- **Discipline declaration** — teams declare their elements/difficulty per apparatus before the competition.
- **Music submission** — upload and validate floor music per team.
- **Score management** — judges enter scores; results are calculated and published.
- **Competition runner** — live, on-the-day tool driving the schedule, calls, and score display.
- **Organisation management** — federations, regions and clubs as a hierarchy, with delegated administration.

### Non-goals (for now)

- Individual artistic or rhythmic gymnastics competitions.
- Payment processing (fees may be tracked, but not collected).
- Native mobile apps (the web app should be responsive instead).

## Glossary

| Term | Meaning |
| --- | --- |
| **Troop gymnastics / TeamGym** | A team-based gymnastics form where a team performs on floor, tumbling and mini trampoline. |
| **Discipline / apparatus** | Floor, tumbling or mini trampoline (trampette). |
| **Organisation** | A federation, region or club. Clubs own teams. |
| **Team** | The competing unit — a group of gymnasts from a club entered in a class. |
| **Class** | A competition category, typically by age group, level and gender composition. |
| **Competition** | An event hosted by an organiser, with one or more classes. |
| **Application** | A club's request to enter a team into a competition class. |
| **Schedule** | The running order: sessions, rotations, warm-ups and apparatus assignments. |

The glossary will evolve into the project's ubiquitous language and should be kept in sync with the code.

## Technology (intended)

Final choices are made through ADRs, but the starting preference is:

- **Backend:** .NET (ASP.NET Core)
- **Frontend:** React, possibly via Next.js
- **Hosting:** Microsoft Azure
- **Source control, issues and CI/CD:** GitHub (Issues, Pull Requests, Actions)

## Repository layout

This repository is the **public, open-source application**. Deployment infrastructure for the hosted
service (Infrastructure as Code, environment configuration, secrets references) lives in a separate
**private** repository. Nothing in this repository should depend on that private repository — anyone
should be able to build, test and run the application locally from here alone.

```text
docs/
  adr/            Architecture Decision Records
.github/
  ISSUE_TEMPLATE/ Issue forms (including ADR proposals)
```

The rest of the layout (`src/`, `tests/`, …) will be defined by an ADR.

## Architecture decisions

Significant decisions are recorded as **Architecture Decision Records (ADRs)** in [`docs/adr`](docs/adr),
using the [MADR](https://adr.github.io/madr/) format. The process is described in
[ADR-0001](docs/adr/0001-record-architecture-decisions.md):

1. A decision that needs to be made is opened as a GitHub issue with the `adr` label.
2. Discussion and option analysis happens on the issue.
3. The decision is proposed as a pull request adding an ADR file.
4. When the pull request is merged the ADR is `Accepted`. Superseded ADRs are never deleted — they are
   marked as superseded and linked to their replacement.

## Working with AI agents

Much of the code in this project is written with the help of AI coding agents. To keep the project
trustworthy:

- Every change goes through a pull request and is reviewed by a human before merging.
- Agents work from issues and ADRs; if a decision isn't recorded, it is raised as an issue rather than
  made silently in code.
- Changes must be covered by automated tests and pass CI.
- Pull requests should be small and focused enough to review in one sitting.

## Contributing

Contributions are very welcome — especially from people who run, judge or compete in troop gymnastics.
Right now the most valuable contributions are:

- Commenting on the open [ADR issues](https://github.com/gyback/gymnastics-competition/issues?q=is%3Aissue+is%3Aopen+label%3Aadr)
  with requirements, experience or alternatives.
- Describing how competitions are run in your club, region or country (rules, classes, schedules).

A `CONTRIBUTING.md` and code of conduct will be added as the project takes shape.

## License

[MIT](LICENSE) © Gustav Gybäck
