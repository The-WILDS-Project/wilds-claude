---
name: Weizmann SMB shares
description: How to reach //mast-ns-control/mast-share and //last0/catalogs from the Mac; credentials in WILDS/.wilds-vault/smb-shares.txt
type: reference
---

Two Samba (Ubuntu) servers on the Weizmann network (`*.weizmann.ac.il`), SMB on TCP 445. Checked 2026-11-04.

| Share | Contents (top level) |
|---|---|
| `//mast-ns-control/mast-share` | `2025-04-16 pointing models`, `Downloads`, `MAST`, `solve-test`, `test` |
| `//last0/catalogs` | astrophysics catalogs: `GAIA`, `GLADE`, `MergedCat`, `PGC`, `QSO`, `data` |

- **Login:** user `mast`; the password is in `/Users/blumzi/src/WILDS/.wilds-vault/smb-shares.txt` (`password: …` line; parse with `split(":", 1)[1].strip()`). Never print it, and never put it in this repo, which is public.
- **Guest:** can list the shares (`smbutil view -N //guest@<host>`) but not open them. Samba reports that as "No such file or directory".
- **Mount read-only** under the session scratchpad, then unmount when done:
  ```sh
  mount_smbfs -o rdonly "//mast:$PW@mast-ns-control/mast-share" "$MNT"   # or last0/catalogs
  umount "$MNT"
  ```
  `mount_smbfs` error messages echo the URL with the password, so pipe output through `sed` to mask it.
- `mast-ns-control` also has `mast-provisioning`: "MAST provisioning payloads (read-only; units pull from here)".
