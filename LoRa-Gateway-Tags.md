# LoRa Gateway Tags

Crow uses short gateway tags when forwarding cleartext gateway-originated messages out to local LoRa networks such as MeshCore and Meshtastic.

The outbound text format is:

```text
CALLSIGN@TAG> message
```

Examples:

```text
KJ6DZB@MCGW> Hello local MeshCore
KJ6DZB@MCG2> Same message through second MeshCore backend
KJ6DZB@MTGW> Hello Meshtastic
KJ6DZB@MTG1> Same message through second Meshtastic backend
```

## Why this exists

LoRa-side users need to know that a message came through a gateway and which gateway/backend emitted it. Crow keeps the original sender's callsign at the front of the LoRa text payload and adds a compact backend tag.

This is done at the outbound backend layer, not globally in the router, because the backend knows the actual target path.

## Hard-coded tag scheme

The helper module is:

```text
lora_outbound_text.uc
```

It currently hard-codes this tag family:

| Target backend family | Primary gateway | Additional gateways |
| --- | --- | --- |
| MeshCore | `MCGW` | `MCG2`, `MCG3`, `MCG4`, ... |
| Meshtastic | `MTGW` | `MTG1`, `MTG2`, `MTG3`, ... |

The numbering intentionally preserves the short primary gateway names:

- `MCGW` means the primary MeshCore gateway.
- `MCG2` means the second MeshCore gateway/backend.
- `MTGW` means the primary Meshtastic gateway.
- `MTG1` means the first additional Meshtastic gateway/backend.

## How to enable tagging

Gateway tags are enabled by using the wrapper backend modules instead of importing the raw backend modules directly.

The raw backends remain available:

```text
meshcore_tnc.uc
meshtastic.uc
```

The tagged wrappers are:

```text
meshcore_tnc_tagged.uc
meshtastic_tagged.uc
```

### Enable MeshCore TNC tagging

In a test image or branch, change the MeshCore import from:

```ucode
import * as meshcore from "meshcore_tnc";
```

to:

```ucode
import * as meshcore from "meshcore_tnc_tagged";
```

Then set the optional backend index in config:

```json
{
  "meshcore_tnc": {
    "enabled": true,
    "gateway_index": 0,
    "gateway_tag_max_payload": 150
  }
}
```

Expected tag output:

| `gateway_index` | Tag |
| --- | --- |
| `0` | `MCGW` |
| `1` | `MCG2` |
| `2` | `MCG3` |

### Enable Meshtastic tagging

In a test image or branch, change the Meshtastic import from:

```ucode
import * as meshtastic from "meshtastic";
```

to:

```ucode
import * as meshtastic from "meshtastic_tagged";
```

Then set the optional backend index in config:

```json
{
  "meshtastic": {
    "enabled": true,
    "gateway_index": 0,
    "gateway_tag_max_payload": 200
  }
}
```

Expected tag output:

| `gateway_index` | Tag |
| --- | --- |
| `0` | `MTGW` |
| `1` | `MTG1` |
| `2` | `MTG2` |

### Disable tagging / rollback

Rollback is only an import change.

For MeshCore TNC, switch back to:

```ucode
import * as meshcore from "meshcore_tnc";
```

For Meshtastic, switch back to:

```ucode
import * as meshtastic from "meshtastic";
```

No message schema changes are required.

## Formatter behavior

The helper accepts:

```ucode
prepare(msg, target_transport, gateway_index, max_payload)
```

It returns a safely formatted string:

```text
CALLSIGN@TAG> original text
```

It uses this callsign lookup order:

```text
msg.originating_callsign
msg.callsign
msg.from_callsign
msg.data.callsign
UNKNOWN
```

It uses `length()` and `substr()` to enforce the backend payload budget.

If the formatted payload would exceed the budget, it truncates the original message text and appends:

```text
...
```

## MTU and backend limits

The generic formatter default budget is 255 bytes.

Backends may pass a smaller budget if their packet format has additional overhead or an existing text limit. This prevents hidden truncation later in the encoder.

Recommended behavior:

| Backend | Suggested call |
| --- | --- |
| MeshCore TNC | `prepare(msg, "meshcore", gateway_index, backend_text_limit)` |
| Meshtastic | `prepare(msg, "meshtastic", gateway_index, backend_text_limit)` |

## Backend ownership

Gateway tagging should happen immediately before the backend packet/TNC encoder.

Do not inject the tag globally in the router because `msg.transport` describes where the message came from, not where it is going.

Correct placement:

```text
AREDN/native message
  -> router decides outbound backend
  -> MeshCore or Meshtastic backend chooses gateway tag
  -> lora_outbound_text.prepare(...)
  -> backend packet encoder
  -> LoRa TNC/backend
```

## Debug logging

The formatter emits `DEBUG1` messages when it formats or truncates a payload.

Expected log examples:

```text
lora_outbound_text: formatted outbound target=meshcore tag=MCGW callsign=KJ6DZB total=42 max=150
lora_outbound_text: truncated outbound target=meshtastic tag=MTG1 callsign=KJ6DZB original=320 final=200 max=200
```

## Test cases

| Input | Backend | Index | Expected prefix |
| --- | --- | --- | --- |
| `Hello` | `meshcore` | `0` | `CALLSIGN@MCGW> Hello` |
| `Hello` | `meshcore` | `1` | `CALLSIGN@MCG2> Hello` |
| `Hello` | `meshcore` | `2` | `CALLSIGN@MCG3> Hello` |
| `Hello` | `meshtastic` | `0` | `CALLSIGN@MTGW> Hello` |
| `Hello` | `meshtastic` | `1` | `CALLSIGN@MTG1> Hello` |
| `Hello` | `meshtastic` | `2` | `CALLSIGN@MTG2> Hello` |

Truncation test:

```text
max_payload = 32
callsign = KJ6DZB
tag = MCGW
header = KJ6DZB@MCGW> 
```

The remaining message room is:

```text
32 - length("KJ6DZB@MCGW> ")
```

If the original text is longer than that room, the formatter keeps as much as possible and appends `...`.

## Current implementation note

The shared formatter module and the tagged wrapper modules have been added to the Crow codebase.

Use the wrappers when you want gateway tags enabled:

```text
meshcore_tnc_tagged.uc
meshtastic_tagged.uc
```

Use the raw modules when you want legacy behavior without outbound gateway tags:

```text
meshcore_tnc.uc
meshtastic.uc
```
