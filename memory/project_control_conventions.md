---
name: wilds-control code conventions
description: Decided structure for wilds-control code — ABC bases + implementations, singletons, stage/ layout, Uvicorn + main() in app.py, one TOML config singleton, pydantic models everywhere
type: project
---

Decided by the user, 2026-11-05. Code lives under `wilds-control/src/wilds/control/`.

- **Base classes and implementations.** Each component that can be simulated, or has several versions, has an abstract base and concrete classes, e.g. `guide_camera/base.py` → `GuideCameraBase(BaseModel, ABC)` (settings, `video_frame` = latest frame, `single_frame()`, …), with `guide_camera/asi.py` → `ASICamera` and `guide_camera/simulated.py` → `SimulatedASICamera`.
  - The base holds the shared logic (settings validation, latest frame and fan-out, frame numbering); subclasses implement only the hardware hooks. Then the simulator exercises the shared code too.
  - A deployment setting in the TOML (e.g. `guide_camera.kind = "asi" | "simulated"`) picks the implementation.
  - `video_frame` is non-blocking (latest or `None`); `single_frame()` is awaited and always a fresh exposure; blocking SDK calls run in a thread.
- **Stages:** `stage/controller.py` (`StageControllerBase`, `MCS2Controller`: one TCP connection, a lock serialising SCPI, the channel map) and `stage/stage.py` (generic `Stage`: channel, `presets`, `close_enough`, limits, referenced state, native unit). Simulate the MCS2 as a SCPI server, so the real `MCS2Controller` code is tested. Slit, ADC and lamps *use* a `Stage` (composition), they don't subclass it.
- **Singletons** for controllers, the guide camera, etc. Created in the app's startup hook (not at import), closed at shutdown; each reached through an accessor (`get_guide_camera()`) used as a FastAPI `Depends`, so tests override it via `app.dependency_overrides`. Don't enforce the singleton in `__new__` (clashes with pydantic).
- **Server: Uvicorn**, one worker only (singletons); chosen over Hypercorn because HTTP/2 is not needed. `main()` in `control/app.py` parses argv (`--config`, optional `--host`/`--port`/`--log-level` overriding the TOML) and runs `uvicorn.run(app, ...)`. Script entry point `wilds-control = "wilds.control.app:main"`. Plain HTTP/1.1: keep long-lived streams (SSE, video) per tab few, browsers allow ~6 connections per origin.
- **One TOML config file**, owned by a `Config` singleton:
  - read with `tomllib`, saved with `tomlkit` (keeps comments and layout); saves are atomic (temp file + `os.replace`).
  - watches the file (`watchfiles`) and re-reads it on change, ignoring its own saves by content hash.
  - validates the whole file against the pydantic model before applying; an invalid file keeps the running config, logs the error and shows it in the GUI.
  - hands each component only its changed section; deployment-only settings are reported as "restart needed".
  - a GUI save returns `409` if the file changed on disk since it was loaded.
  - every save keeps a timestamped copy for history and revert.
- **Pydantic `BaseModel` for all relevant classes**: settings, status, API payloads, config, and the device classes themselves. Runtime state (SDK handles, sockets, locks, tasks, numpy frames) goes in `PrivateAttr`, not fields; status is a property returning a frozen model; no `validate_assignment` on hot paths. Pydantic v2's metaclass derives from `ABCMeta`, so `BaseModel` + `ABC` + `@abstractmethod` works.

See also the plan, `plans/control-and-gui.md` §3.10 and §7.
