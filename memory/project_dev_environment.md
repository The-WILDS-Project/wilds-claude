---
name: Dev Environment
description: Shared Python 3.14 venv at the WILDS workspace root; wilds is a PEP 420 namespace package; sdss-archon from PyPI
type: project
---
All WILDS repos share one venv at `/Users/blumzi/src/WILDS/.venv` (Python 3.14, installed via uv),
set up 2026-10-12. Each repo's `pyrightconfig.json` uses `venvPath: ".."`; the workspace file sets
`python.defaultInterpreterPath` to `${workspaceFolder}/../.venv/bin/python`.

**Why:** One environment with all four repos (bridge, solving, control, science-camera) installed
editable, so cross-repo imports resolve.

**How to apply:**
- The tree is set up by the **`wilds-workspace`** repo (github.com/The-WILDS-Project/wilds-workspace, created 2026-10-30): `WILDS.code-workspace` (folder paths `../wilds-*`) plus `setup.sh`, which clones missing repos beside it and builds/refreshes the venv with the install command below. Change editor settings or the install list there, not in a loose file in `WILDS/`.
- Install/reinstall with `uv pip install --no-sources --python .venv/bin/python -e ./wilds-bridge
  -e ./wilds-solving -e ./wilds-control -e ./wilds-science-camera sdss-archon pytest pytest-asyncio ruff`.
  `--no-sources` is needed because `wilds-science-camera` points `sdss-archon` at `../archon`,
  which isn't checked out; archon comes from PyPI instead.
- `wilds` is a PEP 420 namespace package: no repo may add `src/wilds/__init__.py`, or it shadows
  the others (removed from bridge and solving on 2026-10-12).
