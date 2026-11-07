---
name: Memory Location
description: Claude's memory, plans and notes for WILDS live in the wilds-claude repo, not in ~/.claude
type: feedback
---
Claude's memory files, plans and working notes for the WILDS project belong in the
`wilds-claude` repo (`/Users/blumzi/src/WILDS/claude`), not in Claude Code's
per-project memory directory under `~/.claude/projects/`.

**Why:** User stated it explicitly (2026-10-13). The repo is version-controlled and shared
across machines and sessions, unlike `~/.claude`.

**How to apply:**
- Write new memories to `claude/memory/` and index them in `claude/memory/MEMORY.md`.
- Put plans and longer notes in `claude/` (e.g. alongside `project-notes.md`).
- Read `claude/memory/MEMORY.md` at the start of WILDS work.
- `~/.claude/projects/*/memory/` should hold only a pointer to this repo.
