# Configuration

This page documents the operator-facing Crow configuration patterns for selecting backends and testing the new TCP/API backends.

Crow now uses backend selector modules so the original UDP backends remain the default while new API backends can be enabled explicitly.

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
| MeshCore | `meshcore.uc` UDP/multicast | `meshcore_tcp_api.uc` TCP API |

Default rule:

- If the normal `meshtastic` block is enabled, Crow uses Meshtastic UDP.
- If the normal `meshcore` block is enabled, Crow uses MeshCore UDP.
- API backends are opt-in.
- Disable the UDP block before enabling the matching API block, unless using the explicit `backend` selector.

## Keep original UDP backends enabled

Use this when you want the original behavior.

```json
{
  "meshtastic": {
    "enabled": true
  },
  "meshcore": {
    "enabled": true
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

## MeshCore TCP API backend

Use this section to test the experimental MeshCore TCP API backend instead of the original MeshCore UDP backend.

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

Expected backend behavior:

- Crow should try to use `meshcore_tcp_api.uc`.
- MeshCore UDP should not also be active.
- Reconnect failures should log cleanly and not crash Crow.
- Inbound TCP API text should enter Crow as MeshCore transport traffic when supported by the backend.

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

For test deployments, the safer form is still:

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

### Confirm Crow stays up

```sh
/etc/init.d/crow status
logread | tail -100
```

Expected:

- no crash loop;
- no syntax error;
- reconnect attempts are bounded;
- API failure does not break Meshtastic, APRS, or MeshIP.

## Switching back to MeshCore UDP

To return to the original MeshCore UDP backend:

```json
{
  "meshcore": {
    "enabled": true
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

- UDP remains the default compatibility path.
- API backends are experimental until field validation passes.
- Only one backend per transport family should be active.
- Keep raw PSKs out of logs.
- Do not treat read-only discovery as persistent channel sync.
- Do not push channel config to radios until an explicit write-sync phase is designed and tested.
