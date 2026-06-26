# Configuration

This page documents the operator-facing Crow configuration patterns for selecting backends and testing the TCP/API backends.

Crow uses backend selector modules so the original UDP backends remain the default while API backends can be enabled explicitly.

## Backend selector model

`router.uc` imports selector modules:

```ucode
import * as meshtastic from "meshtastic_backend";
import * as meshcore from "meshcore_backend";
```

The selector modules choose the concrete backend from config.

| Transport | Default backend | Experimental API backend |
| --- | --- | --- |
| Meshtastic | `meshtastic.uc` UDP/multicast | `meshtastic_API.uc` TCP Port-API |
| MeshCore | `meshcore.uc` UDP/multicast | `meshcore_tcp_api.uc` TCP Companion API |

Default rule:

- If the normal `meshtastic` block is enabled, Crow uses Meshtastic UDP.
- If the normal `meshcore` block is enabled and no explicit API selector is set, Crow uses MeshCore UDP.
- API backends are opt-in.
- For test deployments, disable the UDP block before enabling the matching API block, unless using the explicit `backend` selector.

## Keep original UDP backends enabled

Use this when you want the original compatibility behavior.

```json
{
  "meshtastic": {
    "enabled": true
  },
  "meshcore": {
    "enabled": true,
    "bridgekey": "REPLACE_WITH_MESHCORE_BRIDGE_PUBLIC_KEY"
  }
}
```

Expected logs:

```text
meshtastic_backend: selected udp backend
meshcore_backend: selected udp backend
```

## Meshtastic TCP Port-API backend

Use this only for Meshtastic API testing.

```json
{
  "meshtastic": {
    "enabled": false
  },
  "meshtastic_api": {
    "enabled": true,
    "host": "192.168.4.1",
    "port": 4403,
    "channel_discovery": true,
    "channel_sync": "read_only",
    "channel_refresh_seconds": 600
  }
}
```

Replace `192.168.4.1` with the Meshtastic node IP.

Expected logs:

```text
meshtastic_backend: selected tcp-port-api backend
meshtastic_API: connected tcp-port-api 192.168.4.1:4403
meshtastic_API: config request sent id=... reason=connect
```

Important:

- `channel_discovery` is disabled by default.
- `channel_sync` currently supports `off` or `read_only` only.
- Read-only discovery does not write Crow config files.
- Read-only discovery does not push channel config back to the Meshtastic radio.
- Raw PSKs must not be logged.

## MeshCore backends

See [MeshCore Backends](MeshCore-Backends.md) for the full operator and code-behavior page.

Crow has two MeshCore paths:

| Path | Module | Status |
|---|---|---|
| Original UDP bridge | `meshcore.uc` | Compatibility path; supports inbound and outbound MeshCore bridge packets. |
| TCP Companion API | `meshcore_tcp_api.uc` | Experimental receive/decode path for cleartext direct and group text frames. Outbound send is not implemented yet. |

The active path is selected by `meshcore_backend.uc`.

## MeshCore UDP backend

The UDP backend uses the original MeshCore bridge packet path:

```text
multicast: 224.0.0.69
port:      4402
```

Use this when you want the original behavior and outbound MeshCore sending.

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

Optional multicast interface binding:

```json
{
  "meshcore": {
    "enabled": true,
    "bridgekey": "REPLACE_WITH_MESHCORE_BRIDGE_PUBLIC_KEY",
    "address": "192.168.1.10"
  }
}
```

Expected selector log:

```text
meshcore_backend: selected udp backend
```

## MeshCore TCP Companion API backend

Use this section to test the experimental MeshCore TCP Companion API backend instead of the original MeshCore UDP backend.

### 1. Disable the original MeshCore UDP backend

Set the normal `meshcore` block to disabled:

```json
{
  "meshcore": {
    "enabled": false
  }
}
```

This prevents Crow from also opening the legacy MeshCore UDP/multicast socket.

### 2. Enable the MeshCore TCP API backend

Add a `meshcore_tcp_api` block:

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

Use `127.0.0.1` when the MeshCore companion service is on the same node. Use the radio/service IP when it is reachable over the network.

### 3. Restart Crow

On an AREDN/OpenWrt node:

```sh
/etc/init.d/crow restart
```

Watch logs:

```sh
logread -f | grep -Ei 'crow|meshcore_backend|meshcore_tcp_api|meshcore'
```

Expected selector log:

```text
meshcore_backend: selected tcp-api backend
```

Expected TCP API behavior:

- Crow tries to connect to the configured TCP Companion host/port.
- On connect, Crow sends a MeshCore Companion `HELLO` frame.
- The radio is expected to auto-push cleartext direct and group message frames.
- Direct message frames use command `0x07`.
- Group/channel message frames use command `0x08`.
- Decoded inbound messages enter Crow as `transport: "meshcore"` with `backend: "tcp_api"`.
- Reconnect failures should log cleanly and not crash Crow.

Current TCP API limitations:

- Outbound send is not implemented in `meshcore_tcp_api.uc`.
- Production outbound MeshCore traffic still requires the UDP backend.
- Unknown, oversized, malformed, encrypted, advert, telemetry, and handshake frames are dropped rather than routed.
- Automatic MeshCore group discovery is not complete yet.

### 4. Alternative explicit selector form

Instead of disabling the legacy block, you can explicitly request the API backend:

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

Accepted selector values for the MeshCore API path:

```text
api
tcp-api
companion-api
```

For field testing, the safer form is still:

```json
"meshcore": { "enabled": false }
```

plus:

```json
"meshcore_tcp_api": { "enabled": true, "host": "...", "port": 4403 }
```

That makes it obvious that only one MeshCore backend should run.

## MeshCore TCP API validation checklist

Run these checks after enabling `meshcore_tcp_api`.

### Confirm config selection

```sh
logread | grep -Ei 'meshcore_backend|meshcore_tcp_api'
```

Expected:

```text
meshcore_backend: selected tcp-api backend
```

### Confirm UDP backend is not selected

There should be no new log line saying:

```text
meshcore_backend: selected udp backend
```

for the same run.

### Confirm TCP reachability

If the TCP API host is not local, test it from the node:

```sh
nc -vz MESHCORE_API_HOST 4403
```

Replace `MESHCORE_API_HOST` with the configured host.

### Confirm inbound decode expectations

For TCP API testing, expect only cleartext text frames to route:

| Frame | Command | Current behavior |
|---|---:|---|
| direct text | `0x07` | decoded into Crow message |
| group/channel text | `0x08` | decoded into Crow message with group slot metadata |
| encrypted DM/blob | `0x90`, `0x91` | dropped early under strict mode; otherwise dropped as unknown |
| advert / telemetry / handshake / vendor frames | varies | dropped as unknown |
| oversize frame | payload > 256 bytes | dropped early |

### Confirm Crow stays up

```sh
/etc/init.d/crow status
logread | tail -100
```

Expected:

- no crash loop
- no syntax error
- reconnect attempts are bounded
- API failure does not break Meshtastic, APRS, or MeshIP

## MeshCore TCP discovery status

`meshcore_tcp_discovery.uc` contains the parser and planned slot-discovery flow for MeshCore group memory slots `0-7`.

The intended radio query is:

```text
CMD_GET_CHANNEL = 0x1F
```

The expected response is:

```text
PACKET_CHANNEL_INFO = 0x12
50-byte response
slot index 0-7
32-byte null-padded group name
16-byte secret key
```

Current limitation: `queryDeviceGroups()` still contains the slot loop and parser notes, but the actual radio command send is stubbed. It currently returns an empty group list. Do not treat MeshCore discovery as production-ready yet.

## Switching back to MeshCore UDP

To return to the original MeshCore UDP backend:

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

Expected log:

```text
meshcore_backend: selected udp backend
```

## Backend mode matrix

| Desired mode | Meshtastic config | MeshCore config |
| --- | --- | --- |
| Both legacy UDP | `meshtastic.enabled=true` | `meshcore.enabled=true` |
| Meshtastic API + MeshCore UDP | `meshtastic.enabled=false`, `meshtastic_api.enabled=true` | `meshcore.enabled=true` |
| Meshtastic UDP + MeshCore API | `meshtastic.enabled=true` | `meshcore.enabled=false`, `meshcore_tcp_api.enabled=true` |
| Both API backends | `meshtastic.enabled=false`, `meshtastic_api.enabled=true` | `meshcore.enabled=false`, `meshcore_tcp_api.enabled=true` |
| Disable Meshtastic | `meshtastic.enabled=false`, no API block | any MeshCore mode |
| Disable MeshCore | any Meshtastic mode | `meshcore.enabled=false`, no API block |

## Safety notes

- UDP remains the MeshCore compatibility path.
- MeshCore TCP Companion API is experimental.
- MeshCore TCP API outbound send is not implemented yet.
- Only one backend per transport family should be active.
- Keep raw PSKs and MeshCore secret keys out of logs.
- Do not treat discovery as persistent channel sync until live radio slot querying is implemented and tested.
- Do not bridge encrypted MeshCore traffic into AREDN.
