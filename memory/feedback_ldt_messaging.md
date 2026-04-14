---
name: LDT Messaging — Use pydantic-xml models
description: All LDT broker message serialization/deserialization must use pydantic models with XML serialization (pydantic-xml), not raw xmltodict or xmlschema
type: feedback
---

Use **pydantic-xml** (v2.19+) for all LDT ActiveMQ message models. Define each broker topic as a Pydantic model class with XML serialization.

**Why:** User's explicit preference. Pydantic-xml gives typed models, validation, and clean `model.to_xml()` / `Model.from_xml(bytes)` round-tripping — preferred over raw xmltodict or xmlschema.

**How to apply:** Whenever writing code that parses or produces LDT broker messages (tcs.loisTelemetry, TCS.TCSSharedVariables.*, wilds.loisTelemetry, command topics, etc.), define the message structure as a pydantic-xml model class, not a dict or dataclass. This applies to both inbound (subscribe/parse) and outbound (publish/serialize) messages.
