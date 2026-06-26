# Crow Wiki

This repository is the editable documentation landing area for the Crow mesh messaging project.

Crow is the upgraded Raven mesh messaging platform for AREDN, APRS, Meshtastic, MeshCore, Winlink-style forms, and related mesh workflows.

## Start here

- [Home](Home.md) — main wiki landing page and full page index
- [Change Log](Change-Log.md) — Raven-to-Crow feature and documentation tracking
- [Configuration](Configuration.md) — operator configuration reference
- [Configuring Channels](Configuring-Channels.md) — channel setup workflow
- [Command Reference](Command-Reference.md) — user-facing slash commands

## Bridges and backends

- [APRS Bridge](APRS.md) — APRS-IS, KISS TCP, APRS passcode, groups, and Part 97-safe behavior
- [Backend Selection and Test Deployment](Backend-Selection-and-Deployment.md) — backend selector and deployment validation
- [LoRa Gateway Tags](LoRa-Gateway-Tags.md) — outbound LoRa gateway tag format
- [Meshtastic API Backend](Meshtastic-API.md) — experimental Meshtastic TCP Port-API backend notes
- [Winlink](Winlink.md) — Winlink-style form UI, form storage, and current implementation notes

## Storage and operations

- [Memory Use](Memory-Use.md) — internal flash, RAM mode, USB persistent storage, degraded mode, and message-store storage behavior
- [USB Storage](USB-Storage.md) — AREDN USB setup, `/storage` commands, image quota, and degraded-mode behavior
- [Text Stores](Text-Stores.md) — Raven-style text stores / Crow node message stores
- [Supernodes](Supernodes.md) — AREDN supernode relay behavior and storage/bridge restrictions
- [Strict Gatekeeper](Strict-Gatekeeper.md) — fail-closed bridge filtering and Part 97 auto-forwarding guidance

## Markdown file inventory

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
Supernodes.md
Text-Stores.md
USB-Storage.md
Winlink.md
_Sidebar.md
```

## Relationship to GitHub's built-in wiki

This repo, `mathisono/Crow.wik`, is a normal GitHub repository used as the writable documentation source.

GitHub's built-in wiki for the main Crow repo is a separate hidden wiki repository. Content here does not automatically appear at `mathisono/Crow/wiki` unless a sync job or manual push copies these Markdown files into the built-in wiki repo.

## Maintenance

Keep these files in sync when adding or removing pages:

1. `README.md` — repo landing page
2. `Home.md` — wiki landing page
3. `_Sidebar.md` — GitHub wiki sidebar/navigation
4. `Change-Log.md` — documentation and behavior changes
