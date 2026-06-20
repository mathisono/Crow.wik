# Backend Selection and Test Deployment

Crow now supports backend selector modules for LoRa transports.

The goal is to keep the original UDP backends as the default while allowing new API backends to be enabled for testing and validation.

## Current selector modules

```text
meshtastic_backend.uc
meshcore_backend.uc
```

`router.uc` imports the selectors:

```ucode
import * as meshtastic from "meshtastic_backend";
import * as meshcore from "meshcore_backend";
```

The selectors then choose which concrete backend to use.

## Available backend modules

| Transport family | Legacy/default backend | New experimental backend |
| --- | --- | --- |
| Meshtastic | `meshtastic.uc` UDP/multicast | `meshtastic_API.uc` TCP Port-API |
| MeshCore | `meshcore.uc` UDP/multicast | `meshcore_tcp_api.uc` TCP Companion/API |

## Default behavior

The original UDP backends remain the default.

If the normal legacy blocks are enabled, Crow should use:

```text
Meshtastic -> meshtastic.uc
MeshCore   -> meshcore.uc
```

Example default UDP config:

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

## Opting into the Meshtastic API backend

Use this form when testing the new Meshtastic TCP Port-API backend:

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

Alternative explicit selector form:

```json
{
  "meshtastic": {
    "enabled": true,
    "backend": "api"
  },
  "meshtastic_api": {
    "enabled": true,
    "host": "192.168.4.1",
    "port": 4403
  }
}
```

## Opting into the MeshCore API backend

Use this form when testing the new MeshCore TCP Companion/API backend:

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

Alternative explicit selector form:

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

## Safety rules

- Keep UDP as the default until the API backends pass real-node validation.
- Only one backend per transport family should be active at a time.
- Do not enable Meshtastic UDP and Meshtastic API at the same time during validation.
- Do not enable MeshCore UDP and MeshCore API at the same time during validation.
- Do not write Meshtastic channel changes back to the radio yet.
- Meshtastic API channel discovery is read-only and runtime-only.
- Do not log raw PSKs.
- Do not treat API backend validation as a production switch until UDP regression tests still pass.

## Build artifacts

The AREDN build script copies all root `*.uc` files into the package, so the selector modules are included automatically:

```text
meshtastic_backend.uc
meshcore_backend.uc
```

Expected package outputs:

```text
crow_alpha.ipk
crow-alpha.apk
```

## Pre-deploy local checks

Run these from the Crow repo root before pushing to nodes:

```sh
# Confirm selector imports
 grep -n 'meshtastic_backend\|meshcore_backend' router.uc

# Confirm backend selector files exist
ls -l meshtastic_backend.uc meshcore_backend.uc

# Confirm legacy UDP backends still exist
ls -l meshtastic.uc meshcore.uc

# Confirm API backends still exist
ls -l meshtastic_API.uc meshcore_tcp_api.uc

# Confirm build packaging copies root ucode files
 grep -n 'cp \$SRC/\*.uc' platforms/aredn/build.sh

# Syntax checks where ucode is available
ucode -R -L . router.uc
ucode -R -L . meshtastic_backend.uc
ucode -R -L . meshcore_backend.uc
ucode -R -L . meshtastic.uc
ucode -R -L . meshtastic_API.uc
ucode -R -L . meshcore.uc
ucode -R -L . meshcore_tcp_api.uc

# Package script checks
sh -n platforms/aredn/build.sh platforms/aredn/postinst platforms/aredn/postinstall platforms/aredn/postupgrade platforms/aredn/prerm
```

## Build

From the Crow repo root:

```sh
cd platforms/aredn
sh ./build.sh
ls -lh crow_alpha.ipk crow-alpha.apk
```

## Deploy to an AREDN test node

Replace `NODE_IP` with the test node IP.

```sh
scp platforms/aredn/crow_alpha.ipk root@NODE_IP:/tmp/crow_alpha.ipk
ssh root@NODE_IP 'opkg install /tmp/crow_alpha.ipk || opkg install --force-reinstall /tmp/crow_alpha.ipk'
ssh root@NODE_IP '/etc/init.d/crow restart'
ssh root@NODE_IP 'logread -f | grep -Ei "crow|meshtastic_backend|meshcore_backend|meshtastic_API|meshcore_tcp_api"'
```

For APK-based targets, use `crow-alpha.apk` and the node's package tooling instead of `opkg`.

## Validation matrix

Test these modes before calling the selector work stable:

| Mode | Meshtastic config | MeshCore config | Expected |
| --- | --- | --- | --- |
| Legacy default | `meshtastic.enabled=true` | `meshcore.enabled=true` | UDP backends selected |
| Meshtastic API | `meshtastic.enabled=false`, `meshtastic_api.enabled=true` | `meshcore.enabled=true` | Meshtastic API + MeshCore UDP |
| MeshCore API | `meshtastic.enabled=true` | `meshcore.enabled=false`, `meshcore_tcp_api.enabled=true` | Meshtastic UDP + MeshCore API |
| Both API | `meshtastic.enabled=false`, `meshtastic_api.enabled=true` | `meshcore.enabled=false`, `meshcore_tcp_api.enabled=true` | Both API backends selected |
| Disabled Meshtastic | `meshtastic.enabled=false`, no API | any MeshCore config | No Meshtastic socket |
| Disabled MeshCore | any Meshtastic config | `meshcore.enabled=false`, no API | No MeshCore socket |

For each mode verify:

- Crow starts without crashing.
- Logs show the selected backend.
- Only the selected backend opens sockets.
- RX works where hardware is present.
- TX works where implemented.
- Strict Gatekeeper still applies to bridged LoRa traffic.
- UDP regression still passes after API tests.

## Rollback

If a node fails after installing the selector build:

```sh
ssh root@NODE_IP '/etc/init.d/crow stop'
ssh root@NODE_IP 'cp /usr/local/crow/crow.conf /tmp/crow.conf.failed 2>/dev/null || true'
ssh root@NODE_IP 'opkg remove crow'
```

Then reinstall the last known-good package.

## Release note

The selector layer is intended to preserve compatibility. The old UDP backends should remain the default until field testing proves the new API backends are ready for normal operation.
