---
name: WILDS Hardware Options
description: Hardware options and decisions for WILDS instrument computer and peripherals
type: reference
originSessionId: 40881ff9-65ee-4a5c-9a0e-41dd681b2bb2
---
## TriggerIn GPIO — FT232H

**Status: may not be needed.** The two Archons may be daisy-chained so starting exposure on one triggers the other — confirm with STA before investing in GPIO hardware.

If external trigger IS needed:
- **Device:** FTDI FT232H breakout (e.g. Adafruit FT232H)
- **Why:** E302 has no exposed GPIO pins. FT232H is USB, cheap (~$15), works on Linux without drivers, Python `pyftdi` library gives sub-millisecond GPIO toggling
- **Use:** Single USB cable + breakout board; TTL output to first Archon TriggerIn (daisy-chain handles second)
- **Alternative:** Measurement Computing USB-DIO96H (more I/O lines), or serial port RTS line (crude, ~1-2ms jitter, no extra hardware)
