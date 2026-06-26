# Configuration

This page is the operator-facing overview of Crow configuration. It explains the main config blocks and points to the deeper pages that own each area.

Use this page when you need to answer:

```text
What config block controls this feature?
Where should I go for the detailed setup page?
```

For backend selector details, UDP/API transport setup, and validation commands, see [Backend Configuration](Backend-Configuration.md).

For channel creation, joining, AREDN-only channels, APRS groups, and channel mapping, see [Configuring Channels](Configuring-Channels.md).

## Configuration layers

Crow configuration is easiest to understand in layers:

| Layer | What it controls | Main page |
|---|---|---|
| Node identity | local callsign / gateway identity | this page, [Strict Gatekeeper](Strict-Gatekeeper.md) |
| Backends | how Crow talks to Meshtastic, MeshCore, APRS, and API transports | [Backend Configuration](Backend-Configuration.md) |
| Channels | which message groups, keys, and channel mappings Crow routes | [Configuring Channels](Configuring-Channels.md) |
| Bridge safety | fail-closed handling for bridged LoRa traffic | [Strict Gatekeeper](Strict-Gatekeeper.md) |
| Storage | internal flash, USB storage, image quota, degraded mode | [USB Storage](USB-Storage.md), [Memory Use](Memory-Use.md) |
| Message stores | node text/message stores and retained channel history | [Text Stores](Text-Stores.md) |
| Forms | Winlink-style form inventory and storage | [Winlink](Winlink.md) |

## Common config files

Crow deployments commonly use a base config plus an override file. The exact file location depends on the platform package, but the practical rule is:

- keep package defaults in the base config;
- put local/operator changes in the override config;
- keep secrets and local callsign choices out of generic docs/examples.

On AREDN/OpenWrt nodes, restart Crow after changing config:

```sh
/etc/init.d/crow restart
```

Then watch logs:

```sh
logread -f | grep -Ei 'crow|meshtastic|meshcore|aprs|gatekeeper|storage'
```

## Minimal node identity

Most real deployments should set a callsign at the top level.

```json
{
  "callsign": "KJ6DZB"
}
```

If Strict Gatekeeper is enabled, `strict_gatekeeper.gateway_callsign` can override the top-level callsign for gateway annotation.

See: [Strict Gatekeeper](Strict-Gatekeeper.md).

## Top-level config blocks

| Block | Purpose | Detailed page |
|---|---|---|
| `callsign` | Local node/gateway callsign. | this page, [Strict Gatekeeper](Strict-Gatekeeper.md) |
| `channels` | Local channel list, namekeys, channel metadata, optional backend mappings. | [Configuring Channels](Configuring-Channels.md) |
| `meshtastic` | Original Meshtastic UDP backend and selector options. | [Backend Configuration](Backend-Configuration.md), [Meshtastic API Backend](Meshtastic-API.md) |
| `meshtastic_api` | Experimental Meshtastic TCP Port-API backend. | [Backend Configuration](Backend-Configuration.md), [Meshtastic API Backend](Meshtastic-API.md) |
| `meshcore` | Original MeshCore UDP backend and selector options. | [Backend Configuration](Backend-Configuration.md), [MeshCore Backends](MeshCore-Backends.md) |
| `meshcore_tcp_api` | Experimental MeshCore TCP Companion API backend. | [Backend Configuration](Backend-Configuration.md), [MeshCore Backends](MeshCore-Backends.md) |
| `aprs` | APRS-IS, KISS TCP, APRS groups, APRS backend selection. | [APRS Bridge](APRS.md), [Backend Configuration](Backend-Configuration.md) |
| `strict_gatekeeper` | Fail-closed inbound bridge policy and gateway annotation. | [Strict Gatekeeper](Strict-Gatekeeper.md) |
| `storage` | Internal/USB storage mode, mount point, quota, degraded mode. | [USB Storage](USB-Storage.md), [Memory Use](Memory-Use.md) |
| `messages` | Message behavior, including RAM mode where supported. | [Memory Use](Memory-Use.md) |
| `textstore` | Node message/text stores for retained channel history. | [Text Stores](Text-Stores.md) |
| `winlink` | Winlink-style form behavior and form storage. | [Winlink](Winlink.md) |

## Minimal backend examples

For the original compatibility path:

```json
{
  "callsign": "KJ6DZB",
  "meshtastic": {
    "enabled": true
  },
  "meshcore": {
    "enabled": true,
    "bridgekey": "REPLACE_WITH_MESHCORE_BRIDGE_PUBLIC_KEY"
  }
}
```

For TCP/API backend testing, do not put the long setup here. Use [Backend Configuration](Backend-Configuration.md) so the selector details, limitations, and validation commands stay in one place.

## Minimal channel example

Channels determine which message groups/keys are available after the backend is running.

```json
{
  "channels": [
    { "namekey": "AREDN og==", "telemetry": false }
  ]
}
```

For channel naming, shared-key channels, AREDN-only channels, APRS groups, and MeshCore slot mapping, see [Configuring Channels](Configuring-Channels.md).

## Strict Gatekeeper example

Use Strict Gatekeeper on public, event, or unattended bridge gateways.

```json
{
  "callsign": "W6XYZ",
  "strict_gatekeeper": {
    "enabled": true,
    "gateway_callsign": "W6XYZ",
    "allowed_callsigns": ["KN6PLV", "KJ6DZB"]
  }
}
```

See: [Strict Gatekeeper](Strict-Gatekeeper.md).

## Storage example

USB storage is configured separately from channels and backends.

```json
{
  "storage": {
    "mode": "usb",
    "mountpoint": "/mnt/crow",
    "label": "CROWDATA",
    "image_quota_mb": 128,
    "min_free_mb": 16
  }
}
```

See: [USB Storage](USB-Storage.md) and [Memory Use](Memory-Use.md).

## Text store example

Node message stores are configured under `textstore`.

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

See: [Text Stores](Text-Stores.md).

## Validation checklist

After changing configuration:

1. Restart Crow.
2. Watch logs for parse errors or backend selection errors.
3. Confirm the intended backend is selected.
4. Confirm channels appear in the UI or `/channels` output.
5. Confirm Strict Gatekeeper behavior if bridge traffic is enabled.
6. Confirm no raw PSKs, MeshCore secret keys, or private local details are logged.

Useful log command:

```sh
logread -f | grep -Ei 'crow|meshtastic_backend|meshcore_backend|aprs|gatekeeper|storage'
```

## Where to go next

| Task | Go to |
|---|---|
| Configure Meshtastic UDP/API, MeshCore UDP/API, APRS backend selection | [Backend Configuration](Backend-Configuration.md) |
| Join or map channels | [Configuring Channels](Configuring-Channels.md) |
| Configure APRS-IS, KISS TCP, APRS groups, group repeat | [APRS Bridge](APRS.md) |
| Configure Meshtastic TCP Port-API discovery | [Meshtastic API Backend](Meshtastic-API.md) |
| Configure MeshCore UDP/TCP API behavior | [MeshCore Backends](MeshCore-Backends.md) |
| Configure storage or USB mode | [USB Storage](USB-Storage.md), [Memory Use](Memory-Use.md) |
| Configure node message stores | [Text Stores](Text-Stores.md) |
| Configure bridge safety | [Strict Gatekeeper](Strict-Gatekeeper.md) |
| Configure Winlink-style forms | [Winlink](Winlink.md) |
