# Strict Gatekeeper

Strict Gatekeeper is Crow's fail-closed safety layer for bridge traffic entering Crow from Meshtastic or MeshCore before that traffic can be forwarded into AREDN or other amateur-radio paths.

It is intended for gateways where Crow may act like a message-forwarding station. The basic idea is simple:

> If the gateway cannot identify the sender and cannot confirm the traffic is plain text that fits local operator policy, Crow should not forward it.

Strict Gatekeeper is not a legal-compliance engine and it does not replace the control operator. It is an operator safety control that helps prevent Crow from becoming a blind automatic relay for unknown, encrypted, non-text, or non-amateur traffic.

This page combines the earlier `Strict-Gatekeeper` and `Strict-Gatekeeper-Mode` wiki notes into one canonical operator page.

## Plain-English purpose

Strict Gatekeeper answers one basic question before Crow forwards traffic from Meshtastic or MeshCore into the amateur/AREDN side:

> “Is this something I am willing to let my gateway station retransmit?”

If the answer is no, the packet is dropped before it enters the normal routing queue.

The current policy is intentionally conservative:

- encrypted Meshtastic packets are dropped before Crow attempts to decrypt/forward them;
- non-text bridged packets are dropped;
- the sender must have a callsign-looking identity;
- if `allowed_callsigns` is configured, the extracted sender callsign must be on that list;
- accepted messages are rewritten so the gateway identifies itself in the message body.

## What Strict Gatekeeper does

When enabled, Strict Gatekeeper checks bridged Meshtastic and MeshCore traffic before the normal Crow router forwards it onward.

| Check | What Crow looks for | Pass behavior | Fail behavior |
| --- | --- | --- | --- |
| Strict mode enabled | `strict_gatekeeper.enabled` is `true` | Gatekeeper policy is applied. | If disabled, normal routing behavior applies. |
| Gateway callsign | `strict_gatekeeper.gateway_callsign` or fallback `callsign` | Accepted messages can be marked with the gateway callsign. | If strict mode is on and no valid gateway callsign exists, traffic is dropped. |
| Sender identity | Sender name or metadata contains a simple callsign-looking token. | Sender callsign is extracted and used in annotation. | Traffic is dropped as unidentified. |
| Whitelist | Sender callsign is in `allowed_callsigns`, when the list is not empty. | Traffic may continue. | Traffic is dropped as not whitelisted. |
| Text only | Message contains `data.text_message`. | Text may continue. | Non-text bridge traffic is dropped. |
| No encrypted bridge forwarding | Meshtastic encrypted packet is detected before bridge forwarding. | Plain packets continue to text validation. | Encrypted packet is dropped. |
| Gateway annotation | Accepted traffic is rewritten as `[SENDER via GATEWAY] message`. | Downstream users see source and gateway. | Not applicable. |

## Why this matters for Part 97 automatic forwarding

Crow can connect non-amateur mesh transports, such as Meshtastic or MeshCore, to AREDN or other amateur-radio paths. When Crow forwards a message automatically, the gateway station may transmit content that did not originate from the control operator sitting at the gateway.

That creates a practical Part 97 concern:

- amateur transmissions need responsible station control;
- messages should not be obscured/encrypted for the purpose of hiding meaning;
- station identity matters;
- third-party and automatically forwarded messages need operator attention;
- a gateway should not blindly retransmit unknown traffic.

Strict Gatekeeper gives the Crow gateway an enforceable policy point before forwarding occurs.

## Part 97 auto-forwarding explained

A Crow bridge can look like a message-forwarding station. A message may start on Meshtastic or MeshCore, enter Crow through IP/multicast bridge traffic, and then be forwarded toward AREDN or another amateur-radio path. That means the gateway station may be the first amateur-side station that accepts the message for forwarding.

The practical rule of thumb is:

> Do not configure Crow as a blind bridge for traffic you would not be comfortable transmitting under your station callsign.

Strict Gatekeeper does not decide whether your specific band, mode, channel, bandwidth, or control arrangement is legal. It does give you a local policy gate so automatic forwarding is limited to plain-text, callsign-identifiable, operator-approved traffic.

## Part 97 concepts in plain language

| Part 97 concept | Plain-English meaning | Why a Crow gateway cares | How Strict Gatekeeper helps |
| --- | --- | --- | --- |
| Control operator | A licensed operator is responsible for station transmissions. | A bridge does not remove operator responsibility. | Requires/uses a gateway callsign and creates visible gateway attribution. |
| Automatic control | A station may transmit without a human pressing send only where the rules allow it. | Crow may automatically forward traffic once configured. | Filters traffic before it reaches the forwarding path. |
| Message forwarding system | A system that accepts and forwards messages between stations. | Crow can behave like one when bridging traffic. | Adds sender checks, text-only filtering, and gateway annotation. |
| First forwarding station | The first amateur-side station accepting a message has special responsibility. | Crow may be the first amateur/AREDN hop for Meshtastic or MeshCore traffic. | Whitelists and callsign checks reduce unknown-origin forwarding. |
| No obscured meaning | Amateur traffic should not hide meaning with encryption or codes, except where specifically permitted. | Encrypted mesh packets should not be blindly forwarded into Part 97 traffic. | Drops encrypted Meshtastic packets before bridge forwarding. |
| No arbitrary automatic retransmission | Part 97 limits automatic retransmission of other amateur stations' signals. | The gateway should not be a blind repeating machine. | Only admits traffic that passes policy before it can be routed onward. |
| Station identification | Transmissions should identify responsible stations. | Downstream users need to know who originated and who forwarded. | Rewrites messages as `[SENDER via GATEWAY] ...`. |
| Operator accountability | The gateway owner should be able to explain what crossed the bridge. | Logs and message annotation matter during events. | Drops policy failures and makes accepted traffic easier to audit. |

## Current Crow implementation

Crow currently has Strict Gatekeeper code in `gatekeeper.uc` and wires it into startup and routing.

The gatekeeper:

1. reads `strict_gatekeeper` configuration;
2. extracts a gateway callsign;
3. builds an optional callsign whitelist;
4. checks inbound Meshtastic/MeshCore bridge messages before normal routing;
5. drops traffic that fails policy;
6. annotates accepted traffic as gateway-forwarded text.

## Example configuration

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

## Configuration fields

| Field | Type | Required? | Purpose | Recommended use |
| --- | --- | --- | --- | --- |
| `enabled` | boolean | Yes | Turns Strict Gatekeeper on or off. | Use `true` for public, event, or unattended bridge gateways. |
| `gateway_callsign` | string | Strongly recommended | Callsign used to identify the forwarding gateway. | Use the gateway/control-operator callsign. |
| `allowed_callsigns` | array of strings | Optional but recommended | Sender whitelist. Empty means “accept any valid-looking callsign.” Non-empty means “only these callsigns may pass.” | Use a whitelist for real deployments. |

## Forwarding decision table

| Incoming traffic | Strict Gatekeeper off | Strict Gatekeeper on | Why |
| --- | --- | --- | --- |
| Plain text from Meshtastic/MeshCore with valid callsign-like sender | Routed normally. | Allowed and annotated as `[SENDER via GATEWAY] text`. | Plain-text, identifiable bridge traffic. |
| Plain text from allowed callsign | Routed normally. | Allowed and annotated. | Sender is on the operator-approved list. |
| Plain text from callsign not in whitelist | Routed normally. | Dropped when whitelist is configured. | Gateway policy restricts who may use the bridge. |
| Plain text with no callsign-looking sender | Routed normally. | Dropped. | Sender is not sufficiently identified. |
| Encrypted Meshtastic packet | May be handled by normal Meshtastic logic. | Dropped before bridge forwarding. | Avoids forwarding obscured traffic into amateur paths. |
| MeshCore/Meshtastic non-text payload | Routed or handled normally depending on app behavior. | Dropped at gatekeeper. | Strict bridge policy is text-only. |
| Native Crow/AREDN-originated message | Routed normally. | Not treated as Meshtastic/MeshCore ingress. | Gatekeeper is for bridge ingress, not all local messages. |
| Already annotated gateway traffic | Routed normally. | Allowed if it is already from this gateway identity. | Avoids double-wrapping accepted gateway text. |

## How messages are rewritten

Accepted bridge traffic is rewritten so the forwarded text identifies both the apparent sender and the gateway station.

Original bridged text:

```text
radio check from the hill
```

Forwarded by gateway `W6XYZ` after identifying sender `KJ6DZB`:

```text
[KJ6DZB via W6XYZ] radio check from the hill
```

This gives operators and downstream readers two important facts:

1. the apparent originating station/operator, `KJ6DZB`;
2. the amateur gateway that accepted and forwarded it, `W6XYZ`.

It also makes after-action review easier because accepted bridge traffic carries visible attribution.

## Callsign matching limits

The current callsign matcher intentionally uses a simple U.S.-style callsign token:

```text
1 or 2 letters + 1 number + 1 to 3 letters
```

Examples likely to pass:

```text
KJ6DZB
KN6PLV
W6XYZ
```

Examples likely to fail:

```text
Tactical 1
Net Control
Portable Hill
DX1ABC/P
```

This is conservative by design. For international callsigns, special-event callsigns, tactical labels, or suffix-heavy formats, the validator should be intentionally expanded rather than casually loosened.

## Recommended operating modes

| Deployment | Recommended Strict Gatekeeper setting | Notes |
| --- | --- | --- |
| Private software lab with no RF or AREDN forwarding | Optional | Useful for testing, but lower risk if nothing leaves the lab. |
| AREDN-only amateur deployment | Enable with `gateway_callsign` | Helps keep gateway identity visible. |
| Public Meshtastic-to-AREDN bridge | Enable with `gateway_callsign` and `allowed_callsigns` | Avoids becoming a public blind relay. |
| MeshCore-to-AREDN bridge | Enable with whitelist | MeshCore identity should be constrained before forwarding. |
| Emergency/event bridge | Enable with a preloaded whitelist | Tie the bridge to assigned operators/callsigns. |
| Unattended gateway | Enable, whitelist, and monitor logs | Best fit for fail-closed behavior. |

## What Strict Gatekeeper does not do

Strict Gatekeeper is a bridge policy tool, not a full identity or legal compliance system.

It does **not**:

- prove that a person legally owns a callsign;
- verify a callsign against FCC/ULS records;
- bind a callsign to a Meshtastic node ID;
- bind a callsign to a MeshCore public key;
- decide whether a band, emission, bandwidth, or automatic-control segment is legal;
- approve third-party traffic;
- make encrypted content lawful for amateur forwarding;
- replace the station licensee or control operator.

For stronger event or public deployments, pair `allowed_callsigns` with known Meshtastic node IDs or MeshCore public keys as a future hardening step.

## Logging and review

Strict Gatekeeper uses Crow's debug logging paths for drop messages. On OpenWrt or AREDN nodes, confirm that logs are captured by the service manager, `logd`, or your deployment wrapper before relying on logs for after-action review.

Useful things to log or verify during operations:

| Item | Why it matters |
| --- | --- |
| Gateway callsign loaded | Confirms forwarded messages will identify the gateway. |
| Whitelist loaded | Confirms the bridge is not open to everyone. |
| Drop counts | Shows whether unknown or encrypted traffic is trying to cross. |
| Accepted message examples | Confirms annotation is readable and useful. |
| Operator review time | Supports after-action review and troubleshooting. |

## Source/code release checks

Use these checks in the Crow repo before claiming Strict Gatekeeper is active in a build:

```text
grep -n "strict_gatekeeper" gatekeeper.uc config.uc
grep -n "allowed_callsigns" gatekeeper.uc
grep -n "filterInboundBridge" gatekeeper.uc router.uc
grep -n "setGatekeeper\|config._gatekeeper" config.uc router.uc meshtastic.uc
grep -n "drop encrypted Meshtastic" meshtastic.uc
```

Expected wiring:

| File | Expected role |
| --- | --- |
| `gatekeeper.uc` | Implements strict-mode callsign checks, whitelist checks, annotation, and bridge filtering. |
| `config.uc` | Imports and initializes gatekeeper, stores it as `config._gatekeeper`, and passes it to the router. |
| `router.uc` | Calls `gatekeeper.filterInboundBridge(msg)` before queuing Meshtastic/MeshCore ingress. |
| `meshtastic.uc` | Drops encrypted Meshtastic packets early when strict mode is enabled. |
| `STRICT_GATEKEEPER.md` | Source-repo operator note. |
| `Strict-Gatekeeper.md` | Canonical wiki operator documentation page. |

## Operator checklist

Before enabling a real bridge:

1. Set the gateway callsign.
2. Decide whether the bridge is public, event-only, or private.
3. Populate `allowed_callsigns` for any real public/event deployment.
4. Confirm Meshtastic/MeshCore users have callsign-bearing names.
5. Confirm logs are captured.
6. Send a test message and verify the `[SENDER via GATEWAY]` annotation.
7. Confirm encrypted traffic is not forwarded.
8. Confirm non-text packets are dropped.
9. Review Part 97 responsibilities for your actual deployment path.

## Part 97 rule anchors for operators

Review the current FCC/eCFR text directly for operational decisions. Commonly relevant sections include:

- 47 CFR § 97.109 — station control and automatic control
- 47 CFR § 97.113 — prohibited transmissions, including messages intended to obscure meaning and automatic retransmission limits
- 47 CFR § 97.115 — third-party communications
- 47 CFR § 97.119 — station identification
- 47 CFR § 97.219 — message forwarding systems
- 47 CFR § 97.221 — automatically controlled digital stations
- 47 CFR § 97.309 — RTTY/data emission codes and public documentation of digital techniques

Main Part 97 page:

https://www.ecfr.gov/current/title-47/chapter-I/subchapter-D/part-97

## Short version

Strict Gatekeeper is the switch that makes Crow stop acting like a blind bridge. It forces Meshtastic/MeshCore ingress to be plain-text, callsign-identifiable, and optionally whitelisted before Crow forwards it toward AREDN/amateur-radio paths.
