---
name: WILDS Project Context
description: This workspace is for WILDS — Weizmann Institute Lowell Discovery Spectrograph — built on top of the Archon actor framework
type: project
originSessionId: 53fd8017-61d6-4f21-b337-bd5f7dcf3631
---
This project is named **WILDS** (Weizmann Institute Lowell Discovery Spectrograph).

The codebase is the `archon` Python actor framework (originally from SDSS), being used/adapted as the controller software for the WILDS spectrograph instrument. It is intended to replace the DeVeny spectrograph at LDT.

**Hardware:**
- Detector: STA4850 — 4080×4080 pixels, 15 µm pixel size, full-frame CCD
- Controller: STA Archon CCD controller
- Telescope: Lowell Discovery Telescope (LDT), 4.3m, Anderson Mesa, AZ

**Software base:** `sdss-archon` Python actor library (https://github.com/sdss/archon), editable install via `pip install -e .`
**NOTE:** The sdss-archon SDK will likely be replaced by an SDK provided directly by STA. Design the ScienceCamera interface to be SDK-agnostic where possible.

**Key dependencies:** `sdss-clu` ≥2.5.3 (AMQP/RabbitMQ actor framework), `numpy` ≥2.0, `astropy` ≥6.0

**Why:** New instrument project at Weizmann Institute, deployed at Lowell Discovery Telescope. The Archon library handles communication with STA Archon CCD controllers.

**How to apply:** When discussing the project, refer to it as WILDS. Suggestions and changes should be framed in the context of a spectrograph instrument control system integrating with LDT infrastructure. The archon codebase is being adapted (not used generically) for the WILDS Archon controller component. The ScienceCamera singleton should be designed to be SDK-agnostic in anticipation of switching to the STA-provided SDK.

---

## LDT Infrastructure

- **Message broker:** Apache ActiveMQ Classic 5.x at telescope site
  - STOMP on port 61613 (use this from Python)
  - AMQP on 5672, OpenWire on 61616
  - Auth not currently required
- **Python stack:** `stomp.py` ≥4.1.21, `xmltodict`, `xmlschema`, `ligmos` (https://github.com/LowellObservatory/ligmos)
- **XML serialization:** `pydantic-xml` v2.19+

### Broker Hosts
- **Real LDT broker (Anderson Mesa):** hostname `joe`, STOMP port 61613 (no auth required)
- **Test broker (Mars Hill):** `tanagra.lowell.edu`, port 61616 (OpenWire) — use STOMP port 61613 for Python
  - Admin UI: http://tanagra.lowell.edu:8161/admin/topics.jsp (admin/admin)
  - Has **MCB (Message Conversion Bridge)** — intermediary between Shared Variable Engine and broker; has UI to select which topics are bridged and in which direction
  - Has **MCU simulator** — simulates the mount; TCS will not start without it
  - Simulator startup order: (1) Start MCB, (2) Start MCU simulator, (3) Start TCS
  - `tcs.loisTelemetry` is **NOT** published by the simulator — it comes from JOE only; not available in simulator sessions
  - Individual `TCSLowLevelStatusSV.*` topics only appear once MCB is running
  - STOMP on port 61613 confirmed working with stomp.py; `reconnect_attempts_max` must be **1** (not 0 — 0 means don't connect at all in stomp.py)
- **Test broker (WebRat):** `lo-webrat.lowell.edu` / `webrat.lowell.edu` (IP: 10.10.130.155)
  - Ubuntu 20.04 VM; login: `dlytle` + Lowell password
  - Runs Apache ActiveMQ **Artemis** (NOT Classic 5.x) in `/opt/artemis`; broker instance `test-broker` in `/var/lib`; credentials `admin`/`admin`
  - Also runs Redis (compiled from source, open to all connections)

### Key ActiveMQ Topics

**Subscribe (status):**

| Topic | Rate | Source | Notes |
|-------|------|--------|-------|
| `tcs.loisTelemetry` | ~2 Hz | JOE / FacilitySummary.java | Strings/sexagesimal; easiest to use. BUG: sends each message twice ~1/min |
| `TCS.TCSSharedVariables.TCSHighLevelStatusSV.TCSTcsStatusSV` | ~1 Hz | LabVIEW shared vars | Structured XML; has offset/rate info not in loisTelemetry. Schema: **v1.9** (live on tanagra 2026-04-29) |
| `TCS.TCSSharedVariables.TCSLowLevelStatusSV.*` | ~1 Hz | LabVIEW shared vars | Individual plain-text SVs (CurrentAzSV, CurrentElSV, CurrentParAngleSV, CurrentZD_degSV, etc.) — no XML, each is its own topic |
| `deveny.loisTelemetry` / `wilds.loisTelemetry` | ~1 Hz | JOE | Instrument state (WILDS schema TBD with LIG) |
| `aos.loisTelemetry` | ~1 Hz | JOE | Active optics |
| `AOS.AOSPubDataSV.AOSDataPacket` | ~1 Hz | LabVIEW AOS | Structured XML; has totalFocusOffset (m), summaryState/detailedState, M1/M2 piston/tilt demands, comaPointingOffset, settled flags. XSD: ligmos/schemas/AOS.AOSPubDataSV.AOSDataPacket.xsd |
| `instrumentCube.loisTelemetry` | ~1 Hz | JOE | Instrument cube |
| `wrs.loisTelemetry` | ~1 Hz | JOE / WRSTelemetry.java | Weather/environment |

**tcs.loisTelemetry key fields:** Root tag `TCSTelemetry`. TimeStamp (Unix s, capital S), TCSLST, TCSUTC, DemandRa, DemandDec, CurrentHourAngle, TCSCurrentAzimuth, TCSCurrentElev, TCSCurrentZenithDistance, TCSCurrentRotatorPA, TCSCurrentRotatorIPA, TCSCurrentRotatorIAA, RotatorFrame, TargetFrame, equinox, TCSState, TCSHealth, TCSHeartBeat, TCSAccessMode, InPosition, ScienceTargetName, MountTemperature, m1CoverState, MountDomeAzimuthDifference (can be string "UNKNOWN" — use str|None), DomeOccultationWarning, CurrentParAngle, CLSLowBankState, DSSPositionStatus. Published by JOE only (~2 Hz); not available from simulator.

**aos.loisTelemetry key fields:** Root tag `AOSTelemetry`. TimeStamp (Unix s), FocusOffset (**microns** — note: AOSDataPacket uses meters). Minimal content; prefer AOSDataPacket for AOS status.

**deveny.loisTelemetry** is the template for `wilds.loisTelemetry`. WILDS will publish: Timestamp, SlitWidthMM, SlitWidthASEC, ADC stage positions, 2 calibration lamps (name, on/off state, In/Out position), shutter, detector and guide-camera state. Draft model: `wilds-bridge` `WildsTelemetry`. Schema TBD with LIG.

**instrumentCube.loisTelemetry:** Root `InstrumentCubeTelemetry`. TimeStamp, InstrumentCoverState (Unknown/Open/Closed), InstrumentCoverPosition (mm), FMA–FMD State (Unknown/Home/Extended) and Position (mm). Relevant for FITS headers.

**Weather topics:**
- `WRS.WRSPubDataSV.WRSDataPacket` — metric, sea-level corrected; only when telescope operational. Fields: airTemp_C, barPressure_mbar, dewPointCurrentValue, relativeHumidity, windSpeed (m/s), windDirection_deg, tenMinWindGustSpeed, temperatureStatistics, windSpeedStatistics. Preferred for FITS headers.
- `ldtWeatherStream` → `DCTweather` root — always-on fallback; **imperial units**, raw from station.

**Publish (commands):**

- `TCS.TCSSharedVariables.TCSSubData.TCSTcsCommandSV` — telescope offsets (XML)
  - Reply on: `TCS.TCSSharedVariables.TCSLowLevelStatusSV.TCSTcsCommandResponseSV`
  - **MCB ISSUE (2026-04-29):** Broker→SVE direction causes MCB fault (LabVIEW cluster type mismatch). MCB fault kills ALL topic bridging. Publishing works; TCS receipt is unconfirmed. Need to ask Tim/Ryan about correct command path.
  - Three XML command types (root tag varies):
    - `<scienceTargetOffset>`: commandID (µs since midnight UT), offsetDef {num1: User|Handset, off1, off2 (arcsec), offsetType: Simple|TPLANE}
    - `<scienceTargetClearOffset>`: commandID, num2: User|Handset
    - `<scienceTargetAbsorbOffset>`: commandID, num2: User|Handset — bakes offset into target coords

- `TCS.TCSSharedVariables.TCSLowLevelStatusSV.NewScienceTargetSV` — slew to new science target
  - Root tag: `<newScienceTarget>` containing demandRADec, pointingOriginX/Y, scienceTargetConfiguration (fracrate, rotator {rotPA, rotatorFrame}, target {RA/Dec, proper motion, parallax, rv, targetName, difftrack}, wl_microns), trackID

- **Rotator:** no standalone command — JIRA TCSD-131 folded it into the target command's `rotPA` (0–360°, E of N; the slit's sky PA) and `rotFrame`. `target` (TCS default) holds the field still on the instrument; `fixed` tracks in Az/El (sky rotates). `rotoffset iaa` (alignment angle) and `mccrot*` (move/stop/stow/track) exist as TCS commands but aren't on the broker interface — ask LIG whether instruments may use them. Source: TCS External Interface Definitions DCT-0720S-005-I §3.3.3.2, 3.6.3 (see [[reference_confluence]])

- **AOS Focus commands** — plain-text single-value messages (NOT XML), units **meters**:
  - `AOS.AOSSubDataSV.AbsoluteFocusOffset` — e.g. `3.1E-3`
  - `AOS.AOSSubDataSV.RelativeFocusOffset` — relative delta in meters
  - `AOS.AOSSubDataSV.ClearFocusOffset` — send `true` to reset to 0
  - **Focuser class publishes to AOS topics, not TCS**

### XML Parsing Guidance (from LIG)
- Use `xmlschema` (preferred, typed) over raw `xmltodict` for production
- XSD schemas in https://github.com/LowellObservatory/ligmos/tree/master/ligmos/schemas
- **JOE startup bug:** float fields can contain string `"UNKNOWN"` at night start — don't parse messages right at startup
- **JOE duplicate bug:** `tcs.loisTelemetry` sent twice ~once/min (known, unfixed)
- For Redis storage: flatten nested dicts with recursive flattener, period-delimited keys

### Locutus config (reference — real LDT connection)
```yaml
broker_host: [joe, 61613]
redis_host: [localhost, 6379]
agents_to_monitor:
  - mount:
      incoming_topic: TCS.TCSSharedVariables.TCSHighLevelStatusSV.TCSTcsStatusSV
  - weather:
      incoming_topic: wrs.loisTelemetry
```

### Existing LDT systems
- **LOIS** — C/TCL instrument control daemon
- **LOUI** — Java/Eclipse RCP observer UI
- **JOE** — Java Operation Executive (drives OMS motors, publishes telemetry to broker)

### LIG Contacts
- **Timothy Ellsworth-Bowers (Tim)** — tbowers@lowell.edu (user's primary LDT contact)
- Ryan Hamilton — rhamilton@lowell.edu (LDT software; MCB/command path questions)
- Georgi Mandushev — gmand@lowell.edu
- Dyer Lytle — dlytle@lowell.edu (WebRat login)
- Stephen Levine — sel@lowell.edu

---

## File Storage Convention

**FITS path (archival):**
```
/lois/data/dct/<InstrumentID>/YYYYMMDD/<abbr>.NNNN.fits
```
- WILDS needs agreed instrument ID/abbreviation (e.g., `WILDS`/`wlds`) from LIG
- UT rollover at local noon (UT−5h = 19:00 UT) — "noon rule"

---

## Hardware Design (in progress)

### Detectors / Controllers
- Two STA4850 CCDs + two STA Archon controllers (one VIS, one UV)
- **Exposure sync:** Two Archons may be daisy-chained — possibly no external trigger needed at all. Starting exposure on one Archon via software may automatically trigger the second through the daisy-chain. Confirm with STA; if true, eliminates GPIO hardware requirement entirely.
- **Separate exposure durations per detector** — each Archon runs its own countdown; orchestrator fires trigger then waits on both readouts via asyncio.gather()
- Key open question with STA: does TriggerIn start integration only, or gate the full exposure?

### Motorized Stages (confirmed by user 2026-10-13)
- **ADC:** 2 stages on a SmarAct MCS2. Driven autonomously by TCS telemetry (parallactic angle + elevation)
- **Slit:** a stage sets the slit **width**; the spectrograph (science) cameras see the light that passes through the slit. **SmarAct** stage (confirmed by user 2026-10-25)
- **Calibration lamps:** **2 lamps**, each switched on/off **and** moved In/Out of the optical path by its own stage. Stage controller and power switching **TBD**. Lamp types not yet known
- **Shutter:** single, driven by the Archon controller
- **Three SmarAct stages in total: slit + 2 ADC**, all on **one MCS2-C-0006** — tabletop housing, 3 channels, **Ethernet** interface (decided by user 2026-10-25). Channels: **0 = slit, 1 = ADC 1, 2 = ADC 2** (so ADC channel = `AdcIndex`)
- **Stage part numbers** (user, 2026-10-25):
  - Slit (ch 0): SmarAct `EX-0045566-000-CLS3232-4th` — CLS-3232 linear stage. CLS is newer than the 2018 MCS2 positioner-type list, so its MCS2 type code isn't in that list; `-4th` suffix not yet understood
  - ADC 1 & 2 (ch 1, 2): SmarAct `EX-0018664-003-SRM120-S` — SR-120-class rotary stage with S-sensor. Per the 2018 MCS2 Positioner Types list, SR120 maps to MCS2 type code **320** (`SR...T1S8S`, large rotary, single reference mark) — inferred, confirm against the controller's configured type
  - `EX-…` prefixes look like SmarAct custom/order numbers
- **Control path: SCPI over TCP from Linux** — no SmarAct SDK on the instrument computer. Ethernet-only model means the ASCII interface is available; the USB variant (C-0002) would have required the native SDK. Pulling the network cable stops all positioners, so use a private/direct wired link; change the default IP before deployment
- Software: SmarAct's SDK (SmarActCTL) is a native library (Windows DLL / Linux .so, no macOS) plus a generated CFFI Python file; it comes only from SmarAct with the controller, under their EULA — not on PyPI. MCS2 units with an Ethernet port also speak **SCPI over raw TCP, port 55551** (`<CR><LF>` termination, default IP 192.168.1.200; MCS2 Programmer's Guide ch. 6), which needs no vendor library

### Guide Camera
- **ZWO ASI294-class** camera. Sees the sky surrounding the slit, reflected by a **static mirror** — i.e. a slit-viewing camera used for acquisition and guiding
- Open item: the slit's reference position (pixel) on the guide-camera image, needed to put targets on the slit
- `wilds-solving` plate-solves its frames (scale hints assume ASI294MM, 4.63 µm pixels)

## Proposed Software Architecture

Three main components, all running on the instrument computer:

1. **Archon Actor** (sdss-clu/AMQP → RabbitMQ) — CCD control (expose, read, flush, status, etc.)
2. **LDT Bridge** (stomp.py/ligmos → ActiveMQ STOMP:61613) — subscribe to telemetry topics, publish wilds.loisTelemetry, send TCS commands
3. **WILDS GUI** (Django + Django Channels, ASGI) — TO-facing web interface

FITS headers assembled from tcs/aos/instrumentCube/wrs telemetry topics + Archon internal state.

### Singleton Control Objects
- **Telescope** — wraps TCS commands (offsets, focus, etc.) via LDT Bridge
- **Focuser** — wraps focus commands
- **ADC** — 2 channels of the shared MCS2-C-0006; manual control initially, auto mode using `currentParAngle` from TCS telemetry when confidence is established
- **SlitStage** — slit width stage, 1 channel of the shared MCS2-C-0006 (one SCPI connection shared with ADC)
- **CalibrationLamps (×2)** — power on/off + In/Out stage per lamp (controller TBD)
- **ScienceCamera (×2)** — Archon VIS + UV CCD controllers
- **GuideCamera** — ZWO ASI294-class slit-viewing camera

### GUI Architecture
`wilds-gui`, probably Django — **details still to be discussed with the user**.
Current design (`wilds-control/docs/architecture.md`): Django (:8000) serves the page; control
(POST /acquire /slit /lamps /adc /focus /solve), telemetry (SSE) and guide-camera video (WS, JPEG
frames) go to `wilds-control` (FastAPI/Hypercorn :8001). This supersedes an earlier idea of
Django + Channels owning the process and calling the hardware singletons directly.

**GUI features (TO-facing):**
- Guide camera canvas: start/stop streaming, acquisition onto the slit, focus checks
- Stage controls: slit width, ADC rotation (manual override)
- Calibration lamps: on/off and In/Out for each of the 2 lamps
- ADC auto mode: drives stages from `currentParAngle` telemetry once validated
- Component status panel: TCS state, health, Az/El, PA, LST; camera status; stage positions
- Science exposure controls: parameters (exptime, etc.), start/stop

---

## Confluence Resources
- **Overall software interfaces overview:** https://confluence.lowell.edu/display/LIGS/DCT+Science+Instrument+Software+Interface
- **TCS Simulator guide (Operator Training):** https://confluence.lowell.edu/display/DCTDOC/Operator+Training — useful for testing against simulator and on-the-hill broker; ignore "Other LDT Applications" section (operator training only)
- **LIG Software Products + Interface Documents:** https://confluence.lowell.edu/display/LIGS/Lowell+Instrument+Group+Software+Products — includes Interface Documents section describing TCS and AOS (active optics) packet formats (send/receive)
- deveny.loisTelemetry schema: https://confluence.lowell.edu/pages/viewpage.action?pageId=44597486
- tcs.loisTelemetry: https://confluence.lowell.edu/pages/viewpage.action?pageId=44597457

## External Repos
- https://github.com/sdss/archon — Archon actor (this codebase)
- https://github.com/LowellObservatory/ligmos — LDT Python messaging + XSD schemas
- https://github.com/LowellObservatory/LDTObserverTools — LDT observer tools

---

## Instrument Computer
Recommended: Supermicro SYS-E300-13AD (mini-1U, Intel 13th Gen, up to 64 GB DDR4, IPMI).
Fanless option: SYS-E302-12D-8C (Xeon D, 8 cores, passive cooling — preferred for observatory).
