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

LoRa-side users need to know that a message came through a gateway and which gateway/backend emitted it. Crow keeps the source callsign at the front of the LoRa text payload and adds a compact backend tag.

This is done at the outbound backend layer, not globally in the router, because the backend knows the actual target path.

## Relationship to Strict Gatekeeper

Strict Gatekeeper and LoRa Gateway Tags are separate layers:

| Layer | Direction | Job |
|---|---|---|
| Strict Gatekeeper | inbound from Meshtastic/MeshCore into Crow/AREDN | decide whether bridge ingress may be forwarded, then annotate accepted text |
| LoRa Gateway Tags | outbound from Crow/AREDN toward Meshtastic/MeshCore | mark the LoRa-side packet with the gateway/backend that emitted it |

When Strict Gatekeeper is enabled, accepted inbound bridge text is rewritten as:

```text
[SENDER via GATEWAY] message
```

For weak-identity MeshCore group messages, accepted inbound bridge text is rewritten as:

```text
[SENDER@MCGW-GroupName via GATEWAY] message
```

When that already-annotated message is later sent out through a tagged LoRa backend, the outbound wrapper adds the LoRa gateway tag in front:

```text
GATEWAY@MCGW> [SENDER via GATEWAY] message
```

Example:

```text
W6XYZ@MCGW> [KJ6DZB via W6XYZ] radio check from the hill
```

Example for a weak-identity MeshCore group message:

```text
W6XYZ@MCGW> [KJ6DZB@MCGW-TacNet via W6XYZ] radio check
```

This is intentionally redundant: the first tag identifies the backend/gateway emitting the LoRa packet, while the bracketed Strict Gatekeeper annotation identifies the apparent sender and gateway policy point that accepted the bridged traffic.

Current implementation caveat: the formatter itself is not conditional on Strict Gatekeeper. If a tagged wrapper is wired in, it tags outbound text handled by that wrapper. Strict Gatekeeper decides whether bridged ingress is allowed and how it is annotated before that outbound step.

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

Gateway tags are enabled by using the wrapper backend modules instead of importing the raw backend modules directly, or by updating the backend selector to use the tagged wrapper.

The raw backends remain available:

```text
meshcore.uc
meshtastic.uc
```

The tagged wrappers are:

```text
meshcore_tagged.uc
meshtastic_tagged.uc
```

### Enable MeshCore UDP tagging

The current MeshCore selector imports the raw UDP backend as `meshcore`. To make MeshCore UDP outbound tagging active, wire the selector or router path to use:

```ucode
import * as meshcore from "meshcore_tagged";
```

instead of:

```ucode
import * as meshcore from "meshcore";
```

Then set the optional backend index in config:

```json
{
  "meshcore": {
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

Rollback is only an import/selector change.

For MeshCore UDP, switch back to:

```ucode
import * as meshcore from "meshcore";
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

This matters with Strict Gatekeeper enabled because `gatekeeper.uc` sets `msg.originating_callsign` to the gateway callsign after accepting bridged ingress. That means an accepted/annotated bridge message that later exits through a tagged LoRa backend will normally use the gateway callsign in the `CALLSIGN@TAG>` prefix.

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
| MeshCore UDP tagged wrapper | `prepare(msg, "meshcore", gateway_index, backend_text_limit)` |
| Meshtastic tagged wrapper | `prepare(msg, "meshtastic", gateway_index, backend_text_limit)` |

## Backend ownership

Gateway tagging should happen immediately before the backend packet encoder.

Do not inject the tag globally in the router because `msg.transport` describes where the message came from, not where it is going.

Correct placement:

```text
AREDN/native or gatekeeper-accepted bridge message
  -> router decides outbound backend
  -> MeshCore or Meshtastic backend chooses gateway tag
  -> lora_outbound_text.prepare(...)
  -> backend packet encoder
  -> LoRa backend
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
meshcore_tagged.uc
meshtastic_tagged.uc
```

Use the raw modules when you want legacy behavior without outbound gateway tags:

```text
meshcore.uc
meshtastic.uc
```
