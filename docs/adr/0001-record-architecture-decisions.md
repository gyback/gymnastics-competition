---
status: accepted
date: 2026-11-08
decision-makers: Gustav Gybäck
---

# Record architecture decisions using MADR

## Context and Problem Statement

The project is open source, started by a single maintainer, intended to welcome more contributors over time,
and developed largely with AI coding agents. Decisions about architecture, technology and process need to be
visible, reviewable and durable, so that:

- humans can verify *why* the code looks the way it does,
- new contributors can get up to speed without oral history,
- AI agents have an authoritative source to follow instead of guessing or re-deciding.

How should we record architecturally significant decisions?

## Decision Drivers

- Must live next to the code and be versioned with it.
- Must be a widely recognised industry practice.
- Must be easy for both humans and agents to read and write (plain Markdown).
- Must support decisions changing over time without losing history.

## Considered Options

- Markdown Architectural Decision Records (MADR) in the repository
- Michael Nygard's original lightweight ADR format in the repository
- Decisions documented in a wiki or in GitHub Discussions only
- No formal decision records

## Decision Outcome

Chosen option: **MADR (v4) in `docs/adr`**, because it is a well-known, actively maintained Markdown format
that captures options and trade-offs explicitly (not just the outcome), which is what reviewers and agents
need to understand a decision.

### Process

1. **Raise** — a decision that needs to be made is opened as a GitHub issue using the *ADR proposal* issue
   form, labelled `adr`.
2. **Discuss** — requirements, options and trade-offs are collected on the issue.
3. **Propose** — a pull request adds `docs/adr/NNNN-title-with-dashes.md` (copied from
   [`template.md`](template.md)) with status `proposed`, and references the issue.
4. **Decide** — merging the pull request accepts the decision; the status is set to `accepted` in the same PR.
   Rejected proposals may be merged with status `rejected` when the reasoning is worth keeping.
5. **Change** — accepted ADRs are immutable apart from typo fixes and status updates. A changed decision gets
   a new ADR; the old one is set to `superseded by ADR-NNNN`.

Numbers are sequential, four digits, and assigned when the pull request is opened.

### Consequences

- Good, because the reasoning behind the code is reviewable and searchable in the repository.
- Good, because agents can be instructed to read `docs/adr` before working and to raise an issue instead of
  silently making a new architectural decision.
- Bad, because writing ADRs takes time; we mitigate this by only recording decisions that are hard to reverse
  or affect several parts of the system.

## More Information

- MADR: <https://adr.github.io/madr/>
- Michael Nygard, *Documenting Architecture Decisions* (2011)
