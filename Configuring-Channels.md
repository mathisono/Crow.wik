# Configuring Channels

This page is the operator how-to for Crow channel setup.

Use this page after the bridge backends are selected. Backends decide **how Crow talks to radios/transports**. Channels decide **which message groups, keys, and mappings Crow routes**.

For backend selector details, UDP/API setup, APRS backend setup, and transport validation, see [Backend Configuration](Backend-Configuration.md).

## Configuration order

Use this order when setting up a node:

1. Set node identity and basic config. See [Configuration](Configuration.md).
2. Select bridge backends. See [Backend Configuration](Backend-Configuration.md).
3. Configure bridge safety if needed. See [Strict Gatekeeper](Strict-Gatekeeper.md).
4. Join or define local Crow channels.
5. Add APRS groups or MeshCore slot mappings if needed.
6. Validate message routing.

## Backend setup vs channel setup

| Layer | Question it answers | Example |
|---|---|---|
| Backend setup | How does Crow talk to the outside transport? | Meshtastic UDP, MeshCore UDP, MeshCore TCP API, APRS-IS, KISS TCP |
| Channel setup | Which group/key/channel should this message use? | `AREDN og==`, `#TacNet`, `%TacNet`, APRS group, MeshCore slot 0 |

Do backend setup first. If the backend is not active, channel membership can look correct but messages still will not leave or enter through that transport.

## Channel types

Crow uses channel names and `namekey` values to decide routing and encryption behavior.

| Channel form | Meaning | Typical use |
|---|---|---|
| `AREDN og==` | AREDN public/default unencrypted channel | local AREDN mesh traffic |
| `#Name` through `/join` | shared-key Crow channel | general mesh chat where backend rules allow it |
| `%Name` through `/join` | AREDN-only unencrypted channel | APRS groups / Part 97-safe AREDN-only traffic |
| `Name <base64-key>` | explicit namekey config | manual channel config or imported/discovered channel |
| MeshCore slot mapping | maps group slot `0-7` to a Crow channel | MeshCore TCP API group messages |

## AREDN-only channels

Channels prefixed with `%` are AREDN-only and should not be bridged into encrypted LoRa networks.

APRS group channels should use the unencrypted `og==` key.

Example:

```text
%TacNet og==
```

When APRS groups are created through `/join`, Crow should use AREDN-only behavior for that group.

## Joining channels with slash commands

If the running build supports slash commands, use the channel join command from the chat/UI.

Create or join a normal shared-key channel:

```text
/join #EmComm
```

Create or join an AREDN-only channel:

```text
/join %TacNet
```

Create an APRS group/channel with callsigns:

```text
/join #TacNet KN6PLV KJ6DZB-4 radio check
```

Create an APRS group/channel and bind it to a named APRS backend:

```text
/join #TacNet backend=direwolf1 KN6PLV KJ6DZB-4 radio check
```

Notes:

- Callsigns must contain at least one digit, such as `KN6PLV` or `KJ6DZB-4`.
- Plain words such as `radio`, `check`, or `hello` are treated as message text, not callsigns.
- `/join` creates APRS groups with `repeat_member_messages: false` by default.
- Use [APRS Bridge](APRS.md) for APRS group repeat and backend details.

## Leaving channels

Use `/leave` to leave a channel and remove the related APRS group when one exists.

```text
/leave #TacNet
```

## Listing channels and groups

Useful commands:

```text
/channels
/groups
/backend
/backends
```

`/groups` shows APRS groups and marks repeat-capable groups with `[repeat]`. Backend-bound groups show `[backend=NAME]`.

## Manual channel config pattern

A Crow channel entry normally needs enough information to identify:

- channel name or namekey;
- symmetric key material or shorthand key;
- telemetry preference;
- optional backend binding;
- optional access-control policy.

Minimal example:

```json
{
  "channels": [
    { "namekey": "AREDN og==", "telemetry": false }
  ]
}
```

Example with access-control intent:

```json
{
  "channels": [
    {
      "namekey": "AREDN-Local og==",
      "telemetry": false,
      "access_control": {
        "require_callsign": true,
        "allowed_callsigns": ["KJ6DZB", "W6*"],
        "deny_callsigns": []
      }
    }
  ]
}
```

Exact channel object shape should follow the current Crow config schema in the deployed build.

## Channel keys and namekeys

A `namekey` is the channel name plus key material.

Examples:

```text
AREDN og==
TacNet <base64-key>
```

Practical rules:

- Keep raw PSKs and private keys out of logs and public docs.
- Use `og==` only where unencrypted AREDN/Part 97 behavior is intended.
- Do not bridge APRS traffic into encrypted channels.
- Be deliberate when copying channel keys from radio/API discovery output.

## Backend-bound channels

Some channels may be bound to a specific backend.

Example shape:

```json
{
  "channels": [
    { "namekey": "APRS og==", "telemetry": false, "backend": "aprsis" },
    { "namekey": "%TacNet og==", "telemetry": false, "backend": "direwolf1" }
  ]
}
```

This is most useful when APRS has multiple named backends or when a test deployment needs explicit routing.

Backend configuration itself belongs in [Backend Configuration](Backend-Configuration.md) and [APRS Bridge](APRS.md).

## APRS groups as channels

APRS group behavior is channel-adjacent because Crow needs somewhere to display and route group messages.

Typical APRS group config shape:

```json
{
  "aprs": {
    "groups": [
      {
        "name": "TacNet",
        "members": ["KN6PLV", "KJ6DZB-4"],
        "repeat_member_messages": false,
        "rate_limit_seconds": 20,
        "max_members": 10
      }
    ]
  }
}
```

When repeat mode is enabled, Crow can repeat messages from one APRS group member to the other group members. That belongs to APRS behavior, not general channel setup.

See: [APRS Bridge](APRS.md).

## Meshtastic API discovered channels

When `meshtastic_api.channel_discovery=true`, discovered channels are runtime-only for now.

The backend extracts:

- channel index;
- channel name;
- PSK bytes;
- runtime `namekey` form.

Current limits:

- discovered channels are not written to config;
- discovered channels do not automatically overwrite operator-managed channels;
- discovered channels do not push changes back to the Meshtastic radio;
- raw PSKs are not logged.

Use discovery to verify the radio's channel state, then deliberately add or map channels in Crow config when needed.

See: [Meshtastic API Backend](Meshtastic-API.md).

## MeshCore group slot mapping

MeshCore TCP API group messages carry a group slot index from `0-7`. Crow needs that slot mapped to a Crow channel before the message can route locally.

Current code has support for slot-to-channel mapping:

```text
channelsByMeshcoreSlot[slot]
```

and helper behavior around:

```ucode
getChannelByMeshcoreSlot(slot)
setMeshcoreSlotChannel(slot, channel)
registerGroupChannel(slot, channelObj)
```

Current limitation: MeshCore TCP discovery has parser/design code, but live radio slot querying is not production-ready yet. Manual slot/channel registration or static config may be needed during testing.

See: [MeshCore Backends](MeshCore-Backends.md).

## Text stores for channels

Text stores retain recent channel history on normal Crow nodes.

Example:

```json
{
  "textstore": {
    "stores": [
      { "namekey": "AREDN og==", "size": 100 },
      { "namekey": "*" }
    ]
  }
}
```

Text stores use Crow storage through `platform.store("textstore.<namekey>")`, so USB storage automatically places them under `/mnt/crow/data` when USB mode is active.

Do not configure text/message stores on AREDN supernodes; Crow disables text/message stores on supernodes.

See: [Text Stores](Text-Stores.md), [USB Storage](USB-Storage.md), and [Supernodes](Supernodes.md).

## Strict Gatekeeper and channel access

Strict Gatekeeper is bridge safety, not channel setup, but channel access rules can depend on callsigns.

For real public/event bridges:

- set `strict_gatekeeper.enabled=true`;
- set a gateway callsign;
- use `allowed_callsigns` for the bridge policy;
- use channel-level access control only when the deployed build supports it.

See: [Strict Gatekeeper](Strict-Gatekeeper.md).

## Channel validation

After changing channel configuration, restart Crow:

```sh
/etc/init.d/crow restart
```

Watch logs:

```sh
logread -f | grep -Ei 'crow|channel|aprs|meshcore|meshtastic|gatekeeper'
```

Validation checklist:

1. Crow starts without crashing.
2. Intended backend is already selected. See [Backend Configuration](Backend-Configuration.md).
3. Channels appear in the UI or `/channels` output.
4. `/groups` shows expected APRS groups.
5. APRS group repeat is enabled only where intentionally configured.
6. MeshCore group slot messages have a matching channel mapping before testing group RX.
7. Joined channels route as expected.
8. No raw PSKs appear in logs.
9. Read-only API discovery does not modify persistent channel config.
10. Strict Gatekeeper still filters bridged LoRa traffic when enabled.

## Safety notes

- Backend setup comes before channel joining.
- Channels are routing/permission/key objects; backends are radio/transport objects.
- `og==` means unencrypted key material and should be used intentionally.
- APRS and Part 97 traffic should stay on unencrypted AREDN/APRS-safe paths.
- API discovery is read-only until explicit write-sync is designed and tested.
- Do not push channel config to radios until an explicit write-sync phase is designed and tested.
