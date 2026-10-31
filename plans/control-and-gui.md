# Plan: wilds-control API and wilds-gui panels

*Status: draft for discussion, 2026-10-31. Nothing here is implemented yet.*

## 1. Principles

- **wilds-control owns all hardware and every procedure.** Acquisition, autofocus, slit
  rotation and exposures run in the backend as *jobs*. The GUI starts them, shows their
  progress and can cancel them. A procedure survives a closed browser tab, and the same
  API serves the GUI, scripts and later automation.
- **WILDS drives its own workflow** and talks to LDT only through the ActiveMQ broker
  (facility broker `joe`; test broker `tanagra`). This answers Tim's open question: WILDS
  integrates target submission into its own software.
- **One controller, many viewers.** One session holds control; others are read-only.
- **Every device has a simulator** (TCS on nysa, guide-camera image simulator, MCS2 SCPI
  server, Archon stub), so the GUI can be built before hardware arrives.
- **Measured constants are configuration, never code**: plate scale, slit pixel, IAA,
  pointing origin, LDT↔WILDS focus offset, ADC zero points.
- **Parfocal cameras**: the guide camera and the science cameras are parfocal by design, so
  focusing on the guide camera focuses the science cameras.

## 2. API conventions (wilds-control, FastAPI on :8001)

| Kind | Form |
|---|---|
| Snapshot | `GET /status`: one model with telescope, instrument, cameras, jobs and the controlling session |
| Live updates | `GET /status/stream` (SSE): `status` every 1 s, plus `job` and `log` events when they happen |
| Short command | `POST`/`PUT` returns `200` with the resulting state |
| Long procedure | `POST` returns `202 Accepted` with `{job_id}`; progress arrives as `job` events |
| Jobs | `GET /jobs`, `GET /jobs/{id}`, `POST /jobs/{id}/cancel`. States: queued → running → done, failed or cancelled |
| Control | `POST /session/control` (take), `DELETE /session/control` (release) |
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
| → | `TCS…TCSLowLevelStatusSV.NewScienceTargetSV` *(path unresolved)* | `newScienceTarget` XML: `demandRADec`, `scienceTargetConfiguration` (`rotator.rotPA`, `rotatorFrame=Target`, `wl_microns`, `fracrate`, target block), `pointingOriginX/Y`, `trackID` | **Blocked:** on 2026-10-31 this reached the broker but not the TCS. In the default MCB configuration this topic only flows TCS → broker. Ask LIG which topic CAT/OTL use; it may be `TCS.TCSSharedVariables.TCSSubData.TCSTcsCommandSV`. |
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

There is **no standalone rotator command** (JIRA TCSD-131). The slit's sky PA is set by
re-sending the target with a new `rotPA`, with `rotatorFrame=Target` to hold the field
still on the slit.

**LDT messages**

| Dir | Topic | Content | Notes |
|---|---|---|---|
| → | as in §3.2 | the target re-sent with a new `rotPA` | Inherits §3.2's blocker. LIG: is the TCS "Rot Offset" command usable over the broker instead? |
| ← | `TCSTcsStatusSV` | `objectPA.EqPA` (slit sky PA), `currentRotatorPositions.rotPA` (mechanical), `rotIPA` (demanded), `iaa`, `inPositionRotIsTrue`, `limits.timeToRotLimit_min`, `rotCurrentWrap` | Semantics verified live, 2026-10-31 |

Common choices: parallactic angle (from `currentParAngle`) or a user PA.

**Endpoints:** `POST /slit/rotation` `{sky_pa_deg | "parallactic"}` → job.

### 3.6 Slit width, calibration lamps, ADC

These are WILDS mechanisms with no LDT commands. The stages are on one SmarAct MCS2-C-0006
over SCPI on TCP 55551 (channel 0 slit, 1 ADC 1, 2 ADC 2).

**LDT messages**

| Dir | Topic | Content | Notes |
|---|---|---|---|
| ← | `TCSTcsStatusSV` | `currentParAngle`, elevation | ADC auto-tracking |
| ← | `WRSDataPacket` | temperature, pressure, humidity | Atmospheric dispersion model |
| → | `wilds.loisTelemetry` | slit width (mm and arcsec at 8.02″/mm), lamp states and positions, ADC angles | |

**Endpoints:** `PUT /slit/width` `{width_mm | width_arcsec}`; `POST /slit/width/jog`
`{delta_mm}`; `PUT /lamps/{n}` `{on, position: in|out}`; `PUT /adc/{n}` `{angle_deg}`;
`PUT /adc/tracking` `{enabled}`.

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

**Endpoints:** `POST /commissioning/{procedure}` → job; `GET /config/calibration`;
`PUT /config/calibration` (save reviewed values; changes are logged).

## 4. Order of work

| # | Step | Testable with | Blocked by |
|---|---|---|---|
| 1 | Status model and stream; GUI status panel. `TelescopeStatus` falls back to `TCSTcsStatusSV`. | TCS simulator, now | none |
| 2 | Guide-camera simulator and live view with overlays | image simulator | none |
| 3 | Jobs framework and control session | unit tests | none |
| 4 | Slit width, lamps and ADC panels | MCS2 SCPI simulator | lamp and stage controllers for the lamps (TBD) |
| 5 | Target submission and slit rotation | TCS simulator | **LIG: target path** |
| 6 | Acquisition | TCS simulator and image simulator | **LIG: offset path (MCB fault)**; commissioning values |
| 7 | Autofocus | AOS simulator | **AOS simulator (Georgi)** |
| 8 | Science exposures and Locutus headers | Archon stub; Locutus on `joe` | STA ACF; hardware; a WILDS Locutus dictionary |
| 9 | Commissioning procedures | the real telescope | hardware at LDT |

## 5. Questions for LIG

1. Which topic and format do CAT and the OTL use to send a science target? Should
   `NewScienceTargetSV` be broker → TCS on nysa's MCB?
2. How should instruments send offsets without faulting the MCB (the April `TCSTcsCommandSV` issue)?
3. Is a rotator-only change (the TCS "Rot Offset") possible over the broker?
4. Must the outer `demandRADec` values be integers? Does `pointingOriginX/Y` override the
   instrument's pointing origin?
5. The AOS simulator's status, for autofocus testing.
6. The `wilds.loisTelemetry` schema, the instrument ID and abbreviation, a TCS instrument
   entry, and a Locutus FITS dictionary for WILDS.
7. What does `rotI` mean in an instrument definition?

## 6. Open decisions (ours)

- **GUI technology.** Django (accounts, observing log, admin; MAST_gui precedent) or a
  single-page app served by FastAPI. Either way the GUI talks to wilds-control over REST,
  SSE and WebSocket only.
- **Users.** Telescope operator, observer, or both? This decides what sits behind an
  engineering or commissioning mode.
