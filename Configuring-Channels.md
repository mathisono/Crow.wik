# Configuring Channels

This page is the operator how-to for Crow channel setup.

Start with the backend/bridge setup first, then join or configure channels. The order matters: Crow needs to know which bridge backends are active before channel membership and routing behavior make sense.

## Configuration order

Use this order when setting up a node:

1. [Initial setup](#initial-setup)
2. [Bridges](#bridges)
3. [Meshtastic UDP Bridge](#meshtastic-udp-bridge)
4. [MeshCore UDP Bridge](#meshcore-udp-bridge)
5. [APRS Backend](#aprs-backend)
6. [Experimental: Meshtastic API Backend](#experimental-meshtastic-api-backend)
7. [Experimental: MeshCore TCP API Backend with Auto-Discovery](#experimental-meshcore-tcp-api-backend-with-auto-discovery)
8. [Joining channels](#joining-channels)
9. [Validation](#validation)

## Initial setup

Crow ships with bridge/channel concepts for the default bridge paths. Name these clearly before changing backend configuration.

The default bridge channels are:

| Default bridge channel | What it means | Default backend |
| --- | --- | --- |
| **Meshtastic UDP Device Bridge Channel** | Channel/path used for the original Meshtastic UDP device bridge. | `meshtastic.uc` |
| **MeshCore UDP Device Bridge Channel** | Channel/path used for the original MeshCore UDP device bridge. | `meshcore.uc` |
| **APRS Backend Channel** | Channel/path used for APRS/KISS/APRS-IS backend traffic when APRS is enabled. | APRS backend config |

These are bridge/channel roles, not proof that a user has joined a chat channel yet. Configure the bridge backend first, then join or map specific operator channels.

## Bridges

Crow can bridge messages between several transport families. Start here before joining channels.

| Bridge section | Backend module | Status | Default? |
| --- | --- | --- | --- |
| [Meshtastic UDP Bridge](#meshtastic-udp-bridge) | `meshtastic.uc` | Legacy UDP/multicast device bridge | Yes |
| [MeshCore UDP Bridge](#meshcore-udp-bridge) | `meshcore.uc` | Legacy UDP/multicast device bridge | Yes |
| [APRS Backend](#aprs-backend) | APRS/KISS/APRS-IS backend config | Operator-configured APRS bridge | Optional |
| [Experimental: Meshtastic API Backend](#experimental-meshtastic-api-backend) | `meshtastic_API.uc` | Experimental TCP Port-API backend | No |
| [Experimental: MeshCore TCP API Backend with Auto-Discovery](#experimental-meshcore-tcp-api-backend-with-auto-discovery) | `meshcore_tcp_api.uc` | Experimental TCP API backend | No |

The bridge setup layer decides **how Crow talks to each transport**. Channel setup decides **which message groups/keys are allowed to route**.

The router imports selector modules:

```ucode
import * as meshtastic from "meshtastic_backend";
import * as meshcore from "meshcore_backend";
```

The selector modules preserve the original UDP defaults while allowing API backends to be enabled for testing.

Default behavior:

- `meshtastic.enabled=true` selects the **Meshtastic UDP Device Bridge Channel** through `meshtastic.uc`.
- `meshcore.enabled=true` selects the **MeshCore UDP Device Bridge Channel** through `meshcore.uc`.
- APRS is independent and is enabled through APRS config.
- API backends are not selected unless explicitly requested.
- Only one backend per bridge family should be active at a time.

## Meshtastic UDP Bridge

This is the original Meshtastic backend and remains the default compatibility path.

```text
Bridge channel: Meshtastic UDP Device Bridge Channel
Backend module: meshtastic.uc
Transport: UDP/multicast
Port: 4403
```

Use it when you want Crow to behave like the original UDP Meshtastic bridge.

```json
{
  "meshtastic": {
    "enabled": true
  }
}
```

Expected selector log:

```text
meshtastic_backend: selected udp backend
```

## MeshCore UDP Bridge

This is the original MeshCore backend and remains the default compatibility path.

```text
Bridge channel: MeshCore UDP Device Bridge Channel
Backend module: meshcore.uc
Transport: UDP/multicast
Port: 4402
```

Use it when you want Crow to behave like the original UDP MeshCore bridge.

```json
{
  "meshcore": {
    "enabled": true
  }
}
```

Expected selector log:

```text
meshcore_backend: selected udp backend
```

## APRS Backend

APRS is independent of the Meshtastic and MeshCore backend selectors. Configure APRS separately using the APRS/KISS/APRS-IS settings for the deployed build.

```text
Bridge channel: APRS Backend Channel
Backend: APRS/KISS/APRS-IS config
Transport: APRS-IS, KISS TCP, or local TNC depending on deployment
```

Example placeholder shape:

```json
{
  "aprs": {
    "enabled": true
  }
}
```

Operator notes:

- APRS should continue to work whether Meshtastic/MeshCore are using UDP or API backends.
- APRS should be included in regression tests after changing LoRa backend selection.
- Do not mix APRS channel behavior with Meshtastic/MeshCore backend selection; they are separate layers.

See also: [APRS Bridge](APRS).

## Experimental: Meshtastic API Backend

This is the experimental Meshtastic TCP Port-API backend.

```text
Backend module: meshtastic_API.uc
Transport: TCP Port-API
Default Port-API port: 4403
Status: experimental / opt-in
Discovery: read-only runtime channel discovery
```

Use it only when deliberately testing the TCP API path.

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

Important Meshtastic API notes:

- `channel_discovery` is disabled by default.
- `channel_sync` currently supports `off` or `read_only` only.
- Read-only discovery does not write Crow config files.
- Read-only discovery does not push channel configuration back to the radio.
- Raw PSKs must not be logged.
- Do not run Meshtastic UDP and Meshtastic API at the same time unless explicitly testing selector failure handling.

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

Accepted API selector values:

```text
api
tcp-api
port-api
```

For field testing, the clearer and safer pattern is still to set `meshtastic.enabled=false` and `meshtastic_api.enabled=true`.

## Experimental: MeshCore TCP API Backend with Auto-Discovery

This is the experimental MeshCore TCP API backend.

```text
Backend module: meshcore_tcp_api.uc
Transport: TCP API
Status: experimental / opt-in
Discovery: companion/API discovery where supported by the backend
```

Use it only when deliberately testing the TCP API path.

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

Use `127.0.0.1` when the MeshCore companion/API service is on the same node. Use the service IP if it is reachable over the network.

Expected log:

```text
meshcore_backend: selected tcp-api backend
```

Validation command:

```sh
logread -f | grep -Ei 'crow|meshcore_backend|meshcore_tcp_api|meshcore'
```

Important MeshCore API notes:

- MeshCore UDP should be disabled while testing MeshCore TCP API.
- The TCP API backend should fail/reconnect cleanly if the service is unavailable.
- Do not run MeshCore UDP and MeshCore TCP API at the same time unless explicitly testing selector failure handling.

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

Accepted API selector values:

```text
api
tcp-api
companion-api
```

For field testing, the clearer and safer pattern is still to set `meshcore.enabled=false` and `meshcore_tcp_api.enabled=true`.

## Joining channels

After the backend/bridge setup is correct, configure or join channels.

Channels determine where messages are allowed to route. Backends determine how Crow talks to each radio/transport.

### Join by command

If the running build supports slash commands, use the channel join command from chat/UI:

```text
/join #channel-name key=passphrase
```

For MeshCore group/channel testing, a slot-based group may need to be mapped before messages can route to a Crow channel.

### Manual channel config pattern

A Crow channel entry normally needs enough information to identify:

- channel name;
- `namekey`;
- symmetric key;
- any Meshtastic or MeshCore hash/index mapping used by the backend.

Example conceptual channel record:

```json
{
  "name": "AREDN-Local",
  "namekey": "AREDN-Local <base64-key>",
  "access_control": {
    "require_callsign": true,
    "allow": ["KJ6DZB", "W6*"],
    "deny": []
  }
}
```

Exact channel object shape should follow the current Crow config schema in the deployed build.

### Meshtastic API discovered channels

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

## Validation

After changing backend or channel configuration, restart Crow:

```sh
/etc/init.d/crow restart
```

Watch logs:

```sh
logread -f | grep -Ei 'crow|meshtastic_backend|meshcore_backend|meshtastic_API|meshcore_tcp_api|aprs'
```

Confirm selected backends:

```text
meshtastic_backend: selected udp backend
meshcore_backend: selected udp backend
```

or, when testing APIs:

```text
meshtastic_backend: selected tcp-port-api backend
meshcore_backend: selected tcp-api backend
```

Validation checklist:

1. Crow starts without crashing.
2. Only the intended backend for each family is selected.
3. UDP backends remain default when legacy blocks are enabled.
4. API backends connect or fail cleanly.
5. APRS behavior is unchanged.
6. Strict Gatekeeper still filters bridged LoRa traffic.
7. Joined channels route as expected.
8. No raw PSKs appear in logs.
9. Read-only discovery does not modify persistent config.

## Quick mode examples

Both legacy UDP backends:

```json
{
  "meshtastic": { "enabled": true },
  "meshcore": { "enabled": true }
}
```

Meshtastic API + MeshCore UDP:

```json
{
  "meshtastic": { "enabled": false },
  "meshtastic_api": {
    "enabled": true,
    "host": "192.168.4.1",
    "port": 4403,
    "channel_discovery": true,
    "channel_sync": "read_only"
  },
  "meshcore": { "enabled": true }
}
```

Meshtastic UDP + MeshCore API:

```json
{
  "meshtastic": { "enabled": true },
  "meshcore": { "enabled": false },
  "meshcore_tcp_api": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 4403
  }
}
```

Both API backends:

```json
{
  "meshtastic": { "enabled": false },
  "meshtastic_api": {
    "enabled": true,
    "host": "192.168.4.1",
    "port": 4403,
    "channel_discovery": true,
    "channel_sync": "read_only"
  },
  "meshcore": { "enabled": false },
  "meshcore_tcp_api": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 4403
  }
}
```

## Safety notes

- Backend setup comes before channel joining.
- The default bridge channels are the Meshtastic UDP Device Bridge Channel, MeshCore UDP Device Bridge Channel, and APRS Backend Channel.
- UDP remains the compatibility default for Meshtastic and MeshCore.
- API backends are experimental until field validation passes.
- Only one backend per transport family should be active.
- Do not push channel config to radios until an explicit write-sync phase is designed and tested.
