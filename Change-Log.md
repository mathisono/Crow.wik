# Crow Change Log

This page tracks the practical changes from Raven to Crow, including code work, wiki/documentation work, and operator-facing behavior.

## Current focus

Crow is the rebranded and expanded home for the Raven mesh messaging work. The goal is to keep Raven's useful AREDN messaging base while adding clearer operator controls, APRS support, USB-attached storage, and better Part 97-aware bridge behavior.

## 2026-06-25

### Wiki: MeshCore TCP API protocol correction notes
**Wiki status:** committed to `mathisono/Crow.wik`

- Added `MeshCore-TCP-API-Protocol-Notes.md` to record the protocol mismatch in Crow's current `meshcore_tcp_api.uc` scaffold.
- Documented that Crow currently assumes `[0x3E][CmdID][PayloadLen BE][Payload]`.
- Documented stock MeshCore serial/Wi-Fi framing as radio-to-client `>` plus little-endian 2-byte length and client-to-radio `<` plus little-endian 2-byte length.
- Documented the message-waiting receive model: `0x83` push/tickle, client sends `CMD_SYNC_NEXT_MESSAGE = 0x0A`, then decodes queued response codes `0x07`, `0x08`, `0x10`, and `0x11`.
- Documented that `meshcore_tcp_discovery.uc` has the correct `CMD_GET_CHANNEL = 0x1F` and `RESP_CODE_CHANNEL_INFO = 0x12` discovery idea, but `queryDeviceGroups()` still does not send the command and returns an empty array.
- Linked the new protocol notes page from `README.md`, `Home.md`, and `_Sidebar.md`.

### Wiki: verify Winlink USB storage behavior
**Wiki status:** committed to `mathisono/Crow.wik`

- Verified `winlink.uc` uses the relative form path `winlink/forms` and reads form assets through platform helpers such as `platform.dirtree()` and `platform.loadbinary()`.
- Verified the AREDN platform maps any path beginning with `winlink/` under the active storage root.
- Documented that internal storage maps Winlink forms to `/usr/local/crow/winlink/forms`.
- Documented that healthy USB storage maps Winlink forms to `/mnt/crow/winlink/forms`.
- Documented that degraded USB mode falls back to the internal form path while images fall back to `/tmp/apps/crow/images`.
- Updated `USB-Storage.md` to align degraded-mode and Winlink/form storage wording with the code.

### Wiki: split configuration and channel setup pages
**Wiki status:** committed to `mathisono/Crow.wik`

- Added `Backend-Configuration.md` as the operator page for backend selector setup, Meshtastic UDP/API, MeshCore UDP/API, APRS backend setup, backend mode examples, and backend validation.
- Refocused `Configuration.md` into a top-level configuration overview and index of config blocks.
- Refocused `Configuring-Channels.md` into a channel-only operator guide covering `/join`, `/leave`, AREDN-only channels, APRS groups, MeshCore slot mapping, text stores, and channel validation.
- Preserved the backend/API content by moving it into `Backend-Configuration.md` instead of deleting it.
- Updated `README.md`, `Home.md`, and `_Sidebar.md` so the new page is discoverable.

### Wiki: outbound LoRa gateway tagging with Strict Gatekeeper context
**Wiki status:** committed to `mathisono/Crow.wik`

- Updated `MeshCore-Backends.md` to explain outbound MeshCore gateway tagging when a gatekeeper-accepted message later exits through a tagged MeshCore backend.
- Updated `LoRa-Gateway-Tags.md` to document the relationship between Strict Gatekeeper annotations and outbound LoRa gateway tags.
- Documented the layered format:
  - outbound backend tag: `GATEWAY@MCGW> message`
  - strict inbound annotation: `[SENDER via GATEWAY] message`
  - combined example: `W6XYZ@MCGW> [KJ6DZB via W6XYZ] radio check`
- Documented weak-identity MeshCore group example: `W6XYZ@MCGW> [KJ6DZB@MCGW-TacNet via W6XYZ] radio check`.
- Clarified that `lora_outbound_text.uc` is not itself conditional on Strict Gatekeeper. If the tagged wrapper is wired in, it tags outbound text handled by that wrapper. Strict Gatekeeper decides whether bridge ingress is allowed and how accepted text is annotated before the outbound tag step.
- Noted that the current MeshCore selector uses the raw UDP backend unless `meshcore_tagged.uc` is explicitly wired in or the selector is updated to use it.

### Wiki: MeshCore backend documentation audit
**Wiki status:** committed to `mathisono/Crow.wik`

- Added `MeshCore-Backends.md` to document what the current code actually does for MeshCore.
- Documented the backend selector in `meshcore_backend.uc`: default UDP, explicit `api` / `tcp-api` / `companion-api`, and `meshcore_tcp_api.enabled=true` with `meshcore.enabled=false`.
- Documented the original UDP backend behavior: multicast `224.0.0.69:4402`, bridge key requirement, inbound/outbound support, shared-key persistence, direct/group text handling, adverts, and ACK/routing behavior.
- Documented the TCP Companion API backend behavior: TCP `host:port`, HELLO on connect, direct `0x07` decode, group `0x08` decode, Smart Accumulator gates, reconnect behavior, and `backend: "tcp_api"` message tagging.
- Corrected the wiki to state that MeshCore TCP API outbound send is not implemented yet; production outbound still requires the UDP backend.
- Corrected the wiki to state that MeshCore TCP discovery is not production-ready yet because `queryDeviceGroups()` still has the radio command send stubbed.
- Linked the new page from `README.md`, `Home.md`, and `_Sidebar.md`.
- Updated `Configuration.md` so the MeshCore section matches the selector and backend code.

### Crow: fix USB setup script Crow path
**Crow commit:** `dd82ff90`

- Updated `platforms/aredn/usb-setup.sh` usage text from the legacy Raven path to the current Crow path:
  - old: `/usr/local/raven/platforms/aredn/usb-setup.sh`
  - new: `/usr/local/crow/platforms/aredn/usb-setup.sh`
- No storage behavior changed. This was a documentation/path cleanup inside the script.
- **Files changed:** `platforms/aredn/usb-setup.sh`

### Crow: fix README wiki link display
**Crow commit:** `18e1cae`

- Fixed the visible README documentation URL so it points to `mathisono/Crow/wiki` instead of the misspelled `mathison/Crow/wiki`.
- The link target was already correct; the displayed text was corrected.
- **Files changed:** `README.md`

### Wiki: storage, supernode, text store, and Winlink refresh
**Wiki status:** committed to `mathisono/Crow.wik`

- Rewrote `Memory-Use.md` around Crow's current storage behavior: internal storage, USB storage, degraded mode, image quota, RAM message mode, Winlink form storage, and message-store behavior.
- Added/updated `Supernodes.md` using the original Raven source material and Crow code behavior. Supernodes auto-detect from AREDN config, relay MeshIP traffic, and intentionally do not act as message/text stores.
- Added/updated `Text-Stores.md` / node message store documentation using the original Raven Text Stores source material and Crow code behavior. Text stores use `platform.store("textstore.<namekey>")`, so they automatically use `/mnt/crow/data` when USB storage is active.
- Updated `Winlink.md` from placeholder status into a Crow-specific form UI and storage page based on current `winlink.uc` behavior.
- Updated `Home.md` and `_Sidebar.md` to include `Supernodes.md` and `Text-Stores.md` as canonical pages.
