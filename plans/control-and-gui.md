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

## 2. API conventions (wilds-control, FastAPI on :8001)

| Kind | Form |
|---|---|
| Snapshot | `GET /status`: one model with telescope, instrument, cameras, jobs and the control lease |
| Live updates | `GET /status/stream` (SSE): `status` every 1 s, plus `job` and `log` events when they happen |
| Short command | `POST`/`PUT` returns `200` with the resulting state |
| Long procedure | `POST` returns `202 Accepted` with `{job_id}`; progress arrives as `job` events |
| Jobs | `GET /jobs`, `GET /jobs/{id}`, `POST /jobs/{id}/cancel`. States: queued → running → done, failed or cancelled |
| Control | `POST /control/requests` (request, returns a pairing code), `DELETE /control` (release); TO actions in §3.11. Every command and configuration change needs the lease token, or gets `403`. |
| Configuration | `GET /config`, `GET`/`PUT /config/{component}`, `GET /config/{component}/schema`, `GET /config/{component}/history` (see §3.10) |
| Video | `WS /guide-camera/stream`: binary JPEG frames, each preceded by a JSON metadata frame |

The Python client is generated from FastAPI's OpenAPI schema. It replaces the hand-written
`client/api.py`, whose `/acquire/cancel` and `/solve` paths have no router behind them.
The skeleton's `/telemetry` router becomes `/status`.

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
object}` → job; `POST /jobs/{id}/cancel` (abort); `GET /exposures` (recent files).

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
model in the backend. wilds-control validates them, stores them (a versioned file per
component), and applies them to the device. The Django panels render their forms from
`GET /config/{component}/schema`, the model's JSON Schema, so fields, units, limits and
help text are defined once, in the backend. Django's own database holds only the
observing log; there are no user accounts (§3.11).

**Three kinds of setting**, shown separately in each panel:

| Kind | Examples | Who may change | When it applies |
|---|---|---|---|
| Operational | guide-camera exposure, gain; stage presets | the control holder | immediately, or rejected with `409` if the device is busy |
| Calibration | plate scale, slit pixel, IAA, focus offset, ADC zero points | the control holder, in engineering mode | on save; every change logged with the holder's name and the time, and revertible |
| Deployment | broker host (`joe`/`tanagra`), device addresses, data paths, simulator on/off | file only, not editable in the GUI | on restart |

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

- **Viewers:** anyone who can reach the GUI. It is hosted at LDT and reachable only over
  the VPN.
- **Identity:** an anonymous Django session (cookie) per browser, plus a name the person
  types (e.g. "Arie (Rehovot)"), kept in the browser. It's a label for screens and the log,
  not authentication.
- **The control lease:** one browser session at a time holds it. wilds-control issues and
  checks the lease token on every command and configuration change, so scripts or a second
  GUI can't bypass it. Django shows the prompts and keeps the presence heartbeat.

**Granting control (pairing code):**

1. The requester clicks *Request control*; their screen shows a short code (`K7-42`).
2. On Zoom they tell the TO their name and code.
3. The TO's page lists pending requests with names and codes; the TO approves the match.

The name proves nothing; the code ties the approval to the browser of the person on the
call. If someone already holds control, the TO's approval preempts them and they are told
who has it now.

**The TO page** is the only credentialed part: a shared role passphrase set in deployment
configuration (preferred), or an allowlist of TO console hosts. A second passphrase for an
engineer role can follow; that is configuration, not user management.

**Rules:**

- **Automatic refusal or deferral:** while an exposure or acquisition runs, a new request
  shows why ("exposure in progress, 4 min left") and can be queued until the job ends. The
  TO can still override.
- **Running jobs at handover** continue under the new holder rather than being aborted;
  the new holder may cancel them.
- **Lost holder:** if the holder's browser stops sending heartbeats, the lease lapses after
  a timeout, so a closed laptop can't lock everyone out.
- **Same browser, second tab:** shares the lease; no prompt.
- **Engineering mode** (a TO-page switch, for daytime commissioning without a TO): anyone
  may take control; changes to calibration settings are allowed.
- **Audit:** every request, grant, refusal, preemption and lapse is logged with the typed
  name, time and reason.

**Shown on every page:** "WILDS control: held by X" and, if the TCS reports it, whether
the TCS is accepting external commands. The first time someone takes control each night,
the GUI reminds them to confirm with the TO before commanding the telescope.

**LDT messages:** none for the lease itself. ← `TCSTcsStatusSV` for the TCS's
external-command lockout, if it's there (field to be identified).

**Endpoints:** `POST /control/requests` `{name}` → `{request_id, code}`;
`GET /control/requests` (TO); `POST /control/requests/{id}/approve` and `/refuse`
`{reason}` (TO); `DELETE /control` (release); `PUT /control/mode` `{engineering}` (TO);
`POST /control/heartbeat`. TO endpoints need the TO passphrase.

## 4. Order of work

| # | Step | Testable with | Blocked by |
|---|---|---|---|
| 1 | Status model and stream; GUI status panel. `TelescopeStatus` falls back to `TCSTcsStatusSV`. | TCS simulator, now | none |
| 2 | Guide-camera simulator and live view with overlays | image simulator | none |
| 3 | Jobs framework and the control lease, with the TO page | unit tests | none |
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
   commands?
4. The AOS simulator's status, for autofocus testing.
5. The `wilds.loisTelemetry` schema, the instrument ID and abbreviation, a TCS instrument
   entry, and a Locutus FITS dictionary for WILDS.
6. What does `rotI` mean in an instrument definition?
7. Will a TO-page passphrase and the pairing-code flow (§3.11) suit the TOs?

## 6. Open decisions (ours)

- **Lease details:** heartbeat interval and lapse timeout; whether engineering mode needs
  the TO passphrase to switch on (yes, as drafted) and whether it switches off by itself at
  sunset.

## 7. Decided

- **GUI: Django** (2026-10-31). Django (`wilds-gui`, :8000) serves the pages and keeps the
  observing log. The browser talks to wilds-control (:8001) directly over REST, SSE and
  WebSocket for commands, status, configuration and video. Django keeps no hardware state.
- **No user accounts** (2026-11-04). Anonymous browser sessions with a typed name; one
  control lease, granted by the TO with a pairing code; a shared passphrase for the TO
  page. wilds-control enforces the lease (§3.11).
- **Target and rotator path** (2026-11-04, from Tim): `TCSTcsCommandSV`, re-sending the
  current target with the changed fields (§3.2, §3.5).
