---
name: Work via pull requests
description: WILDS repo changes go on a feature branch and land through a GitHub PR, not direct commits to the default branch
metadata:
  type: feedback
---
Changes to the WILDS repos (github.com/the-wilds-project/*) are made on a feature branch and
submitted as a pull request, not committed directly to `main`/`master`.

**Why:** User asked for it (2026-10-19, "lets work with pull requests").

**How to apply:**
- Before committing, create a descriptively named branch off the default branch.
- Push and open the PR with `gh pr create`; give the PR link back to the user.
- One PR per coherent change; split unrelated work into separate PRs.
- Don't merge PRs unless the user says to.
- Leave untracked files the user added (e.g. reference PDFs in `docs/`) out unless asked.
