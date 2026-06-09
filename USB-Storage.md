# Crow USB Storage

Crow keeps the application core on the node and can use an attached USB storage volume as the data layer on AREDN nodes.

External storage is optional. When enabled, it is used for Crow data, forms, and persistent images. The Crow runtime, package files, and minimal configuration remain on internal node storage so the node can still boot and Crow can report a degraded storage state if the external volume is missing or unhealthy.

## External storage can be enabled

On supported AREDN builds, external USB storage can be enabled from Crow's command channel instead of manually editing configuration files.

When using the `/cmd` path, send the command text shown below:

| Command text | Purpose |
| --- | --- |
| `storage status` | Show the active storage mode, root path, image path, mountpoint, and degraded reason if present. |
| `storage usb scan` | List removable USB storage candidates detected by the platform. |
| `storage usb enable` | Enable the configured external USB storage volume. |
| `storage usb disable` | Return Crow to internal node storage. |
| `storage quota images <mb>` | Set the persistent image quota in megabytes. |

In the UI slash-command form, these same actions appear as `/storage status`, `/storage usb scan`, `/storage usb enable`, `/storage usb disable`, and `/storage quota images <mb>`.

## Enabling a new USB drive

Use this process for a new USB storage device on an AREDN node running a supported Crow build.

1. Plug the USB drive into the node.
2. Open Crow and use the command channel.
3. Run `/storage usb scan` to confirm Crow can see the removable drive.
4. Run `/storage usb enable` to activate the configured external storage volume.
5. Run `/storage status` and confirm the state reports active USB storage, the expected mountpoint, and the `/mnt/crow` data paths.
6. Set or confirm the image quota with `/storage quota images <mb>` if persistent image storage needs a site-specific limit.

Equivalent `/cmd` command text:

```text
storage usb scan
storage usb enable
storage status
storage quota images 64
```

Expected success result:

```text
USB storage active.
```

If the result reports degraded storage, Crow should continue running from internal node storage. Check that the USB device is removable, mounted as expected, writable, and has enough free space. Then re-run `/storage usb scan` and `/storage usb enable`.

Do not move the Crow core application files to the USB drive. External storage is only the data layer.

## Storage layout

When USB storage is healthy, Crow uses:

- `/mnt/crow/data` for JSON state, message stores, node database, and other platform data
- `/mnt/crow/winlink/forms` for Winlink and form storage
- `/mnt/crow/images` for persistent uploaded images

The Crow runtime, package files, and minimal configuration remain on internal node storage.

## Degraded mode

Crow periodically verifies that the USB volume is mounted, writable, and has enough free space. If the check fails, the platform switches to degraded mode:

- the core service remains running
- the user is alerted that storage is degraded
- temporary fallback storage is used under `/tmp/apps/crow/degraded`
- image uploads fall back to `/tmp/apps/crow/images`

This protects the node from a hard failure when a USB stick is removed, fails, or fills up.

## Persistent image quota

When USB storage is enabled, images are persistent under `/mnt/crow/images`. A timer prunes old image files so the image directory remains under the configured quota.

Default values:

- image quota: 64 MB
- minimum free space: 16 MB
- health check interval: 1 minute
- image prune interval: 5 minutes

The cleanup policy removes the oldest files first and only acts inside the Crow-managed image directory.

## Runtime configuration

Example configuration override:

```json
{
  "storage": {
    "mode": "usb",
    "mountpoint": "/mnt/crow",
    "label": "CROWDATA",
    "image_quota_mb": 64,
    "min_free_mb": 16
  }
}
```

To return to internal node storage:

```json
{
  "storage": {
    "mode": "internal"
  }
}
```

## Platform API

The AREDN platform exposes storage helper functions:

- `storageStatus()` returns active storage state
- `storageScan()` lists removable storage candidates
- `storageMount()` attempts to activate configured USB storage
- `storageDisable()` switches back to internal storage
- `storageImageQuota(mb)` updates the image quota and prunes if needed

The platform storage path function routes normal data, Winlink/form storage, and images through the active storage root. Existing code should continue using `platform.load()`, `platform.store()`, `platform.loadbinary()`, and `platform.storebinary()`.

## Operator notes

External storage should only be used as the data layer. The Crow core should remain on the node.

If external storage is not available, Crow should keep running from internal storage and clearly report that persistence is degraded until the external storage is restored.
