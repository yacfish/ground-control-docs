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

## Built-in app decks and matching

A deck matches the frontmost app only when its `bundleId` is exactly equal to that app's bundle id. There is no prefix or wildcard match. Most apps keep one bundle id across versions (`com.adobe.Photoshop`, `com.adobe.illustrator`, `com.adobe.AfterEffects`). Premiere Pro puts the major version in its id, so there is one file per version: `com.adobe.PremierePro.25` (2025) and `com.adobe.PremierePro.26` (2026), with the same buttons. Another Premiere version needs a copy of that file with its own `id` and `bundleId`. To check an app's id on the Mac: `osascript -e 'id of app "Adobe Premiere Pro 2025"'` or `mdls -name kMDItemCFBundleIdentifier /Applications/<App>.app`.

Apple apps (Finder, Preview, QuickTime Player, Keynote) and Max use `menu` buttons, because AXPress by title path survives shortcut changes. Pro apps whose menus are not standard or whose titles change with state (Photoshop tools, Premiere, After Effects, Resolve, Blender, Ableton Live, Zoom, Slack) mostly use `key` buttons with each app's default shortcuts. A user who remapped a shortcut edits the deck on the phone or drops an override file.

## Not yet verified on a Mac

These decks were written from each app's documented defaults and menu titles, not tested on a Mac. Check them before relying on them:

- **Bundle ids**: `com.adobe.PremierePro.25` / `.26` (versioned), `com.adobe.AfterEffects`, `com.blackmagic-design.DaVinciResolve`, `com.ableton.live`, `us.zoom.xos`, `com.tinyspeck.slackmacgap`, `org.blenderfoundation.blender`.
- **QuickTime Player**: menu paths `View > Loop`, `Edit > Split Clip`, `File > New Audio Recording`, `File > Export As > 1080p…` (the export submenu only lists sizes the movie supports). Arrow keys step a frame only while paused.
- **Finder**: `View > Show Preview` and `View > as Gallery` (macOS 10.14 and later). Toggle items change title (Show / Hide Path Bar), so the second press of Path Bar and Preview Pane may need the Hide title.
- **Preview**: `View > Zoom to Fit`, `View > Slideshow`, `View > Show Markup Toolbar`, `Tools > Crop` (dimmed without a selection).
- **Photoshop**: menu titles `View > 100%` (older versions say Actual Pixels), `Edit > Free Transform`, `Layer > Merge Down` (says Merge Layers with several layers selected). Tool keys cycle through grouped tools when Use Shift Key for Tool Switch is off.
- **Illustrator**: `View > Fit Artboard in Window`, `View > Actual Size`; Artboard tool ⇧O.
- **Premiere Pro**: Ripple Trim Q / W, Maximize Panel `` ` ``, Export ⌘M (opens the Export mode in 2023 and later), `\` to fit the sequence.
- **After Effects**: frame step with Page Up / Page Down, Trim In / Out ⌥[ / ⌥], Split Layer ⌘⇧D, menu titles `Composition > New Composition…` and `Composition > Add to Render Queue`.
- **DaVinci Resolve**: page keys ⇧4 Edit, ⇧6 Color, ⇧8 Deliver; Insert F9, Overwrite F10 (macOS may take F9 to F10 as media keys unless Use F1, F2 as standard function keys is on); Split Clip ⌘B; Zoom to Fit ⇧Z.
- **Blender**: keys follow the default keymap with the mouse over the 3D viewport or timeline; Blender has no native macOS menus, so this deck uses keys only. Frame All uses Home.
- **Max**: object keys N, M, C, T, B work only in an unlocked patcher. Existing menu titles (`File > Save…`, `View > Zoom in`) came from the earlier deck and are still unchecked against Max 8 and 9.
- **Ableton Live**: Record F9 (same function key caveat), Draw Mode B, MIDI Clip ⌘⇧M, Rename ⌘R, Detail View ⇧Tab.
- **Keynote**: `Play > Play Slideshow`, `Play > Rehearse Slideshow`, `Slide > Skip Slide`, `View > Navigator` / `Light Table`, `View > Show Presenter Notes`.
- **Zoom**: default meeting shortcuts (⌘⇧A, ⌘⇧V, ⌘⇧S, ⌘⇧T, ⌘⇧W, ⌘⇧F, ⌘⇧H, ⌘U, ⌘⇧R, ⌘⇧P, ⌥Y). They work only in a meeting window, and some can be turned off in Zoom's settings.
- **Slack**: Next Unread ⌥⇧↓, Huddle ⌘⇧H, Set Status ⌘⇧Y, Activity ⌘⇧M, and the React +1 text button (types `+:+1:`, which reacts to the last message only when the message box is focused; press Return after it).

The Safari deck is unchanged in this set: `scripts/smoke-test.js` pins it to its 10 buttons on one page, so growing it needs that test updated first.
