# MeshCore Backends

Crow has two MeshCore backend paths:

1. the original MeshCore UDP/multicast backend, `meshcore.uc`
2. the experimental MeshCore TCP Companion API backend, `meshcore_tcp_api.uc`

The active backend is selected by `meshcore_backend.uc`.

## Backend selector

`router.uc` imports `meshcore_backend`, not the concrete UDP or TCP API module directly.

`meshcore_backend.uc` chooses one backend at startup:

| Condition | Selected backend |
|---|---|
| `meshcore.backend` or `meshcore.transport` is `api`, `tcp-api`, or `companion-api` | TCP Companion API backend |
| `meshcore_tcp_api.enabled=true` and `meshcore.enabled=false` or no `meshcore` block exists | TCP Companion API backend |
| `meshcore.enabled` is not false | UDP/multicast backend |
| none of the above | MeshCore disabled |

The selector logs one of:

```text
meshcore_backend: selected tcp-api backend
meshcore_backend: selected udp backend
meshcore_backend: disabled
```

## Original MeshCore UDP backend

The original backend is `meshcore.uc`.

It uses the MeshCore bridge packet format over UDP multicast:

```text
address: 224.0.0.69
port:    4402
```

When enabled, Crow:

- opens a UDP socket on port `4402`
- joins multicast group `224.0.0.69`
- optionally binds multicast to `meshcore.address`
- disables multicast loopback
- decodes native MeshCore packets into Crow messages
- sends Crow messages back out as MeshCore bridge packets
- supports direct messages, group text, adverts, and ACK/routing behavior that is implemented in `meshcore.uc`
- persists MeshCore shared-key material through `platform.store("meshcore.sharedkeys", ...)`

The UDP backend is still the compatibility path and the only MeshCore backend that currently implements outbound send.

### UDP config

```json
{
  "meshcore": {
    "enabled": true,
    "bridgekey": "REPLACE_WITH_MESHCORE_BRIDGE_PUBLIC_KEY"
  }
}
```

Optional address binding:

```json
{
  "meshcore": {
    "enabled": true,
    "bridgekey": "REPLACE_WITH_MESHCORE_BRIDGE_PUBLIC_KEY",
    "address": "192.168.1.10"
  }
}
```

## Outbound tagging with Strict Gatekeeper

Strict Gatekeeper and outbound LoRa gateway tags are related, but they are not the same step.

Strict Gatekeeper handles inbound bridge policy. When strict mode accepts Meshtastic or MeshCore ingress, it rewrites accepted text as:

```text
[SENDER via GATEWAY] original message
```

For MeshCore group messages with weak group identity, it rewrites accepted text as:

```text
[SENDER@MCGW-GroupName via GATEWAY] original message
```

Outbound LoRa gateway tagging is handled later, at the outbound backend wrapper layer, by `lora_outbound_text.uc` and `meshcore_tagged.uc`.

When the MeshCore tagged wrapper is wired in and a text message is sent out to MeshCore, the outbound text is formatted as:

```text
CALLSIGN@MCGW> message
```

For example, if Strict Gatekeeper has accepted and annotated a bridged message, and that message is later sent out over a tagged MeshCore backend, the LoRa-side text may look like:

```text
W6XYZ@MCGW> [KJ6DZB via W6XYZ] radio check from the hill
```

For a weak-identity MeshCore group bridge message, the text may look like:

```text
W6XYZ@MCGW> [KJ6DZB@MCGW-TacNet via W6XYZ] radio check
```

That gives the LoRa side two layers of attribution:

1. `W6XYZ@MCGW>` — the gateway/backend that emitted the LoRa packet
2. `[KJ6DZB via W6XYZ]` or `[KJ6DZB@MCGW-TacNet via W6XYZ]` — the sender/gateway annotation added by Strict Gatekeeper

Current implementation caveat: the tagged MeshCore wrapper is present in code, but `meshcore_backend.uc` currently imports the raw UDP backend as `meshcore`. Therefore outbound tagging is only active when the MeshCore tagged wrapper is explicitly wired in or the selector is updated to use it.

The MeshCore tagged wrapper:

```text
meshcore_tagged.uc
```

wraps only the outbound text path. It does not change UDP receive behavior, channel/key handling, or router-level gatekeeper behavior.

### MeshCore tag config

When using the tagged wrapper, these optional `meshcore` fields control tag selection and payload budget:

```json
{
  "meshcore": {
    "enabled": true,
    "bridgekey": "REPLACE_WITH_MESHCORE_BRIDGE_PUBLIC_KEY",
    "gateway_index": 0,
    "gateway_tag_max_payload": 150
  }
}
```

Expected tags:

| `gateway_index` | Tag |
|---:|---|
| `0` | `MCGW` |
| `1` | `MCG2` |
| `2` | `MCG3` |

See also: [LoRa Gateway Tags](LoRa-Gateway-Tags) and [Strict Gatekeeper](Strict-Gatekeeper).

## MeshCore TCP Companion API backend

The TCP API backend is `meshcore_tcp_api.uc`.

It speaks the MeshCore Companion Protocol over TCP, defaulting to:

```text
host: 127.0.0.1
port: 4403
```

This is not the Meshtastic TCP Port-API. Meshtastic and MeshCore both commonly use port `4403`, but their wire protocols are different.

| Backend | Port | Framing |
|---|---:|---|
| Meshtastic TCP Port-API | 4403 | protobuf stream with Meshtastic framing |
| MeshCore Companion API | 4403 | `0x3E` magic, command id, 2-byte big-endian length, binary payload |

### What the TCP API backend does now

The current TCP API backend is a receive/decode backend for cleartext text messages.

It currently:

- connects to the configured TCP host/port
- sends a MeshCore Companion `HELLO` frame on connect
- relies on the radio's auto-push model for direct and group message frames
- decodes direct message frames, command `0x07`
- decodes group/channel message frames, command `0x08`
- queues decoded text messages into Crow as `transport: "meshcore"`
- marks TCP API messages with `backend: "tcp_api"`
- sets `hop_limit: 1`
- reconnects on a 5-second timer
- uses a Smart Accumulator to reject unsafe or irrelevant frames before allocating payload buffers

### What the TCP API backend does not do yet

The current TCP API backend does **not** implement outbound send.

`meshcore_tcp_api.uc` currently logs send attempts as not implemented and returns `false`. Production outbound MeshCore traffic still uses the UDP backend.

The TCP API backend also does not currently decode adverts, telemetry, encrypted blobs, unknown vendor frames, or handshake responses into Crow messages.

## Smart Accumulator behavior

The TCP API backend has a defensive buffer/parser called the Smart Accumulator.

It drops frames early when:

| Gate | Trigger |
|---|---|
| oversize | payload length is greater than 256 bytes |
| encrypted early-drop | Strict Gatekeeper is on and the command is an encrypted/blocked command |
| unknown command | command is not direct message `0x07` or group message `0x08` |
| malformed text | the direct or group payload does not match the expected text structure |
| resync cap | garbage before the next `0x3E` magic byte exceeds the resync cap |

The TCP API backend does not run the full gatekeeper filter itself. It only uses a cached strict-mode hook for early encrypted-frame dropping. The normal router/gatekeeper path handles canonical inbound bridge filtering later.

## Direct TCP API frame

Command:

```text
0x07
```

Payload:

```text
sender node id      uint32 little-endian
recipient node id   uint32 little-endian
text length         uint8
text bytes          UTF-8 text
```

Crow decodes this into a message with:

```json
{
  "transport": "meshcore",
  "backend": "tcp_api",
  "metadata": {
    "is_group_message": false,
    "identity_strength": "strong"
  }
}
```

## Group TCP API frame

Command:

```text
0x08
```

Payload:

```text
sender node id      uint32 little-endian
group slot index    uint8, 0-7
text bytes          UTF-8 text, rest of payload
```

Crow decodes this into a message with:

```json
{
  "transport": "meshcore",
  "backend": "tcp_api",
  "group_slot": 0,
  "metadata": {
    "is_group_message": true,
    "group_slot": 0,
    "identity_strength": "weak",
    "symmetric_key": true,
    "requires_slot_lookup": true
  }
}
```

## MeshCore group slot mapping

Crow has support code for mapping MeshCore group memory slots to Crow channels.

`channel.uc` keeps:

```text
channelsByMeshcoreSlot[slot]
```

and exposes:

```ucode
getChannelByMeshcoreSlot(slot)
setMeshcoreSlotChannel(slot, channel)
```

`router.uc` also exposes a temporary helper:

```ucode
registerGroupChannel(slot, channelObj)
```

This is intended for testing slot-to-channel routing before full automatic discovery is complete.

## MeshCore TCP discovery status

`meshcore_tcp_discovery.uc` documents the intended group discovery flow:

- query MeshCore memory slots `0-7`
- use `CMD_GET_CHANNEL = 0x1F`
- parse a `PACKET_CHANNEL_INFO = 0x12` response
- response structure is expected to be exactly 50 bytes
- slot index is `0-7`
- group name is a 32-byte null-padded UTF-8 field
- secret key is 16 bytes

However, the actual slot query path is still a stub. `queryDeviceGroups()` currently iterates the slots but does not send the command to the radio yet, so it returns an empty list.

That means automatic MeshCore group discovery is not production-ready yet. Manual slot/channel registration or static channel configuration is still needed for testing.

## Recommended test configuration: UDP compatibility path

Use this when you want the original behavior.

```json
{
  "meshcore": {
    "enabled": true,
    "bridgekey": "REPLACE_WITH_MESHCORE_BRIDGE_PUBLIC_KEY"
  },
  "meshcore_tcp_api": {
    "enabled": false
  }
}
```

Expected log:

```text
meshcore_backend: selected udp backend
```

## Recommended test configuration: TCP API receive path

Use this for TCP Companion API receive/decode testing.

```json
{
  "meshcore": {
    "enabled": false
  },
  "meshcore_tcp_api": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 4403
  }
}
```

Expected log:

```text
meshcore_backend: selected tcp-api backend
meshcore_tcp_api: connected tcp companion 127.0.0.1:4403
```

## Explicit selector form

You can also force the API path with:

```json
{
  "meshcore": {
    "enabled": true,
    "backend": "api"
  },
  "meshcore_tcp_api": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 4403
  }
}
```

Accepted API selector values:

```text
api
tcp-api
companion-api
```

For field testing, the clearer form is still to disable the UDP block and enable `meshcore_tcp_api`.

## Validation checklist

### Confirm selector

```sh
logread | grep -Ei 'meshcore_backend|meshcore_tcp_api|meshcore'
```

Expected for UDP:

```text
meshcore_backend: selected udp backend
```

Expected for TCP API:

```text
meshcore_backend: selected tcp-api backend
```

### Confirm TCP reachability

```sh
nc -vz MESHCORE_API_HOST 4403
```

### Confirm current limitations

For TCP API mode, expect:

- inbound direct text messages may decode from `0x07`
- inbound group messages may decode from `0x08`
- outbound send is not implemented
- discovery may report no groups because radio slot querying is still stubbed
- unknown, oversized, malformed, and encrypted frames are dropped early

## Switching back to UDP

```json
{
  "meshcore": {
    "enabled": true,
    "bridgekey": "REPLACE_WITH_MESHCORE_BRIDGE_PUBLIC_KEY"
  },
  "meshcore_tcp_api": {
    "enabled": false
  }
}
```

Restart Crow:

```sh
/etc/init.d/crow restart
```

## Safety notes

- UDP remains the compatibility path.
- TCP API is experimental.
- TCP API outbound send is not implemented yet.
- Outbound LoRa gateway tags require the tagged wrapper path or a selector update that uses `meshcore_tagged.uc`.
- Do not treat discovery as complete until `queryDeviceGroups()` actually sends `CMD_GET_CHANNEL` to the radio and parses live responses.
- Keep raw keys out of logs.
- Do not bridge encrypted MeshCore traffic into AREDN.
