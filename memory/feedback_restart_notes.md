---
name: Say what needs restarting
description: After any change, tell the user which servers or sessions need restarting (and which don't)
type: feedback
---
After making changes, say explicitly what needs restarting: control, GUI, the browser page, or nothing.

**Why:** User request (2026-11-05). The dev servers run from VS Code launch entries and not all pick up changes: control runs with `--reload` (Python changes apply, but `launch.json` env vars such as the dev passphrases only on a fresh start); the GUI debug session did run `--noreload`, so it served stale templates after a merge. Browser pages may also need a hard refresh.

**How to apply:** End every reply that changes code, templates, static files, `launch.json` or config with a short line per affected server (wilds-control :8001, wilds-gui :8000, browser) saying restart / hard refresh / nothing needed, and why. Check the checkout's current branch, since a server only runs what is checked out.
