# Strict Gatekeeper Mode

Strict Gatekeeper mode is an optional fail-closed bridge policy for sites that want tighter control over Meshtastic or MeshCore text traffic before it is forwarded to AREDN.

It is intended for bridge deployments where Crow should reject traffic unless it looks like valid amateur-radio text traffic and, when configured, comes from an allowed callsign.

## What it controls

When enabled, Strict Gatekeeper checks Meshtastic and MeshCore ingress before the message is admitted to the routing queue.

The current policy:

- drops encrypted Meshtastic protobuf packets instead of decrypting them for bridge forwarding;
- drops non-text bridged packets;
- requires the sender name to contain a simple US-style amateur callsign token such as `KN6PLV`, `W6XYZ`, or `KJ6DZB`;
- if `allowed_callsigns` is non-empty, requires the extracted sender callsign to appear in that whitelist;
- rewrites forwarded text so that it originates from the gateway node and prefixes the body as `[SENDER via GATEWAY] message`.

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

| Field | Purpose |
| --- | --- |
| `enabled` | Turns Strict Gatekeeper mode on or off. |
| `gateway_callsign` | Callsign used to identify the AREDN gateway when text is forwarded. |
| `allowed_callsigns` | Optional whitelist. Leave empty for callsign-format checking only, or populate it to restrict accepted senders. |

## Operating notes

Strict Gatekeeper mode is a transport safety control, not an identity proof system. Meshtastic and MeshCore display names are user-controlled, so a node can be renamed to look like a valid callsign.

For real deployments, use `allowed_callsigns`. A future hardening step should bind allowed operators to a stable Meshtastic node ID or MeshCore public key.

## Callsign matching limits

The built-in callsign validator intentionally matches only a simple US amateur callsign form: one or two letters, one numeral, and one to three letters.

It will reject international callsigns, special-event formats, and suffix forms that do not contain an extractable US callsign token.

## Logging

Dropped packet logging uses the existing `DEBUG0` and `DEBUG1` debug channels.

On headless OpenWrt deployments, confirm that log output is captured by the service manager or `logd` before relying on logs for troubleshooting.

## Related source document

The code repository includes `STRICT_GATEKEEPER.md`. This wiki page mirrors that feature documentation for operator-facing use.
