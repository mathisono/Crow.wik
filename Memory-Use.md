# Memory Use and Storage

Crow inherits Raven's lightweight message-storage model, but Crow adds a stronger operator-facing storage path: USB storage.

Crow can run entirely from node internal storage. For busy nodes, event hubs, Winlink/form nodes, image/file sharing, or node message stores, the recommended setup is to keep the Crow application on the node and move the Crow data layer to USB storage.

## Storage locations

Crow uses these main storage areas:

| Data | Internal/default path | USB mode path |
|---|---|---|
| Crow JSON data, messages, node DB, and text stores | `/usr/local/crow/data` | `/mnt/crow/data` |
| Uploaded images and shared binary files | `/tmp/apps/crow/images` | `/mnt/crow/images` |
| Winlink form files/data | `/usr/local/crow/winlink/forms` | `/mnt/crow/winlink/forms` |
| Temporary runtime files | `/tmp/apps/crow` | still `/tmp/apps/crow` for temporary runtime use |

The Crow program, package files, and base configuration stay on the node. USB storage is for Crow data, not for moving the application itself.

## Why USB storage matters

Internal node flash is fine for light use. Text messages are small, and Crow does not need a large database to run.

USB storage is recommended when the node is expected to handle:

- image sharing
- shared files
- Winlink forms
- high-traffic channels
- event operations
- long-running public channels
- node message store / textstore service
- persistent data across reboots or firmware work

A USB-backed Crow node is a much better place for history and files than a small flash-only field node.

## USB storage setup

Crow includes an AREDN USB storage setup script:

```bash
ssh -p 2222 root@<node-name-or-ip>
sh /usr/local/crow/platforms/aredn/usb-setup.sh
```


Setup examples:

```bash
sh /usr/local/crow/platforms/aredn/usb-setup.sh
sh /usr/local/crow/platforms/aredn/usb-setup.sh format
sh /usr/local/crow/platforms/aredn/usb-setup.sh format sda1
```

The script installs USB mass-storage support packages and optional filesystem tools. With `format`, it formats the selected removable drive as ext4 and labels it:

```text
CROWDATA
```

The script refuses to format non-removable devices.

## Using the system

After the USB packages are installed and the drive is attached, use Crow slash commands from the message composer.

Scan for USB storage:

```text
/storage usb scan
```

Enable USB storage:

```text
/storage usb enable
```

Check current storage state:

```text
/storage status
```

Set the image/file quota in megabytes:

```text
/storage quota images 128
```

Disable USB mode and return to internal node storage:

```text
/storage usb disable
```

## Storage states

Crow reports these storage states:

| State | Meaning |
|---|---|
| `internal` | Crow is using the node's built-in storage. |
| `usb` | Crow is using `/mnt/crow` for persistent Crow data and images. |
| `degraded` | USB mode was requested, but the USB device is missing, unsafe, not mounted, not writable, or below the minimum free-space threshold. |

A healthy USB setup should look similar to:

```text
Crow storage: usb
Mode: usb
Root: /mnt/crow
Images: /mnt/crow/images
Mountpoint: /mnt/crow
```

If the USB device is missing or unhealthy, Crow keeps the core service running from internal/degraded storage and reports the reason in `/storage status`.

## What Crow does when USB is enabled

When USB mode is active, Crow:

- mounts the drive at `/mnt/crow`
- expects or creates `/mnt/crow/data`
- expects or creates `/mnt/crow/images`
- expects or creates `/mnt/crow/winlink/forms`
- verifies the storage root is writable
- checks minimum free space
- migrates existing internal message data where possible
- migrates existing temporary image data where possible
- writes a persistent AREDN/OpenWrt fstab entry for the Crow mount
- installs a block hotplug helper so the Crow USB device can be mounted when it appears

The default USB label is `CROWDATA`.

## Minimum free space and quota

Crow's AREDN platform code defaults to:

```text
minimum free space: 16 MB
image quota:        64 MB
```

The image quota can be changed from the UI with:

```text
/storage quota images <mb>
```

Crow prunes the oldest stored image/binary files when the quota is exceeded.

## Internal flash behavior

By default, Crow stores JSON data in internal flash under `/usr/local/crow/data`. Uploaded images and binary files use temporary node storage unless USB mode is active.

Internal flash is acceptable for:

- low-traffic nodes
- testing
- short events
- normal chat with little or no image sharing
- bridge nodes that are not acting as message stores

Use USB storage for:

- high-traffic nodes
- public/event nodes
- image/file sharing
- Winlink form use
- node message stores
- nodes expected to retain data after reboot or upgrade work

## RAM message mode

Crow still supports RAM-backed message storage:

```json
{
  "messages": {
    "ram": true
  }
}
```

In RAM mode, message files are stored under Crow's temporary runtime tree instead of persistent data storage. This reduces writes to internal flash, but messages do not survive reboot unless they can be restored from a node message store.

Do not use RAM mode as a substitute for USB storage when persistence matters.

## Node message stores / text stores

Crow's node message store feature still uses the Raven config name `textstore`.

A message store keeps recent channel messages available so users who join later, restart, or miss a message can recover recent channel history.

Generic message store for all local channels:

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

Specific channel with a larger retained history:

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

If `size` is not set, Crow keeps 50 recent messages by default.

A message store should be placed on a normal node with reliable storage. A USB-backed Crow node is preferred.

## Supernode behavior

Crow can run on an AREDN supernode, but a supernode is for relay, not persistent storage.

When Crow detects that it is running on an AREDN supernode, it ignores or removes these runtime config areas:

- `meshtastic`
- `meshcore`
- `textstore`
- `messages`

That means a supernode does not act as a Meshtastic bridge, MeshCore bridge, RAM-message client, or node message store. Put USB-backed message history on a normal Crow node, and put relay work on the supernode.

## Example storage config

Most operators should use `/storage usb enable`. Advanced users can preconfigure storage in `/usr/local/crow/crow.conf.override`:

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

Then restart Crow.

## Recommended layouts

Small node:

```text
Crow app:  internal node storage
Crow data: internal node storage
Images:   temporary node storage
Use for:   light chat/testing
```

Event or hub node:

```text
Crow app:  internal node storage
Crow data: /mnt/crow/data
Images:   /mnt/crow/images
Winlink:  /mnt/crow/winlink/forms
Use for:  chat, files, forms, local history
```

Message-store node:

```text
Crow app:  internal node storage
Crow data: /mnt/crow/data
Role:      textstore/node message store
Use for:   recent channel history
```

Supernode:

```text
Crow app:  internal node storage
Crow data: minimal local runtime data
Role:      Crow/MeshIP relay
Do not use for: textstore, Meshtastic bridge, MeshCore bridge, message archive
```
