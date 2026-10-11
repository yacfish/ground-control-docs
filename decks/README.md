# Shortcut decks

JSON button grids (SPEC §6.2). The server loads these at startup. The system deck is the title popup. An app deck is the Deck tab until the phone edits it; the edited copy then lives on the phone. `deck.run` still resolves a built-in button, including Sleep and Shut Down.

## Locations

1. **Built-ins** — this directory (`decks/*.json` in the repo).
2. **User overrides** — `~/.config/ground-control/decks/*.json` (or `$GC_CONFIG_DIR/decks/`).

A user file replaces a built-in when they share the same `id` or the same non-null `bundleId`. Built-ins win only when no user file matches that key.

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

### Confirmation (dangerous buttons)

Buttons whose action is `sleep` / `shutdown`, or any action with `"confirm": true` (including a `sequence` containing one), are sent to the phone with `confirm: true`. The Deck tab shows a confirm sheet (optional per-button `"confirmText"`); only **OK** sends `{ "t": "deck.run", deckId, buttonId, "confirmed": true }`, which then executes immediately. The server answers an unconfirmed run with `error confirm_required` and sends nothing to the helper; the helper itself also refuses `system.sleep` / `system.shutdown` without `confirmed: true`. Clients cannot send `system.*` directly (only via a deck button). Owner rules are the same as any `deck.run` (implicit engage; non-owners get `not_owner`).

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
