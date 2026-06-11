# Crow APRS Bridge

Crow can bridge APRS text messages through a configurable APRS backend. APRS traffic is public amateur-radio traffic, so keep transmit disabled until the station callsign, APRS-IS login/passcode or local TNC path, and operator-control requirements are understood.

This page folds the Raven `pt97-compliance` APRS notes into the Crow wiki and documents the current Crow APRS backend behavior.

## Current code status

Crow's current `aprs.uc` supports a single configured APRS backend under `aprs.backend`.

The current code supports APRS-IS passcode use through:

```json
"aprs": {
  "backend": {
    "type": "aprsis",
    "passcode": "REPLACE_WITH_APRS_IS_PASSCODE"
  }
}
```

When the backend type is `aprsis`, Crow reads `backend.passcode`, falls back to `-1` when no passcode is configured, and sends the APRS-IS login line using that passcode. Use `tx_enabled: true` only after a valid callsign/passcode/operator-control setup is ready.

The older Raven `pt97-compliance` branch also documented multi-backend APRS support using `aprs.backends` and per-channel backend bindings. That is important design history, but the current Crow `aprs.uc` is still using the older single-backend implementation. Treat the multi-backend examples below as a target/compatibility note until Crow's code is updated to match the old Raven multi-backend implementation.

## Quick start

1. Add an `aprs` block to `crow.conf`.
2. Add an APRS channel to the `channels` array.
3. Start Crow and open the web UI.
4. Type `/help` in the message box for available slash commands.
5. Keep APRS transmit off until configuration is verified.

## Basic configuration: receive-only APRS-IS

```json
{
  "callsign": "N0CALL-10",
  "aprs": {
    "enabled": true,
    "callsign": "N0CALL-10",
    "channel": "APRS og==",
    "default_group": "APRSgroup1",
    "inline_max_members": 10,
    "backend": {
      "type": "aprsis",
      "host": "rotate.aprs2.net",
      "port": 14580,
      "tx_enabled": false
    },
    "groups": [
      {
        "name": "APRSgroup1",
        "members": [ "N0CALL-4", "N0CALL-7" ],
        "repeat_member_messages": false,
        "rate_limit_seconds": 20,
        "max_members": 10
      }
    ]
  },
  "channels": [
    { "namekey": "AREDN og==", "telemetry": false },
    { "namekey": "APRS og==", "telemetry": false }
  ]
}
```

## APRS-IS transmit with passcode

For APRS-IS transmit, configure a valid APRS-IS passcode and set `tx_enabled` to `true` only when the station callsign, control operator, and transmit path are correct.

```json
{
  "callsign": "N0CALL-10",
  "aprs": {
    "enabled": true,
    "callsign": "N0CALL-10",
    "channel": "APRS og==",
    "backend": {
      "type": "aprsis",
      "host": "rotate.aprs2.net",
      "port": 14580,
      "passcode": "REPLACE_WITH_APRS_IS_PASSCODE",
      "filter": "b/N0CALL-4/N0CALL-7",
      "tx_enabled": true
    }
  },
  "channels": [
    { "namekey": "APRS og==", "telemetry": false }
  ]
}
```

Notes:

- `passcode` belongs inside `aprs.backend` for the current Crow code.
- `tx_enabled: false` is the safer default.
- `passcode: "-1"` or missing passcode should be treated as receive/login-only behavior, not a ready-to-transmit configuration.
- Do not use APRS to carry encrypted content.

## Backend types

### APRS-IS

```json
"backend": {
  "type": "aprsis",
  "host": "rotate.aprs2.net",
  "port": 14580,
  "passcode": "REPLACE_WITH_APRS_IS_PASSCODE",
  "filter": "b/N0CALL-4/N0CALL-7",
  "tx_enabled": true
}
```

### Dire Wolf KISS TCP

```json
"backend": {
  "type": "kiss_tcp",
  "host": "127.0.0.1",
  "port": 8001,
  "kiss_port": 0,
  "path": [],
  "tx_enabled": true
}
```

### Xastir, YAAC, or another APRS/TNC2-style TCP server

```json
"backend": {
  "type": "tcp_text",
  "host": "127.0.0.1",
  "port": 14580,
  "tx_enabled": true
}
```

## Slash command reference

These commands are typed in the Crow message box with a slash prefix.

### `/help`

Shows available commands.

### `/join` — create or join a channel / APRS group

```text
/join #name
/join %name
/join #name CALL1 CALL2 message text
/join #name backend=NAME CALL1 CALL2 message text
```

Channel-only join:

- `#name` creates or joins a shared-key channel.
- `%name` creates or joins an AREDN-only channel.

APRS group join:

- Supplying callsigns creates an APRS group and an AREDN-only channel using the unencrypted `og==` key.
- Callsigns must contain at least one digit, such as `KN6PLV` or `KJ6DZB-4`.
- Plain words such as `radio`, `check`, or `hello` are treated as message text, not callsigns.

Examples:

```text
/join #EmComm
```

```text
/join #TacNet KN6PLV KJ6DZB-4 radio check
```

```text
/join #TacNet backend=direwolf1 KN6PLV KJ6DZB-4 radio check
```

The `backend=NAME` form is part of the Raven multi-backend design. Current Crow documentation keeps it visible because the command reference includes it, but current `aprs.uc` still needs the Raven multi-backend code restored before backend-specific APRS group routing is fully supported.

### `/leave` — leave a channel and remove APRS group

```text
/leave #name
```

Removes the channel and deletes the APRS group if one exists for that name.

### `/groups` — list APRS groups

```text
/groups
```

Shows configured APRS groups and members.

### `/backend` or `/backends` — list APRS backends

```text
/backend
/backends
```

Shows configured APRS backend names. This is most useful after the multi-backend APRS implementation is restored.

### `/channels` — channel management

```text
/channels
/channels local
/channels world
/channels join #name
/channels join name key
/channels leave name
```

The newer `/join` and `/leave` forms are recommended for most channel/APRS group workflows.

## AREDN-only channels and Part 97 behavior

Channels prefixed with `%` are AREDN-only and should never be bridged into encrypted networks.

| Prefix | Key | Bridges to | Use case |
| --- | --- | --- | --- |
| `#Name` | SHA-256 derived | Meshtastic + MeshCore + AREDN | General mesh chat |
| `%Name` | `og==` unencrypted | AREDN only | APRS groups / Part 97 traffic |

APRS group messages must stay unencrypted because APRS is amateur-radio traffic. When an APRS group is created, Crow should use the `og==` unencrypted key and keep that traffic AREDN-only.

## APRS chat commands

These are typed as normal chat text in the APRS channel, without a leading slash.

### Direct message

```text
@N0CALL-4 message text
```

### Send to an existing group

```text
#APRSgroup1 message text
```

### Send to an inline list

```text
#APRSgroup1 N0CALL-4, N0CALL-7 message text
```

### Create/update runtime group and send

```text
join #APRSgroup1 N0CALL-4, N0CALL-7 message text
```

This creates `APRSgroup1` if missing, replaces the group member list, and sends the message. This in-chat form does not create a separate Crow channel; use `/join` for channel creation.

## Group repeat mode

A group can optionally repeat received APRS messages from one group member back out to the other members:

```json
{
  "name": "APRSgroup1",
  "members": [ "N0CALL-4", "N0CALL-7" ],
  "repeat_member_messages": true,
  "rate_limit_seconds": 20,
  "max_members": 10
}
```

Crow applies duplicate suppression and rate limiting to reduce loops.

## Raven pt97-compliance features still to restore in Crow code

The old Raven APRS code included features the current Crow APRS file does not yet fully match:

- `aprs.backends` plural configuration
- multiple named backend instances
- per-channel backend binding with `channels[].backend`
- backend-specific inbound channel routing
- backend-specific group routing
- `/join #group backend=NAME ...` fully bound to the selected backend
- `getBackendNames()` for UI/command listing
- runtime backend binding updates

The passcode requirement is already covered for the current single APRS-IS backend through `aprs.backend.passcode`. The next code cleanup should restore Raven's multi-backend registry while preserving Crow naming and the current passcode behavior.
