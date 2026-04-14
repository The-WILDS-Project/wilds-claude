---
name: WILDS Project Context
description: This workspace is for WILDS — Weizmann Institute Lowell Discovery Spectrograph — built on top of the Archon actor framework
type: project
---

This project is named **WILDS** (Weizmann Institute Lowell Discovery Spectrograph).

The codebase is the `archon` Python actor framework (originally from SDSS), being used/adapted as the controller software for the WILDS spectrograph instrument. It is intended to replace the DeVeny spectrograph at LDT.

**Hardware:**
- Detector: STA4850 — 4080×4080 pixels, 15 µm pixel size, full-frame CCD
- Controller: STA Archon CCD controller
- Telescope: Lowell Discovery Telescope (LDT), 4.3m, Anderson Mesa, AZ

**Software base:** `sdss-archon` Python actor library (https://github.com/sdss/archon), editable install via `pip install -e .`

**Key dependencies:** `sdss-clu` ≥2.5.3 (AMQP/RabbitMQ actor framework), `numpy` ≥2.0, `astropy` ≥6.0

**Why:** New instrument project at Weizmann Institute, deployed at Lowell Discovery Telescope. The Archon library handles communication with STA Archon CCD controllers.

**How to apply:** When discussing the project, refer to it as WILDS. Suggestions and changes should be framed in the context of a spectrograph instrument control system integrating with LDT infrastructure. The archon codebase is being adapted (not used generically) for the WILDS Archon controller component.

---

## LDT Infrastructure

- **Message broker:** Apache ActiveMQ Classic 5.x at telescope site
  - STOMP on port 61613 (use this from Python)
  - AMQP on 5672, OpenWire on 61616
  - Auth not currently required
- **Python stack:** `stomp.py` ≥4.1.21, `xmltodict`, `xmlschema`, `ligmos` (https://github.com/LowellObservatory/ligmos)
- **XML serialization:** `pydantic-xml` v2.19+

### Broker Hosts
- **Real LDT broker:** hostname `joe`, STOMP port 61613 (no auth required)
- **Test broker (WebRat):** `lo-webrat.lowell.edu` / `webrat.lowell.edu` (IP: 10.10.130.155)
  - Ubuntu 20.04 VM; login: `dlytle` + Lowell password
  - Runs Apache ActiveMQ **Artemis** (NOT Classic 5.x) in `/opt/artemis`; broker instance `test-broker` in `/var/lib`; credentials `admin`/`admin`
  - Also runs Redis (compiled from source, open to all connections)
  - Telescope simulator available on request from LIG (Ryan Hamilton)

### Key ActiveMQ Topics

**Subscribe (status):**

| Topic | Rate | Source | Notes |
|-------|------|--------|-------|
| `tcs.loisTelemetry` | ~2 Hz | JOE / FacilitySummary.java | Strings/sexagesimal; easiest to use. BUG: sends each message twice ~1/min |
| `TCS.TCSSharedVariables.TCSHighLevelStatusSV.TCSTcsStatusSV` | ~1 Hz | LabVIEW shared vars | Structured XML; has offset/rate info not in loisTelemetry. Schema: v1.8 |
| `deveny.loisTelemetry` / `wilds.loisTelemetry` | ~1 Hz | JOE | Instrument state (WILDS schema TBD with LIG) |
| `aos.loisTelemetry` | ~1 Hz | JOE | Active optics |
| `instrumentCube.loisTelemetry` | ~1 Hz | JOE | Instrument cube |
| `wrs.loisTelemetry` | ~1 Hz | JOE / WRSTelemetry.java | Weather/environment |

**tcs.loisTelemetry key fields:** Timestamp (Unix s), TCSLST, TCSUTC, DemandRa, DemandDec, CurrentHourAngle, TCSCurrentAzimuth, TCSCurrentElev, TCSCurrentZenithDistance, TCSCurrentRotatorPA, TCSCurrentRotatorIPA, TCSCurrentRotatorIAA, RotatorFrame, TargetFrame, equinox, TCSState, TCSHealth, TCSHeartBeat, TCSAccessMode, InPosition, ScienceTargetName, MountTemperature, m1CoverState, MountDomeAzimuthDifference, DomeOccultationWarning, CurrentParAngle, CLSLowBankState, DSSPositionStatus

**Publish (commands):**
- `TCS.TCSSharedVariables.TCSSubData.TCSTcsCommandSV` — telescope offset commands
  - Reply on: `TCS.TCSSharedVariables.TCSLowLevelStatusSV.TCSTcsCommandResponseSV`

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
- Ryan Hamilton — rhamilton@lowell.edu (primary contact)
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
- **Exposure sync:** Investigating TriggerIn hardware signal from Supermicro GPIO to both Archon TriggerIn ports simultaneously — hardware sync preferred over software timing
- **Separate exposure durations per detector** — each Archon runs its own countdown; orchestrator fires trigger then waits on both readouts via asyncio.gather()
- Key open question with STA: does TriggerIn start integration only, or gate the full exposure?

### Motorized Stages
- Slit linear stage, 2× ADC linear stages, 1+ shutters
- Investigating single-vendor/single-controller solution for all stages (TBD)
- ADC stages driven autonomously by TCS telemetry (parallactic angle + elevation)
- Shutter timing must be coordinated with TriggerIn pulse; may be dedicated shutter driver (e.g. Uniblitz) rather than on stage controller

### Guide Camera
- TBD

## Proposed Software Architecture

Two main Python async components:
1. **Archon Actor** (sdss-clu/AMQP → RabbitMQ) — CCD control (expose, read, flush, status, etc.)
2. **LDT Bridge** (stomp.py/ligmos → ActiveMQ STOMP:61613) — subscribe to telemetry topics, publish wilds.loisTelemetry, send TCS commands

FITS headers assembled from tcs/aos/instrumentCube/wrs telemetry topics + Archon internal state.

---

## Confluence Resources
- Start here: https://confluence.lowell.edu/display/LIGS/DCT+Science+Instrument+Software+Interface
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
