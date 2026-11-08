# Plan: wilds-control API and wilds-gui panels

*Status: draft for discussion, 2026-10-31; updated 2026-11-04 with Tim's answers on the
rotator and on users. Nothing here is implemented yet.*

## 1. Principles

- **wilds-control owns all hardware and every procedure.** Acquisition, autofocus, slit
  rotation and exposures run in the backend as *jobs*. The GUI starts them, shows their
  progress and can cancel them. A procedure survives a closed browser tab, and the same
  API serves the GUI, scripts and later automation.
- **WILDS drives its own workflow** and talks to LDT only through the ActiveMQ broker
  (facility broker `joe`; test broker `tanagra`). This answers Tim's open question: WILDS
  integrates target submission into its own software.
- **One controller, many viewers, no user accounts.** Anyone who can reach the GUI (VPN
  only) may watch. One browser session at a time holds the *control lease*; the Telescope
  Operator grants it (§3.11).
- **Every device has a simulator** (TCS on nysa, guide-camera image simulator, MCS2 SCPI
  server, Archon stub), so the GUI can be built before hardware arrives.
- **Measured constants are configuration, never code**: plate scale, slit pixel, IAA,
  pointing origin, LDT↔WILDS focus offset, ADC zero points.
- **Parfocal cameras**: the guide camera and the science cameras are parfocal by design, so
  focusing on the guide camera focuses the science cameras.

## 2. API conventions (wilds-control, FastAPI on Uvicorn, :8001)

| Kind | Form |
|---|---|
| Snapshot | `GET /status`: one model with telescope, instrument, cameras, jobs, activities (§3.13) and the control lease |
| Live updates | `GET /status/stream` (SSE): `status` every 1 s, plus `job` and `log` events when they happen |
| Short command | `POST`/`PUT` returns `200` with the resulting state |
| Long procedure | `POST` returns `202 Accepted` with `{job_id}`; progress arrives as `job` events |
| Jobs | `GET /jobs`, `GET /jobs/{id}`, `POST /jobs/{id}/cancel`. States: queued → running → done, failed or cancelled |
| Control | `POST /sessions`, `POST /control/requests` (request control), `DELETE /control` (release), `POST /control/engineering`; TO actions in §3.11. Every command and configuration change needs the lease token, or gets `403`. |
| Configuration | `GET /config`, `GET`/`PUT /config/{component}`, `GET /config/{component}/schema`, `GET /config/{component}/history` (see §3.10) |
| Video | `WS /guide-camera/stream`: binary JPEG frames, each preceded by a JSON metadata frame |

The Python client is generated from FastAPI's OpenAPI schema. It replaces the hand-written
`client/api.py`, whose `/acquire/cancel` and `/solve` paths have no router behind them.
The skeleton's `/telemetry` router becomes `/status`.

### GUI stack and look (wilds-gui)

The look and the component kit are **copied from the MAST GUI** (`The-MAST-project/MAST_gui`,
mostly the same author), noting where each piece came from, and adapted. There is no shared
package for now; one could follow if both GUIs keep changing the same parts.

**Stack:**
- Django;
- Bootstrap 5.3 and Bootstrap Icons;
- HTMX for page fragments and forms;
- Alpine.js for live components.

Every library is pinned to an exact version and served from Django's static files, never a
CDN, so the GUI still loads over a slow VPN or from LDT without internet.

**Taken from MAST:**
- **Colour tokens:** the subdued info/success/warning/danger/primary/secondary palette, as
  CSS variables, renamed `--mast-*` → `--ui-*`, and the severity classes for borders and text.
- **Status dots** (ok / warning / error, with warning and error pulsing) and the pulsing
  **activity heart**, whose tooltip lists what's running. Used for the TCS access badge, the
  wilds-control link and running jobs.
- **Toasts** for notices, the **confirm modal** (e.g. "Slew now" with Preview off), the
  `x-tooltip` Alpine directive (for why a control is greyed out), and the link-status
  dropdown.
- **Layout:**
  - an 80 px header with the Weizmann and WILDS logos;
  - a collapsible sidebar and the main area for the multi-page `/io` (Operations, Products) and
    `/eng` (Monitoring, Calibration, Configuration);
  - a single page with no sidebar for `/to`.
- **Form and modal conventions:** horizontal `col-4` label, `col-8` field; Cancel on the
  left, then the primary action.

**Changed for WILDS:**
- **Dark theme.** WILDS is used at night in a dark control room.
  - Every colour is a variable with light and dark values.
  - Bootstrap's `data-bs-theme` switches between them; dark is the default, and the choice
    is remembered per browser.
  - A dim red night mode may follow.
- **No accounts:** sign-in, sign-up, the user menu, allauth and the site selector are
  dropped. JS9 loads on the product-browsing pages, not per user.
- **Live data** comes from wilds-control's SSE and WebSocket streams into Alpine
  components, not from HTMX polling.
- **Safe text:** session names are typed by anonymous visitors and shown on `/to`, so toasts,
  tooltips and the confirm modal insert text with `textContent`, never as raw HTML. MAST's
  versions insert HTML.
- **The confirm dialog** is its own function returning a promise (`uiConfirm()`), rather
  than replacing `window.confirm`, with Cancel left and OK right per the convention above.
- **Clean-ups:** no global `* { margin: 0; padding: 0 }` reset (it works against
  Bootstrap's spacing), fewer `!important` rules, and none of the stray editor `filepath`
  comments.

## 3. Features

Message directions: **→ LDT** = WILDS publishes; **← LDT** = WILDS subscribes. Topic
names are given in full once; `TCS…` abbreviates `TCS.TCSSharedVariables`.

### 3.1 Status monitoring

Panels: TCS state and health, pointing, limits, weather, WILDS mechanisms, cameras, jobs.

**LDT messages**

| Dir | Topic | Content | Notes |
|---|---|---|---|
| ← | `TCS.TCSSharedVariables.TCSHighLevelStatusSV.TCSTcsStatusSV` | `tcsTCSStatus` XML, v1.9, ~1 Hz | Main source. Verified live on the simulator. |
| ← | `TCS…TCSLowLevelStatusSV.*` | plain scalars (`CurrentParAngleSV`, `InPositionIsTrueSV`, …) | Optional, for faster single values |
| ← | `tcs.loisTelemetry` | `TCSTelemetry` XML, ~2 Hz | JOE only: real telescope, not the simulator. `TelescopeStatus` must fall back to `TCSTcsStatusSV` when it's absent. |
| ← | `AOS.AOSPubDataSV.AOSDataPacket` | focus readback (`totalFocusOffset`, m), M1/M2 settled flags | AOS simulator offline (ask Georgi) |
| ← | `WRS.WRSPubDataSV.WRSDataPacket` | weather, metric | Real LDT weather even in simulation |
| ← | `instrumentCube.loisTelemetry` | instrument cover, fold mirrors FMA–FMD | Confirms the light path is set to WILDS's port |
| → | `wilds.loisTelemetry` | `WildsTelemetry` XML, ~1 Hz | WILDS state for LDT (slit, lamps, ADC, shutter, cameras). Schema to be agreed with LIG. |

**Endpoints:** `GET /status`, `GET /status/stream`.

### 3.2 Target selection and submission to the TCS

Panel: target entry (name, RA/Dec, frame, equinox, proper motion, slit sky PA), a target
list, submit, and a "slewing / in position" indicator.

**LDT messages**

| Dir | Topic | Content | Notes |
|---|---|---|---|
| → | `TCS.TCSSharedVariables.TCSSubData.TCSTcsCommandSV` | a full `ScienceTargetConfiguration`: target block, `rotator` (`rotPA`, `rotatorFrame=Target`), `wl_microns`, `fracrate`, `pointingOriginX/Y`, `trackID` | **Topic confirmed by Tim (2026-11-02).** Our 2026-10-31 attempt went to `TCSLowLevelStatusSV.NewScienceTargetSV`, which is TCS → broker only, so it never reached the TCS; `Telescope.slew_to` in wilds-bridge must change topic. First try on nysa with Georgi: the April test on this topic faulted the MCB. |
| ← | `TCS…TCSLowLevelStatusSV.NewScienceTargetSV` | the TCS's current `newScienceTarget` | The starting point for re-sending the target (§3.5, §3.3) |
| ← | `TCS…TCSLowLevelStatusSV.TCSTcsCommandResponseSV` | command result, `tcsErrorResponse` | Accept or reject |
| ← | `TCSTcsStatusSV` | `pointingPositions.demandRADec`, `targetName`, `inPositionIsTrue`, `externalTargetCfgCmdPreviewIsTrue` | Confirms arrival. With preview on, the operator must approve. |

Open questions: must the outer `demandRADec` values be integers? Does `pointingOriginX/Y`
in the command override the instrument's pointing origin? WILDS should send its effective
wavelength, not the 0.5 µm default.

**Endpoints:** `POST /telescope/target` → job (sent → accepted → in position);
`GET /status` shows the current and demanded target.

### 3.3 Target acquisition

Procedure: after the slew, take guide-camera frames, plate-solve, compute the offset that
puts the target on the slit, send it, then repeat until within tolerance.

**LDT messages**

| Dir | Topic | Content | Notes |
|---|---|---|---|
| → | `TCS.TCSSharedVariables.TCSSubData.TCSTcsCommandSV` | `scienceTargetOffset` (`commandID` = µs since midnight UT; `offsetDef`: `num1=User`, `off1`/`off2` arcsec, `offsetType=TPLANE`/`Simple`); also `scienceTargetAbsorbOffset`, `scienceTargetClearOffset` | **Risk:** in April, broker → TCS on this topic faulted the MCB and stopped all bridging. Must be resolved with LIG first. |
| ← | `TCSTcsCommandResponseSV` | result | |
| ← | `TCSTcsStatusSV` | `OffsetStatus.userOff1/2`, `inPositionIsTrue`, `pointingPositions.objectPA.EqPA`, `currentParAngle` | Offset applied and settled. Slit sky PA for the pixel → sky transform. |

Local: guide camera, `wilds-solving` (Series 5200 indices), configuration (plate scale,
orientation, parity, slit pixel). Fallback when the ~2.5′ field has too few stars to solve:
centroid the target and use the known slit pixel.

**Endpoints:** `POST /acquisition` `{tolerance_arcsec, exposure_s, max_iterations}` → job,
with per-iteration events (frame, solution, offset sent, residual);
`POST /telescope/offset` `{d1_arcsec, d2_arcsec, type}`;
`POST /telescope/offset/absorb`; `POST /telescope/offset/clear`;
`POST /guide-camera/solve` (one-off solve of the latest frame).

### 3.4 Autofocus

Procedure: step the telescope focus through a range centred on *LDT focus + WILDS offset*,
measure the star size (HFD/FWHM) on guide-camera frames, fit a curve (V or hyperbola), move
to the minimum and log the result.

**LDT messages**

| Dir | Topic | Content | Notes |
|---|---|---|---|
| → | `AOS.AOSSubDataSV.AbsoluteFocusOffset` / `.RelativeFocusOffset` | plain-text number, **metres** | Applied on top of the AOS model (temperature, elevation) |
| → | `AOS.AOSSubDataSV.ClearFocusOffset` | `true` | |
| ← | `AOS.AOSPubDataSV.AOSDataPacket` | `totalFocusOffset` (m), `focusOffsetDemandOutOfRange`, `M2PSettled` | Wait for settled before each frame. AOS simulator offline. |
| ← | `TCSTcsStatusSV`, `WRSDataPacket` | elevation, air temperature | Logged with each result |

**Endpoints:** `POST /autofocus` `{range_um, steps, exposure_s}` → job, with per-step
events (focus, HFD) and the final fit; `POST /focus` `{offset_m, relative}` (manual);
`DELETE /focus` (clear offset).

### 3.5 Slit rotation

There is **no standalone rotator command** (JIRA TCSD-131; confirmed by Tim, 2026-11-02:
"Our TCS does not, at present, expose a direct rotator-change API"). The slit's sky PA is
set by re-sending the whole target. LDT's own software does it as read, modify, write:

1. Take the latest `NewScienceTarget` from TCS telemetry.
2. Build a `ScienceTargetConfiguration` from it, replacing only the rotator (`rotPA`,
   `rotatorFrame`; `Target` holds the field still on the slit).
3. Send it to `TCSTcsCommandSV`.

Copying everything else preserves any offsets and differential tracking rates.

**LDT messages**

| Dir | Topic | Content | Notes |
|---|---|---|---|
| ← | `TCS…TCSLowLevelStatusSV.NewScienceTargetSV` | the current target | Step 1. If it's stale or missing, refuse rather than guess. |
| → | `TCS.TCSSharedVariables.TCSSubData.TCSTcsCommandSV` | the same target with a new `rotator` | As in §3.2, including the MCB caution |
| ← | `TCSTcsStatusSV` | `objectPA.EqPA` (slit sky PA), `currentRotatorPositions.rotPA` (mechanical), `rotIPA` (demanded), `iaa`, `inPositionRotIsTrue`, `limits.timeToRotLimit_min`, `rotCurrentWrap` | Semantics verified live, 2026-10-31 |

Common choices: parallactic angle (from `currentParAngle`) or a user PA.

**Endpoints:** `POST /slit/rotation` `{sky_pa_deg | "parallactic"}` → job.

### 3.6 Slit width, calibration lamps, ADC

These are WILDS mechanisms with no LDT commands. All five stages are on one 6-channel
SmarAct MCS2 (Ethernet; part number to confirm; replaces the 3-channel MCS2-C-0006), over
SCPI on TCP 55551, sharing one connection:

| Channel | Stage | Kind | Native unit |
|---|---|---|---|
| 0 | slit width | linear (CLS-3232) | pm |
| 1, 2 | ADC 1, ADC 2 | rotary (SR-120) | n° (nano-degrees) |
| 3, 4 | lamp 1, lamp 2 In/Out | to choose | per stage |
| 5 | spare | | |

One controller means one failure point: a controller fault or a pulled cable stops all five.

**Common stage settings.** Every stage has:

- `presets: dict[str, int]`: name → position in the stage's native unit, so values are
  exact and match what SCPI reads back. The API and GUI accept and show physical units
  (mm and arcsec for the slit, degrees for the ADC) and convert at the edge. Names are
  URL-safe slugs. Every preset must lie within the travel limits.
- `close_enough: int`: native-unit tolerance. Stages may stop a few ticks short of or past
  the target while reporting it reached; a move counts as done, and the stage as *at* a
  preset, when it is within `close_enough`. Status reports `at_preset` (a name or `null`).

Moves to a preset are refused until the stage is referenced. Lamp stages must define the
presets `in` and `out`. On an ADC, a preset move (e.g. `zero`, `park`) is `409` while
auto-tracking is on.

**LDT messages**

| Dir | Topic | Content | Notes |
|---|---|---|---|
| ← | `TCSTcsStatusSV` | `currentParAngle`, elevation | ADC auto-tracking |
| ← | `WRSDataPacket` | temperature, pressure, humidity | Atmospheric dispersion model |
| → | `wilds.loisTelemetry` | slit width (mm and arcsec at 8.02″/mm), lamp states and positions, ADC angles | |

**Endpoints:** `PUT /slit/width` `{width_mm | width_arcsec}`; `POST /slit/width/jog`
`{delta_mm}`; `PUT /lamps/{n}` `{on, position: in|out}` (a move to that preset);
`PUT /adc/{n}` `{angle_deg}`; `PUT /adc/tracking` `{enabled}`; for any stage,
`POST /stages/{stage}/preset/{name}` → job.

### 3.7 Science exposures (two cameras)

Panel: exposure type, separate VIS and UV exposure times, count, object name, lamp
selection for arcs, progress, and the list of recent files.

**LDT messages**

| Dir | Topic | Content | Notes |
|---|---|---|---|
| → | `lorax.ldt.locutus.inforequest` | `agentCommand` (`INFO`) | At startup: readiness and dictionary names |
| ← | `lorax.ldt.locutus.infobroadcast` | `READY`, `available_dictionaries` | |
| → | `lorax.ldt.locutus.dictrequest` | `locutusDictRequest` (`requester`, a new `request_id`, `dictionary_name`) | Just before each header is written: `ldt_facility`, plus a WILDS dictionary LIG would create |
| ← | `lorax.ldt.locutus.dictresponse` | `locutusDict`: per keyword `val`/`comment`/`valid` | Filter on `recipient` and `request_id`. Keep `KWDICT` in the header. |
| → | `wilds.loisTelemetry` | exposure state, last file per camera | |

Locutus runs on the facility broker `joe`; it isn't on tanagra. Files follow
`/lois/data/dct/<ID>/YYYYMMDD/<abbr>.NNNN.fits` with the noon rollover rule. The instrument
ID and abbreviation are to be agreed with LIG.

**Endpoints:** `POST /exposures` `{type: bias|dark|flat|arc|object, vis_s, uv_s, count,
object}` → job; `POST /jobs/{id}/cancel`; `GET /exposures` (recent files).

### 3.8 Guide camera live view

Shared by acquisition, autofocus, slit rotation and commissioning.

**LDT messages:** none directly. Overlays use `TCSTcsStatusSV` (target name, slit sky PA,
parallactic angle).

**Endpoints:** `PUT /guide-camera/settings` `{exposure_s, gain, roi, binning}`;
`POST /guide-camera/start`; `POST /guide-camera/stop`; `WS /guide-camera/stream`;
`GET /guide-camera/frames/{id}` (raw FITS). Overlays (slit, target marker, centroids,
HFD) are drawn in the browser from the metadata frames, never burned into the JPEG.

### 3.9 Commissioning support

Each procedure is a job that measures something and proposes a configuration value. The
operator reviews and saves it.

| Procedure | Method | LDT messages |
|---|---|---|
| Guide-camera plate scale, orientation, parity | known offsets, then solve | → `TCSTcsCommandSV` offsets; ← `TCSTcsStatusSV` |
| Slit pixel on the guide camera | lamp-illuminated slit image | none |
| IAA (slit orientation vs rotator) | rotate in steps, track a star across the slit | → target with stepped `rotPA`; ← `TCSTcsStatusSV` rotator fields. Procedure on Confluence: "IAA and center of rotation measurement". |
| Pointing origin (slit centre) | centre of rotation vs slit | ← `TCSTcsStatusSV`; the result goes to LIG for WILDS's instrument entry |
| LDT ↔ WILDS focus offset | autofocus after LDT's own focus | → AOS focus; ← `AOSDataPacket` |
| Parfocality check | focus on the guide camera, inspect science frames | AOS focus |
| ADC zero points | lamps or star | none |

**Endpoints:** `POST /commissioning/{procedure}` → job. Accepted results are saved to the
`calibration` component through `PUT /config/calibration` (§3.10).

### 3.10 Configuration panels

One Django panel per component, for viewing and changing the settings that wilds-control
applies to hardware and procedures.

**Who owns the configuration:** wilds-control. Each component's settings are a Pydantic
model in the backend, which validates them and applies them to the device. Two stores, by who
writes:

- **Deployment settings** are hand-edited in `/etc/wilds/config.toml` (root-owned, read at
  start; passphrases in their own file, §3.11). wilds-control never writes it.
- **Operational and calibration settings** are written by the application, from the GUI or
  from a procedure, so they live in **one SQLite file** (`/var/lib/wilds/settings.db`, the
  `[store]` section of config.toml; `wilds/control/settings_store.py`). Not MongoDB: MAST uses
  it for a fleet of units sharing one configuration; one machine needs no server to run,
  secure, back up and keep running before the instrument can start. SQLite is in the standard
  library, one file, transactional (a power cut leaves the last whole version).
  - A *section* is one component's settings model, stored as JSON and versioned as a whole.
    `put` records who (name and code), when and why; a save that changes nothing records
    nothing; every version is kept, and `history` lists them with the keys each changed.
  - `revert` restores an older version *as a new version*, so the history only grows.
  - `expected_version` is optimistic concurrency: a form loaded at version 4 that finds 5 is
    refused with `409`, never silently overwritten.
  - Keys added to a model later take their defaults; a stored section that no longer
    validates is reported, not guessed at; a database from a newer wilds-control is refused.
  - If a second machine ever needs the same calibration (the MAST fleet, say), a network
    database becomes worth its cost; access goes through this one module.

The Django panels render their forms from `GET /config/{component}/schema`, the model's
JSON Schema, so fields, units, limits and help text are defined once, in the backend. Django's
own database holds only the observing log; there are no user accounts (§3.11).

**Three kinds of setting**, shown separately in each panel:

| Kind | Examples | Who may change | When it applies |
|---|---|---|---|
| Operational | guide-camera exposure, gain; stage presets | the control holder | immediately, or rejected with `409` if the device is busy; kept in the settings store |
| Calibration | plate scale, slit pixel, IAA, focus offset, ADC zero points | the control holder, in engineering mode | on save; every change recorded in the settings store with the holder's name, the time and the reason, and revertible |
| Deployment | broker host (`joe`/`tanagra`), device addresses, data paths, simulator on/off | `/etc/wilds/config.toml` only, not editable in the GUI | on restart |

**Components and their settings**

| Component | Settings |
|---|---|
| Guide camera (ASI294MM) | exposure, gain, offset, binning (1×1/2×2), ROI, USB bandwidth, cooler setpoint (Pro model), frame rate limit; display stretch (zscale/asinh, black/white points) |
| Science cameras (Archon VIS, UV), each | default exposure time; window (lines, pixels, pre/post-skip, overscan); binning (h, v); CCD temperature setpoint; autoflush and flush count; readout speed, amplifiers and gain, depending on what the STA ACF exposes; ACF file in use (read-only) |
| Every MCS2 stage | `presets` (native units, shown in physical units), `close_enough`, speed, acceleration, travel limits, reference/calibrate actions |
| Slit stage (MCS2 ch 0) | presets shown as width in mm and arcsec at 8.02″/mm |
| ADC (MCS2 ch 1, 2), each | presets shown in degrees; zero point, travel limits, speed; tracking model parameters, update rate and dead-band (minimum change before moving) |
| Calibration lamps (×2) | name/type, warm-up time, maximum on-time; In/Out stage (MCS2 ch 3, 4) with required presets `in` and `out` |
| Autofocus | scan range, steps, exposure, star-size metric (HFD/FWHM), fit model, settle wait; LDT ↔ WILDS focus offset (calibration) |
| Acquisition | tolerance (arcsec), maximum iterations, exposure, solver scale hints, centroid fallback on/off, offset type (TPLANE/Simple) |
| Telescope interface | target defaults (rotator frame `Target`, effective wavelength), Locutus dictionary names; broker host (deployment) |
| Data storage | instrument ID and abbreviation, file naming, rollover rule (noon UT−5 h), data directories (deployment) |

**LDT messages:** none. Configuration is local to WILDS. Some values change what is sent
(target wavelength, offset type, Locutus dictionary names).

**Endpoints:** `GET /config` (all components); `GET /config/{component}`;
`PUT /config/{component}` (validated; `409` if not applicable now);
`GET /config/{component}/schema`; `GET /config/{component}/history` and
`POST /config/{component}/revert/{version}`; for presets,
`GET`/`POST`/`DELETE /config/{stage}/presets[/{name}]` and
`POST /stages/{stage}/preset/{name}` (move to a preset).

### 3.11 Control and access

**LDT's model** (Tim, 2026-11-02): the TCS doesn't authenticate commands; anything on the
local network can send them. Observing machines use a shared, role-based observer login,
and remote observers VNC in and agree with the Telescope Operator (TO) over Zoom before
commanding the telescope. The TO can lock out external commands at the TCS. EXPRES, with a
web GUI hosted at LDT, is the suggested model for WILDS; how observers and spectators get
access is up to us. Operations rely on a "good actor" model.

**WILDS keeps no user accounts.**

**Two GUIs**, served by Django under one host, so there's one origin and no CORS:

- **`/to`, the Telescope Operator's page.** It shows instrument status and current
  activities (information only) and the control panel:
  - the current holder;
  - the sessions requesting control, with name, short code and time of request;
  - a count of read-only viewers, expandable into the full list.

  The TO grants control to exactly one session, or revokes it so nobody holds it. The page
  shows "**engineering mode: <name> has control**" prominently and can revoke that too.
- **`/io`, the Instrument Operator's GUI.** Multi-page, with tabbed panels for
  calibration, operations, and browsing products and intermediates. Every session shows its
  short code and its state: viewer, requesting, or in control. It has a *Request control*
  button, which can be pressed again to re-apply.

For HTTP/2 over TLS the host needs a fully qualified name (e.g. `wilds.lowell.edu`); a bare
`wilds` can't be given a certificate.

**Sessions:**

- **One per browser.** Tabs in the same browser share it, and its lease.
- **Any number of read-only sessions, one holder.** Each session can come from anywhere:
  - a browser on the wilds computer, locally or over VNC;
  - a computer at Weizmann or elsewhere, over the VPN.

  Where a session comes from gives it no rights, so a local or VNC browser is just another
  session.
- **Identity:** an anonymous Django session (cookie) plus a name the person types (e.g.
  "Arie (Rehovot)"), kept in the browser. wilds-control gives each session a UUID
  internally and a short code for people (`K7-42`). The name is a label for screens and the
  log, not authentication.
- **Dead sessions** drop off after a missed-heartbeat timeout. If a dead session held
  control, the lease lapses and the TO page says so, so a closed laptop can't lock everyone
  out.
- **Read-only** means seeing everything (status, video, product and intermediate browsers)
  while anything that changes state gets `403`.
- **Viewer cap:** a configurable maximum number of sessions (e.g. 20), with a clear message
  when it's reached, to protect the wilds computer.

**The control lease:** one session at a time holds it. wilds-control issues and checks the
lease token on every command and configuration change, so scripts or a second GUI can't
bypass it.

**Granting control (regular mode):**

1. The instrument operator clicks *Request control*. The request appears on `/to`;
   clicking again re-applies and moves it to the top.
2. On Zoom they tell the TO their name and short code.
3. The TO grants the matching session.

The name proves nothing; the code ties the grant to the browser of the person on the call.
Sessions appear in the TO's request list only when they ask. If someone already holds
control, the grant moves it: the previous holder is told who has it now and drops to
read-only immediately.

**Engineering mode (superuser):** a separate page, `/eng` (on :8000, with the aliases
`/engineering`), titled "WILDS - Engineering". The engineering passphrase unlocks it and takes
control without the TO's approval, e.g. for daytime commissioning. It is checked by
wilds-control; the page stays unlocked while the session holds the lease, so a reload does not
ask again. Engineering mode covers WILDS mechanisms only: stages, lamps, cameras and
calibration settings (`require_engineering`: lease token *and* engineering mode). Commands to
the TCS (targets, offsets, rotator, focus) still need a TO grant, since only the TO sees what
the telescope is doing. Taking control this way is **not coordinated by the software**: it
takes the lease at once and the previous holder drops to read-only (running jobs continue);
people coordinate by Zoom or in person. The TO can still revoke. The page has monitoring
(components, link, holder) and the calibration access points (lamps, slit, ADC, focus, bias,
dark, flat), greyed out until wilds-control implements them. Page titles: "WILDS - Instrument
Operator" for `/io`, "WILDS - Telescope Operator" for `/to`.

**TO presence:** the TO counts as online while a TO page is open and unlocked (it polls
wilds-control with the passphrase). wilds-control reports `to.online` in the status stream,
offline after `presence.to_timeout_s` (75 s, since browsers throttle background tabs) without
contact, and logs the transitions. The IO pages show "TO: online/offline" and, when offline,
that control requests wait for the TO. This replaces the IO page polling the TO page every
15 s: sessions already heartbeat to wilds-control, which tells everyone.

**The TO page** is protected by its own shared passphrase, also in the configuration file
(`/etc/wilds/passphrases.toml`, `[passphrases]`, a file of its own so it can have its own
owner and mode; in development `WILDS/.vault/passphrases.toml`; engineering defaults to
`physics`, which is accepted as it is (no warning), TO has no default and stays closed until
set). Both passphrases are configuration, not user management.

**Rules:**

- **Automatic refusal or deferral:** while an exposure or acquisition runs, a new request
  shows why ("exposure in progress, 4 min left") and can be queued until the job ends. The
  TO can still override.
- **Running jobs at handover** continue under the new holder rather than being cancelled; the
  new holder may cancel them.
- **Audit:** every request, grant, refusal, handover, revoke, engineering-mode entry and
  lapse is logged with the typed name, short code, time and reason.

**Many viewers, little load:**

- **Video:** each guide-camera frame is encoded to JPEG once and the same frame goes to
  every viewer. Each WebSocket gets only the newest frame and stale ones are dropped, so a
  slow VPN viewer gets a lower frame rate, not a growing delay, and never slows the holder.
- **Viewer settings:** a viewer can choose a lower frame rate or JPEG quality. This is per
  session and never changes the camera.
- **Status:** one status model is built each second and sent to all sessions.

**Shown on every page:** "WILDS control: held by X" (or "engineering mode: X") and the
TCS access badge (§3.12). The first time someone takes
control each night, the GUI reminds them to confirm with the TO before commanding the
telescope.

**LDT messages:** none for the lease itself; the TCS access badge is in §3.12.

**Endpoints:**

- Sessions:
  - `POST /sessions` `{name}` → `{session_id, code}`
  - `POST /sessions/{id}/heartbeat`
- Requests and release:
  - `POST /control/requests` (request or re-apply)
  - `DELETE /control` (the holder releases)
  - `POST /control/engineering` `{passphrase}` (take control in engineering mode)
- TO only (needs the TO passphrase):
  - `GET /control/sessions` (holder, requests and viewers)
  - `POST /control/requests/{id}/grant` and `/refuse` `{reason}`
  - `POST /control/revoke`

### 3.12 Telescope commands: access, preview and confirmation

This applies to **every operation that changes telescope state**:
- targets (`slew_to`) and slit rotation (`set_rotator`);
- offsets (apply, clear, absorb);
- focus (AOS);
- the steps of acquisition and autofocus, and anything added later.

They all go through one path in wilds-control rather than each handling it alone.

**TCS access badge**, on every page and for every session, with or without control:

| Part | Source |
|---|---|
| TCS state and health | `TCSTcsStatusSV` |
| TCS user mode, Operator or Engineer (the TCS GUI's own login, §4.12 of the TCS manual; not a lockout) | `accessMode` |
| External Command Preview on/off | `externalTargetCfgCmdPreviewIsTrue` |
| External commands locked out by the TO | field to be identified (§5) |
| Telemetry freshness | age of the last `TCSTcsStatusSV`; stale means the badge is grey or "unknown" |

**External Command Preview** (TCS manual §5.4.2.1.7) is the TCS's own switch.
- **On** (the default at TCS start-up): an external sidereal science target only fills the
  TCS's preview panel; the telescope doesn't move until the TO accepts it.
- **Off**: the target becomes current at once, and a tracking telescope slews immediately.

Per the manual it covers science targets only, so a re-sent target (`set_rotator`) is
previewed but offsets and focus presumably are not. Whether that holds is to be confirmed (§5).

**Availability:** the status model carries a map with one entry per action, e.g.
`{"slew": {"ok": false, "why": "TCS locked out external commands"}}`. GUIs grey out the
control and show `why` as a tooltip. wilds-control enforces the same rules, with `403`
(no lease) or `409` (state) and the same reason, so greying is a convenience, never the
guard. A telescope action is unavailable when any of these apply:
- there is no TO-granted lease (engineering mode doesn't count);
- the TO has locked out external commands;
- the TCS isn't in a state to accept the command;
- telemetry is stale.

**Effect label:** each control says what will happen.
- **"Send to TO preview"** when the operation is previewed and Preview is on.
- **"Applies immediately"** otherwise. A slew with Preview off asks for confirmation.
- If Preview is unknown, assume the TO must approve.

**Progress is confirmed from telemetry, never from sending.** Each operation is a job:

`sent → waiting for TO (previewed only) → in progress → done`, or `not accepted` after a
timeout. Each kind of operation has its own confirmation:

| Operation | Confirmed by |
|---|---|
| target, rotator | `NewScienceTargetSV` and the status `targetName` change to ours, then `inPositionIsTrue`, and the rotator PA |
| offsets | `offsetStatus` |
| focus | AOS focus status (topic to be identified) |

Every operation is logged with the session's name and code, the effect label shown, and the
outcome.

### 3.13 Activities

What each component is doing *right now*, adopted from MAST's `common/activities.py` with the
parts that hurt there left out. Jobs (§2) are requests with a life cycle; activities are
component state. A job raises one or more activities, and an activity can exist without a job
(a mount that is tracking).

- **Where:** `wilds.control.activities`, in wilds-control. Control owns all hardware and every
  procedure (§1), so it is the only place activities start and end. wilds-bridge, -solving and
  -science-camera know nothing of them: control raises `Slewing` around its call into bridge.
  wilds-gui sees activities only as JSON. The module imports no FastAPI, so it can move into
  wilds-bridge, the way science-camera already depends on it, if a second repo ever needs to
  start activities itself. A separate common repo only when three repos share types.
- **Each component defines its own activity enum** next to its code and registers it:
  `activities = registry.register("mount", MountActivity)`. No central table of components.
- **Names on the wire, never a bitmask.** MAST numbers its flags with `auto()` and sends the
  bitmask, which forced an append-only rule and a cross-repo check before removing a member.
  Here the state is the set of running members, kept in start order, and `/status` carries
  their names, so adding, reordering or removing a member cannot change what a consumer reads.
- **Calls:** `start(activity, details=, existing_ok=)`, `end(activity)`, `is_active`,
  `is_idle`, `names`, and `wait_clear(activity, timeout=)` (sync) / `await_clear` (async).
  A wait that times out returns `False` and **leaves the activity set**, so the component still
  reports it and the caller reports the failure instead of hiding it.
- **Timing:** the duration is measured on the monotonic clock, so an NTP step cannot skew it
  or make it negative. The wall-clock start is reported as `since_ms`.
- **Everything under one lock**, including the timings and details (MAST mutates those outside
  it).
- **A listener can never fail the operation it announces.** `on_change` listeners get an
  `ActivityEvent` (start or end, with details and duration). They must not block; an exception
  in one is logged and swallowed. The `/status` stream needs no listener: it reads the registry
  each second, and listeners are for toasts and logs later.
- **Conventions kept from MAST:**
  - a `Cancelling` member on every component that can move, so cancel is a visible, waitable
    state;
  - umbrella members (`Calibrating`, `ExecutingPlan`) for work spanning components, tied to
    the job that raised them;
  - **no member before something sets it.** A member nobody starts reads like a completion
    signal, and a waiter on it never returns. So the registry ships with no components: each
    arrives with the code that sets its activities.
- **`/status`:** `activities: {component: [{name, since_ms, details}]}`, listing only
  components with something running. Running jobs at handover (§3.11) show up here per
  component.
- **In the GUI:** the *activity board*. A row of component rectangles; each running activity
  hangs below its component as a pill with its elapsed time. A job touching two or more
  components is an umbrella bar across them (kind, who started it, progress, Cancel); a
  single-component job puts its progress in the pill. An ended pill or bar tints by outcome
  (done, failed, cancelled), then collapses and the stack closes up. It replaces the old
  component cards and the activity heart, and is on `/to`, `/io` and `/eng`. A **failed job**
  also leaves a red card in the *attention band* under the header until dismissed, since its
  pill is gone; nothing shows there when all is well. Only the session holding control sees an
  enabled Cancel (`/io`, `/eng`); `/to` is read-only.
- **`opmode`** (`config.toml`, top level): `development` (the default) or `production`.
  Reported in `/status`. The `/dev` helpers answer `404` unless it is `development`; the file is
  re-read when it changes. `POST /dev/jobs/demo` ("testing 1 2 3") is a made-up job across
  the slit, ADC and guide camera for watching the board, started from a card on `/io` Operations.
  A deployed `config.toml` says `production`.

### 3.14 Jobs and errors

A *job* is one long procedure that control runs (acquisition, autofocus, slit rotation, an
exposure series). It lives in the backend, so it survives a closed tab and the same API
serves the GUI, scripts and automation (§1). Jobs are requests with a life cycle; the
components' activities (§3.13) are what they raise while they run.

**Errors.** MAST wraps every reply in a `CanonicalResponse` (`value` or `errors`, always HTTP
200). WILDS does not: HTTP carries the outcome, so the generated client (§2) is typed per
endpoint and the OpenAPI schema shows the error cases. MAST's `succeeded` is also
`value is not None`, which reads a successful call returning nothing as a failure. What is
kept is one error shape that a caller can always parse, used by every error response *and* by
failed jobs:

`{"code": "busy", "message": "stage is moving (job j-41)", "details": {...}}`

- `code` is machine-readable, so the GUI never parses text. Initial set: `lease_required`
  (403), `not_found` (404), `busy` (409), `invalid` (422), `refused` (the TCS or a device said
  no), `timeout`, `cancelled`, `internal`.
- `message` is for people. `details` is optional and structured (e.g. the blocking `job_id`).

**Model:**

| Field | Meaning |
|---|---|
| `id`, `kind` | e.g. `j-41`, `autofocus` |
| `state` | `queued` → `running` → `done`, `failed` or `cancelled` |
| `started_by` | session name and short code that created it (provenance, §3.11) |
| `created_at`, `started_at`, `ended_at` | UTC; durations come from the monotonic clock |
| `progress` | `{fraction: 0..1 or null, message}` |
| `activities` | the `{component: [names]}` it has raised, from §3.13 |
| `result` | on `done`, per kind |
| `error` | on `failed` or `cancelled`: the error shape above |
| `cancel_requested` | set by `POST /jobs/{id}/cancel`; not a state of its own |

**Rules:**

- **Start:** `POST` of a procedure needs the lease token (`403 lease_required`), creates the
  job and returns `202 {job_id}`.
- **A job declares the components it uses.** Two jobs may not use the same component at once:
  the second is refused `409 busy` with `details.job_id` of the first, or queued if the
  request asks for it (§3.11, requests while busy).
- **Activities follow the job.** It starts its activities as it needs them and *always* ends
  them (done, failed or cancelled), so a component never stays marked busy by a dead job.
- **Cancel:** cooperative. It sets `cancel_requested`, raises `Cancelling` on the job's
  components, and the job becomes `cancelled` once they have stopped. Cancel needs the lease
  token, so it is the *current holder* who may cancel, not the session that started it.
- **Handover (§3.11):** a running job keeps running under the new holder. A job still
  `queued` is dropped and logged as `cancelled` with reason "holder changed", since the new
  holder never asked for it.
- **Events:** every state or progress change is a `job` event on `/status/stream`, carrying
  the whole job. `/status` lists active jobs and the last few finished ones.
- **Audit:** create, start, end and cancel are logged with the session's name and code.
- **Retention:** in memory, the last 100 finished jobs. A restart of wilds-control drops
  them, as it drops sessions and the lease; running jobs do not survive it.

### 3.15 Filing: local staging and delivery

Everything the instrument produces is written on the **local staging disk** and delivered from
there to the facility's data folder on the **share** (`/lois/data/dct`, §3.7). The share may be
down, slow or not yet mounted; the night must not depend on it, and no frame may be lost to it.
Code: `wilds/control/filing.py`. Adapted from MAST's `Filer` (RAM disk to share, deferred moves,
a sweeper), without its `MoveGuardian`, which guards against a mover taking a file a producer is
still writing: here a file is visible to the sweeper only when it is whole, so nothing needs
guarding. MAST's `common` package is not shared (§2, "GUI stack and look" says the same of its
GUI).

- **The staging folder is the queue.** A producer calls `filing.writing("<ID>/YYYYMMDD/<name>")`
  and writes to `.<name>.partial`; on a clean exit the file is renamed into place, whole, and the
  sweeper is woken. A failed write leaves nothing. What is in staging under a normal name is
  complete and waiting, and survives a restart of wilds-control (no in-memory queue to lose).
  Hidden names are the mechanism, never a product. At startup, partials left by a crash are
  removed and what waits is counted.
- **Delivery** runs in one background thread, every `sweep_interval_s` (30 s) and whenever a file
  is added: probe the share; if it answers, copy each waiting file (oldest first) to
  `.<name>.partial` beside its target, check the size, rename into place, then remove it from
  staging (and any empty folders). The final name on the share is therefore always a whole file,
  which matters because facility software watches that folder. A file already there with the
  same bytes counts as delivered. A *different* file with the same name is never overwritten:
  the source stays, the clash is logged once and shown as a warning.
- **The probe** creates and removes a folder in the share's root, in a thread joined with a
  timeout (`probe_timeout_s`, 2 s), so a hung mount costs one timeout, not a stuck sweeper. While
  an earlier probe is still hung no second thread is started. The share's root is **never
  created**, and with `share_is_mount = true` (production) it must be a mount point: an unmounted
  mount point is an empty local folder, and delivering into it would fill the local disk and look
  like success.
- **Disk ceiling:** a new file is refused (`StagingFull`) when the staging disk is 90 % full.
- **Status:** `/status` has `filing`: `share` (`unknown | online | offline`), `pending` files and
  bytes, the age of the oldest, `delivered` since start, `last_error`, `staging_used`,
  `staging_full`. While files wait and the share is offline, and when the disk is full or a
  delivery is stuck, `warnings` says so (shown on the pages as other warnings are).
- **Configuration** (`[filing]`, deployment settings, read at start): `staging_root`
  (`/var/lib/wilds/staging`), `share_root` (`/lois/data/dct`), `share_is_mount`,
  `sweep_interval_s`, `probe_timeout_s`, `disk_ceiling`. In development both roots are folders
  under `~/wilds-dev`; renaming the share folder "unplugs" it.
- **Not decided here:** the numbering of files (`<abbr>.NNNN.fits`) belongs to whoever names
  them and must not restart when staging empties (MAST's counter lived on a RAM disk and did
  exactly that); whether anything else (LIG's pipeline, Locutus headers) must happen before a
  file may be delivered; the GUI display of `filing` (a line on the board or a badge); the
  Products page (§2) reads both staging and the share.

## 4. Order of work

| # | Step | Testable with | Blocked by |
|---|---|---|---|
| 1 | Status model and stream; GUI status panel. `TelescopeStatus` falls back to `TCSTcsStatusSV`. | TCS simulator, now | none |
| 2 | Guide-camera simulator and live view with overlays | image simulator | none |
| 3 | Jobs framework and error shape (§3.14) and the control lease, with the TO page | unit tests | none |
| 3b | Activities registry and `/status` field (§3.13); components add theirs as they arrive | unit tests | none |
| 3a | Configuration framework (models, storage, history, schema endpoints) and the generic Django form renderer; first panels for the guide camera and stage presets | simulators | none |
| 4 | Slit width, lamps and ADC panels | MCS2 SCPI simulator (6 channels) | 6-channel MCS2 part number; lamp stage models; lamp power switching (TBD) |
| 5 | Target submission and slit rotation (`TCSTcsCommandSV`; fix `slew_to`'s topic) | TCS simulator | a session on nysa with Georgi, in case the MCB faults |
| 6 | Acquisition | TCS simulator and image simulator | **LIG: offset path (MCB fault)**; commissioning values |
| 7 | Autofocus | AOS simulator | **AOS simulator (Georgi)** |
| 8 | Science exposures and Locutus headers | Archon stub; Locutus on `joe` | STA ACF; hardware; a WILDS Locutus dictionary |
| 9 | Commissioning procedures | the real telescope | hardware at LDT |

## 5. Questions for LIG

Answered by Tim (2026-11-02):

- *Which topic carries a science target?* `TCS.TCSSharedVariables.TCSSubData.TCSTcsCommandSV`,
  with a full `ScienceTargetConfiguration`.
- *Is a rotator-only change possible?* No; re-send the target with a new rotator (§3.5).
- *Users and roles?* No central authentication; role-based observer logins, coordination
  with the TO, and access to a WILDS web GUI left to us (§3.11).

Still open:

1. How should instruments send to `TCSTcsCommandSV` without faulting the MCB (April)?
   Was that a malformed message, or something in the MCB?
2. Must the outer `demandRADec` values be integers? Does `pointingOriginX/Y` override the
   instrument's pointing origin?
3. Which `TCSTcsStatusSV` field, if any, shows that the TO has locked out external
   commands? Does the lockout also cover offsets and AOS focus?
8. Is a re-sent target that changes only the rotator previewed like a new target? Are
   offsets and focus never previewed?
4. The AOS simulator's status, for autofocus testing.
5. The `wilds.loisTelemetry` schema, the instrument ID and abbreviation, a TCS instrument
   entry, and a Locutus FITS dictionary for WILDS.
6. What does `rotI` mean in an instrument definition?
7. Will a TO-page passphrase and the short-code flow (§3.11) suit the TOs?

## 6. Open decisions (ours)

- **Lease details:** heartbeat interval and dead-session timeout; the viewer cap; whether
  engineering mode ends by itself (e.g. at sunset, or when the TO grants control).
- **TO cancelling jobs:** for now only the lease holder (including engineering mode) can
  cancel; the TO page is read-only. Later the TO may be allowed to cancel any job with their
  passphrase (their Revoke does not stop a running job, which carries on under the next holder).

## 7. Decided

- **GUI: Django** (2026-10-31). Django (`wilds-gui`, :8000) serves the pages and keeps the
  observing log. The browser talks to wilds-control (:8001) directly over REST, SSE and
  WebSocket for commands, status, configuration and video. Django keeps no hardware state.
- **No user accounts** (2026-11-04). Anonymous browser sessions with a typed name; one
  control lease, granted by the TO with a short code; a shared passphrase for the TO
  page. wilds-control enforces the lease (§3.11).
- **Two GUIs, one holder, many viewers** (2026-11-05): `/to` for the TO, `/io` for
  instrument operators; any number of read-only sessions from anywhere (local, VNC, VPN);
  engineering mode takes control with a passphrase, for WILDS mechanisms only, never TCS
  commands (§3.11).
- **One path for telescope commands** (2026-11-05): every telescope-state-changing
  operation shows the TCS access badge and its effect ("Send to TO preview" or "Applies
  immediately"), is greyed out with a reason when unavailable, and is confirmed from
  telemetry rather than from sending (§3.12).
- **GUI look copied from MAST** (2026-11-05): Django, Bootstrap 5.3, HTMX, Alpine.js; MAST's
  palette and components, plus a dark theme, safe text, pinned and bundled libraries, and no
  accounts (§2, "GUI stack and look").
- **Target and rotator path** (2026-11-04, from Tim): `TCSTcsCommandSV`, re-sending the
  current target with the changed fields (§3.2, §3.5).
- **Code structure** (2026-11-05): abstract base plus implementations per component (e.g.
  `GuideCameraBase` → `ASICamera`, `SimulatedASICamera`), chosen by a deployment setting;
  `stage/{controller,stage}.py`; singletons created at startup and injected with
  `Depends`; pydantic `BaseModel` for all relevant classes. Details in
  `memory/project_control_conventions.md`.
- **Server: Uvicorn** (2026-11-05; reverses an earlier Hypercorn choice made for HTTP/2, which
  is not needed); one worker. `main()` in `control/app.py` parses argv and runs it. Plain
  HTTP/1.1, so keep the number of long-lived streams per tab low (6 connections per origin).
- **One TOML configuration file** (2026-11-05), replacing a file per component (§3.10). Refined
  2026-11-08: it holds *deployment* settings only; operational and calibration settings are
  written by the app into a SQLite settings store, with history and revert (§3.10).
- **Engineering page and TO presence** (2026-11-07): `/eng` replaces the engineering page in
  `/io`; no software coordination of engineering access (Zoom or in person); passphrases live
  in `/etc/wilds/passphrases.toml` (re-read when the file changes; moved out of `config.toml`
  on 2026-11-07); the default engineering
  passphrase `physics` is accepted without a warning, with no rate limiting for now. The TO is online while
  a TO page is open and unlocked. Aliases: `/telescope-operator`, `/telop` → `/to`;
  `/instrument-operator`, `/instop` → `/io`; `/engineering` → `/eng`. `/eng` stays on :8000;
  a reverse proxy on :80 for `http://wilds/...` is not decided.
- **Local staging with probe and sweep** (2026-11-08): data files are written to a local staging
  disk and delivered to the share by a sweeper, which can be built and tested before the share
  exists (§3.15).
- **Calibration and Configuration live on `/eng`** (2026-11-07), not `/io`: `/io` keeps Operations
  and Products; `/io/calibration/` and `/io/configuration/` redirect to the `/eng` pages.
