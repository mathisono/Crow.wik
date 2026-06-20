# Meshtastic API Backend

Status: **experimental / not router-wired by default**.

The Meshtastic API backend is the separate TCP Port-API backend in Crow:

```text
meshtastic_API.uc
```

It is intentionally separate from the existing production UDP/multicast backend:

```text
meshtastic.uc
```

## Current support

The current `meshtastic_API.uc` backend supports first-pass TCP Port-API plumbing:

- connects to a Meshtastic node over TCP;
- default TCP Port-API port: `4403`;
- reads Meshtastic Port-API frames using the `0x94 0xc3` frame header;
- decodes `FromRadio.packet` into Crow-style Meshtastic messages;
- sends Crow-originated text back through a `ToRadio.packet` envelope;
- keeps the original `meshtastic.uc` UDP/multicast backend unchanged.

Example experimental config:

```json
{
  "meshtastic_api": {
    "enabled": true,
    "host": "192.168.4.1",
    "port": 4403
  }
}
```

## Auto channel discovery

Current status: **not implemented yet**.

The Meshtastic API backend does **not** currently query the node for its channel list, channel names, PSKs, modem presets, or channel indexes.

It does not currently create Crow channels automatically from the Meshtastic node configuration.

## Channel sync

Current status: **not implemented yet**.

The Meshtastic API backend does **not** currently sync channel changes between the Meshtastic node and Crow.

It does not currently:

- pull the node's configured channels into Crow;
- push Crow channel changes back to the Meshtastic node;
- watch for channel changes on the node;
- periodically refresh channel state;
- resolve channel indexes into newly discovered Crow channel records.

## What works instead

For now, Meshtastic API traffic uses Crow's existing channel model.

Inbound packets are decoded and then mapped through Crow's existing channel/hash logic. That means channels must already exist in Crow config or be manually configured before reliable channel mapping can happen.

The existing `meshtastic.uc` backend remains the stable production path unless the router is deliberately changed to import the API backend.

## Future work

A later Meshtastic API phase should add channel discovery and sync by using Meshtastic admin/config protobuf support.

Needed future behavior:

1. Query the Meshtastic node's channel configuration.
2. Decode channel names, indexes, roles, PSKs, and modem preset context.
3. Convert discovered channels into Crow `namekey` entries.
4. Maintain a mapping of Meshtastic channel index/hash to Crow channel.
5. Detect node-side channel changes.
6. Avoid overwriting operator-managed Crow channels without an explicit setting.
7. Add read-only discovery mode before enabling bidirectional sync.

Suggested future config shape:

```json
{
  "meshtastic_api": {
    "enabled": true,
    "host": "192.168.4.1",
    "port": 4403,
    "channel_discovery": true,
    "channel_sync": "read_only"
  }
}
```

Suggested sync modes:

| Mode | Meaning |
| --- | --- |
| `off` | No discovery or sync. Current safe behavior. |
| `read_only` | Read channels from the Meshtastic node and expose them to Crow without pushing changes back. |
| `bidirectional` | Future mode only. Sync selected Crow channel changes back to the node. Requires extra safety checks. |

## Release note

Do not document auto channel discovery or channel sync as supported until code exists and has been tested against real Meshtastic hardware.
