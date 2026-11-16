---
name: reburp-ui
description: Drive and see the Burp Suite window itself over the reburp API - screenshots that work while Burp is minimized, switching tabs including extension sub-tabs, clicking buttons, selecting table rows, typing into fields, combo boxes, dialogs and context menus. Use when a feature has no REST endpoint, when it lives in another extension's UI, or to show the user what Burp currently shows.
---

# Driving the Burp window

Read `reburp-api` first. The `/api/ui` group walks Burp's live Swing tree from inside the JVM, so
it needs no OS accessibility or screen capture and works while Burp is minimized, occluded or on
another desktop. Use it over desktop screenshots and clicking, which Java/Swing blocks.

Reach for a purpose-built endpoint when one exists (`reburp-traffic`, `reburp-scanning`); use
`/api/ui` for what the REST API does not name: another extension's panel, a Burp dialog, a context
menu, or showing the user the result in Burp's own view.

## Seeing

- `GET /api/ui/screenshot` returns a PNG (save it with `-o`). `window=` renders a dialog or popup
  instead of the suite frame, `id=` crops to one element. This call is not written to the activity
  log, so the image bytes never land there.
- `GET /api/ui/tabs` lists the main tab bar plus the nested tab bars showing inside the open tab,
  and what is selected.
- `GET /api/ui/windows` lists every showing window. A dialog is its own window (`w1`), not part of
  `w0` - check here whenever a click seems to do nothing.
- `GET /api/ui/elements?text=&type=` inventories the actionable elements of one window with ids of
  the form `w0:3/1/2`, plus class name, visible text, bounds, and a chooser's items and selection.
  `text` matches text, name, tooltip and items; `type` matches the class name; `all=true` includes
  plain containers and `hidden=true` includes elements that are not showing.

**Ids are paths in the current layout.** They stay valid only until the layout changes, so list,
then act immediately, and list again after switching a tab or opening a dialog.

## Acting

| Goal | Call |
|------|------|
| Switch a tab (Burp's bar, or a nested one) | `POST /api/ui/select-tab` with `name` (substring) or `index` (main bar only) |
| Press a button, checkbox, radio or menu item | `POST /api/ui/click-button` with its label or tooltip |
| Pick a row in the open tab's main table | `POST /api/ui/select-row` with `index`, or `last: true` for the newest |
| Click anything else, or a point | `POST /api/ui/click` with `id` (or `x`/`y` + `window`), `button: "right"` for its context menu |
| Type into a field, area or editable combo | `POST /api/ui/type` with `id`, `text`, `replace`, `enter` |
| Choose in a combo, list, tab bar, table or tree | `POST /api/ui/select` with `id` and `text` or `index` |
| Dismiss or confirm a dialog | `POST /api/ui/key` with `key` (`ESCAPE`, `ENTER`) and optional `id` |

The usual loop is: `select-tab`, `select-row` with `{"last": true}`, `screenshot` - that shows the
user the request and response of the newest entry. `select-row` dispatches a real mouse click so
Burp's detail panes update, which a plain selection change would not do.

## Gotchas

- Burp's own controls (Repeater's **Send**, the message editors' icon-only toggles) are obfuscated
  custom widgets, not `JButton`s: find them by `text`, never by `type`. `click-button` matches
  tooltips too, so `{"text": "line wrapping"}` works.
- A toggle's tooltip names the *next* action, so `GET /api/ui/elements?text=line wrapping` reporting
  "Enable line wrapping" tells you it is currently off - read it instead of clicking to find out.
- Never drive reburp's own **Loaded** checkbox through `/api/ui`: unloading stops the server
  mid-call and wedges the reload.
- A click can open a modal that blocks everything afterwards. If calls start timing out or doing
  nothing, check `GET /api/ui/windows` and clear the dialog.
- You are moving the user's actual Burp window. Put it back on the tab they were on, and do not
  click anything destructive (Delete, Clear, Remove) without asking.
