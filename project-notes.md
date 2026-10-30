# WILDS Project Notes

**WILDS** — Weizmann Institute Lowell Discovery Spectrograph  
A new optical spectrograph being built at the Weizmann Institute for use at the Lowell Discovery Telescope (LDT), Anderson Mesa, Arizona.

WILDS is intended to replace the DeVeny spectrograph at LDT.

---

## Hardware

- **Detector:** STA4850 — 4080×4080 pixels, 15 µm pixel size, full-frame CCD
- **Controller:** STA Archon CCD controller
- **Telescope:** Lowell Discovery Telescope (LDT), 4.3m, Anderson Mesa, AZ

### Getting the ACF file
The Archon controller requires an ACF (configuration) file specific to the STA4850 detector.
Options to obtain:
- Request from STA directly (should be supplied with hardware purchase): http://www.sta-inc.net/archon/
- Contact MAROON-X team at Gemini/U. Chicago (Jacob Bean, jbean@uchicago.edu) — they use identical STA4850 + Archon hardware
- Contact Lowell LIG (Ryan Hamilton, rhamilton@lowell.edu)

---

## Software: Archon Controller

The codebase in `C:\Users\User\WILDS\archon` is `sdss-archon`, a Python actor library for
communicating with STA Archon controllers, developed by SDSS. It is the basis for WILDS
controller software.

- **Repo:** https://github.com/sdss/archon
- **PyPI:** `sdss-archon`
- **Docs:** https://sdss-archon.readthedocs.io
- **Install (editable):** `pip install -e .` or `uv pip install sdss-archon`
- **Python:** 3.10+
- **Build backend:** hatchling

### Key dependencies
| Package | Purpose |
|---------|---------|
| `sdss-clu` ≥2.5.3 | Actor/messaging framework (RabbitMQ/AMQP) |
| `numpy` ≥2.0 | Array handling |
| `astropy` ≥6.0 | FITS, coordinates |
| `click-default-group` | CLI command parsing |

### Actor commands
| Command | Description |
|---------|-------------|
| `init [ACF-FILE]` | Initialise controller, load ACF config |
| `status` | Report controller status |
| `system` | Report backplane status |
| `power [on\|off]` | Power CCD array |
| `flush [COUNT]` | Flush CCD |
| `reset` | Reset controller, discard exposures |
| `reconnect` | Restart socket connection |
| `disconnect` | Disconnect controller |
| `recover [PATH]` | Recover a failed exposure |
| `talk COMMAND` | Send raw command to controller |
| `expose [EXPTIME]` | Take exposure (see options below) |
| `read` | Finish ongoing exposure early |
| `abort [--flush]` | Abort current exposure |
| `wait-until-idle` | Block until controller is IDLE |
| `get-window` | Show current readout window |
| `set-window` | Set readout window (lines, pixels, binning, overscan) |
| `frame status CTRL` | Read frame buffer status |
| `frame fetch CTRL` | Fetch buffer and write to disk |
| `config read CTRL` | Read ACF from controller |
| `config write CTRL FILE` | Write ACF file to controller |

#### `expose` options
```
expose [EXPTIME] [--bias|--dark|--flat|--arc|--object]
       [--window-mode MODE]
       [--readout/--no-readout] [--async-readout]
       [--header JSON]
       [-d DELAY_READOUT]
       [-n COUNT]
       [--no-write] [--no-shutter] [--with-dark]
       [-s SEQNO]
```

### sdss-clu actor framework
`sdss-clu` is an asyncio-based actor framework using **RabbitMQ** (AMQP 0-9-1) as its message broker.
- Actors inherit from `AMQPActor`
- Commands are Click functions decorated with `@parser.command()`
- Replies: `command.finish(key=value)`, `command.fail(error="...")`, `command.info(...)`
- Client: `AMQPClient` for scripting/inter-actor messaging

---

## LDT Infrastructure

### Message Broker
LDT uses **Apache ActiveMQ Classic 5.x** as its message broker (server at telescope site — hostname provided by LIG on request).

Authentication is currently **not required**.

Supported protocols:
| Protocol | Port | Notes |
|----------|------|-------|
| OpenWire | 61616 | Java/legacy |
| **STOMP** | **61613** | **Use this for Python** |
| AMQP | 5672 | |
| WebSockets | 61614 | |
| MQTT | 1883 | |

A **test broker + telescope simulator** is available from LIG for development. Contact Ryan Hamilton.

### Python integration stack (new LIG standard)
```
stomp.py >= 4.1.21      # STOMP protocol client
xmltodict >= 0.12.0     # XML ↔ dict
xmlschema >= 1.1.0      # XSD validation
ligmos                  # LIG helper library (connection, heartbeat, reconnect)
```

**ligmos repo:** https://github.com/LowellObservatory/ligmos  
**XML schemas:** https://github.com/LowellObservatory/ligmos/tree/master/ligmos/schemas

Recommended Python XML serialization: **pydantic-xml** (v2.19+)
- Define message models as Pydantic classes
- `model.to_xml()` / `Model.from_xml(bytes)`
- https://pydantic-xml.readthedocs.io/

### LIG Contacts
| Name | Email | Role |
|------|-------|------|
| Ryan Hamilton | rhamilton@lowell.edu | Primary contact |
| Georgi Mandushev | gmand@lowell.edu | |
| Dyer Lytle | dlytle@lowell.edu | |
| Stephen Levine | sel@lowell.edu | |

### LOIS / LOUI / JOE
- **LOIS** — Lowell Observatory Instrumentation System (C/TCL, instrument control daemon)
- **LOUI** — Lowell Observatory User Interface (Java/Eclipse RCP, observer UI)
- **JOE** — Java Operation Executive (drives OMS motor controllers, publishes telemetry)
- Communication: all via ActiveMQ topics, fully decoupled

---

## ActiveMQ Topics

### Status topics (subscribe to receive)

#### `deveny.loisTelemetry` — instrument state (~1 Hz, published by JOE)
XSD: https://github.com/LowellObservatory/ligmos/blob/master/ligmos/schemas/deveny.loisTelemetry.xsd

| Field | Type | Notes |
|-------|------|-------|
| `TimeStamp` | long | Unix epoch (seconds) |
| `Lamp1Name`–`Lamp4Name` | string | Cd, Ar, Ne, Hg |
| `Lamp1State`–`Lamp4State` | boolean | on/off |
| `SlitWidthMM` | decimal | 0.075–15 mm |
| `SlitWidthASEC` | decimal | arcseconds |
| `GratingTilt` | decimal | 0–47° |
| `CollimatorFocus` | decimal | 7.75–19 mm |
| `Grating` | string | from MasterInstrumentConfiguration.xml |
| `RearFilter` | string | from MasterInstrumentConfiguration.xml |

WILDS equivalent: **`wilds.loisTelemetry`** — schema to be agreed with LIG.

#### `tcs.loisTelemetry` — telescope state (~1 Hz, for FITS headers)
XSD: https://github.com/LowellObservatory/ligmos/blob/master/ligmos/schemas/tcs.loisTelemetry.xsd

Key fields: `DemandRa`, `DemandDec`, `CurrentHourAngle`, `TCSCurrentElev`, `TCSCurrentAzimuth`,
`TCSCurrentRotatorPA`, `TCSCurrentRotatorIAA`, `TCSCurrentRotatorIPA`, `RotatorFrame`,
`TCSState`, `TCSHealth`, `TCSHeartBeat`, `InPosition`, `ScienceTargetName`, `MountTemperature`,
`m1CoverState`, `DomeOccultationWarning`, `CurrentParAngle`, `equinox`, `TCSLST`, `TCSUTC`

#### `TCS.TCSSharedVariables.TCSHighLevelStatusSV.TCSTcsStatusSV` — higher cadence TCS
Includes offset and differential rate info not in `tcs.loisTelemetry`. Use for ~1 Hz cadence
or when offset/rate data needed.

#### Other status topics
| Topic | Content |
|-------|---------|
| `aos.loisTelemetry` | Active optics |
| `instrumentCube.loisTelemetry` | Instrument cube state |
| `wrs.loisTelemetry` | Weather/environmental |
| `RC[1,2].loisTelemetry` | GWAVES probe telemetry |
| `lmi.loisTelemetry` | LMI camera telemetry |
| `LOUI.<machine>.loisTelemetry.@camera_struct` | CCD/exposure state from LOIS |

#### `LOUI.<machine>.loisTelemetry.@camera_struct` — CCD state
Key fields: `exp-time`, `num-exp`, `exp-num`, `row`, `col`, `row-bin`, `col-bin`,
`x1`, `xsize`, `y1`, `ysize`, `camtemp`, `coldhead`, `settemp`, `gain`, `speed`, `amp`,
`start-time`, `end-time`, `ut-date`, `lst-obs`, `title`, `object`, `comment1`, `comment2`

### Command topics (publish to send)

#### `TCS.TCSSharedVariables.TCSSubData.TCSTcsCommandSV` — telescope offset commands
Reply on: `TCS.TCSSharedVariables.TCSLowLevelStatusSV.TCSTcsCommandResponseSV`

```xml
<TCSTcsCommand>
  <commandID>59909236</commandID>   <!-- microseconds since midnight UT -->
  <newTarget>
    <tcsId>0</tcsId>
    <isNewTarget>false</isNewTarget>
  </newTarget>
  <offset>
    <offsetDef>User</offsetDef>     <!-- User | Handset -->
    <off1>-8.133</off1>             <!-- arcsec (RA or Az component) -->
    <off2>12.0</off2>               <!-- arcsec (Dec or El component) -->
    <offsetType>TPLANE</offsetType> <!-- TPLANE | SIMPLE -->
  </offset>
</TCSTcsCommand>
```

`commandID` construction:
```python
import datetime
now = datetime.datetime.utcnow()
commandID = round((now.hour*3600 + now.minute*60 + now.second + now.microsecond/1e6) * 1e6)
```

### Topic naming (LOUI/LOIS)
LOIS publishes/subscribes using machine-name-scoped topics:
```
LOUI.<machineName>.<function>
```
e.g. with machine `deveny.lowell.edu`:
- `LOUI.deveny.lowell.edu.loisCommand` — send commands to LOIS (synchronous TCL queue)
- `LOUI.deveny.lowell.edu.loisCommandResult` — command completion status
- `LOUI.deveny.lowell.edu.loisImage` — image available notification
- `LOUI.deveny.lowell.edu.loisLog` — log messages
- `LOUI.deveny.lowell.edu.loisTelemetry` — general LOIS telemetry

---

## DeVeny Hardware Details (reference for WILDS design)

### Motorized stages (driven by JOE via OMS cards)
OMS card hostname: `devenymot1.lowell.edu:4321`

| Axis | Function | Range | Resolution |
|------|----------|-------|-----------|
| DGT | Grating tilt | 0–47° | <0.001° (encoder) |
| DSW | Slit width | 0.49–85.62 steps (0.075–15 mm) | 0.08 µm, 10 µm backlash |
| DCF | Collimator focus | 7.75–18.25 mm | encoder |

### Arc lamps
PDU at `devenypdu.lowell.edu:23` (telnet). Lamps: Cd, Ar, Ne, Hg.
Controlled via JOE → ActiveMQ → PDU socket.

### Grating
Changed manually by telescope operator (not motorized).

---

## DeVeny Acquisition Flow (reference)

```
Observer (LOUI) → loisCommand topic
                → JOE (stage commands via OMS TCP)
                → LOIS (exposure via DSP/PCI)
                       → FITS file on disk
                       → loisCommandResult
                       → loisImage (notification)
```

FITS headers assembled from 7 telemetry streams at exposure time:
`deveny.loisTelemetry`, `tcs.loisTelemetry`, `aos.loisTelemetry`,
`RC1/RC2.loisTelemetry`, `instrumentCube.loisTelemetry`, `wrs.loisTelemetry`,
plus LOIS internal `camera_struct`.

Header version tracked in FITS keyword `KWDICT` (DeVeny current: `Deveny1.2/4`).

---

## File Storage Convention

**FITS files** written by LOIS to local disk on instrument host, then copied to archive.

### Path format (archival mode)
```
/lois/data/dct/<InstrumentID>/YYYYMMDD/<abbr>.NNNN.fits
```

DeVeny examples:
```
/lois/data/dct/DeVeny/20131124/dvny.11839.fits
/lois/data/dct/DeVeny/20131212/dvny.01_0232.fits   (subframe 1)
/lois/data/dct/DeVeny/20131212/dvny.02_0232.fits   (subframe 2)
```

Test exposures → `test.fits` in current store directory (not archived).

### UT date rollover — "noon rule"
Rundate rolls over at **local noon** (UT−5h at Lowell = 19:00 UT).  
A full night's data (e.g., sunset ~01:00 UT to dawn ~13:00 UT) stays under one rundate.
`store.cfg` setting: `rollover=-5`

### WILDS will need
- Agreed instrument ID and abbreviation (e.g., `WILDS` / `wlds`) with LIG
- Noon-rule UT rollover implemented
- Archive copy daemon equivalent to `loiscopy`

---

## WILDS Software Architecture (proposed)

```
┌─────────────────────────────────────────────────────────┐
│                  WILDS Orchestration                    │
│  (Python, asyncio)                                      │
│                                                         │
│  ┌──────────────────┐   ┌───────────────────────────┐  │
│  │  Archon Actor    │   │   LDT Bridge              │  │
│  │  (sdss-clu/AMQP) │   │   (stomp.py / ligmos)     │  │
│  │  → RabbitMQ      │   │   → ActiveMQ (STOMP:61613)│  │
│  └────────┬─────────┘   └─────────┬─────────────────┘  │
│           │                       │                     │
│     Archon commands         Subscribe:                  │
│     expose, read,           tcs.loisTelemetry           │
│     flush, status, etc.     aos.loisTelemetry           │
│                             instrumentCube.loisTelemetry│
│                             wrs.loisTelemetry           │
│                       Publish:                          │
│                             wilds.loisTelemetry         │
│                       Send commands:                    │
│                             TCS offset commands         │
└──────────────┬──────────────────────────────────────────┘
               │
    ┌──────────▼──────────┐
    │  STA Archon          │
    │  Controller          │
    │  (TCP/Ethernet)      │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │  STA4850 CCD         │
    └──────────────────────┘
```

### Integration notes
- **RabbitMQ ↔ ActiveMQ:** Use STOMP (port 61613) directly from Python — no bridge needed.
  sdss-clu uses AMQP/RabbitMQ internally; the LDT bridge is a separate async process using `stomp.py`.
- **Stage control:** DeVeny stages driven by JOE via OMS cards. WILDS needs own mechanism
  or agreement with LIG on who drives WILDS mechanisms. WILDS stages: 3 SmarAct (slit width + 2 ADC) on one MCS2-C-0006 (3-ch, Ethernet, SCPI) —
  and an In/Out stage for each of the 2 calibration lamps (lamp stage controller TBD).
- **FITS headers:** Subscribe to `tcs.loisTelemetry` and other topics; inject keywords at
  exposure start via `expose --header` JSON option (or extend the actor).
- **`wilds.loisTelemetry`:** New topic schema to be agreed with LIG. Should include
  instrument state analogous to `deveny.loisTelemetry`.

---

## Confluence Resources

Lowell LIG Confluence: https://confluence.lowell.edu/display/LIGS/Lowell+Instrument+Group+Software+Products

Key pages:
- [DCT Science Instrument Software Interface](https://confluence.lowell.edu/display/LIGS/DCT+Science+Instrument+Software+Interface) — **start here**
- [deveny.loisTelemetry schema](https://confluence.lowell.edu/pages/viewpage.action?pageId=44597486)
- [tcs.loisTelemetry](https://confluence.lowell.edu/pages/viewpage.action?pageId=44597457)
- [TCS command topic](https://confluence.lowell.edu/pages/viewpage.action?pageId=44597608)
- [LOIS File Storage and Naming](https://confluence.lowell.edu/pages/viewpage.action?pageId=11010075)
- [DeVeny FITS Header Versions](https://confluence.lowell.edu/pages/viewpage.action?pageId=22610268)
- [LOUI DeVeny](https://confluence.lowell.edu/pages/viewpage.action?pageId=56557699)
- [Messaging Architecture](https://confluence.lowell.edu/display/LIGS/) (DCTIC space)

---

## External Repositories

| Repo | Purpose |
|------|---------|
| https://github.com/sdss/archon | Archon actor (this codebase) |
| https://github.com/LowellObservatory/ligmos | LDT Python messaging helpers + XSD schemas |
| https://github.com/LowellObservatory/LDTObserverTools | LDT observer tools |

---

## Instrument Computer

Recommended: **Supermicro SYS-E300-13AD** (fanless alternative: SYS-E302-12D-8C)
- Mini-1U, 10"×1.7"×8.9"
- Intel 13th Gen Core (up to i9, 16 cores)
- Up to 64 GB DDR4, 2× 2.5" drives
- Dedicated IPMI LAN port (out-of-band management, iKVM, web interface)
- 120W AC-DC adapter

Fanless option (SYS-E302-12D-8C): Intel Xeon D, 8 cores, passive cooling —
preferable for dusty/harsh observatory environment.
