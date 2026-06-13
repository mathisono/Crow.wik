# Memory Use

Crow is designed to run on small AREDN/OpenWrt-class nodes where internal flash and RAM are limited. The main rule is:

> Keep the Crow core on the node, but move Crow's growing data layer to external USB storage when the node supports it.

External USB storage does **not** replace RAM, and it should not be treated as the place to run the whole application from. It is used to keep persistent data, message stores, forms, and uploaded images from consuming the node's limited internal flash.

## Why memory and storage matter

Small mesh nodes often have three separate limits:

| Resource | What it is | Why Crow cares |
| --- | --- | --- |
| RAM | Working memory while Crow is running. | Too little RAM can cause crashes, failed parsing, slow UI, or killed processes. |
| Internal flash | Built-in storage on the node. | Filling internal flash can break config writes, package updates, logs, and service startup. |
| External USB storage | Optional attached storage volume. | Best place for persistent Crow data that can grow over time. |

The USB storage feature mostly protects **internal flash**, not RAM. It gives Crow a safer place to store things that grow.

## What should stay on internal node storage

Keep these on the node:

- Crow runtime files
- package files
- service/init files
- minimal configuration needed to boot Crow
- storage configuration that tells Crow where external storage should be

This lets the node boot and lets Crow report a degraded storage state even if the USB drive is missing or unhealthy.

## What should move to external USB storage

When USB storage is enabled and healthy, Crow should use it for the data layer:

| Data type | Preferred USB path | Why |
| --- | --- | --- |
| JSON state | `/mnt/crow/data` | Keeps node database, message state, and local data off internal flash. |
| Message stores | `/mnt/crow/data` | Message history can grow over time. |
| Node database | `/mnt/crow/data` | Mesh node state is persistent and can become large. |
| Winlink/forms storage | `/mnt/crow/winlink/forms` | Form packs can take space and are easier to maintain externally. |
| Uploaded images | `/mnt/crow/images` | Images can quickly fill internal flash. |

The Crow source/runtime stays on internal storage. USB is the persistence/data layer.

## Recommended storage model

| Mode | Use case | Behavior |
| --- | --- | --- |
| Internal storage | Lab testing, very small deployments, no image uploads, no persistent form packs. | Crow stores data on the node. Keep retention short. |
| USB storage | Real AREDN node, field/event deployment, image uploads, Winlink-style forms, or longer message retention. | Crow stores data under `/mnt/crow` and leaves internal flash mostly alone. |
| Degraded fallback | USB missing, full, not mounted, or not writable. | Crow keeps running with temporary fallback storage and warns that persistence is degraded. |

## Enable external USB storage

Use the Crow command channel or slash-command UI.

| Command | Purpose |
| --- | --- |
| `/storage status` | Show active storage mode, root path, image path, mountpoint, and any degraded reason. |
| `/storage usb scan` | List removable USB storage candidates. |
| `/storage usb enable` | Enable the configured external USB storage volume. |
| `/storage usb disable` | Return Crow to internal node storage. |
| `/storage quota images <mb>` | Set the persistent image quota. |

Equivalent `/cmd` command text:

```text
storage usb scan
storage usb enable
storage status
storage quota images 64
```

## Basic setup process

1. Plug in the USB storage device.
2. Open Crow.
3. Run `/storage usb scan`.
4. Confirm the drive is visible.
5. Run `/storage usb enable`.
6. Run `/storage status`.
7. Confirm Crow reports USB storage active.
8. Set an image quota with `/storage quota images <mb>`.
9. Send a test message and upload a small image if image storage is used.
10. Reboot the node and confirm `/storage status` still shows healthy USB storage.

Expected success message:

```text
USB storage active.
```

## Example configuration

Crow can also be configured through storage settings.

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

## Degraded mode

If USB storage is enabled but unhealthy, Crow should not hard-fail. It should enter degraded mode.

Common degraded causes:

| Cause | What happens | Operator action |
| --- | --- | --- |
| USB drive removed | Crow cannot use `/mnt/crow`. | Reattach drive and re-run storage status/enable. |
| USB not mounted | Storage path is missing. | Check mountpoint and platform support. |
| USB not writable | Crow cannot persist data safely. | Check filesystem, permissions, and device health. |
| USB full | Crow cannot safely write new data. | Free space, increase drive size, or reduce quotas. |
| Low free space | Crow may reject persistent image writes or enter degraded state. | Increase free space or lower stored data volume. |

Degraded fallback paths may use temporary storage such as:

```text
/tmp/apps/crow/degraded
/tmp/apps/crow/images
```

Temporary fallback storage is not a long-term persistence plan. Treat degraded mode as a warning that the USB data layer needs operator attention.

## Persistent image quota

Images can consume storage quickly. When external storage is active, use an image quota.

Recommended starting point:

```text
/storage quota images 64
```

That sets the image quota to 64 MB. Crow should prune older images first so the Crow-managed image directory stays within the configured limit.

Suggested settings:

| Deployment | Suggested image quota |
| --- | --- |
| Low-storage node / testing | 16 MB |
| Normal field node | 64 MB |
| Event node with frequent images | 128–256 MB |
| Large USB drive with image-heavy use | Site policy, but still set a quota |

Do not leave image storage unlimited on a small node.

## RAM guidance

USB storage helps with persistent data, but it does not fix RAM pressure.

To keep RAM use under control:

- avoid very large images;
- avoid huge form packs with heavy embedded assets;
- keep message history reasonable;
- avoid excessive debug logging during normal operation;
- restart the service after major configuration or form-pack changes;
- prefer small, text-first Winlink-style forms for field use.

## Flash protection guidance

Internal flash wear and space exhaustion are common risks on small OpenWrt/AREDN nodes.

Use external USB storage when:

- uploaded images are enabled;
- forms are installed;
- message history should survive reboots;
- the node is used for an event or unattended deployment;
- the node has limited internal free space;
- multiple users will be sending traffic through the gateway.

Avoid writing high-churn data to internal flash when USB storage is available.

## Health-check checklist

Run this before a real deployment:

```text
/storage status
/storage usb scan
/storage quota images 64
```

Confirm:

- storage mode is `usb` when intended;
- mountpoint is `/mnt/crow` or the configured mountpoint;
- data path points to external storage;
- image path points to external storage;
- free space is above the configured minimum;
- storage is not degraded;
- image quota is set;
- Crow still starts if the USB drive is temporarily missing.

## Troubleshooting

| Symptom | Likely issue | Fix |
| --- | --- | --- |
| `/storage status` shows internal mode | USB storage has not been enabled. | Run `/storage usb scan`, then `/storage usb enable`. |
| USB drive not listed | Platform does not see the drive or it is not considered removable. | Re-seat drive, check power, filesystem, and platform support. |
| Storage degraded after reboot | Mountpoint or label mismatch, USB slow to mount, or failed health check. | Run `/storage status`, verify label/mountpoint, and re-enable. |
| Images disappear after reboot | Crow is using temporary fallback storage. | Fix USB storage and confirm image path points to `/mnt/crow/images`. |
| Internal flash still filling | Some code path may be writing outside platform storage helpers. | Check image path, logs, and any custom scripts. |
| UI feels slow or service restarts | RAM pressure, not storage pressure. | Reduce image size, message history, form size, or debug load. |

## Related pages

- [USB Storage](USB-Storage)
- [Winlink](Winlink)
- [Strict Gatekeeper](Strict-Gatekeeper)

## Short version

Use USB storage for Crow's growing data: messages, node state, forms, and images. Keep the Crow core on the node. If USB storage fails, Crow should keep running in degraded mode, but operators should fix the external storage before relying on persistence.
