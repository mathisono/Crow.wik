# Winlink Forms

Crow has a lightweight Winlink-style form system for composing and viewing structured messages from the web UI.

The form system is intentionally simple:

- forms live in the Crow install tree under `winlink/forms`;
- each form has a small `.txt` descriptor file;
- the descriptor points to an HTML **post/composer** form and an HTML **view/display** form;
- the UI asks Crow for the form menu and opens the selected HTML form in the browser;
- submitted fields are stored as structured text inside the Crow message;
- received structured messages can be reopened with the matching view template.

This page documents how to use the form UI, how Crow discovers available forms, and how to add more forms.

## Operator workflow in the UI

Typical form workflow:

1. Open the Crow web UI.
2. Choose the channel or direct message target where the form should be sent.
3. Open the Winlink/forms menu in the message composer.
4. Select a form category.
5. Select a specific form.
6. Fill out the form fields.
7. Submit or attach the completed form to the outgoing message.
8. Send the message normally.
9. When a received message contains Winlink structured text, use the UI's form/view action to open the rendered form.

The UI receives the available form menu from Crow at websocket connection time. The backend sends a `winmenu` event containing the form menu. When the user selects a form, the UI requests `winform`. When the user opens a received form, the UI requests `winshow`.

## Backend event flow

| UI/backend event | Direction | Purpose |
| --- | --- | --- |
| `winmenu` | backend → UI | Sends the list of available form categories and form names. |
| `winform` | UI → backend → UI | Requests the HTML composer/post form for a selected form ID. |
| `post` | UI → backend | Sends a normal Crow text message, optionally with structured Winlink form data attached. |
| `winshow` | UI → backend → UI | Opens the view/display template for a received structured form message. |

## How Crow discovers forms

Crow does **not** hard-code the available forms in the UI. The form list is discovered at startup by scanning this directory:

```text
winlink/forms
```

The expected layout is:

```text
winlink/forms/
  Category Name/
    FormName.txt
    FormName_post.html
    FormName_view.html
```

Each subdirectory becomes a menu category. Each `.txt` descriptor file inside that category becomes a selectable form.

Crow builds the form ID as:

```text
Category Name/FormName
```

So this file:

```text
winlink/forms/ICS/ICS-213.txt
```

would become this form ID:

```text
ICS/ICS-213
```

## Form descriptor format

Every form needs a `.txt` descriptor file.

The first line must be:

```text
Form:POST_HTML_FILE,VIEW_HTML_FILE
```

Example:

```text
Form:ICS-213_post.html,ICS-213_view.html
<var msgsender>
<var seqnum>
<var incidentname>
<var to>
<var from>
<var subject>
<var message>
```

The descriptor does two jobs:

1. tells Crow which HTML file to use when composing the form;
2. tells Crow which HTML file to use when viewing a submitted/received form;
3. lists the allowed data fields with `<var FIELDNAME>` entries.

Crow lowercases field names internally, so keep form field names simple and consistent.

## Form file table

| File | Required? | Purpose |
| --- | --- | --- |
| `FormName.txt` | Yes | Descriptor. First line maps post and view templates. `<var ...>` lines define stored fields. |
| `FormName_post.html` | Yes | Composer form shown in the UI when filling out the form. |
| `FormName_view.html` | Yes | Read-only display template used when opening a received/submitted form. |

## Supported automatic variables

When Crow opens the composer/post form, it replaces several common Winlink-style variables before sending the HTML to the UI.

| Variable | Meaning |
| --- | --- |
| `{MsgSender}` or `{var MsgSender}` | Current node/station callsign, if known. |
| `{SeqNum}` or `{var SeqNum}` | Local incrementing sequence number for new forms. |
| `{Latitude}` or `{var Latitude}` | Current station latitude from Crow location data. |
| `{Longitude}` or `{var Longitude}` | Current station longitude from Crow location data. |
| `{GridSquare}` or `{var GridSquare}` | Current station grid square, if known. |
| `{Location_Source}` or `{var Location_Source}` | `GPS` when location came from GPS, otherwise `SPECIFIED`. |
| `{GPS_SIGNED_DECIMAL}` or `{var GPS_SIGNED_DECIMAL}` | Latitude/longitude pair when GPS is available, otherwise `(Not available)`. |

## Submitted data behavior

When a form is submitted, Crow only keeps fields declared in the `.txt` descriptor file.

That means:

- undeclared fields are ignored;
- empty fields are dropped;
- saved field names are normalized to lowercase;
- the saved structured object contains the form ID and the cleaned field data.

The resulting structured text object looks conceptually like this:

```json
{
  "winlink": {
    "id": "ICS/ICS-213",
    "data": {
      "to": "KJ6DZB",
      "from": "W6XYZ",
      "subject": "Status update",
      "message": "Net is active."
    }
  }
}
```

## Viewing received forms

When a received message includes Winlink structured text, Crow uses the form ID stored in the message to load the matching view template.

For each stored field, Crow replaces this placeholder pattern in the view template:

```text
{var FIELDNAME}
```

Any leftover `{var ...}` placeholders are blanked before the rendered form is returned to the UI.

## Available forms

Crow lists available forms dynamically from installed files under `winlink/forms`.

To list the forms installed in a local checkout or on a node, run:

```sh
find winlink/forms -name '*.txt' -type f | sort
```

Or, to show them in the same `Category/FormName` style Crow uses internally:

```sh
find winlink/forms -name '*.txt' -type f | sort | sed 's#^winlink/forms/##; s#\.txt$##'
```

The UI menu should match this inventory after Crow restarts.

### Inventory table

Keep this table updated when forms are added to the repo or deployment image.

| Category | Form name | Form ID | Notes |
| --- | --- | --- | --- |
| _Discovered at runtime_ | _Discovered at runtime_ | `Category/FormName` | Forms are loaded from `winlink/forms/<category>/*.txt`. |

## How to add a new form

### 1. Choose a category

Pick or create a directory under `winlink/forms`.

Examples:

```text
winlink/forms/ICS
winlink/forms/ARES
winlink/forms/Shelter
winlink/forms/Local
```

### 2. Add the descriptor file

Create:

```text
winlink/forms/Local/Field-Report.txt
```

Example descriptor:

```text
Form:Field-Report_post.html,Field-Report_view.html
<var msgsender>
<var seqnum>
<var reporttime>
<var location>
<var operator>
<var summary>
<var needs>
```

### 3. Add the post/composer HTML

Create:

```text
winlink/forms/Local/Field-Report_post.html
```

The form should include input fields matching the descriptor variables.

Use simple field names, preferably lowercase:

```html
<label>Operator</label>
<input name="operator">

<label>Location</label>
<input name="location">

<label>Summary</label>
<textarea name="summary"></textarea>
```

The exact UI submit glue depends on the existing Crow UI form wrapper. Keep the HTML simple and avoid external scripts or network dependencies.

### 4. Add the view/display HTML

Create:

```text
winlink/forms/Local/Field-Report_view.html
```

Example view template:

```html
<h1>Field Report</h1>
<p><b>Operator:</b> {var operator}</p>
<p><b>Location:</b> {var location}</p>
<p><b>Summary:</b> {var summary}</p>
<p><b>Needs:</b> {var needs}</p>
```

### 5. Restart Crow

Forms are discovered during `winlink.setup()`, so restart Crow after adding files:

```sh
/etc/init.d/crow restart
```

If the service is still named Raven on a given image:

```sh
/etc/init.d/raven restart
```

### 6. Verify the UI menu

Reconnect to the web UI and confirm:

- the category appears;
- the form appears under that category;
- opening the form loads the composer HTML;
- submitting the form stores only the expected fields;
- opening the received/sent message renders the view HTML.

## Developer checks

Use these checks before committing a form pack:

```sh
# List descriptor files
find winlink/forms -name '*.txt' -type f | sort

# Check descriptor first lines
find winlink/forms -name '*.txt' -type f -exec sh -c 'echo "--- $1"; head -n 1 "$1"' sh {} \;

# Check declared variables
find winlink/forms -name '*.txt' -type f -exec sh -c 'echo "--- $1"; grep -o "<var [^>]*>" "$1"' sh {} \;
```

## Common problems

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Form category does not show in UI | Crow was not restarted after adding files, or directory is not under `winlink/forms`. | Restart Crow and confirm path. |
| Form name does not appear | Missing `.txt` descriptor or bad first line. | Ensure first line starts with `Form:` and references two files. |
| Composer opens blank | Post HTML file listed in descriptor is missing or misnamed. | Check exact filename and case. |
| Received form opens blank | View HTML file listed in descriptor is missing or misnamed. | Check exact filename and case. |
| Submitted field is missing | Field was not declared in the `.txt` descriptor, or the submitted field name does not match. | Add `<var fieldname>` to the descriptor and keep names consistent. |
| Placeholder remains visible in view | View template uses a name that was not stored. | Make the placeholder match the descriptor/submitted field name. |

## Design notes

Crow's Winlink form support is not a full Winlink Express replacement. It is a lightweight structured-message layer for Crow's UI.

Best practice:

- keep form HTML self-contained;
- avoid external JavaScript/CSS dependencies;
- avoid large embedded images;
- keep field names lowercase and simple;
- prefer text-first forms that can survive low bandwidth and small screens;
- keep emergency/event forms short enough for mesh use.

## Source-code anchors

The current backend implementation is in `winlink.uc`.

Important functions:

| Function | Purpose |
| --- | --- |
| `setup(config)` | Scans `winlink/forms`, builds menu categories, and registers form descriptors. |
| `menu()` | Returns the form menu to the UI. |
| `formpost(id)` | Loads the composer/post HTML and substitutes automatic variables. |
| `post(id, formdata)` | Filters submitted fields to those declared by the descriptor. |
| `formshow(id, formdata)` | Loads the display/view HTML and substitutes stored values. |
