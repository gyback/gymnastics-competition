<!--
Title: use Conventional Commits, e.g. `feat(scheduling): generate rotation order`.
The title becomes the squash commit on `main`. See CONTRIBUTING.md.
-->

## Summary

<!-- What does this change and why? Keep it short. Note anything a reviewer should look at first. -->

## Linked issue

<!-- Every pull request references an open issue. Use `Closes #N`, or `Refs #N` for partial work. -->

Closes #

## Architecture decisions

- [ ] I have read the ADRs in [`docs/adr/`](https://github.com/gyback/teamgym-arena/tree/main/docs/adr) relevant to this change, and it follows them.
- [ ] This change makes no new architectural decision, **or** it adds or links an ADR: <!-- ADR-NNNN / #N -->

## Testing

- [ ] Unit tests cover new or changed domain rules and handlers.
- [ ] Integration tests cover new or changed endpoints, persistence and authorisation.
- [ ] Architecture tests pass.
- [ ] E2E tests are added or updated if a critical user journey changed.
- [ ] Not applicable (documentation or configuration only).

<!-- How did you verify the change? List manual checks reviewers can repeat. -->

## Checklist

- [ ] The PR title follows [Conventional Commits](https://www.conventionalcommits.org/).
- [ ] The change is focused on the linked issue, with no unrelated changes.
- [ ] Build, formatting, linting and tests pass locally.
- [ ] Documentation is updated (`README.md`, `AGENTS.md`, `docs/deployment.md`, etc.) where needed.
- [ ] No secrets or real personal data are committed. Test data is synthetic.

## AI assistance

<!-- See "Working with AI coding agents" in CONTRIBUTING.md. -->

- [ ] No AI assistance was used.
- [ ] AI tools assisted with this change: <!-- tool and model, e.g. Claude Code (Claude Opus) --> for
      <!-- what, e.g. "the whole change", "tests only", "review" -->
- [ ] I have reviewed and understood every line of this change and can explain it in review.
