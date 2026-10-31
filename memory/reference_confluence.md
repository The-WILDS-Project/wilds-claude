---
name: LDT Confluence access
description: How to read Lowell's Confluence (confluence.lowell.edu) — VPN required, token location and CRLF quirk, REST recipe, key page IDs
type: reference
---

**Access needs the Lowell VPN.** Off-VPN, `confluence.lowell.edu` resolves but connections
time out. Verified working on the VPN 2026-10-31.

**GitHub on the VPN is unreliable:** the WatchGuard VPN intermittently intercepts HTTPS to GitHub with its own certificate ("Fireware SSLVPN … CA"), so `gh` and `git push`/`fetch` fail TLS verification. Some calls get through, but don't count on it (2026-10-31). Do GitHub work with the VPN disconnected; do Confluence and broker work with it connected.

**Token:** personal access token `firstone`.
- Mac: `WILDS/.wilds-vault/LDT-Confluence.txt` (outside every repo — never commit it)
- Windows: `y:/MAST/.mast-vault/LDT Confluence.txt`
- The file has Windows line endings. Strip `\r` or Apache answers `400 Bad Request`:
  `T=$(awk '/token:/{print $3}' .wilds-vault/LDT-Confluence.txt | tr -d '\r\n ')`
- Use as `Authorization: Bearer $T` against `https://confluence.lowell.edu/rest/api/...`
  (`content/<id>?expand=body.storage`, `content/search?cql=...`). Never print the token.

**Key pages** (id — space — title):
- 44597378 — LIGS — DCT Science Instrument Software Interface (v78, last edited 2022-03-23;
  PDF export in `wilds-bridge/docs/`, untracked)
- 1441800 — GWAVES — Internal Interface Control Document (commands addressed to OCS/TCS)
- 11960930 — DCTDOC — MCB Configuration
- 24608781 — DCTDOC — IAA and center of rotation measurement
- Attachment 2260994 — `0720S-005-I TCS Interface Definitions.pdf`: the authoritative TCS
  command list (target, rotoffset, mcc*). Rotator semantics in §3.3.3.2.
