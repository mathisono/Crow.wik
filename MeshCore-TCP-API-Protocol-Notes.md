# MeshCore TCP API Protocol Notes

This page records the protocol correction for Crow's experimental MeshCore TCP API work.

## Current status

Crow's `meshcore_tcp_api.uc` has been corrected to use the stock MeshCore TCP/Wi-Fi and USB serial outer framing model.

It should still be treated as experimental because outbound text send is not implemented yet and field testing with real MeshCore devices or captured frames is still needed.

## Correct stock outer framing

Stock MeshCore TCP/Wi-Fi serial framing uses direction markers and little-endian length fields:

```text
Radio -> client: [ '>' ][ length LSB ][ length MSB ][ frame payload ]
Client -> radio: [ '<' ][ length LSB ][ length MSB ][ frame payload ]
```

The USB serial interface uses the same `>` / `<` framing model.

Crow now builds and parses frames using that model:

- inbound parser looks for `>`
- inbound length is 2-byte little-endian
- outbound command helper sends `<`
- outbound command length is 2-byte little-endian
- frame payload begins with the MeshCore command/response/push code

## Message receive model

Crow now follows the queued receive model:

```text
0x83 = PUSH_CODE_MSG_WAITING
client sends CMD_SYNC_NEXT_MESSAGE = 0x0A
radio returns queued message response
client decodes response code 0x07 / 0x08 / 0x10 / 0x11
```

Handled codes:

| Code | Role | Current Crow behavior |
|---:|---|---|
| `0x83` | message waiting push/tickle | counted and triggers `CMD_SYNC_NEXT_MESSAGE = 0x0A` |
| `0x0A` | client command to fetch/sync the next queued message | sent through `sendCommand()` |
| `0x07` | older direct-message receive response | decoded as direct text |
| `0x08` | older channel/group-message receive response | decoded as group text with slot metadata |
| `0x10` | newer v3 direct-message receive response | accepted and decoded through the current direct-text envelope |
| `0x11` | newer v3 channel/group-message receive response | accepted and decoded through the current group-text envelope |

If firmware v3 adds fields before the text body, `decodeTextFrame()` is the place to split v3 handling further.

## Discovery status

`meshcore_tcp_discovery.uc` now sends channel discovery requests through the corrected TCP API command helper.

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

The TCP backend caches `0x12` responses, and discovery drains those cached responses through `takeResponse(0x12)`.

Because the TCP socket is non-blocking, discovery is still effectively asynchronous:

1. `queryDeviceGroups()` sends `CMD_GET_CHANNEL = 0x1F` for slots `0-7`.
2. `meshcore_tcp_api.uc` receives and caches `0x12` responses as they arrive.
3. A later discovery sync can parse cached responses and register slot/channel mappings.

## Implemented checklist

Implemented in Crow code:

- stock `>` / `<` frame markers
- little-endian 2-byte frame lengths
- client-to-radio `sendCommand()` helper
- `0x83` message-waiting handling
- `0x0A` sync-next-message command send
- queued response decode for `0x07`, `0x08`, `0x10`, and `0x11`
- response cache for `0x12` channel info
- discovery command send for `CMD_GET_CHANNEL = 0x1F`
- channel registration helper export from `channel.uc`
- updated ucode and Node mirror tests for stock framing

Still needed:

- field test with a stock MeshCore TCP/Wi-Fi or USB serial interface
- captured-frame regression tests from real hardware
- outbound text send for the TCP API backend
- deeper v3 response parsing if `0x10` / `0x11` payloads differ from the current decoded envelope

## Operator note

For now, use the original MeshCore UDP backend for production MeshCore operation.

The TCP API backend is now aligned with the stock outer framing and queued receive model, but it should remain experimental until it is tested against real MeshCore firmware.

See also:

- [MeshCore Backends](MeshCore-Backends.md)
- [Backend Configuration](Backend-Configuration.md)
- [Configuring Channels](Configuring-Channels.md)
