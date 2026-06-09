# Crow Command Reference

This page covers the user-facing slash commands currently wired into Crow's command handler.

## Quick list

| Command | Purpose |
| --- | --- |
| help | Show command help in the UI. |
| join | Join or create a shared channel, and optionally create an APRS group. |
| leave | Leave a channel and remove the matching APRS group if present. |
| groups | List configured APRS groups and members. |
| channels | List, join, or leave public channels. |
| export | Export the current channel log as text or CSV. |
| backend / backends | List configured APRS backends. |
| storage | Show and manage Crow storage state on supported platforms. |

## help

Shows the built-in help text for available slash commands.

## join

Join or create a channel.

Forms:

- join #name
- join %name
- join #name CALL1 CALL2 message
- join #name backend=NAME CALL1 message

Notes:

- #name creates or joins a shared-key channel for Meshtastic, MeshCore, and AREDN.
- %name creates or joins an AREDN-only channel.
- Supplying callsigns creates or updates an APRS group and can send an initial message.
- backend=NAME pins an APRS group/channel to a specific APRS backend.

## leave

Leave a channel and remove the matching APRS group if one exists.

Form: leave #name

## groups

List all APRS groups and members.

## channels

List public channels or join/leave by channel name.

Forms:

- channels
- channels local
- channels world
- channels join #name
- channels join name key
- channels leave name

Notes:

- channels and channels local show local public channels.
- channels world asks an available bridge for public channels across the mesh.
- channels join #name derives the shared channel key for hash-style channels.
- channels join name key joins a channel with an explicit key.

## export

Export the current selected channel log.

Forms:

- export
- export text
- export csv

Notes:

- The default format is plain text.
- CSV export uses timestamp, from, and message fields.
- Export requires a selected channel and at least one message.

## backend and backends

List configured APRS backends. Both forms are aliases.

## storage

Show and manage storage state on supported platforms. The AREDN platform provides the backend for USB data storage, degraded mode, and persistent image quota handling.

Forms:

- storage status
- storage usb scan
- storage usb enable
- storage usb disable
- storage quota images <mb>

Notes:

- storage status shows active storage mode, root path, image path, mountpoint, and degraded reason if present.
- storage usb scan lists removable USB storage candidates detected by the platform.
- storage usb enable attempts to activate configured USB storage.
- storage usb disable returns Crow to internal node storage.
- storage quota images <mb> updates the persistent image quota and prunes old images if needed.

The Crow core remains on the node. USB storage is used only for data, forms, and persistent images.

## APRS chat commands

These are typed in an APRS channel as normal chat text rather than as slash commands.

Forms:

- @CALL message
- #group message
- #group CALL1 CALL2 message
- join #group CALL1 CALL2 message

## Maintenance checklist

When adding or changing a slash command:

1. Update the help output in commands.uc.
2. Update this page.
3. Confirm the command is linked from Home and Sidebar.
