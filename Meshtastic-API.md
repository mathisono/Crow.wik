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

Current implementation commit:

```text
c62497673449a0056ed4e79ef2866ba0487efd92
```

That commit added read-only Meshtastic API channel discovery to `meshtastic_API.uc`.

## Current support

The current `meshtastic_API.uc` backend supports experimental TCP Port-API operation:

- connects to a Meshtastic node over TCP;
- default TCP Port-API port: `4403`;
- reads Meshtastic Port-API frames using the `0x94 0xc3` frame header;
- decodes `FromRadio.packet` into Crow-style Meshtastic messages;
- sends Crow-originated text back through a `ToRadio.packet` envelope;
- keeps the original `meshtastic.uc` UDP/multicast backend unchanged;
- supports read-only Meshtastic channel discovery when explicitly enabled;
- supports periodic read-only channel refresh when explicitly enabled.

Example experimental config:

```json
{
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

Defaults:

- `channel_discovery`: `false`
- `channel_sync`: `"off"`
- `channel_refresh_seconds`: `600`

## Important deployment note

`meshtastic_API.uc` is **not imported by `router.uc` by default**.

The production router path still uses:

```ucode
import * as meshtastic from "meshtastic";
```

That means this backend is safe to keep in the tree while testing. To test it on hardware, temporarily wire the experimental backend only for the test, then revert that wiring unless the project explicitly decides to switch production routing.

## Auto channel discovery

Current status: **implemented in code as experimental read-only runtime discovery**.

When `meshtastic_api.channel_discovery` is enabled, the backend sends a `ToRadio.want_config_id` request after connecting. The request asks the Meshtastic node to send its config state. The backend then scans incoming `FromRadio.channel` frames and extracts:

- channel index;
- channel name;
- PSK bytes;
- Crow-compatible runtime `namekey` form.

The backend stores discovered channels in an internal runtime map named:

```ucode
let discoveredChannels = {};
```

Important limits:

- discovery is disabled by default;
- discovered channels are runtime-only for now;
- Crow config files are not changed;
- `/etc/crow.conf` is not changed;
- override files are not changed;
- raw PSKs are not printed to logs;
- router imports are not changed by default.

## Channel refresh

Current status: **implemented in code as experimental read-only periodic refresh**.

When `channel_discovery` is enabled, the backend can periodically resend `ToRadio.want_config_id`. The refresh interval is controlled by:

```json
"channel_refresh_seconds": 600
```

The refresh uses Crow's existing timer pattern rather than a blocking loop.

Expected refresh behavior:

- no config request is sent when `channel_discovery` is false;
- config request is sent after connect when discovery is true;
- config request is sent again after reconnect;
- config request is sent on the refresh timer while connected.

## Channel sync

Current status: **read-only runtime refresh only**.

The backend does not perform persistent or bidirectional channel sync yet.

It does **not** currently:

- push Crow channel changes back to the Meshtastic node;
- write channel updates to persistent Crow config;
- overwrite operator-managed Crow channels;
- auto-enable routing over newly discovered channels without validation;
- use admin/channel-set requests;
- reboot or reconfigure the physical radio.

## Protobuf tags used

The backend uses targeted protobuf TLV parsing for only the fields needed for packet RX/TX and channel discovery.

| Object | Field | Tag |
| --- | ---: | --- |
| `FromRadio.packet` | 2 | `0x12` |
| `FromRadio.config_complete_id` | 7 | `0x38` |
| `FromRadio.channel` | 10 | `0x52` |
| `ToRadio.packet` | 1 | `0x0A` |
| `ToRadio.want_config_id` | 3 | `0x18` |
| `Channel.index` | 1 | `0x08` |
| `Channel.settings` | 2 | `0x12` |
| `ChannelSettings.name` | 3 | `0x1A` |
| `ChannelSettings.psk` | 4 | `0x22` |

The two important corrected assumptions are:

- `FromRadio.channel` is field 10 / tag `0x52`, not field 3.
- `ToRadio.packet` is field 1 / tag `0x0A`, not field 2.

## Expected log behavior

When discovery is enabled, useful logs should show:

```text
meshtastic_API: connected tcp-port-api HOST:4403
meshtastic_API: config request sent id=... reason=connect
meshtastic_API: channel discovered index=... name=... pskfp=...
meshtastic_API: config complete id=... last_request=...
```

Logs must **not** print raw PSKs.

`pskfp` is a short fingerprint only, not the key itself.

## Hardware validation status

Status: **needs real-node validation**.

The code is committed, but it still needs testing against a real Meshtastic node over TCP Port-API.

Minimum validation checklist:

1. Enable `meshtastic_api.channel_discovery=true`.
2. Manually wire the experimental backend only for the test.
3. Confirm TCP connect to the node.
4. Confirm `want_config_id` is sent after connect.
5. Confirm the node sends `FromRadio.channel` records.
6. Confirm channel names and indexes are logged.
7. Confirm raw PSKs are not logged.
8. Confirm normal inbound text still routes.
9. Confirm outbound text still sends.
10. Confirm reconnect triggers another config request.
11. Confirm no persistent Crow config file is modified.

## Suggested test commands

From the Crow repo root:

```sh
# Confirm production router still imports the UDP backend
grep -n 'import \* as meshtastic from "meshtastic"' router.uc
! grep -n 'import \* as meshtastic from "meshtastic_API"' router.uc

# Confirm the old backend is still UDP/multicast-oriented
grep -n '224.0.0.69\|SOCK_DGRAM\|IP_ADD_MEMBERSHIP' meshtastic.uc

# Confirm discovery code exists in the experimental backend
grep -n 'channel_discovery\|want_config_id\|discoveredChannels\|extractChannels\|FROMRADIO_CHANNEL_TAG' meshtastic_API.uc

# Confirm TCP API-only envelopes are not registered in the UDP proto file
! grep -n 'fromradio\|toradio\|want_config_id\|config_complete_id' meshtasticprotobufs.uc

# Syntax checks where ucode is available
ucode -R -L . meshtastic_API.uc
ucode -R -L . meshtastic.uc
ucode -R -L . router.uc
```

Node reachability check:

```sh
nc -vz 192.168.4.1 4403
```

Replace `192.168.4.1` with the Meshtastic node IP.

## Config-file safety check

Before and after testing, compare config file timestamps and hashes.

Example:

```sh
ls -l /etc/crow.conf /etc/crow.conf.override 2>/dev/null
sha256sum /etc/crow.conf /etc/crow.conf.override 2>/dev/null
```

Expected result:

- read-only discovery does not modify these files;
- discovered channels remain runtime-only;
- no operator-managed channel is overwritten.

## Future work

Future work after read-only discovery is validated:

1. Add a safe runtime helper for registering discovered channels into Crow's broader channel memory.
2. Add operator UI visibility for discovered channels.
3. Add explicit operator confirmation before any radio write.
4. Implement `encodeChannelProto(index, name, psk)` only after hardware validation.
5. Add admin/channel-set request support only after firmware compatibility testing.
6. Add ACK/config-complete verification for any future write path.
7. Consider bidirectional sync only after read-only discovery is stable.

## Release note

Document this as **experimental read-only discovery**, not full channel sync. Persistent channel sync and bidirectional writes are not supported yet.
