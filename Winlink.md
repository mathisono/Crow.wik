# Winlink

Crow includes early Winlink form support.

The original Raven wiki Winlink page was only a placeholder, so there is no completed Raven operator text to import. This page documents what Crow currently does in code and what still needs to be written as the Winlink workflow matures.

> Status: Winlink support is still being built out. Treat this page as operator notes and maintenance guidance, not a complete Winlink client manual.

## What Crow includes

Crow carries Winlink form files under:

```text
winlink/forms/
```

The current form tree includes ICS and general form content, including ICS-213 style forms.

Crow's `winlink.uc` module loads form definitions from:

```text
winlink/forms
```

It scans form directories, builds a menu from form descriptor files, and pairs each form with its initial/post and viewer files.

## Fields Crow fills automatically

When a form is opened, Crow fills common fields from local node information where possible, including:

- sender callsign
- sequence number
- latitude
- longitude
- grid square
- location source
- GPS signed decimal value when GPS-derived location is available

## Storage location

On internal storage, Winlink form storage uses:

```text
/usr/local/crow/winlink/forms
```

When USB storage is enabled, Crow uses:

```text
/mnt/crow/winlink/forms
```

USB storage is recommended for Winlink because forms are larger and more operationally important than normal chat messages.

## Recommended setup

1. Install Crow normally.
2. If the node has USB, set up USB storage first.
3. From the Crow message composer, run:

```text
/storage usb scan
/storage usb enable
/storage status
```

4. Confirm the storage root is `/mnt/crow`.
5. Confirm the Winlink form path exists:

```text
/mnt/crow/winlink/forms
```

## Good Winlink nodes

Good candidates:

- normal AREDN nodes with reliable power
- event hub nodes
- nodes with USB storage
- nodes also used for local forms, images, shared files, and message history

Poor candidates:

- supernodes
- temporary field nodes without persistence
- nodes running in degraded USB mode
- nodes with very limited internal flash

## Supernode note

Do not use a Crow supernode as the Winlink/form storage node. Supernodes should be kept focused on Crow/MeshIP relay work.

## Current limitations

Known current limitations:

- this is not yet a full Winlink client manual
- form support exists, but the complete operator workflow still needs documenting
- transport details need to be documented as the Crow Winlink implementation matures
- storage and persistence are the most important things to document now

## Maintenance TODO

Fill this page in as implementation details are confirmed:

- how to send a form
- how to view received forms
- supported form list
- supported transport paths
- whether APRS, MeshIP, or other backends are valid for specific form workflows
- troubleshooting missing or broken form files
