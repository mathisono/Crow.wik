# Crow Change Log

This page tracks the practical changes from Raven to Crow, including code work, wiki/documentation work, and operator-facing behavior.

## Current focus

Crow is the rebranded and expanded home for the Raven mesh messaging work. The goal is to keep Raven's useful AREDN messaging base while adding clearer operator controls, APRS support, USB-attached storage, and better Part 97-aware bridge behavior.

## 2026-06-25

### Crow: fix USB setup script Crow path
**Crow commit:** `dd82ff90`

- Updated `platforms/aredn/usb-setup.sh` usage text from the legacy Raven path to the current Crow path:
  - old: `/usr/local/raven/platforms/aredn/usb-setup.sh`
  - new: `/usr/local/crow/platforms/aredn/usb-setup.sh`
- No storage behavior changed. This was a documentation/path cleanup inside the script.
- **Files changed:** `platforms/aredn/usb-setup.sh`

### Crow: fix README wiki link display
**Crow commit:** `18e1cae`

- Fixed the visible README documentation URL so it points to `mathisono/Crow/wiki` instead of the misspelled `mathison/Crow/wiki`.
- The link target was already correct; the displayed text was corrected.
- **Files changed:** `README.md`

### Wiki: storage, supernode, text store, and Winlink refresh
**Wiki status:** committed to `mathisono/Crow.wik`

- Rewrote `Memory-Use.md` around Crow's current storage behavior: internal storage, USB storage, degraded mode, image quota, RAM message mode, Winlink form storage, and message-store behavior.
- Added/updated `Supernodes.md` using the original Raven source material and Crow code behavior. Supernodes auto-detect from AREDN config, relay MeshIP traffic, and intentionally do not act as message/text stores.
- Added/updated `Text-Stores.md` / node message store documentation using the original Raven Text Stores source material and Crow code behavior. Text stores use `platform.store("textstore.<namekey>")`, so they automatically use `/mnt/crow/data` when USB storage is active.
- Updated `Winlink.md` from placeholder status into a Crow-specific form UI and storage page based on current `winlink.uc` behavior.
- Updated `Home.md` and `_Sidebar.md` to include `Supernodes.md` and `Text-Stores.md` as canonical pages.

## Raven → Crow overview

| Area | Raven behavior | Crow change | Status / notes |
| --- | --- | --- | --- |
| Project identity | Raven naming and raven icon assets. | Rebranded as Crow with Crow UI/icon assets. | UI assets are being renamed/replaced as `crow.png` and `crow.svg`. |
| Messaging goal | Mesh messaging foundation. | Unified decentralized messaging platform for AREDN and related mesh transports. | Crow is intended to unify AREDN, Meshtastic, MeshCore, legacy MeshChat, Winlink, and APRS workflows. |
| APRS backend | Raven `pt97-compliance` branch had richer APRS work. | Crow needs the Raven APRS backend restored as the current `aprs.uc`. | Target is Raven `pt97-compliance/aprs.uc`. Verify that `main` contains the restored code before release. |
| APRS passcode | Raven APRS-IS backend supported `passcode`. | Crow docs now explicitly document APRS-IS passcode setup. | Use `aprs.backend.passcode` for single-backend compatibility and `aprs.backends.<name>.passcode` for named backends once multi-backend code is active. |
| APRS groups | Raven supported APRS group messaging and repeat behavior. | Crow wiki now documents APRS groups, inline APRS message forms, and `/join` group creation. | Code should be verified against the Raven APRS implementation. |
| Meshtastic API backend | Raven used the UDP/multicast Meshtastic backend. | Crow has an experimental separate `meshtastic_API.uc` TCP Port-API backend. | Current code supports first-pass TCP RX/TX plumbing only. Auto channel discovery and channel sync are **not implemented yet**. See [Meshtastic API Backend](Meshtastic-API). |
| Slash commands | Earlier command set was smaller and less documented. | Crow wiki now has a user-facing command reference. | Includes `/help`, `/join`, `/leave`, `/groups`, `/channels`, `/backend`, `/backends`, `/export`, and `/storage`. |
| Strict Gatekeeper | Not clearly operator-documented. | Crow includes `gatekeeper.uc` and expanded wiki guidance. | Handles fail-closed bridge filtering for Meshtastic/MeshCore ingress. |
| Part 97 bridge policy | Needed clearer operator explanation. | Crow wiki now explains automatic forwarding risk and gateway responsibilities in plain language. | See Strict Gatekeeper page. |
| USB attached storage | Not part of the older Raven baseline. | Crow adds documented USB storage workflows for AREDN nodes. | Supports internal storage fallback, USB scan/enable/disable, and image quota behavior. |
| Auto-update cron | Legacy Raven cron updater existed in packaging. | Crow removed the auto-update cron behavior. | This avoids surprise unattended package replacement. |

## Documentation changes

| Wiki page | Purpose | Status |
| --- | --- | --- |
| [Home](Home) | Main wiki landing page and page index. | Updated to link active documentation pages. |
| [Command Reference](Command-Reference) | User-facing slash commands and APRS chat command forms. | Added/updated. |
| [APRS Bridge](APRS) | APRS-IS, KISS TCP, APRS passcode, groups, Part 97-safe APRS behavior. | Added. |
| [Meshtastic API Backend](Meshtastic-API) | Experimental TCP Port-API backend status and channel discovery/sync limitations. | Added. |
| [Strict Gatekeeper](Strict-Gatekeeper-Mode) | Fail-closed bridge filtering and Part 97 auto-forwarding explanation. | Expanded with tables and operator guidance. |
| [USB Storage](USB-Storage) | AREDN USB data storage and persistent image storage behavior. | Added/updated. |
| [Change Log](Change-Log) | This page. | Added/updated. |

## Meshtastic API backend notes

Crow now has a separate experimental `meshtastic_API.uc` backend for direct TCP Port-API testing. The original `meshtastic.uc` UDP/multicast backend remains the stable production path unless the router is deliberately changed.
