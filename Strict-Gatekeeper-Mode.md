# Strict Gatekeeper Mode

Strict Gatekeeper mode is an optional fail-closed bridge policy for sites that want tighter control over Meshtastic or MeshCore text traffic before it is forwarded to AREDN or other amateur-radio paths.

It is designed for bridge deployments where Crow should **reject bridge traffic unless it is plain text, has an amateur-style sender identity, and, when configured, comes from an allowed callsign**.

This is not legal advice and it does not replace the control operator. It is a software safety guard intended to make Crow's automatic or semi-automatic forwarding behavior easier to supervise under U.S. amateur-radio Part 97 expectations.

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

## Why this matters for Part 97 auto-forwarding

Crow can behave like a forwarding station. That means a message can originate somewhere else, enter Crow through Meshtastic or MeshCore, and then be forwarded into an amateur-radio path. Once Crow is doing that automatically, the gateway operator has to think about Part 97 forwarding and automatic-control responsibilities.

The practical issue is this:

- A human typing directly on an amateur station is easy to supervise.
- A bridge forwarding packets from another network is harder.
- If the bridge blindly forwards everything, the gateway may retransmit encrypted, non-amateur, unidentified, commercial, or otherwise improper content.
- Strict Gatekeeper reduces that risk by refusing traffic that does not meet the configured policy.

## Part 97 concepts in normal language

| Part 97 concept | Plain meaning | Why Crow cares | What Strict Gatekeeper does |
| --- | --- | --- | --- |
| Control operator | A licensed operator is responsible for the station's transmissions. | The gateway cannot be treated as “nobody's responsibility.” | Requires a configured gateway callsign and annotates forwarded traffic with it. |
| Automatic control | A station can transmit without the control operator sitting at the control point only where Part 97 allows it. | A bridge can cause automatic transmissions. | Does not decide band legality, but helps keep bridged content constrained before automatic forwarding. |
| Message forwarding system | A cooperating system that forwards messages between originating and destination stations. | Crow's bridge behavior can look like a message forwarding system. | Adds identity checks and content-type checks before accepting bridge traffic. |
| First forwarding station | The first station that accepts a message into the forwarding system has extra responsibility. | A Crow gateway may be the first amateur-side hop for Meshtastic/MeshCore traffic. | Requires a callsign-looking sender and optionally a whitelist, helping the gateway either authenticate/limit senders or knowingly accept accountability. |
| No obscured messages | Amateur traffic must not use codes/ciphers to hide meaning, except where rules specifically allow. | Encrypted Meshtastic/MeshCore content should not be bridged into Part 97 paths. | Drops encrypted Meshtastic packets and drops non-text bridge packets. |
| No arbitrary automatic retransmission | Part 97 limits automatic retransmission of other amateur stations' signals. | The gateway should not be a blind repeating machine. | Only admits traffic that passes policy before it can be routed onward. |
| Station identification | The source of transmissions must be identifiable. | Bridged messages should show who originated them and which gateway forwarded them. | Rewrites accepted messages as `[SENDER via GATEWAY] message`. |

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
  "strict_gatekeeper": {
    "enabled": true,
    "gateway_callsign": "W6XYZ",
    "allowed_callsigns": ["KN6PLV", "KJ6DZB"]
  }
}
```

## Configuration fields

| Field | Required? | Purpose | Recommended setting |
| --- | --- | --- | --- |
| `enabled` | Yes | Turns Strict Gatekeeper mode on or off. | `true` for public or unattended bridge gateways. |
| `gateway_callsign` | Strongly recommended | Callsign used to identify the AREDN/amateur gateway when text is forwarded. | Use the gateway control operator or station callsign. |
| `allowed_callsigns` | Optional but recommended | Sender whitelist. Empty means “accept any valid-looking callsign.” Non-empty means “only these callsigns may pass.” | Use a whitelist for real deployments. |

## Forwarding decision table

| Incoming traffic | Strict Gatekeeper off | Strict Gatekeeper on | Reason |
| --- | --- | --- | --- |
| Plain text from Meshtastic/MeshCore with valid callsign-like sender | Allowed through normal router behavior | Allowed, annotated as `[SENDER via GATEWAY] text` | Sender identity is visible and traffic is plain text. |
| Plain text with sender callsign not on whitelist | Allowed through normal router behavior | Dropped if `allowed_callsigns` is non-empty | Gateway is configured to accept only named operators. |
| Plain text with no callsign-looking sender | Allowed through normal router behavior | Dropped | Gateway cannot identify the apparent sender. |
| Encrypted Meshtastic packet | May be decoded/handled by normal Meshtastic path | Dropped before bridge forwarding | Avoids forwarding obscured/encrypted content into amateur paths. |
| Non-text packet/data payload | May be handled by normal router behavior | Dropped | Policy is text-only for forwarded bridge traffic. |
| Message already annotated by this gateway | Allowed | Allowed | Prevents re-wrapping gateway-originated forwarded text. |

## What Strict Gatekeeper does not do

Strict Gatekeeper is useful, but it is not magic.

It does **not**:

- prove a person's legal identity;
- prove the sender actually owns the callsign typed into their device name;
- verify Meshtastic node IDs against FCC license records;
- verify MeshCore public keys against operators;
- decide which RF band, emission, bandwidth, or automatic-control segment is legal;
- replace the station licensee or control operator;
- make encrypted traffic legal for amateur forwarding.

For stronger deployments, pair the callsign whitelist with known Meshtastic node IDs or MeshCore public keys. That is a future hardening target.

## Why the gateway rewrites messages

When Strict Gatekeeper accepts a bridged text packet, it rewrites the message body like this:

```text
[SENDER via GATEWAY] original message text
```

Example:

```text
[KJ6DZB via W6XYZ] radio check from the hill
```

That gives operators and downstream readers two important facts:

1. the apparent originating station/operator, `KJ6DZB`;
2. the amateur gateway that accepted and forwarded it, `W6XYZ`.

## Callsign matching limits

The built-in callsign validator intentionally matches a simple U.S.-style amateur callsign token:

```text
1 or 2 letters + 1 numeral + 1 to 3 letters
```

Examples likely to pass:

```text
KJ6DZB
KN6PLV
W6XYZ
```

This simple matcher may reject:

- international callsigns;
- special-event formats;
- tactical-only names;
- suffix-only labels;
- malformed or nonstandard display names.

That is intentional for strict mode. If a deployment needs international or special-event callsigns, the matcher should be expanded deliberately, not loosened accidentally.

## Recommended operating policy

| Deployment type | Suggested setting | Why |
| --- | --- | --- |
| Private lab, no RF forwarding | Strict Gatekeeper optional | Useful for testing but not required if nothing reaches Part 97 RF paths. |
| AREDN-only amateur deployment | Enable with `gateway_callsign` | Keeps gateway identification visible. |
| Public bridge between Meshtastic/MeshCore and AREDN | Enable with `allowed_callsigns` | Avoids becoming a blind bridge for unknown users. |
| Emergency/event deployment | Enable with a preloaded whitelist | Keeps traffic tied to assigned operators and callsigns. |
| Unattended gateway | Enable, whitelist, monitor logs | Reduces risk from automatic forwarding. |

## Logging

Dropped packet logging uses Crow's existing `DEBUG0` and `DEBUG1` debug channels.

On headless OpenWrt/AREDN deployments, confirm that log output is actually captured by the service manager or `logd` before relying on logs for after-action review.

## Part 97 rule anchors for operators

Operators should review the current FCC/eCFR text directly. The most relevant sections are:

- 47 CFR § 97.109 — station control / automatic control
- 47 CFR § 97.113 — prohibited transmissions, including messages intended to obscure meaning and automatic retransmission limits
- 47 CFR § 97.115 — third-party communications
- 47 CFR § 97.119 — station identification
- 47 CFR § 97.219 — message forwarding systems
- 47 CFR § 97.221 — automatically controlled digital stations
- 47 CFR § 97.309 — RTTY/data emission codes and public documentation of digital techniques

Main eCFR page:

https://www.ecfr.gov/current/title-47/chapter-I/subchapter-D/part-97

## Related source document

The code repository includes `STRICT_GATEKEEPER.md`. This wiki page expands the operator-facing explanation and Part 97 forwarding rationale.
