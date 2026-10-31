# LDT simulator test log

Sessions against the LDT TCS simulator and the Mars Hill test broker. Newest first.
Credentials are never recorded here — they live in `WILDS/.wilds-vault/`.

## Setup reference

| What | Where |
|---|---|
| TCS + MCU simulators, MCB | `nysa.lowell.local` (10.10.30.18) — Windows; VNC :5900, RDP :3389 |
| AOS, OCS, WS simulators | `lo-dctsim.lowell.local` (not yet used) |
| Test broker | `tanagra.lowell.local` (10.10.130.138) — STOMP :61613, OpenWire :61616, admin :8161 |
| Network | Lowell VPN required (`*.lowell.local` resolves only on the VPN) |

**Start order on nysa** (Georgi; Confluence "Operator Training"): MCU simulator → TCS →
**Startup** → **Main Display**; then Data Listener Manager, MCB Controller and MCB View, and
**Start/Enable** in the MCB window. Never change the MCB's broker host. The simulator is
shared with LIG's software group — book time with Georgi (subject "Operator training").

**Pointing the telescope:** Mount Control on → Target Config → Science Target → Sidereal
target (name, FK5, RA/Dec) → *Send to preview* → *Go to Target* → Tracking on. The MCU
simulator applies demands instantly (no slewing).

**Screenshots without taking control:** `vncdotool` (run via `uv run --with vncdotool`,
not installed in the venv) connects as a *shared* viewer and only captures the screen.
The VNC password goes in through an environment variable, never on the command line.

## 2026-10-31 — first end-to-end session

**Goal:** read live TCS telemetry with the `wilds-bridge` models; try sending a target.

**Telemetry (TCS → broker): works.**
- Once the MCB was started (it was open but not started — nothing reached tanagra
  before that), `TCSTcsStatusSV` arrived at ~1 Hz plus the `TCSLowLevelStatusSV.*`
  scalars (CurrentAz/El, ParAngle, ZD, time-to-limits, InPosition*) and `TCSIersStatusSV`.
- Every `TCSTcsStatusSV` message parsed with the reworked `TcsStatus` model, parked and
  tracking. A tracking sample is now a test fixture (wilds-bridge#4).
- Not received: `tcs.loisTelemetry` (JOE only — not in the simulator), `AOSDataPacket`,
  `WRSDataPacket` (simulators on lo-dctsim not running / not bridged).

**Target (broker → TCS): did not reach the TCS.**
- `Telescope.slew_to("WILDS-test", 6.0, 20.0)` published a `newScienceTarget` to
  `TCS.TCSSharedVariables.TCSLowLevelStatusSV.NewScienceTargetSV`. It appeared on the
  broker; the TCS demand did not change and no command response came back.
- The MCB did **not** fault (unlike the April test on `TCSTcsCommandSV`).
- Cause (probable): in the MCB's default configuration (Confluence "MCB Configuration")
  `NewScienceTargetSV` is in the *Shared Variables Consumer* list, i.e. TCS → broker only.
  Broker → TCS topics are the AOS focus topics and `TCS.TCSSharedVariables.TCSSubData.*`.
  The v78 interface doc nonetheless lists `NewScienceTargetSV` as a command topic.
- MCB *Tools → Edit Configuration* is greyed out while the MCB runs; we did not edit it.

**Tracking (GUI target RA 06:05, Dec +20, frame Target, PA 0):**
- Az 181–184°, El 75.3°, airmass 1.03, guide mode `OpenLoop`; all limit indicators green.
- Az and El in position; **rotator never in position** (error 0.0006–0.002°, just above
  tolerance), so overall `inPositionIsTrue` stayed false.
- Rotator semantics, from the data: `currentRotatorPositions.rotPA` is the *mechanical*
  angle (moved −2.0° while the parallactic angle moved +2.0°); `objectPA.EqPA` is the actual
  sky PA of the slit (stayed 0); `rotIPA` is the demanded sky PA; IAA = 185.2.
- `tcsState` read `FAULT` for a while after Mount Control was enabled with the default
  "unspecified" target at the horizon; it returned to `ENABLED` once tracking a valid target.

**Instrument definitions** (TCS Command → Instrument Name):
- LMI (selected at start): IAA 185.2°.
- DeVeny: focal length 25 715.4 mm (8.02″/mm), pointing origin (8.5, −0.9) mm, rotI 1,
  IAA 6.0°. Selecting it rotated the rotator from ~351° to 155.26° (the IAA change plus the
  parallactic-angle drift) while the sky PA stayed 0.00.

**Environment notes:**
- TCS health was red throughout: "Humidity (97 %) exceeds maximum (85 %)". The simulator
  uses real LDT weather; the WS simulator on lo-dctsim can override it.
- Default rotator wrap was out of limits (rotator at ~359° vs 89.9°–270.1° on the MCU
  simulator) until a real target was tracked.
- The M1 cover can be opened from the TCS (M1Cover panel) and reports `Open`.
- GitHub over the Lowell VPN is unreliable (TLS interception).

**Open questions for LIG (Georgi / Tim / Ryan):**
1. How do CAT / the Observer Target List send a science target to the TCS — which topic
   and format? Is `NewScienceTargetSV` meant to be broker → TCS on nysa's MCB?
2. Is the rotator never reaching in-position while tracking a simulator artefact, or
   should instruments not wait on overall `inPositionIsTrue`?
3. Does the outer `demandRADec` accept decimal values (`6.0`, `20.0`) or must they be
   integers, as in the documented example?
4. Do `pointingOriginX/Y` in a target command override the selected instrument's pointing
   origin? (We send 0.0, as the documented example does.)
5. What is the `rotI` parameter in an instrument definition (DeVeny: 1)?
6. What does LIG need from us to create a WILDS instrument entry (focal length, pointing
   origin, IAA), and how is the IAA measured for a new instrument?
