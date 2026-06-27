# MeshCore TCP API Protocol Notes

This page records an important correction for Crow's experimental MeshCore TCP API work.

## Current status

Crow's `meshcore_tcp_api.uc` should be treated as a development scaffold, not a stock-compatible MeshCore TCP/Wi-Fi or USB serial client yet.

The current Crow file assumes this outer frame shape:

```text
[ 0x3E ][ CmdID ][ PayloadLen BE ][ Payload ]
```

That does not match the stock MeshCore serial/Wi-Fi outer framing described for the firmware interface.

## Correct stock outer framing

Stock MeshCore TCP/Wi-Fi serial framing uses different direction markers and little-endian length fields:

```text
Radio -> client: [ '>' ][ length LSB ][ length MSB ][ frame payload ]
Client -> radio: [ '<' ][ length LSB ][ length MSB ][ frame payload ]
```

The USB serial interface uses the same `>` / `<` framing model.

That means Crow's current TCP parser must be changed before it can talk to stock MeshCore Companion / Serial Wi-Fi framing.

## Message receive model correction

Crow's current file treats `0x07` and `0x08` like asynchronous direct/group message command IDs.

The stock receive flow is different:

```text
0x83 = PUSH_CODE_MSG_WAITING
client sends CMD_SYNC_NEXT_MESSAGE = 0x0A
radio returns queued message response
client decodes response code 0x07 / 0x08 / 0x10 / 0x11
```

Crow should eventually handle:

| Code | Role |
|---:|---|
| `0x83` | message waiting push/tickle |
| `0x0A` | client command to fetch/sync the next queued message |
| `0x07` | older direct-message receive response |
| `0x08` | older channel/group-message receive response |
| `0x10` | newer v3 direct-message receive response |
| `0x11` | newer v3 channel/group-message receive response |

Until that sync flow is implemented, the TCP API backend should not be described as a working stock MeshCore receive backend.

## Discovery status

`meshcore_tcp_discovery.uc` has the right high-level discovery idea.

The MeshCore channel discovery constants are:

```text
CMD_GET_CHANNEL = 0x1F
RESP_CODE_CHANNEL_INFO = 0x12
```

The response payload layout is:

```text
0x12
channel index, 0-7
32-byte channel name, null-padded UTF-8
16-byte secret
```

That matches the parser documented in `meshcore_tcp_discovery.uc`.

The missing piece is command send. `queryDeviceGroups()` currently loops slots `0-7` but does not send the TCP command, so it returns an empty array.

The next missing implementation step is equivalent to:

```text
meshcore_tcp_api.sendCommand(0x1F, slot)
```

using the corrected client-to-radio frame:

```text
[ '<' ][ length LSB ][ length MSB ][ frame payload ]
```

## Implementation checklist

Before calling the TCP API backend stock-compatible, Crow needs:

1. Replace the current outer frame parser with stock `>` / `<` framing.
2. Read and write 2-byte little-endian frame lengths.
3. Add a command send helper for client-to-radio frames.
4. Handle `PUSH_CODE_MSG_WAITING = 0x83`.
5. Send `CMD_SYNC_NEXT_MESSAGE = 0x0A` when a message-waiting tickle arrives.
6. Decode queued message responses `0x07`, `0x08`, `0x10`, and `0x11`.
7. Wire discovery to send `CMD_GET_CHANNEL = 0x1F` for slots `0-7`.
8. Keep group-slot-to-channel mapping after discovery returns real groups.
9. Add tests using captured stock MeshCore TCP or USB serial frames.

## Operator note

For now, use the original MeshCore UDP backend for working MeshCore operation.

The TCP API backend may connect to a TCP port, but stock MeshCore frames are not expected to decode correctly until this protocol work is done.

See also:

- [MeshCore Backends](MeshCore-Backends.md)
- [Backend Configuration](Backend-Configuration.md)
- [Configuring Channels](Configuring-Channels.md)
