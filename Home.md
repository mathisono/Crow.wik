# Crow Wiki

Welcome to the Crow project wiki.

Crow is the rebranded home for the Raven mesh messaging work. This wiki is the working documentation area for setup notes, feature design, operational guidance, and maintenance checklists.

## Page index

| Wiki page | Markdown file | Purpose |
|---|---|---|
| [Home](Home) | `Home.md` | Main wiki landing page and index of all wiki Markdown files. |
| [Change Log](Change-Log) | `Change-Log.md` | Raven-to-Crow feature and documentation change tracking. |
| [Configuration](Configuration) | `Configuration.md` | Operator configuration overview and index of top-level config blocks. |
| [Backend Configuration](Backend-Configuration) | `Backend-Configuration.md` | Backend selector setup for Meshtastic UDP/API, MeshCore UDP/API, APRS, mode examples, and backend validation. |
| [Configuring Channels](Configuring-Channels) | `Configuring-Channels.md` | Channel setup workflow: joining channels, AREDN-only channels, APRS groups, MeshCore slot mapping, and channel validation. |
| [Backend Selection and Test Deployment](Backend-Selection-and-Deployment) | `Backend-Selection-and-Deployment.md` | Backend selector behavior, default UDP compatibility, package rebuild, deployment, and validation workflow. |
| [Command Reference](Command-Reference) | `Command-Reference.md` | User-facing slash commands and APRS chat command forms. |
| [APRS Bridge](APRS) | `APRS.md` | APRS-IS, KISS TCP, APRS passcode, APRS group messaging, and Part 97-safe channel behavior. |
| [LoRa Gateway Tags](LoRa-Gateway-Tags) | `LoRa-Gateway-Tags.md` | Outbound LoRa gateway tag format and hard-coded MeshCore/Meshtastic tag scheme. |
| [Meshtastic API Backend](Meshtastic-API) | `Meshtastic-API.md` | Experimental Meshtastic TCP Port-API backend status, including channel discovery/sync limitations. |
| [MeshCore Backends](MeshCore-Backends) | `MeshCore-Backends.md` | Original MeshCore UDP backend, experimental TCP Companion API backend, selector behavior, receive-only API status, and discovery limitations. |
| [Memory Use](Memory-Use) | `Memory-Use.md` | Internal flash, RAM, USB persistent storage, degraded mode, image quota, Winlink form storage, and message-store storage behavior. |
| [USB Storage](USB-Storage) | `USB-Storage.md` | AREDN USB data storage, setup commands, degraded-mode behavior, persistent images, and storage quota handling. |
| [Text Stores](Text-Stores) | `Text-Stores.md` | Raven-style text store/message store behavior, including USB-backed persistent stores on normal Crow nodes. |
| [Supernodes](Supernodes) | `Supernodes.md` | Supernode behavior, MeshIP relay purpose, and why bridges/message stores are disabled on supernodes. |
| [Strict Gatekeeper](Strict-Gatekeeper) | `Strict-Gatekeeper.md` | Fail-closed bridge filtering, gateway identification, and Part 97 auto-forwarding guidance. |
| [Winlink](Winlink) | `Winlink.md` | Winlink-style form UI, form inventory, storage behavior, and how to add forms. |
| Sidebar | `_Sidebar.md` | GitHub wiki navigation sidebar. This is not a normal content page, but it must be kept in sync with this index. |

## Current Markdown inventory

These are the `.md` files currently expected in the wiki repository:

```text
APRS.md
Backend-Configuration.md
Backend-Selection-and-Deployment.md
Change-Log.md
Command-Reference.md
Configuration.md
Configuring-Channels.md
Home.md
LoRa-Gateway-Tags.md
Meshtastic-API.md
MeshCore-Backends.md
Memory-Use.md
Strict-Gatekeeper.md
Supernodes.md
Text-Stores.md
USB-Storage.md
Winlink.md
_Sidebar.md
```

## Maintenance status

All Markdown pages currently expected in this wiki should be linked from this Home page and from the sidebar.

When adding, renaming, or removing a `.md` page, update both:

1. `Home.md` under Page index and Current Markdown inventory
2. `_Sidebar.md` under Pages

This keeps every wiki page discoverable from the GitHub wiki UI and prevents orphan pages.
