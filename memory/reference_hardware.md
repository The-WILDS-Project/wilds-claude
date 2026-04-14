---
name: WILDS Hardware Options
description: Hardware options and decisions for WILDS instrument computer and peripherals
type: reference
---

## TriggerIn GPIO — FT232H

For generating the Archon TriggerIn synchronization pulse from the Supermicro E302-12D-8C:

- **Device:** FTDI FT232H breakout (e.g. Adafruit FT232H)
- **Why:** E302 has no exposed GPIO pins. FT232H is USB, cheap (~$15), works on Linux without drivers, Python `pyftdi` library gives sub-millisecond GPIO toggling
- **Use:** Single USB cable + breakout board; TTL output splits to both Archon TriggerIn ports via BNC/TTL splitter
- **Alternative:** Measurement Computing USB-DIO96H (more I/O lines), or serial port RTS line (crude, ~1-2ms jitter, no extra hardware)
