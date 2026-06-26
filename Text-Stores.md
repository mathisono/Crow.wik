# Node Message Stores

Crow still uses the original Raven `textstore` mechanism in code and configuration. This page uses the operator-facing phrase **node message store**, but the config key is still:

```json
"textstore"
```

This page is based on Raven's original Text Stores page, updated with Crow storage behavior and current code notes.

## What a message store does

When you create a brand-new channel, it should be empty. When you join an existing channel, you may expect recent channel history to appear.

A node message store provides that recent history. It remembers recent text messages for configured channels and can resend them to a node that joins later, restarts, or notices that it is missing a message.

## Why not make every node a store?

Every node could store message history, but that puts extra storage and write load on the whole mesh.

A better design is to place a few reliable message stores on the network. Then recent history remains available even if one store is offline, without making every small node preserve every channel.

## Best nodes for message stores

Use nodes with persistent storage and reliable power.

Good choices:

- A normal AREDN node with USB storage enabled
- A local hub node
- An event node that remains online for the operation
- A VM or device with durable storage
- A device with plenty of internal flash, if USB is not available

Avoid:

- AREDN supernodes
- temporary battery-only nodes
- very small flash devices
- nodes in degraded USB mode
- nodes that are frequently rebooted or reflashed

## Recommended: use USB storage

Crow can store message-store data under the active Crow storage root.

With internal storage, Crow data is under:

```text
/usr/local/crow/data
```

With USB storage enabled, Crow data is under:

```text
/mnt/crow/data
```

USB storage is preferred for message stores because it gives Crow more room, reduces pressure on internal flash, and makes images/files/Winlink data easier to keep across event operations.

Before enabling a message store on a USB-capable node, run these commands from the Crow message composer:

```text
/storage usb scan
/storage usb enable
/storage status
```

A healthy USB-backed store should report:

```text
Crow storage: usb
Mode: usb
Root: /mnt/crow
Images: /mnt/crow/images
Mountpoint: /mnt/crow
```

## Generic message store

Add this to `/usr/local/crow/crow.conf.override`:

```json
{
  "textstore": {
    "stores": [
      {
        "namekey": "*"
      }
    ]
  }
}
```

This creates a generic store for local channels. In Crow code, `namekey: "*"` becomes the default store advertisement and default store size source.

## Channel-specific message store

You can also create a store for a specific channel.

Example:

```json
{
  "textstore": {
    "stores": [
      {
        "namekey": "AREDN og==",
        "size": 100
      },
      {
        "namekey": "*"
      }
    ]
  }
}
```

In this example, Crow keeps up to 100 recent messages for the `AREDN og==` channel. Other local channels use the generic `*` store.

If `size` is not defined, Crow uses the default store size of 50 recent messages.

## How it works in code

Crow's `textstore.uc` module:

- enables message-store behavior when `config.textstore` exists
- uses `DEFAULT_STORE_SIZE = 50`
- loads each store from platform storage as `textstore.<namekey>`
- stores messages by channel `namekey`
- ignores direct/private channels for store capture
- keeps an in-memory index of message IDs to avoid duplicates
- sorts stored messages by receive time
- trims each store to its configured `size`
- saves dirty stores to platform storage every five minutes
- requests recent history from a discovered store shortly after startup
- asks a store for a missing message when a received message references a previous ID that is not present locally

## How other nodes find a store

When MeshIP is enabled and text stores are configured, Crow advertises store support in its AREDN service announcement.

The platform code advertises either:

- the configured channel namekeys, or
- `*` for a generic store

Other Crow nodes discover these published stores through AREDN service discovery. When more than one store is available, Crow tries to order stores by AREDN/Babel route metric so the closest store is preferred.

## Local traffic only

Message stores are intended for local channel history. They are not designed to archive every remote mesh carried through a supernode.

Crow follows the original Raven limitation: text stores store local network traffic, not all traffic received by way of supernode relay.

## Supernode limitation

Do not configure a message store on a supernode.

The original Raven guidance said text stores on supernodes are ignored to avoid burdening supernodes with traffic from every network. Crow enforces the same behavior: when the AREDN platform detects supernode mode, it deletes `config.textstore` at runtime.

## Operational checklist

1. Pick a normal node, not a supernode.
2. Prefer a node with USB storage.
3. Enable USB storage and confirm `/storage status` reports `usb`.
4. Add the `textstore` block to `crow.conf.override`.
5. Restart Crow.
6. Join the desired channels.
7. Test from another node by joining the same channel and confirming recent messages appear.
