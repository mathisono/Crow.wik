# Crow Wiki

Welcome to the Crow project wiki.

Crow is the rebranded home for the Raven mesh messaging work. This wiki is the working documentation area for setup notes, feature design, operational guidance, and maintenance checklists.

## Page index

| Wiki page | Markdown file | Purpose |
| --- | --- | --- |
| [Home](Home) | `Home.md` | Main wiki landing page and index of all wiki Markdown files. |
| [Change Log](Change-Log) | `Change-Log.md` | Raven-to-Crow feature and documentation change tracking. |
| [Configuration](Configuration) | `Configuration.md` | Operator configuration how-to for backend selection, Meshtastic API, and MeshCore TCP API setup. |
| [Configuring Channels](Configuring-Channels) | `Configuring-Channels.md` | Channel setup workflow: bridges/backends first, then Meshtastic, MeshCore, APRS, and joining channels. |
| [Backend Selection and Test Deployment](Backend-Selection-and-Deployment) | `Backend-Selection-and-Deployment.md` | Backend selector behavior, default UDP compatibility, package rebuild, deployment, and validation workflow. |
| [Command Reference](Command-Reference) | `Command-Reference.md` | User-facing slash commands and APRS chat command forms. |
| [APRS Bridge](APRS) | `APRS.md` | APRS-IS, KISS TCP, APRS passcode, APRS group messaging, and Part 97-safe channel behavior. |
| [LoRa Gateway Tags](LoRa-Gateway-Tags) | `LoRa-Gateway-Tags.md` | Outbound LoRa gateway tag format and hard-coded MeshCore/Meshtastic tag scheme. |
| [Meshtastic API Backend](Meshtastic-API) | `Meshtastic-API.md` | Experimental Meshtastic TCP Port-API backend status, including channel discovery/sync limitations. |
| [Memory Use](Memory-Use) | `Memory-Use.md` | RAM/internal-flash guidance and using external USB storage as Crow's data layer. |
| [Strict Gatekeeper](Strict-Gatekeeper) | `Strict-Gatekeeper.md` | Fail-closed bridge filtering, gateway identification, and Part 97 auto-forwarding guidance. |
| [Winlink](Winlink) | `Winlink.md` | Winlink-style form UI, form inventory, and how to add forms. |
| [USB Storage](USB-Storage) | `USB-Storage.md` | AREDN USB data storage, degraded-mode behavior, persistent images, and storage quota handling. |
| Sidebar | `_Sidebar.md` | GitHub wiki navigation sidebar. This is not a normal content page, but it must be kept in sync with this index. |

## Current Markdown inventory

These are the `.md` files currently expected in the wiki repository:

```text
APRS.md
Backend-Selection-and-Deployment.md
Change-Log.md
Command-Reference.md
Configuration.md
Configuring-Channels.md
Home.md
LoRa-Gateway-Tags.md
Meshtastic-API.md
Memory-Use.md
Strict-Gatekeeper.md
USB-Storage.md
Winlink.md
_Sidebar.md
```

## Maintenance status

All Markdown pages currently present in this wiki are linked from this Home page and from the sidebar.

When adding, renaming, or removing a `.md` page, update both:

1. `Home.md` under **Page index** and **Current Markdown inventory**
2. `_Sidebar.md` under **Pages**

This keeps every wiki page discoverable from the GitHub wiki UI and prevents orphan pages.
