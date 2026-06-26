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

Winlink forms use Crow's platform storage path instead of hard-coding one physical directory in `winlink.uc`.

The Winlink module asks the platform for relative paths such as:

```text
winlink/forms/<form-directory>/<form-file>
```

On internal storage, the AREDN platform maps that to:

```text
/usr/local/crow/winlink/forms
```

When USB storage is enabled and healthy, the AREDN platform changes the active storage root to `/mnt/crow`, so the same Winlink relative path maps to:

```text
/mnt/crow/winlink/forms
```

USB storage is recommended for Winlink because forms are larger and more operationally important than normal chat messages.

## Code-verified USB behavior

The current code is consistent with USB-backed Winlink form storage.

`winlink.uc` uses the relative constant:

```ucode
const WINLINK_FORMS_DIR = "winlink/forms";
```

and reads form assets through platform helpers:

```ucode
platform.dirtree(WINLINK_FORMS_DIR)
platform.loadbinary(`${WINLINK_FORMS_DIR}/${dir}/${file}`)
platform.loadbinary(`${WINLINK_FORMS_DIR}/${forms[id]?.post}`)
platform.loadbinary(`${WINLINK_FORMS_DIR}/${forms[id]?.view}`)
```

On AREDN, the platform storage path function treats any path beginning with `winlink/` as storage-root relative:

```ucode
if (index(name, "winlink/") === 0) {
    return `${storageRoot}/${name}`;
}
```

That means:

| Storage state | `storageRoot` | Winlink forms path |
|---|---|---|
| internal | `/usr/local/crow` | `/usr/local/crow/winlink/forms` |
| USB active | `/mnt/crow` | `/mnt/crow/winlink/forms` |
| degraded USB | internal fallback root | `/usr/local/crow/winlink/forms` |

The USB activation path also creates the required form directory:

```text
/mnt/crow/winlink/forms
```

and attempts to migrate existing internal Winlink form subdirectories from:

```text
/usr/local/crow/winlink/forms
```

to:

```text
/mnt/crow/winlink/forms
```

So the operator-facing claim is correct: when Crow USB storage is active, Winlink forms are stored and loaded from `/mnt/crow/winlink/forms`.

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

6. Confirm form directories were copied or installed under the active form path.

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
