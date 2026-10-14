# Shortcut decks

JSON button grids (SPEC §6.2). The server loads these at startup. The system deck is the title popup. An app deck is the Deck tab until the phone edits it; the edited copy then lives on the phone. `deck.run` still resolves a built-in button, including Sleep and Shut Down.

## Locations

1. **Built-ins** — this directory (`decks/*.json` in the repo).
2. **User overrides** — `~/.config/ground-control/decks/*.json` (or `$GC_CONFIG_DIR/decks/`).

A user file replaces a built-in when they share the same `id` or the same non-null `bundleId`. Built-ins win only when no user file matches that key.

Saved phone decks and MIDI layouts are a different folder, `library/` under the same config directory. The filename is the layout name, and two files in that folder cannot share one. A file there is a named copy a phone loads. It does not replace a built-in. See [docs/multi-client.md](../docs/multi-client.md).

## Deck shape

```json
{
  "id": "system",
  "title": "System",
  "bundleId": null,
  "buttons": [
    {
      "id": "spotlight",
      "label": "Spotlight",
      "action": { "type": "key", "key": "space", "mods": ["cmd"] }
    }
  ]
}
```

- `id` — unique deck id (used by `deck.run`).
- `title` — the deck's name. The system deck opens from the Ground Control title; an app deck is the Deck tab.
- `bundleId` — `null` for the system deck; otherwise the app this deck follows (e.g. `com.apple.Safari`).
- `buttons` — 8–12 large hit targets is a good couch size.

## Action types

| `type` | Fields | Helper message |
|---|---|---|
| `key` | `key`, optional `mods` | `key.press` |
| `text` | `text` | `key.text` |
| `activate` | `bundleId` | `apps.activate` |
| `menu` | `path` (string array), optional `id` | `menu.invoke` (AXPress only; SPEC §6.5) |
| `sequence` | `steps` (array of the above, except nested sequence) | each step in order |
| `sleep` | — | `system.sleep` (IOKit `IOPMSleepSystem`, like  → Sleep) — **confirm required** |
| `shutdown` | — | `system.shutdown` (`kAEShutDown` Apple Event to loginwindow, like  → Shut Down; not AppleScript) — **confirm required** |

### Labels

A button whose action is `{ "type": "label" }` is a group heading: the Deck tab shows its `label` as text and a press does nothing (the phone never sends `deck.run` for it). The built-in app decks use labels to group their buttons, for example Tools, Edit, View. A label still takes a grid cell.

### Confirmation (dangerous buttons)

Buttons whose action is `sleep` / `shutdown`, or any action with `"confirm": true` (including a `sequence` containing one), are sent to the phone with `confirm: true`. The Deck tab shows a confirm sheet (optional per-button `"confirmText"`); only **OK** sends `{ "t": "deck.run", deckId, buttonId, "confirmed": true }`, which then executes immediately. The server answers an unconfirmed run with `error confirm_required` and sends nothing to the helper; the helper itself also refuses `system.sleep` / `system.shutdown` without `confirmed: true`. Clients cannot send `system.*` directly (only via a deck button). A deck button does not need the trackpad: another phone can run it while someone else holds the pointer. One phone alone still engages on `deck.run`, as it does for any other action.

Shutdown asks apps to quit normally, so an app with unsaved changes can still interrupt it — same as the Apple menu.

### Mission Control, App Exposé, Show Desktop, Screenshot

These are **macOS keyboard shortcuts**, so they only work if they are enabled in **System Settings → Keyboard → Keyboard Shortcuts… → Mission Control** (Mission Control ⌃↑, Application windows ⌃↓, Show Desktop F11) and **→ Screenshots** ("Save picture of screen as a file" ⌘⇧3). If a shortcut is disabled or remapped there, the button does nothing — re-enable it or edit the deck's `key`/`mods` to match.

macOS stores these hotkeys with the **fn** flag that real keyboards attach to arrows / F-keys (⌃↑ is really ctrl+fn+up, F11 is fn+F11). The helper adds `fn` automatically for arrows, Home/End/PgUp/PgDn, Forward Delete and F1–F12, and the built-in deck also lists `"fn"` explicitly. Dedicated keys `missionControl` and `launchpad` (the F3/F4-row keys on newer Apple keyboards) are available as named keys too.

Screenshot uses ⌘⇧3: an instant full-screen capture saved to the usual location (not the ⌘⇧5 region/toolbar UI). No Screen Recording permission is needed — macOS takes the screenshot, not Ground Control.

Cut (⌘X), Copy (⌘C), Paste (⌘V), Previous (⌘⇧`), Next (⌘`), and Close (⌘W) are keystrokes, so they follow whichever app is in front. The popup does not include Hide, Quit, volume, or playback.

Prefer **`menu`** for per-app shortcuts when a real menu item exists (stable across shortcut changes). Use **`key`** / media keys for system actions that are not menu items (volume, Mission Control, Spotlight, lock screen, screenshot).

Named keys the helper understands include: `space`, `tab`, `return`, `escape`, `delete`, arrows, `f1`–`f12`, `missionControl`, `launchpad`, and media keys `volumeUp`, `volumeDown`, `mute`, `playPause`, `next`, `previous`, `brightnessUp`, `brightnessDown`. Modifiers: `cmd`, `opt`, `ctrl`, `shift`, `fn`.

## Wire

On hello and on `focus.changed`, the server pushes `{ "t": "decks", "system": {...}, "app": {...}|null }`. The client sends `{ "t": "deck.run", "deckId", "buttonId" }`; the server engages the session if needed and talks to the helper. The helper never receives raw `deck.run`.

## Blender deck (`org.blenderfoundation.blender.json`)

The Blender deck follows Blender's own UI order and is split into four pages of exactly 16 entries each, labels and hook slots included:

| Page | Groups, in order | Color |
| --- | --- | --- |
| 1 Navigation and views | View, Viewpoint, Orbit, Pan and Zoom | Navigation `hsl(216 75% 50%)` |
| 2 Modes, tools and shading | Mode | Modes `hsl(288 75% 50%)` |
| | Tools (toolbar top to bottom), Snap and Overlays (header, left to right), Shading (header order) | Tools and shading `hsl(180 75% 50%)` |
| 3 Editing and modeling | Edit, Select, Add, Object and Mesh (header menus left to right) | Editing `hsl(36 75% 50%)` |
| 4 Playback and render | Keying, Transport (Timeline header order), Render | Playback and render `hsl(108 75% 50%)` |

Every group starts with a `label` button. Colors use the existing per-element fields from `src/deckLayout.js` (`DECK_SWATCHES`, `deckSwatch`, `cleanWidget`): `color` on labels and `border` on key buttons, each one of the swatch values. There is no deck or page background color, and none is added.

Blender draws its own menus, so they are not in the macOS menu bar and `menu` buttons cannot reach them. Everything here is a `key` button, and Blender sends a key to the editor under the mouse pointer, so keep the pointer over the 3D Viewport (or the Timeline for its keys).

### Hook slots

Buttons with ids `hook-<controlId>` are `label` placeholders with a blank label (`" "`). Each one marks where a control of the Blender add-on (`blender.deck` from ground-control-app-hooks `blender/`) belongs. Without the add-on they show as empty cells, which keeps the page breaks in place.

| Page | Slot | Hook control |
| --- | --- | --- |
| 1 | after Fly / Walk | `view.all` (Frame All) |
| 1 | after the Viewpoint label | `view` list (Camera, Top, Bottom, Front, Back, Right, Left, the View > Viewpoint order) |
| 1 | Orbit | `orbit.yaw`, `orbit.pitch` (nudge, 15 deg) |
| 1 | Pan and Zoom | `pan.x`, `pan.y`, `zoom` (nudge) |
| 2 | after the Mode label | `mode` list |
| 2 | after the Tools label | `tool` list |
| 2 | after the Shading label | `shading` list (Wireframe, Solid, Material Preview, Rendered) |
| 3 | Edit | `undo`, `redo` |
| 4 | Keying | `key.insert`, `key.delete` |
| 4 | Transport | `frame.start`, (key) Previous Keyframe, `frame.prev`, `play`, `frame.next`, (key) Next Keyframe, `frame.end`, `frame`, `frame.goto` |
| 4 | Render | `render.still`, `render.anim` (sensitive, confirm on the Mac) |

The hook's `location.*`, `rotation_euler.*` and `scale.*` sliders have no slot and stay in the add-on's own Controls view.

### PROPOSAL for the agent wiring the server and phone (not implemented)

Nothing in `server/` or `src/` is changed by this deck. What the loader does today, and what would make the deck work as designed:

1. **Colors are dropped on the wire.** `deckPayload` in `server/decks.js` only sends `id`, `label`, `action` and the confirm fields, so `color` and `border` in a deck file never reach `cleanWidget` on the phone. Passing through `color`, `border`, `alignH`, `alignV` and `fontSize` when present would show them. The phone already validates them against `DECK_SWATCHES`.
2. **Pages are not explicit.** `pagesFromButtons` fills pages by the phone's grid (buttons are `floor(cols / 2)` cells wide, one row high), so 16 entries per page only holds on a grid of 2 columns by 8 rows. A taller phone (the smoke test's 400 x 600 gives 7 x 10, so 20 per page) would shift the breaks. Suggested: an optional `{ "action": { "type": "pageBreak" } }` entry, or a top-level `pages` array, that `pagesFromButtons` honors.
3. **Merging the hook.** When `blender.deck` is live and Blender is frontmost, the phone replaces each `hook-<controlId>` placeholder with the matching hook control (same cell, color taken from the placeholder, since `blender.deck` has no color), and renders it with the Chrome deck widgets for its type. Nudge-style sliders (`next`, `prev`, `nudge` only) are best drawn as two arrow buttons. A placeholder whose control is missing stays blank. Hook controls with no placeholder (the transform sliders) go into a Controls sheet. `blender.deck` also sends `page` and `group` per control, which match this table.
4. A blank label is `" "` because `deckPayload` turns an empty label into the button id.

### Not verified on a Mac

- None of these keys has been tried in Blender through Ground Control. Defaults are from the Blender 4.x keymap.
- Fly / Walk (Shift + `) and View Pie (`): Blender binds the accent grave key. On a Belgian AZERTY layout the helper resolves `` ` `` through the current layout, and that may not be the key Blender expects.
- Wireframe Toggle as Shift Z (Blender 2.8+; older builds used Z alone), X-Ray as Option Z (Alt Z in Blender), Overlays not included.
- Previous / Next Keyframe as Down / Up arrow (Blender 4.x; 2.8 to 3.x also used Up / Down for jumping keyframes, check the direction).
- Deselect All as Option A (Alt A), Invert as Ctrl I, Mode Pie as Ctrl Tab, Snap as Shift Tab, Tool Menu as Shift Space, Zoom Region as Shift B.
- Cmd versus Ctrl: Blender on macOS maps some Ctrl shortcuts to Cmd. Ctrl was kept where the Blender keymap says Ctrl.
