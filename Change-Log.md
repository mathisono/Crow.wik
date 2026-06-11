# Crow Change Log

This page tracks the practical changes from Raven to Crow, including code work, wiki/documentation work, and operator-facing behavior.

## Current focus

Crow is the rebranded and expanded home for the Raven mesh messaging work. The goal is to keep Raven's useful AREDN messaging base while adding clearer operator controls, APRS support, USB-attached storage, and better Part 97-aware bridge behavior.

## Raven → Crow overview

| Area | Raven behavior | Crow change | Status / notes |
| --- | --- | --- | --- |
| Project identity | Raven naming and raven icon assets. | Rebranded as Crow with Crow UI/icon assets. | UI assets are being renamed/replaced as `crow.png` and `crow.svg`. |
| Messaging goal | Mesh messaging foundation. | Unified decentralized messaging platform for AREDN and related mesh transports. | Crow is intended to unify AREDN, Meshtastic, MeshCore, legacy MeshChat, Winlink, and APRS workflows. |
| APRS backend | Raven `pt97-compliance` branch had richer APRS work. | Crow needs the Raven APRS backend restored as the current `aprs.uc`. | Target is Raven `pt97-compliance/aprs.uc`. Verify that `main` contains the restored code before release. |
| APRS passcode | Raven APRS-IS backend supported `passcode`. | Crow docs now explicitly document APRS-IS passcode setup. | Use `aprs.backend.passcode` for single-backend compatibility and `aprs.backends.<name>.passcode` for named backends once multi-backend code is active. |
| APRS groups | Raven supported APRS group messaging and repeat behavior. | Crow wiki now documents APRS groups, inline APRS message forms, and `/join` group creation. | Code should be verified against the Raven APRS implementation. |
| Slash commands | Earlier command set was smaller and less documented. | Crow wiki now has a user-facing command reference. | Includes `/help`, `/join`, `/leave`, `/groups`, `/channels`, `/backend`, `/backends`, `/export`, and `/storage`. |
| Strict Gatekeeper | Not clearly operator-documented. | Crow includes `gatekeeper.uc` and expanded wiki guidance. | Handles fail-closed bridge filtering for Meshtastic/MeshCore ingress. |
| Part 97 bridge policy | Needed clearer operator explanation. | Crow wiki now explains automatic forwarding risk and gateway responsibilities in plain language. | See Strict Gatekeeper page. |
| USB attached storage | Not part of the older Raven baseline. | Crow adds documented USB storage workflows for AREDN nodes. | Supports internal storage fallback, USB scan/enable/disable, and image quota behavior. |
| Auto-update cron | Legacy Raven cron updater existed in packaging. | Crow removed the auto-update cron behavior. | This avoids surprise unattended package replacement. |

## Documentation changes

| Wiki page | Purpose | Status |
| --- | --- | --- |
| [Home](Home) | Main wiki landing page and page index. | Updated to link active documentation pages. |
| [Command Reference](Command-Reference) | User-facing slash commands and APRS chat command forms. | Added/updated. |
| [APRS Bridge](APRS) | APRS-IS, KISS TCP, APRS passcode, groups, Part 97-safe APRS behavior. | Added. |
| [Strict Gatekeeper](Strict-Gatekeeper-Mode) | Fail-closed bridge filtering and Part 97 auto-forwarding explanation. | Expanded with tables and operator guidance. |
| [USB Storage](USB-Storage) | AREDN USB data storage and persistent image storage behavior. | Added/updated. |
| [Change Log](Change-Log) | This page. | Added. |

## APRS backend change notes

Crow's APRS work should use the old Raven `pt97-compliance` APRS backend as the current implementation baseline.

Expected Crow APRS features:

- APRS-IS backend support;
- APRS-IS `passcode` support;
- default `passcode: "-1"` behavior when no passcode is configured;
- KISS TCP backend support for Dire Wolf or similar TNCs;
- TNC2-style TCP text backend support;
- named multi-backend configuration with `aprs.backends`;
- backward compatibility for old single `aprs.backend` configuration;
- per-channel backend binding with `channels[].backend`;
- APRS group membership and group send behavior;
- backend-aware `/join #group backend=NAME CALL1 CALL2 message` behavior;
- `/backend` and `/backends` listing support;
- AREDN-only / `og==` behavior for APRS-related channels.

Important release check:

```text
grep -n "passcode" aprs.uc
grep -n "cfg.backends\|cfg.backend" aprs.uc
grep -n "getBackendNames\|updateChannelBackend\|sendToGroup" aprs.uc
```

If these checks fail, Crow is still using the old single-backend APRS file and the Raven APRS restoration is not complete.

## APRS group messaging changes

Crow's intended APRS workflow supports both direct messages and group-like behavior.

| User action | Form | Meaning |
| --- | --- | --- |
| Direct APRS message | `@CALL message` | Send one APRS message to one station. |
| Group send | `#group message` | Send text to members of an existing APRS group. |
| Inline group list | `#group CALL1 CALL2 message` | Send to listed callsigns without permanently changing all config. |
| Runtime group create/update | `join #group CALL1 CALL2 message` | Create/update an APRS group and send. |
| Slash-command group/channel create | `/join #group CALL1 CALL2 message` | Create an AREDN-only channel and APRS group together. |
| Backend-specific group/channel create | `/join #group backend=NAME CALL1 CALL2 message` | Bind a group/channel to a named APRS backend. |

## Slash command changes

| Command | Purpose |
| --- | --- |
| `/help` | Show available commands. |
| `/join #name` | Join or create a shared channel. |
| `/join %name` | Join or create an AREDN-only channel. |
| `/join #name CALL1 CALL2 message` | Create APRS group/channel and send an initial message. |
| `/join #name backend=NAME CALL1 CALL2 message` | Create APRS group/channel bound to a named backend. |
| `/leave #name` | Leave a channel and remove the matching APRS group when present. |
| `/groups` | List APRS groups. |
| `/backend` / `/backends` | List configured APRS backends. |
| `/channels` | List local public channels. |
| `/channels world` | List mesh-wide public channels through the bridge path. |
| `/channels join #name` | Join a named channel using derived key behavior. |
| `/channels join name key` | Join a channel using explicit name/key. |
| `/channels leave name` | Leave a channel. |
| `/export`, `/export text`, `/export csv` | Export current channel log. |
| `/storage status` | Show active storage mode and paths. |
| `/storage usb scan` | Find removable USB storage candidates. |
| `/storage usb enable` | Enable USB storage mode. |
| `/storage usb disable` | Return to internal storage. |
| `/storage quota images <mb>` | Set persistent image quota. |

## Strict Gatekeeper changes

Strict Gatekeeper is now treated as a first-class Crow safety feature instead of a short code note. It is present in the codebase and documented as a **fail-closed bridge control** for Meshtastic and MeshCore traffic entering AREDN or other amateur-radio paths.

The goal is to keep Crow from becoming a blind automatic forwarding gateway. When a non-amateur mesh network feeds Crow, the gateway should not automatically retransmit anything that is encrypted, non-text, unidentified, or outside the local operator policy.

### What changed

| Feature | Crow behavior | Why it matters |
| --- | --- | --- |
| Fail-closed filtering | Drops traffic that does not pass policy before it enters the routing queue. | Safer default for public or unattended gateways. |
| Encrypted packet handling | Drops encrypted Meshtastic packets before bridge forwarding. | Avoids carrying obscured content into Part 97 amateur paths. |
| Text-only forwarding | Drops non-text bridged packets. | Keeps the gateway focused on human-readable traffic. |
| Sender callsign check | Requires a callsign-looking sender identity. | Makes forwarded traffic more traceable to an apparent operator. |
| Callsign whitelist | Allows `allowed_callsigns` to restrict bridge senders. | Prevents unknown stations from using the gateway as a public relay. |
| Gateway annotation | Rewrites accepted messages as `[SENDER via GATEWAY] message`. | Identifies both the apparent originator and the forwarding gateway. |
| Operator documentation | Wiki now includes Part 97 auto-forwarding explanation and operating tables. | Gives control operators practical guidance instead of burying the behavior in code. |

### Part 97 auto-forwarding rationale

Crow can act like a message forwarding system when it accepts traffic from Meshtastic or MeshCore and then forwards it onto AREDN/amateur-radio paths. That creates an operator-control problem: the gateway station may transmit content that the control operator did not personally type.

Strict Gatekeeper does not make legal decisions by itself, but it gives the operator a software control point:

- do not forward encrypted or hidden content;
- do not forward non-text payloads;
- do not forward traffic from unknown or unapproved senders;
- visibly identify the gateway that accepted the message;
- make the forwarding path easier to audit after an event.

### Recommended use

| Deployment | Recommended Strict Gatekeeper setting |
| --- | --- |
| Lab testing with no RF/AREDN forwarding | Optional. |
| AREDN-only amateur deployment | Enable with `gateway_callsign`. |
| Public Meshtastic/MeshCore-to-AREDN bridge | Enable with `gateway_callsign` and `allowed_callsigns`. |
| Emergency/event bridge | Enable with a preloaded callsign whitelist. |
| Unattended gateway | Enable, whitelist, and monitor logs. |

### Release check

```text
grep -n "strict_gatekeeper" gatekeeper.uc config.uc
grep -n "allowed_callsigns" gatekeeper.uc
grep -n "filterInboundBridge" gatekeeper.uc router.uc
grep -n "config._gatekeeper" config.uc meshtastic.uc
```

If these checks fail, the wiki may describe behavior that is not actually wired into the current Crow build.

## USB attached storage changes

Crow adds an AREDN-focused storage workflow so data can live on attached USB storage while the core app remains on the node.

Expected behavior:

- internal storage remains the fallback;
- USB storage can be scanned, enabled, and disabled through `/storage` commands;
- degraded mode is documented for missing or unhealthy USB storage;
- persistent images can be quota-limited;
- storage commands are designed to avoid destructive formatting/wipe behavior.

## Release checklist

Before calling a Crow build/release current, verify:

```text
# APRS code restored
grep -n "cfg.backends\|getBackendNames\|sendToGroup" aprs.uc

# APRS passcode supported
grep -n "passcode" aprs.uc

# Strict Gatekeeper present
grep -n "strict_gatekeeper\|allowed_callsigns\|filterInboundBridge" gatekeeper.uc config.uc router.uc

# USB storage commands present
grep -n "storage usb\|quota images" commands.uc

# Wiki links current
git -C ../Crow.wik grep -n "APRS\|Strict Gatekeeper\|USB Storage\|Change Log" Home.md _Sidebar.md
```

## Known follow-up items

- Confirm the Raven `pt97-compliance/aprs.uc` restoration is actually on Crow `main`.
- Rename any remaining user-facing `Raven` strings to `Crow` where doing so does not break compatibility.
- Verify APRS multi-backend code does not reference undefined Crow symbols.
- Add tests or dry-run checks for `/join #group backend=NAME ...`.
- Add a clear operator example for `allowed_callsigns` and Part 97-safe bridge deployment.
