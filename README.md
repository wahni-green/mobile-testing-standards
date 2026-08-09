# Mobile Testing Standards — Flutter + ERPNext

Reusable testing architecture and baseline invariants for Flutter apps backed by an ERPNext REST API. Meant to be pulled into a project as a git submodule and used as living context for both the team and an AI coding agent.

- **[TESTING_ARCHITECTURE.md](TESTING_ARCHITECTURE.md)** — the two-track testing approach (scripted regression suite + exploratory AI agent), the core verification mechanism, and a phased rollout plan.
- **[TESTING_BASELINE.md](TESTING_BASELINE.md)** — the cross-cutting invariants every project should scaffold before any feature-specific test work (Milestone 0): date/currency formatting, interaction idempotency, device & session integrity, input validation, and more.
- **[TESTING_LESSONS_LEARNED.md](TESTING_LESSONS_LEARNED.md)** — field notes from real rollouts: CI infrastructure gotchas for emulator/simulator E2E, resilient E2E test-writing patterns, backend cross-check pitfalls, and the debugging methodology that paid off when a cross-check went wrong.

## Using this in a project

Add it as a submodule at the path of your choice — `testing-standards/` is a reasonable default:

```
git submodule add https://github.com/wahni-green/mobile-testing-standards.git testing-standards
```

Point your project's `CLAUDE.md` (or equivalent) at it so an AI agent picks it up automatically, e.g.:

```
See testing-standards/TESTING_ARCHITECTURE.md for the testing approach and testing-standards/TESTING_BASELINE.md for baseline invariants to scaffold first.
```

### Tracking your project's rollout progress

Because the submodule is shared, read-only content — the same commit is referenced by every project that pulls it in — **don't check off the Rollout checkboxes in your local submodule copy.** Editing them there only changes what you see until the next `git submodule update`, doesn't persist anywhere meaningful, and risks someone accidentally committing a project-specific edit back into this shared repo.

Instead, track your own project's milestone progress in your own project (e.g. a short `TESTING_ROLLOUT.md` at your project root, or just your issue tracker), referencing the milestone IDs from `TESTING_ARCHITECTURE.md` (`0`, `1a`, `1b`, …).

## Updating a project's copy

```
cd testing-standards
git pull origin main
cd ..
git add testing-standards
git commit -m "chore: update testing-standards submodule"
```

## Contributing back

If a project discovers a baseline check worth adding for everyone (not just itself), open a PR against this repo directly rather than letting it drift as a local-only edit inside the submodule.
