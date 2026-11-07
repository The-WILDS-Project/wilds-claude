---
name: Dev Environment
description: Shared Python 3.14 venv at the WILDS workspace root; wilds is a PEP 420 namespace package; sdss-archon from PyPI
type: project
---
All WILDS repos share one venv at `/Users/blumzi/src/WILDS/.venv` (Python 3.14, installed via uv),
set up 2026-10-12. Each repo's `pyrightconfig.json` uses `venvPath: ".."`; the workspace file sets
`python.defaultInterpreterPath` to `${workspaceFolder}/../.venv/bin/python`.

**Development machine (user, 2026-11-05):** preliminary development is done on the user's Mac,
until the SuperMicro instrument computer is bought and Ubuntu installed on it. So wilds-control
must run on macOS as well as Linux:
- no Linux-only assumptions: no hard-coded `/dev`, `/etc` or systemd paths; the config file and
  data directories come from the TOML/argv; file watching via `watchfiles` (works on both);
- hardware reached over the network (MCS2 SCPI, Archon) works from the Mac as is;
- the ZWO ASI SDK has a macOS build; anything without one runs as its simulator on the Mac;
- systemd units, TLS certificate and udev rules wait for the Ubuntu machine.

**Why:** One environment with all the package repos (bridge, solving, control, gui, science-camera) installed
editable, so cross-repo imports resolve.

**How to apply:**
- The tree is set up by the **`workspace`** repo (GitHub: The-WILDS-Project/wilds-workspace, created 2026-10-30; local folders carry no `wilds-` prefix since 2026-11-07, so `WILDS/{workspace,control,gui,bridge,solving,science-camera,claude}`): `WILDS.code-workspace` (folder paths `../control` etc.) plus `setup.sh`, which clones missing repos beside it and builds/refreshes the venv with the install command below. Change editor settings or the install list there, not in a loose file in `WILDS/`.
- Install/reinstall with `uv pip install --no-sources --python .venv/bin/python -e ./bridge
  -e ./solving -e ./control -e ./gui -e ./science-camera sdss-archon pytest pytest-asyncio ruff`.
  `--no-sources` is needed because `science-camera` points `sdss-archon` at `../archon`,
  which isn't checked out; archon comes from PyPI instead.
- Secrets live in `WILDS/.vault/` (outside every repo, never committed). Dev passphrases: `.vault/passphrases.toml` (`to`, `engineering`), read by wilds-control via `WILDS_CONTROL_PASSPHRASES` in the VS Code launch entry; deployed machines use `/etc/wilds/passphrases.toml`.
- Don't use `uv run` in a repo: it builds a project `.venv` and rewrites `uv.lock`. Use the shared venv. After renaming folders, reinstall the editable packages or `import wilds` fails.
- `wilds` is a PEP 420 namespace package: no repo may add `src/wilds/__init__.py`, or it shadows
  the others (removed from bridge and solving on 2026-10-12).
