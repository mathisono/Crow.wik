# Crow APRS Bridge

Crow can bridge APRS text messages through configurable APRS backends. APRS traffic is public amateur-radio traffic, so keep transmit disabled until the station callsign, APRS-IS login/passcode or local TNC path, and operator-control requirements are understood.

This page folds the Raven `pt97-compliance` APRS notes into the Crow wiki and documents the current Crow APRS backend behavior.

## Current code status

Crow's current `aprs.uc` supports both:

- backward-compatible single backend config under `aprs.backend`
- multi-backend config under `aprs.backends`

If `aprs.backends` is not present, Crow creates a default backend from `aprs.backend`. If neither is present, Crow creates a default APRS-IS backend.

Backend names are used for channel bindings, APRS group bindings, `/join #group backend=NAME ...`, and the UI/command backend list.

## Quick start

1. Add an `aprs` block to `crow.conf` or `crow.conf.override`.
2. Add or create an APRS channel.
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

- `passcode` belongs inside each APRS backend config.
- `tx_enabled: false` is the safer default.
- `passcode: "-1"` or missing passcode should be treated as receive/login-only behavior, not a ready-to-transmit configuration.
- Do not use APRS to carry encrypted content.

## Multi-backend configuration

Crow can define multiple named APRS backends under `aprs.backends`.

```json
{
  "callsign": "N0CALL-10",
  "aprs": {
    "enabled": true,
    "callsign": "N0CALL-10",
    "channel": "APRS og==",
    "backends": {
      "aprsis": {
        "type": "aprsis",
        "host": "rotate.aprs2.net",
        "port": 14580,
        "passcode": "REPLACE_WITH_APRS_IS_PASSCODE",
        "filter": "b/N0CALL-4/N0CALL-7",
        "tx_enabled": false
      },
      "direwolf1": {
        "type": "kiss_tcp",
        "host": "127.0.0.1",
        "port": 8001,
        "kiss_port": 0,
        "path": [],
        "tx_enabled": true
      }
    }
  },
  "channels": [
    { "namekey": "APRS og==", "telemetry": false, "backend": "aprsis" },
    { "namekey": "%TacNet og==", "telemetry": false, "backend": "direwolf1" }
  ]
}
```

Crow uses the first backend in `aprs.backends` as the default backend. A channel can override that by setting `backend` in the channel entry. APRS groups can also carry a `backend` property.

Backend resolution order for a group is:

1. `group.backend`, when present and valid
2. channel backend binding for the current channel
3. default APRS backend

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

Crow treats `tcp_text`, `xastir`, and `yaac` as TNC2-style text connections.

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
- `/join` creates APRS groups with `repeat_member_messages: false` by default.
- Add `backend=NAME` to bind the group/channel to a specific APRS backend.

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

### `/leave` — leave a channel and remove APRS group

```text
/leave #name
```

Removes the channel and deletes the APRS group if one exists for that name.

### `/groups` — list APRS groups

```text
/groups
```

Shows configured APRS groups and members. Groups with `repeat_member_messages: true` are marked with `[repeat]`. Groups bound to a backend show `[backend=NAME]`.

### `/backend` or `/backends` — list APRS backends

```text
/backend
/backends
```

Shows configured APRS backend names and labels.

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

APRS group messages must stay unencrypted because APRS is amateur-radio traffic. When an APRS group is created with callsigns, Crow forces AREDN-only behavior and uses the `og==` unencrypted key.

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

With an explicit backend:

```text
join #APRSgroup1 backend=direwolf1 N0CALL-4, N0CALL-7 message text
```

This creates `APRSgroup1` if missing, replaces the group member list, optionally stores the backend binding, and sends the message. This in-chat form does not create a separate Crow channel; use `/join` for channel creation.

## Group repeat mode

APRS group repeat mode lets Crow act like a small APRS message reflector for a configured group.

When Crow receives an APRS message addressed to the Crow station callsign from a callsign that belongs to a configured group, Crow can repeat that message back out to the other members of the group.

Example group config:

```json
{
  "name": "APRSgroup1",
  "members": [ "N0CALL-4", "N0CALL-7", "N0CALL-9" ],
  "repeat_member_messages": true,
  "rate_limit_seconds": 20,
  "max_members": 10
}
```

With this enabled:

1. `N0CALL-4` sends an APRS message to the Crow station callsign.
2. Crow sees that `N0CALL-4` is a member of `APRSgroup1`.
3. Crow repeats the message to the other group members.
4. Crow does **not** send the repeat back to `N0CALL-4`.
5. Repeated messages are prefixed with the source callsign:

```text
[N0CALL-4] original message text
```

The code uses the group's backend resolution path for repeated messages. If `group.backend` is set and valid, repeats use that backend. Otherwise they use the channel backend or the default backend.

### Loop and spam protection

Crow applies two protections before repeating a member message:

- duplicate suppression: the same `group + source + text + APRS id` is suppressed for 30 minutes
- group rate limit: `rate_limit_seconds` defaults to 20 seconds between repeated messages for the group

The send list also enforces `max_members`, defaulting to 10 when no group-specific value is set.

### Default repeat behavior

Groups created by `/join #name CALL1 CALL2 message text` default to:

```json
"repeat_member_messages": false
```

Enable repeat mode in the APRS group config only for groups where you intentionally want Crow to act as an APRS group reflector.

## Inbound routing behavior

When APRS traffic arrives, Crow tries to place it in the most useful local Crow channel.

If the sending callsign is a member of a configured APRS group, Crow looks for a matching local group channel first:

1. `%GroupName og==`
2. a matching `#GroupName ...` channel
3. the main APRS channel from `aprs.channel`

For non-group traffic, Crow uses the channel/backend map where possible, then falls back to the main APRS channel.

## Outbound behavior

Crow outbound APRS behavior depends on the text pattern and current channel:

| Text pattern | Behavior |
| --- | --- |
| `@CALL text` | Sends direct APRS message to `CALL`. |
| `#Group text` | Sends to an existing group. |
| `#Group CALL1 CALL2 text` | Sends to an inline list. |
| `join #Group CALL1 CALL2 text` | Creates/updates runtime group and sends. |
| normal text in group channel | Sends to that channel's APRS group. |
| normal text in a backend-bound channel with a known last sender | Replies to the last stored APRS sender for that channel. |

## Current implementation notes

The current Crow APRS code includes:

- APRS-IS passcode support per backend
- APRS-IS text/TNC2 backend support
- KISS TCP backend support
- `tcp_text`, `xastir`, and `yaac` style TCP text backend support
- multi-backend registry through `aprs.backends`
- backward-compatible `aprs.backend` single-backend config
- channel-to-backend binding
- group-to-backend binding
- runtime channel backend updates
- `/backend` / `/backends` backend listing through `getBackendNames()`
- APRS group send and repeat behavior
- APRS message acknowledgements when received messages include an APRS message id

## Safety notes

- APRS is public amateur-radio traffic.
- Keep `tx_enabled` off until station identification and operator-control requirements are correct.
- Do not bridge APRS into encrypted channels.
- Use `%Name og==` AREDN-only channels for APRS group traffic.
- Be careful with group repeat mode; it transmits back out to multiple APRS stations.
