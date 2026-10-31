# Memory Index

- [WILDS Project Context](project_wilds.md) — Weizmann Institute Lowell Discovery Spectrograph; Archon codebase adapted for WILDS; LDT infrastructure, ActiveMQ topics, contacts, file storage, architecture
- [LDT Confluence access](reference_confluence.md) — needs Lowell VPN; token in WILDS/.wilds-vault (strip CRLF); REST recipe; key page IDs
- [LDT Messaging Style](feedback_ldt_messaging.md) — All LDT broker messages use pydantic-xml models, not raw xmltodict/xmlschema
- [Hardware Options](reference_hardware.md) — FT232H USB GPIO for Archon TriggerIn; E302 has no native GPIO
- [Python Style](feedback_python_style.md) — PEP 604 unions: `X | None` not `Optional[X]`, no `Union[]`, no `from __future__ import annotations`
- [User Signals](feedback_signals.md) — `???` means "your thoughts/suggestions please"
- [Memory Location](feedback_memory_location.md) — Memory, plans and notes go in this wilds-claude repo, not ~/.claude
- [Dev Environment](project_dev_environment.md) — wilds-workspace repo (workspace + setup.sh); shared Python 3.14 venv at WILDS/.venv; wilds namespace package; archon from PyPI
- [Pull Requests](feedback_pull_requests.md) — WILDS changes go on a feature branch + GitHub PR, never direct to main/master
