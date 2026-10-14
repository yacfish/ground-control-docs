# Ground Control: Specification

Version 0.3 (October 2026: matches main with helper 0.9.0. The Deck tab is the focused app’s editable grid, and the system grid is the title popup. Preferences, auto Keys, Keys echo and auto-repeat, Text composer, Trackpad zoom, and the zoom timeout are in this build. Zoom timeout confirmed on this Mac. Local checks). Author: Yacine Sebti.

Ground Control turns a phone into a remote **controller** for a Mac: trackpad, keyboard, shortcut decks, MIDI, an Apps surface (Dock, running apps, Applications folder) and the frontmost app's menu bar. It is built for the couch/TV case (Mac on a big screen, you across the room) and works equally over Tailscale. The current build is a phone browser talking to a Node server. The final product is the menu-bar Mac app and the iPhone app in §2.1.

*Naming: the working title during design was **PointerBridge**. That name is historical; the product is **Ground Control**.*

---

## 1. Goals and non-goals

### Goals
- Current build: control a Mac from a phone browser, with no account and no cloud.
- Final product: §2.1. A free Mac menu-bar app, distributed outside the App Store, and an iPhone app whose Pro features are an in-app purchase.
- Feel immediate: pointer latency on LAN low enough that it feels like a trackpad, not a remote desktop.
- Be **visible from across the room**: the cursor grows while you're driving and goes back to normal when you stop.
- Be **context-aware**: shortcut decks and the menu tree follow whichever app is in front.
- Stay **safe**: no stuck keys, no stuck modifiers, no stuck giant cursor, ever, after a disconnect.

### Non-goals
- **No screen streaming.** The phone never shows the Mac's screen. You look at the TV/monitor; the phone is the input device. (This keeps it lightweight, private and fast.)
- No cloud relay, no accounts, no public-internet exposure.
- No Windows/Linux host in v1 (the host is macOS-specific by nature: CGEvent, Accessibility, CoreMIDI).
- Not a general automation tool; actions are things a human would do with a keyboard, mouse or menu.

---

## 2. Architecture

Same shape as [ClipBridge](https://github.com/yacfish/clipbridge):

```
┌──────────── phone (Safari / Chrome) ─────────────┐
│ React (Vite) single-page client                   │
│ trackpad · keys · deck · MIDI · apps/dock · menus │
└──────────────────────┬────────────────────────────┘
                       │ one WebSocket (LAN or Tailscale)
                       │ ws://<mac>:8788/ws   (wss via tailscale serve)
┌──────────────────────▼────────────────────────────┐
│ Mac: Node server (v1)                             │
│  • serves the built client (static)               │
│  • token auth, session ownership, idle/drop logic │
│  • routes messages ↔ helper                       │
└──────────────────────┬────────────────────────────┘
                       │ stdin/stdout, newline-delimited JSON
┌──────────────────────▼────────────────────────────┐
│ Swift helper (gc-helper)                          │
│  • CGEvent posting (pointer, scroll, keys, text)  │
│  • cursor scale                                   │
│  • Accessibility: menu tree read + AXPress        │
│  • NSWorkspace: running apps, activate, launch    │
│  • Dock prefs, Applications folder, app icons     │
│  • CoreMIDI virtual source                        │
│  • text-focus watch (auto Keys), Zoom presets     │
└───────────────────────────────────────────────────┘
```

| Concern | Choice |
|---|---|
| Host runtime (v1) | Node (Express-style static + `ws` WebSocket server) |
| Native layer (v1) | Small Swift command-line helper spawned by Node, long-lived |
| Native layer (later) | Swift **menu-bar app** replaces Node + helper entirely (see §10) |
| Phone client | React + Vite, mobile-first dark UI, installable as home-screen web app |
| Transport | **WebSocket** for all input and state; plain HTTP only to load the page and for `/api/health` |
| Network | LAN or Tailscale. Never port-forward to the internet |
| Default port | **8788** (`PORT` env to override). Must not collide with ClipBridge's **8787** so both can run side by side |
| Auth | Random token in the bookmark URL, same model as ClipBridge (§3) |

### Why a WebSocket, not HTTP
Pointer input is a high-rate stream of tiny relative deltas. One request per move would add per-request overhead and reorder under load. A single persistent socket gives ordered, low-overhead delivery, lets the server push state (focused app, menu tree, app list), and gives a clear **liveness signal**: socket gone ⇒ release everything.

### Why a Swift helper
Node cannot post `CGEvent`s, read the Accessibility tree, create CoreMIDI endpoints or change the cursor without native code. A tiny Swift binary is the smallest native surface and keeps Node as the shell for the current build. The final product folds that binary into the menu-bar app (§2.1).

### 2.1 Final product

Two apps. The Mac app is the host. The iPhone app is the full remote. The browser page stays as the free remote.

- **Mac app.** One menu-bar `.app`, notarized, downloaded from the developer, not from the Mac App Store. It opens at login, holds the Accessibility grant, posts the clicks and keystrokes, serves the socket, and shows a QR code of the pairing address. The private zoom and cursor calls stay in this app, which is why it cannot be sandboxed for the store: the store blocks `CGEvent.post`, and review treats that kind of input injection as too much control.
- **iPhone app.** An App Store app. Free mode connects and drives the trackpad and the keys. Text, Deck, MIDI, Apps, Menus and zoom are visible and locked until a Pro in-app purchase. After purchase or restore, the app sends Apple's signed transaction to the Mac. The Mac checks that signature locally and then accepts Pro commands from that phone. No account and no server of ours.
- **Browser page.** The Mac still serves it. It keeps trackpad and keys for a phone that has not installed the app. It cannot buy Pro, and the Mac refuses Pro commands from it. The QR code opens the iPhone app when it is installed, and this page when it is not.
- **Command gate.** Trackpad and key messages stay open to any paired client. Zoom, text, decks, MIDI, apps and menus run only after that connection has presented a valid purchase. The gate lives on the Mac, so hiding a button is not the only lock.

Store remote-desktop apps are viewers. The program that injects input on the controlled Mac is Apple's screen sharing or a host downloaded outside the store. Ground Control is that host, so the paid listing is the iPhone remote and the Mac app stays free and self-distributed.

---

## 3. Auth and connection

Mirrors ClipBridge:

1. First run generates a random token (24 bytes, base64url) at `~/.config/ground-control/token` (mode `0600`). Above the codes it prints the bookmark URL for this Mac (`http://127.0.0.1:8788/?k=<token>` when listening on every interface) as a clickable terminal link. For every LAN / Tailscale address it prints a QR code of the bookmark URL, with the interface and address named on the line above the code. The MagicDNS form is printed as text:
   ```
   http://127.0.0.1:8788/?k=<token>
   http://<mac-ip>:8788/?k=<token>
   http://<mac>.<tailnet>.ts.net:8788/?k=<token>
   ```
2. The client stores the token in `localStorage`. In the browser it leaves `?k=` in the address and shows a banner until that banner is dismissed, so Add to Home Screen saves the pairing link. In the Home Screen web app (`navigator.standalone`, or `display-mode: standalone` / `fullscreen`) it strips `?k=` immediately and shows no banner. That strip does not change the icon's saved address, so the next launch from the icon still arrives with `?k=`.
3. The client opens `ws://<host>/ws` (or `wss://` when served over HTTPS) and sends the token in its **first message** (`hello`). The server closes the socket if no valid `hello` arrives within 2 s. (Keeps the token out of upgrade-URL logs.)
4. Wrong token, missing `hello` or a `hello` timeout ⇒ close with code `4401`; the client shows a "re-open your bookmark link" screen. A bearer token on the upgrade request (`Authorization: Bearer`) is also accepted.

Tailscale already encrypts the path, so plain `ws://` is fine there. For a proper HTTPS name: `tailscale serve --bg 8788`.

Each Mac runs its own server; the phone keeps one bookmark per Mac.

---

## 4. Sessions, pointer ownership and safety

### One trackpad
Only **one phone holds the trackpad**. Move, scroll, buttons, drag lock, screen zoom and the enlarged cursor belong to that phone. Opening the Trackpad tab sends `session.engage` and claims it before a finger is down. That phone keeps it while the tab stays selected. Leaving the tab sends `session.release` for the pointer. A second phone can still open the tab: the pad says "In use by <device name>", and the buttons, drag lock, modifiers and zoom controls are grayed out. The Trackpad tab icon shows a green dot on the phone that holds it and a red dot on the others. No dot means it is free. There is no live / busy pill in the top bar. `session.takeover` still moves the pointer and sends the previous phone `session.revoked`. It does not release anyone's keys or MIDI notes.

Keys, text, deck buttons, menu presses, app actions and MIDI are accepted from any connected phone. They do not take the trackpad. One phone alone still engages on the first of these and grows the cursor, then releases immediately (`idleMs` is 0). Sitting on the Trackpad tab does not time out.

`window.press` and `window.set` stay with the trackpad phone. Their click fallback moves the pointer.

### Engage / release
- `session.engage`: claims the trackpad when it is free. Server marks this client as owner and **grows the cursor** (§5). If another phone holds it, the claim is refused and the phone is told who does. A claim from the Trackpad tab does not time out.
- `session.takeover`: claims it anyway. The previous phone loses the pointer only.
- `session.release`: this phone gives up what it holds. The trackpad phone also gives up the pointer. Leaving the Trackpad tab sends this.
- **Idle**: `idleMs` is 0. It does not give the trackpad up. The phone that has the Trackpad tab open keeps it. A one-phone session started from a key, text, deck, or MIDI action releases as soon as that action is sent. A latched or locked modifier, or a toggled key, stays down until it is released. That one-phone session releases once nothing is held.
- **Socket drop**: heartbeat missing for 10 s, or socket closes ⇒ that phone's release.

### Release
Each phone releases only what it holds:

1. The trackpad phone: every **held mouse button**, drag lock, and the **cursor** back to its pre-session size. A motion zoom returns to the button level.
2. Whichever phone latched or locked a modifier, or is holding a key: **those keys**. A key another phone is also holding stays down.
3. Whichever phone started a MIDI note: **those notes**. A note another phone is also holding stays on. Every phone is told which notes are still down, so the pads that send that channel and note light together and go dark on the last release.

The last phone to leave, and a phone that is alone, runs the full helper release (`session.release`): buttons, keys, modifiers, cursor and all-notes-off. While another phone is connected the helper gets `session.releasePointer` instead: mouse buttons, cursor and a motion zoom, and not that other phone's keys or notes. Server shutdown does the same. `session.state` carries `keysReleased: true` when this phone's keys were lifted. That clears this phone's deck toggles. Modifier chips follow the shared `mods` message, including after the trackpad is given up.

### Heartbeat
Client sends `ping` every 3 s; server replies `pong`. Server considers the client gone after 10 s of silence. The client reconnects with exponential backoff and re-sends `hello`; the session must be re-engaged (never auto-resumed with held keys).

---

## 5. Cursor visibility (couch mode)

- On `session.engage` the helper **enlarges the system cursor** (configurable scale, default ~3×) and records the previous scale.
- On release (leaving the Trackpad tab, the phone locking, disconnect) or socket drop it **restores** the recorded scale. A cursor timeout can shrink it sooner. Idle does not restore it while the Trackpad tab is holding the pointer.
- Persist the "original" scale to disk before changing it, so a crash or forced quit can restore it on next launch.
- Current build: the helper uses the private `CGSSetCursorScale` (looked up with dlsym) and `cursor-restore.json`. Per phone, Preferences pick the engage size and a **cursor timeout** after which the server shrinks the cursor (`cursor.scale {scale:1}`) and re-grows it on the next pointer activity (§6.7). `cursorMode` is `system` or `off`.

---

## 6. Control surfaces

All surfaces share the same socket and session. Bottom tab bar on the phone: **Trackpad · Keys · Text · Deck · MIDI · Apps · Menus** (the Apps tab has three sections: **Dock · Running · Apps**, §6.4). Landscape and portrait layouts; large hit targets for use without looking.

### 6.1 Trackpad + keyboard
**Trackpad**
- One-finger drag ⇒ relative pointer move (`pointer.move {dx, dy}`), batched per animation frame on the client.
- Gain and acceleration applied host-side: a built-in base gain of 2.5x (`POINTER_BASE_GAIN`) x config `sensitivity` x the phone's pointer speed, then the acceleration curve, so it is consistent across devices.
- Tap ⇒ left click. Two-finger tap ⇒ right click. Double-tap ⇒ double click.
- Tap-and-hold-then-drag (or the **Drag lock** toggle) ⇒ left drag.
- Two-finger drag ⇒ scroll (pixel-precise, with momentum computed on the client). The per-phone **Pointer and scroll speed** multiplier scales scroll too (server-side, §6.7).
- **Left / Drag lock / Right** buttons under the pad.
- Zoom row under them: **1X · 2X · 4X** zoom presets (§6.8).
- Haptic tick on click where the browser supports `navigator.vibrate` (not iOS Safari; degrade silently).

**Keyboard**
- Text field bound to the native phone keyboard; typed text is sent as `key.text` and posted as Unicode (no keycode mapping needed for characters, works with any layout/emoji/dictation).
- Special-keys row: Esc, Tab, ⏎, ⌫, ⌦, arrows, Home/End, PgUp/PgDn, Space; a media row (play/pause, previous/next, volume, mute, brightness) with an **F1–12** toggle that shows the function keys.
- Order: esc ↑ ⏎ ⌫ ⌦ ← ↓ → ⇥ home end pg↑ pg↓ space (`SPECIAL_KEYS` in `src/keysModel.js`).
- ⌫ and the four arrows auto-repeat while held: one press on press-down, repeating from 500 ms, interval 300 ms shrinking linearly to 100 ms over the first 1.5 s of repeating, then 100 ms (`repeatInterval`). Each repeat is the same `key.press` as a tap, with the modifiers active at press-down (latched ones turn off once). Stops on release, pointer cancel / leave / lost capture, tab switch, window blur, page hidden, or disconnect. Pointer events with capture; pointerdown / mousedown are cancelled so the type field keeps focus. The phone keyboard's own Backspace keeps its native repeat.
- **Sticky modifiers** ⌘ ⌥ ⌃ ⇧ and fn are one shared set. F1–12 stay ordinary keys.
  - A tap on any phone turns that chip on (*latched*) on every phone, on the Keys tab and on the trackpad row. Holding the chip for 0.8 s (`MOD_LOCK_MS`) *locks* it. The chip fills and grows 3px on every side while the finger is still down, then returns to its normal size when the finger lifts. The chips around it stay put. A tap on a chip that is already locked turns it off when the finger lifts. Holding that chip does not start the lock timer again.
  - The next key from any phone, or the next mouse-up from the trackpad phone, releases latched chips. The key or click reaches the Mac while the latch is still down, and the latch then clears on every phone. Locked chips stay until any phone taps them.
  - The phone sends `mods.set` with only the keys that changed. The server tells every phone the full set (`mods`) and sends the helper a full `mods.set`. It remembers which phone turned each modifier on. That phone leaving lifts those modifiers on every phone.
  - A full session release clears every modifier (§4). Giving up the trackpad leaves them as they are.
- Echo, Return and IME handling: §6.6.

**Text**
- A second way to type, beside Keys. Keys streams each change as it happens. Text holds a draft and sends the whole entry as one `key.text` when Send is pressed, and again when a history row is resent (tap the bubble). The unsent draft stays on the phone (`gc_text_draft_v1`) when you leave the tab or reload. Send and Return clear it. Blank lines inside the entry are kept; a whitespace-only draft is not sent. The server still caps `key.text` at 4000 characters.
- History (last 50, on the phone only, `localStorage`) is grouped Earlier / Yesterday / Today, newest at the bottom, same composer as ClipBridge. Tap a bubble to resend it. Swipe a row left for Edit (loads it back into the composer without sending) and Delete. The row just sent flashes. Preferences → **Delete all sent text** clears the list and does not type anything on the Mac.
- **Return**, immediately left of Send, is always enabled. With text in the composer it sends that text and then a Return key. With an empty composer it sends only the Return key.
- The tab icon is a text field with a caret. The tab is on by default with the others and can be hidden in Preferences.

### 6.2 Shortcut decks (focus-aware)
- The **Deck** tab is one grid: the deck for the frontmost app. There is no System segment and no swipe between System and App.
- **System popup**: the words **Ground Control** in the status bar open it, on every tab. A 25% black scrim covers the screen under the popup. Tap the title again, or tap the scrim, to close it. It is fixed: no edit mode and no pages. Groups, from `decks/system.json`: **Desktop** (Mission Control, App Exposé, Show Desktop, Spotlight, Screenshot, Lock Screen), **Clipboard** (Cut, Copy, Paste), **Windows** (Previous, Next, Close), **Power** (Sleep, Shut Down). Sleep / shut down are *dangerous*: the phone shows a confirm sheet and `deck.run` must carry `confirmed: true`; server and helper both refuse otherwise (helper `system.sleep` via IOKit, `system.shutdown` via a `kAEShutDown` Apple Event to loginwindow; no AppleScript). Mission Control (⌃↑), App Exposé (⌃↓) and Show Desktop (F11) are sent with `fn` as the standard shortcuts, so they must be enabled in System Settings → Keyboard → Keyboard Shortcuts. Cut, Copy, Paste, and the window shortcuts are keystrokes and follow the focused app.
- **Per-app decks**: keyed by bundle ID. The helper watches the frontmost app (`NSWorkspace` activation notifications) and the server pushes `focus.changed {bundleId, name, pid}`; the client switches to that app's deck. The header shows that app's icon beside its name. An app with no deck shows the empty state.
- Built-in decks are JSON in `decks/`, overridable by `~/.config/ground-control/decks/` when `id` or `bundleId` matches. Until the phone edits an app's deck, those buttons are what show, laid out on the grid. After an edit, that deck is stored on the phone (`gc_app_decks_v1`) and the built-in is only the starting copy. The system grid always comes from the server. Share downloads or uploads the phone's copy as JSON.
- **Editable app deck**, the MIDI gestures on a fixed grid. Cells are 50×50, and a button has its own width and height in cells. **Edit** moves, adds, and removes; a press does nothing while editing. Drag moves a button, the corner control copies it, and the title trash deletes it. Dragging a button off the right edge opens the next page when one does not exist yet, and only while another button remains on the page being left. An emptied page collapses forward. More than one page shows a row of dots and a swipe; one page has no dots. Editor kinds: Menu, Key, Extra Keys, Text, Controls, Label. A menu label set to Auto follows that item's title, including when exactly one title in that menu is replaced by one new title and the neighbors match. Sleep / shut down on a built-in button keep the confirm sheet.
- **Controls** opens the front window's Accessibility controls (buttons, sliders, checks, text fields, steppers), grouped by the containers the app names. `window.controls` lists them; `window.press` and `window.set` act on a path of child indexes. A slider follows the finger.
- **Key** and **Extra Keys** can turn on **Toggle mode**. The first tap sends `key.down` (with the button’s modifiers) and the button fills. The second tap sends `key.up`. The same key shows held on the other phones (the deck button, and the matching arrow on Keys). Any phone can release it. After a 500 ms wait, a non-modifier key repeats as further `key.down` events with `repeat: true` until that second tap. Modifiers stay down and do not repeat. Media keys step with `key.press` on each repeat. A long press still opens the button settings. A click, a scroll step, and screen zoom stay on the Trackpad (§6.8). An app’s own zoom is a menu action when that app offers one. Hide, quit, launch, and opening a Dock item stay on the Apps tab (§6.4). MIDI stays on the MIDI tab (§6.3).
- Built-in decks:

  | Deck | `bundleId` | Buttons |
  |---|---|---|
  | System | (always, title popup) | Desktop, Clipboard, Windows, Power — see the groups above |
  | Safari | `com.apple.Safari` | New Tab, Close Tab, Reload, Back, Forward, Full Screen, Reader, Bookmarks, Find, Private Window |
  | Chrome | `com.google.Chrome` | New Tab, Close Tab, Reopen Tab, Reload, Back, Forward, Incognito, Full Screen, Find, Downloads, Bookmarks, Home, Previous Tab, Next Tab, Previous Window, Next Window |
  | Finder | `com.apple.finder` | File (New Window, New Folder, Get Info, Duplicate, Quick Look, Move to Trash); Go (Home, Downloads, Desktop, Applications, Enclosing Folder); View (as Icons, as List, as Columns, as Gallery, Path Bar, Preview Pane) |
  | Max | `com.cycling74.Max` | File (New Patcher, Open, Save); View (Edit / Lock, Presentation, Sidebar, Zoom In, Zoom Out); Objects (New Object, Message, Comment, Toggle, Button); Audio (Overdrive, Audio Status, MIDI Setup, Max Console) |
  | Spotify | `com.spotify.client` | Play / Pause, Next, Previous, Shuffle, Repeat, Volume Up, Volume Down, Full Screen, Like Song, Search |
  | VLC | `org.videolan.vlc` | Play, Pause, Stop, Next, Previous, Fullscreen, Volume Up, Volume Down, Mute, Subtitles |
  | TV | `com.apple.TV` | Play / Pause, Full Screen, Skip Back, Skip Forward, Volume Up, Volume Down, Mute, Home |
  | QuickTime Player | `com.apple.QuickTimePlayerX` | Playback (Play / Pause, Frame Back, Frame Forward, Full Screen, Loop); Edit (Trim, Split Clip, Rotate Left, Rotate Right, Flip Horizontal); Record (Movie Recording, Screen Recording, Audio Recording, Export 1080p) |
  | Preview | `com.apple.Preview` | View (Zoom In, Zoom Out, Actual Size, Zoom to Fit, Full Screen, Slideshow); Image (Markup Toolbar, Adjust Color, Adjust Size, Crop, Rotate Left, Rotate Right, Flip Horizontal); File (Export) |
  | Photoshop | `com.adobe.Photoshop` | Tools (Move, Brush, Eraser, Lasso, Crop, Eyedropper, Zoom Tool); Brush (Smaller, Larger, Swap Colors, Default Colors); Edit (Undo, Redo, Free Transform, Deselect, Merge Down); View (Fit on Screen, 100%, Zoom In, Zoom Out) |
  | Illustrator | `com.adobe.illustrator` | Tools (Selection, Direct Selection, Pen, Type, Rectangle, Ellipse, Paintbrush, Eyedropper, Artboard); Edit (Undo, Redo, Group, Ungroup); View (Fit Artboard, Actual Size, Outline / Preview, Smart Guides, Zoom In, Zoom Out) |
  | Premiere Pro | `com.adobe.PremierePro.25` | Transport (Play / Pause, Reverse, Stop, Forward, Frame Back, Frame Forward, Previous Edit, Next Edit); Edit (Mark In, Mark Out, Add Edit, Ripple Trim Previous, Ripple Trim Next, Undo); Tools (Selection, Razor); View (Zoom In, Zoom Out, Fit Sequence, Maximize Panel, Export) |
  | Premiere Pro | `com.adobe.PremierePro.26` | Transport (Play / Pause, Reverse, Stop, Forward, Frame Back, Frame Forward, Previous Edit, Next Edit); Edit (Mark In, Mark Out, Add Edit, Ripple Trim Previous, Ripple Trim Next, Undo); Tools (Selection, Razor); View (Zoom In, Zoom Out, Fit Sequence, Maximize Panel, Export) |
  | After Effects | `com.adobe.AfterEffects` | Time (Play / Pause, Frame Back, Frame Forward, Start, End, Work Area Start, Work Area End); Layer (Split Layer, Duplicate, Trim In, Trim Out); Properties (Position, Scale, Rotation, Opacity, Keyframed); Comp (New Composition, Render Queue, Undo) |
  | DaVinci Resolve | `com.blackmagic-design.DaVinciResolve` | Transport (Play / Pause, Reverse, Stop, Forward, Frame Back, Frame Forward, Previous Edit, Next Edit); Edit (Mark In, Mark Out, Split Clip, Insert, Overwrite, Undo); Tools (Selection, Blade, Zoom to Fit); Pages (Edit Page, Color Page, Deliver Page) |
  | Blender | `org.blenderfoundation.blender` | Transform (Grab, Rotate, Scale, Extrude, Add, Delete); Mode (Edit Mode, Shading Pie, Frame All); Timeline (Play / Pause, Jump to Start, Render Image); File (Undo, Redo, Save) |
  | Ableton Live | `com.ableton.live` | Transport (Play / Stop, Record, Session / Arrangement, Detail View); Create (Audio Track, MIDI Track, MIDI Clip); Edit (Split, Duplicate, Consolidate, Loop Selection, Quantize, Draw Mode, Undo, Rename) |
  | Keynote | `com.apple.iWork.Keynote` | Slideshow (Play, Rehearse, Previous, Next, Black Screen, White Screen, Exit); Slides (New Slide, Skip Slide); View (Navigator, Light Table, Presenter Notes) |
  | Zoom | `us.zoom.xos` | Me (Mute / Unmute, Video On / Off, Raise Hand); Share (Share Screen, Pause Share); View (Speaker / Gallery, Full Screen, Chat, Participants); Record (Record Locally, Pause Recording) |
  | Slack | `com.tinyspeck.slackmacgap` | Navigate (Jump To, Back, Forward, Previous Channel, Next Channel, Next Unread); Views (All Unreads, Threads, Activity, Search); Actions (Mark Read, Huddle, React +1, Set Status) |

- A deck button that targets a **menu item** uses the same AX invocation as §6.5 (preferred over a keystroke when an app's shortcut is unreliable). An edited button sends that helper message itself (`key.press`, `key.text`, `menu.invoke`, `apps.activate`). Built-in Sleep and Shut Down still go through `deck.run` with `confirmed: true`.

### 6.3 Virtual MIDI
- The helper creates a **CoreMIDI virtual source** named "Ground Control" (overridable via `config.json` → `midiSourceName` / helper `--midi-source-name`), visible to any DAW / synth / VJ app (Logic, Ableton, MainStage, Resolume…) with no driver install.
- Phone surface: layouts saved on the phone (`gc_midi_layouts_v1`). A menu at the top creates or selects a layout. The gear beside that menu, and creating a layout, open a panel for the name, the grid size, the channel mode, a background colour (a checker swatch of the current colour, or the word Theme on the right when none is set; opened, a hue bar, saturation and brightness canvas, and a `#RRGGBB` field; Theme, between Cancel and Apply, returns to the phone theme at once; Cancel discards a colour edit and Apply keeps it), and whether the octave control is shown. One channel covers the whole grid, or each control has its own. A line under the menu shows the size and channel. When show-octave is on, the octave stepper sits just left of **Edit**. While editing, that spot is a **+** instead. **Edit** changes the controls; **Done** returns to playing. Preferences can turn on MIDI stage mode, which hides the gear, New layout, and Edit. Playing a widget sends the messages below. Edit does not. Ending the session still sends all-notes-off.
- **Grid.** Columns and rows start at 2. Preferences set a minimum cell width and height in pixels, and the most columns or rows you can add is how many cells of that size fit in the MIDI area (up to 48). The grid fills that area and does not scroll. Shrinking the grid drops widgets that no longer fit. Widgets occupy a rectangle and cannot overlap.
- **Move, add, remove.** Drag a widget to a free cell. While it moves, an outline the size of that widget shows the cell it will occupy on release. In edit mode a **+** sits just left of **Done** (the octave stepper is hidden while editing). **+** opens Note, Continuous Controller, or System, then a widget, then its settings. Confirming places it on the first free cells and grows the grid downward, up to the rows that fit, when it is full. A widget already on the grid opens that same settings page after a one-second press that stays within 3 pixels. A short tap selects it and does not move it or open its settings. A selected widget shows a small + inset at its bottom-right corner; that + copies the widget into the next free cells when they exist. The trash icon on the right of the edit title deletes the widget; on a tab capsule it deletes the current tab and closes the settings, and deletes the capsule only when that tab is the last one. The title stays centered, and adding a widget shows no trash icon. The same trash icon on the layout panel deletes that layout. Duplicate, in the spot where that panel used to offer Delete, copies the layout: a name of the form Layout and a number becomes the next free number, and any other name gains “ copy”. A one-second press on a tab’s button opens that capsule’s settings. A change that would overlap or leave the grid is refused.
- **Widgets.** The kinds and their settings are in [docs/midi-widgets.md](docs/midi-widgets.md). Notes follow the layout octave (root 60, labeled C4) or a fixed note. Velocity comes from touch force, or from Y when the phone reports no force. Sliders, the slider array, the 2D pad, the rotary knob, the toggle, the toggle group and the radio send CCs. Pitch is 0–16383 and springs back to 8192. Transport sends `start`, `continue` or `stop` and has no channel. Labels and tab capsules send nothing. A label’s text, border, and background each use the layout colour picker plus an alpha; left unset, the text and border follow the theme and the background stays clear. Align H is left, center, or right, and Align V is top, center, or bottom. Text against an edge keeps a small margin. A tab capsule inherits only the layout octave and channel mode; each tab has its own label, colour and inner grid, and new widgets land in the selected tab until the layout outside the capsule is tapped. In per-control mode, each sounding widget has its own channel. A saved fader or horizontal knob opens as a slider or a square rotary knob.
- A new layout is an empty 4×4 grid on channel 1. A phone that already saved one grid (`gc_midi_layout_v2`) keeps it as Layout 1. A phone that only stored channel and octave (`gc_midi_layout_v1`) keeps those on an empty Layout 1.
- Messages (channel `ch` is **0–15** on the wire; the phone UI shows 1–16):
  - `midi.note {ch, note, vel, on}`: `note`/`vel` 0–127
  - `midi.cc {ch, cc, value}`: `cc`/`value` 0–127
  - `midi.pitch {ch, value}`: 14-bit pitch bend 0–16383 (centre 8192)
  - `midi.transport {cmd}`: optional system realtime `start` / `stop` / `continue` (beyond the core three; same ownership rules)
- The phone already sends the channel and the note after its channel and its octave. The server remembers which phone holds each note and tells every phone (`midi`). A control lights when it is sending a note that is down on that channel. The last phone to release it turns every copy off. One phone leaving silences only the notes nobody else is holding. A CC moves every fader, knob, slider, XY axis, toggle and radio on that same channel and CC. Pitch bend matches on the channel. Transport shows the last command from any phone.
- All-notes-off on release (§4): helper lifts tracked notes and sends CC 123 / 120.
- Later: a virtual destination, so the Mac can light pads and move faders on the phone.

### 6.4 Apps: Dock · Running · Apps
The **Apps** tab (formerly "App switcher") is where you get to any app. A segmented control at the top switches between three sections; the last one used is remembered on the phone.

| Section | Shows | Tap | Source on the Mac |
|---|---|---|---|
| **Dock** | What's in the Mac's Dock: Finder + pinned apps, then *Folders & links* (Downloads, stacks, web links), then *Recent* apps if the Dock shows them | opens it: launches or activates an app, opens a folder/file/link like a Dock click | Dock preferences (`com.apple.dock`) |
| **Running** | Running regular apps (the app switcher), frontmost highlighted | activates; long-press ⇒ Bring to front / Hide / Quit (quit asks for confirmation) | `NSWorkspace.runningApplications` |
| **Apps** | Every `.app` in the Applications folder(s), alphabetical, with a search field | launches, or activates if already running | file-system scan of the Applications folders |

Common to all three: large icon tiles (≥ 96 pt tall, 4 per row on a phone), a small dot under apps that are running, a refresh button, and a brief "launching" pulse on the tapped tile until the Mac acknowledges. Icons are small PNGs sent once and **cached client-side** (§6.4.4).

#### 6.4.1 Running apps
- `NSWorkspace.runningApplications` filtered to activation policy `.regular` (what the Dock / ⌘-Tab show), in launch order, with `active` (frontmost) and `hidden` flags.
- `apps.activate {bundleId}` unhides if needed and activates; `apps.hide`, `apps.quit` (polite `terminate()`, never force-quit).
- The helper **pushes** a fresh list (broadcast `apps`) on launch / quit / hide / unhide / activate (debounced ~300 ms), so the Running section and the running dots elsewhere stay live without polling.

#### 6.4.2 Dock
- **Read the Dock's own preferences** through CFPreferences (`CFPreferencesCopyAppValue` on domain `com.apple.dock`), the same data the Dock draws from:
  - `persistent-apps`: pinned apps (`tile-data.file-data._CFURLString`, `file-label`, `bundle-identifier`);
  - `persistent-others`: right-hand side: folders/stacks (`directory-tile`), files, web links (`url-tile`);
  - `recent-apps`: "suggested and recent apps", shown only when `show-recents` is on (default on);
  - spacer tiles are skipped. **Finder** is always the Dock's first item but is not stored in the prefs, so it is prepended (`/System/Library/CoreServices/Finder.app`). Trash is not listed in v1.
- Pinned apps that moved are re-resolved by bundle id through LaunchServices (`urlForApplication(withBundleIdentifier:)`); entries that can't be resolved are sent with `missing: true` and shown disabled.
- Each tile gets an `id` (`finder`, `apps:<n>`, `others:<n>`, `recent:<n>`). `dock.open {id, path?/url?}` re-reads the prefs, re-resolves the id (falling back to the path/url if the Dock changed) and opens it; **only items currently in the Dock** can be opened this way.
- Apps launch via `NSWorkspace.openApplication(at:configuration:)` (activates if already running); folders, files and links via `NSWorkspace.open(_:)`, i.e. what a Dock click does.
- **No AppleScript, no System Events, no UI scripting of the Dock.** Preferences are read-only; Ground Control never writes the Dock's prefs. Reading the Dock process's Accessibility tree (`AXList` children of the Dock app) is the documented *fallback* if a future macOS stops keeping the Dock layout in `com.apple.dock`; it is not used in v1.
- The Dock list is fetched when the section is opened or refreshed (no file watching in v1).

#### 6.4.3 Applications folder
- Scan, in this order, de-duplicating by resolved path and then by bundle id: `/Applications`, `/System/Applications` (Finder merges both into "Applications"), `/System/Cryptexes/App/System/Applications` (Safari since macOS 13; `/Applications/Safari.app` is only a symlink to it), `~/Applications`, plus **one level of sub-folders** (e.g. `Utilities`, vendor folders like `Adobe …/`). Never descend into bundles; don't follow symlinked folders.
- Per app: display name (`FileManager.displayName`, `.app` stripped), `bundleId`, `path`, `folder` (sub-folder name, if any), `iconId`, `running`. Sorted with Finder-style `localizedStandardCompare`.
- Scanned on a background queue on request (`apps.catalog`), so a few hundred `Info.plist` reads never stall pointer input. No caching in the helper in v1.
- `apps.open {path}` (or `{bundleId}`) launches or activates. The server only forwards `.app` paths; the helper re-checks that the bundle exists.
- Client: alphabetical grid with a search box (substring on name / bundle id). A–Z index, folders-as-groups and favourites are later polish.

#### 6.4.4 Icons
- Every list entry with an icon carries an `iconId` derived from the bundle path and its `Info.plist` modification time: stable across restarts, and **new after an app update**, so the client can cache by id forever.
- The client asks only for ids it doesn't have (`icon.get {iconIds}` in chunks of ≤ 24); the helper renders `NSWorkspace.icon(forFile:)` to a 96 × 96 PNG off the main thread and replies one `icon {iconId, png}` each (`png: null` if it has no icon or the id is unknown).
- The phone keeps icons in **IndexedDB** (localStorage is too small for a full Applications folder) plus an in-memory map; falls back to memory-only where IndexedDB is unavailable.

#### 6.4.5 Ownership
Listing (`apps.list`, `dock.list`, `apps.catalog`, `icon.get`) is read-only and allowed for any authenticated client. Actions that change what's on the Mac's screen (`apps.activate` / `hide` / `quit` / `open`, `dock.open`) require pointer ownership, with implicit engage like `deck.run` (§4).

### 6.5 Menu tree via Accessibility
- The phone shows the **frontmost app's menu bar as a navigable tree** (drill-down lists, breadcrumb back), including each item's enabled state, checkmark state and keyboard-shortcut hint.
- The **Apple menu () is always present** as the first top-level entry (About This Mac, System Settings, Sleep, Restart, Lock Screen, …), regardless of which app is in front.
- Read via the Accessibility API: `AXUIElementCreateApplication(pid)` → `kAXMenuBarAttribute` → children recursively (titles, `AXEnabled`, `AXMenuItemMarkChar`, `AXMenuItemCmdChar` / modifiers). Loaded lazily per submenu to stay fast on apps with huge menus; refreshed on `focus.changed` and on demand.
- **Invocation (locked decision):** a menu item is triggered only by its **Accessibility action** (`AXPress` on the `AXMenuItem` element). Ground Control **does not use AppleScript / System Events** for menus, and **never fakes clicks on menu-bar screen coordinates**: coordinate clicking is fragile (resolution, scaling, multiple displays, menu-bar auto-hide, notch, localisation) and visibly opens menus on the TV. If AXPress fails, surface an error; do not fall back to scripting.
- Items are addressed by **path** (e.g. `["View", "Enter Full Screen"]`) plus a per-refresh element id, so a stale id can be re-resolved by path.
- Dynamic menus (Recent items, Windows, etc.) are re-read when opened on the phone.
- **Ownership:** browsing (`menu.get`) is read-only and allowed for any authenticated client; `menu.invoke` changes what's on the Mac, so it requires pointer ownership with implicit engage, like `deck.run` (§6.4.5).
- Phone UI: drill-down list with breadcrumb + back, Apple menu first, ✓ / shortcut hint / dimmed-when-disabled per row, large rows (≥ 56 pt). Items that quit the app or end the session (*Quit …*, *Sleep*, *Restart*, *Shut Down*, *Log Out*, without "…") ask for confirmation on the phone first.

### 6.6 Keys: echo, Return and IME
- The Keys tab types through a hidden textarea. Each change is diffed against the previous value: added characters go out as `key.text`, removed ones as `key.press delete`, so autocorrect, dictation and IME conversions arrive as ⌫ + new text (surrogate pairs never split).
- **Return** sends a real `key.press return`, never a newline in the field: a plain Enter keydown is handled and prevented; soft keyboards without a usable keydown (Android keyCode 229, some iOS IMEs) are caught on `beforeinput` `insertLineBreak` / `insertParagraph`, and any newline that still lands in the field is turned into a Return press (`textSteps`).
- **Echo** (display only, never affects what is sent): a strip of spaced tokens (one per code point, 0.6em gap) showing what was just typed. Named keys render as pills: **ENTER**, **SPACE**, **TAB**, **BACKSPACE**. A pure deletion appends one BACKSPACE token per deleted character instead of erasing the echo; a replacement (IME, autocorrect) rewrites the tail. Special-key buttons add their tokens too.
- **Idle clear**: 1 s after the last keystroke (`ECHO_IDLE_MS`) the echo clears and the hidden field is emptied, unless an IME composition is in progress (then on the next idle after `compositionend`). Nothing is sent when it clears. A **clear** button empties both at once.
- Auto-repeat for ⌫ and the arrows, and the special-key order: §6.1. Opening the phone keyboard from the Keys tab and switching to Keys on Mac text focus: §6.7.

### 6.7 Top menu and Preferences (phone)
The top bar has a **⋮** menu (same look and behavior as ClipBridge: closes on any outside tap, capture phase): **Light mode / Dark mode** (saved on the phone), **Preferences**, **Disconnect** (closes the socket, the server releases everything; a **Connect** button reconnects without a reload).

Preferences (centered Cancel / OK dialog), saved in the phone's `localStorage` and re-sent as `prefs.set` after every `welcome`:
- **Tabs**: which bottom tabs are shown; at least one stays on.
- **Switch to Keys automatically when typing is needed** (off by default, needs helper 0.6.0+; 0.6.1+ adds the auto-Keys fixes, current helper 0.8.2). While any phone has it on (and its Keys tab is visible), the server sends the helper `textfocus.watch {on:true}` and logs `[gc] auto Keys: text focus polling on/off`; the helper acks with `textfocus.status {ok, watching, reason?}` and polls every ~300 ms on a background queue (0.25 s AX timeout per app). Frontmost pid: system-wide `AXFocusedApplication`, else `NSWorkspace.frontmostApplication`, else the owner of the frontmost layer-0 window (CGWindowList). Chromium and Electron apps (Chrome, Brave, Edge, Arc, Vivaldi, Opera, Chromium, anything bundling Electron Framework) only expose web content to AX once asked: the helper reads the app's `AXRole` and sets `AXManualAccessibility=true` once per pid (also on any app whose focused-element query fails). It never sets `AXEnhancedUserInterface` (breaks window animations). Side effect: some Electron apps (VS Code) may switch to their screen-reader mode. It emits `textfocus {editable, role?, subrole?, pid?}` when editable flips or focus moves to a different text input (field to field re-triggers). Text inputs: `AXTextField` (incl. search / secure), `AXTextArea`, `AXSearchField`, editable `AXComboBox`, and web content with `AXEditableAncestor`. The phone switches to Keys and goes back to the previous tab ~0.7 s after focus leaves, unless the user picked a tab meanwhile. No watch, no polling when nobody has it on. Failure modes are visible: phones get `textfocus.status {state: ok | unsupported | unavailable | no_accessibility}` (shown under the toggle in Preferences), and the server logs once when the helper is too old, not running, or lacks Accessibility.
- **Open keyboard when Keys tab is selected** (off by default, phone only, not sent to the server). Every tap on the Keys tab focuses the hidden Keys textarea synchronously in the tab's click handler, because iOS Safari only raises the keyboard for a `focus()` made inside the user gesture. Keys already showing: the field is focused directly (blurred first if it still counts as focused, so a dismissed keyboard comes back). Keys not mounted yet: an always-mounted, offscreen proxy textarea is focused in the tap, and Keys moves focus to its field right after it mounts (the keyboard stays up across that programmatic focus change); the proxy is blurred after 600 ms if Keys never mounts. An automatic switch to Keys requests the same focus without a gesture: Android / desktop focus the field, iOS will not show the keyboard. With the software keyboard down, Keys shows a round button at the bottom right of the panel (a keyboard with an upward chevron). That tap focuses the type field inside the gesture, blurred first if iOS still counts it as focused, and the button hides while `visualViewport` reports a keyboard inset.
- **Mouse**: **Pointer and scroll speed** (`pointerSpeed`, 0.25 to 3, multiplies `POINTER_BASE_GAIN` (2.5) x `sensitivity`; the same multiplier, without the base gain or acceleration, scales `pointer.scroll` dx / dy server-side, so 1.0 leaves scroll unchanged), **Acceleration** (0 to 1.5, pointer only, replaces config `acceleration` for this phone), **Invert scroll direction** (flips both scroll axes server-side).
- **Cursor size**: Off / Small / Medium / Large = engage scale 1 / 2 / 3 / 4, sent to the helper with `session.engage {cursorScale}` and live as `cursor.scale {scale}` while engaged.
- **Cursor timeout**: `cursorTimeout` seconds (0.5 to 10, slider steps 0.5 / 1 / 2 / 3 / 5 / 10) or 0 = *Until I stop driving* (it stays big until the phone leaves the Trackpad tab, locks, or disconnects). `null` = config `cursorTimeoutMs` (default 0). Timed on the server: after the owner's last pointer activity (`pointer.move` / `button` / `click` / `scroll`, or a fresh `session.engage` touch) it sends `cursor.scale {scale:1}`; the next pointer activity sends `cursor.scale {scale}` again. Does not end the session. Needs helper 0.6.0+ (`cursor.scale`); older helpers keep the cursor big until release.
- **Zoom entry method** (phone only, not sent in `prefs.set`): **Pinch** (default), **Slider**, or **Buttons**. Pinch and the slider set a factor from 1.00 to 6.00 (`zoom.set {factor}`). Buttons stay the discrete 1X / 2X / 4X levels (`zoom.set {level}`). Choosing another method on Preferences OK returns the zoom to 1X. Two-finger drag still scrolls while pinch is the entry method.
- **Zoom during motion**: 1X / 1.25X / 1.5X (`zoomMotion` 1 / 1.25 / 1.5, default 1X). While this phone moves the pointer, the Mac screen zooms to that level (private zoom), then returns to the button level `zoom.motionIdleMs` (500 ms) after the last movement; 1X never zooms in further on movement (§6.8).
- **Zoom timeout**: Off / 300s / 60s / 15s (`zoomTimeout` 0 / 300 / 60 / 15, default 60). A 2X or 4X button level returns to 1X after the phone sits still for that long (§6.8). Confirmed on this Mac.

### 6.8 Zoom presets (Trackpad)
- Discrete levels are 1X / 2X / 4X, one action per press. Pinch and the slider (§6.7) send `zoom.set {factor}` from 1 to 6, rounded to 0.01; the helper sets that private factor without the button-level verify. macOS has no public API that sets the Zoom level (UniversalAccess.h only offers `UAZoomEnabled` / `UAZoomChangeFocus`, which move the zoomed view's focus), and on macOS 12.7.6 Accessibility Zoom ignored our synthesized Control + scroll (0.7.0 to 0.7.2, nil-source and HID-state source, HID tap, flags on the event). Helper 0.8.x therefore has three methods, picked by config.json `zoom.method` (sent with every `zoom.set`):
- **`private`**: the WindowServer zoom, `CGSSetZoomParameters(cid, &origin, factor, smooth)` with `CGSMainConnectionID()` (or the SkyLight `SLS` aliases), resolved with dlsym at first use so a missing symbol means "not available", never a crash. Absolute: 1X / 2X / 4X = factor 1 / 2 / 4, origin chosen so the pointer stays put. Verified by `CGSGetZoomParameters` after each set; a set that does not read back marks the method as not working (auto then falls back). This zoom is independent of Accessibility Zoom's state (`closeViewZoomFactor` does not change). The origin is chosen around the pointer when the level is set. The view stays there until the pointer reaches an edge of the magnified area, then the helper pans the origin only enough to keep the pointer on that edge. The factor does not change. Edge panning is the only later change to the origin. 0.8.0 / 0.8.1 re-applied the last successful private factor around the pointer on every `pointer.move` (pointer-follow) until a later private Set(1) succeeded, so a 1X that went through keys / scroll (auto fallback after an unverified smooth set), a failed Set(1) or a zoom-out done elsewhere left that factor at 2 / 4 and moving the pointer zoomed the screen back in; 0.8.2 removed the follow entirely. A smooth set is verified by polling `CGSGetZoomParameters` for up to ~0.4 s; before a `keys` / `scroll` press the helper puts a private zoom it holds back to 1 (no stacking). A 1X / 2X / 4X button level stays across `session.release` (Release, disconnect, idle, takeover) and across other tabs, including a `key.text` send. `session.release` only drops a motion zoom back to that button level. The private zoom goes back to factor 1 on helper exit, since nothing could undo it once the helper is gone. A helper that is killed or crashes cannot run that reset, so when the server starts or restarts the helper, the first `zoom.get` reply with `privateAPI: true` and `privateFactor > 1.001` makes it send `zoom.resetPrivate`: the helper re-reads the factor with `CGSGetZoomParameters` and, if still above 1x, sets it to 1 and replies `zoom.state {mode: "reset"}`; the server then sets its tracked level to 1X. Accessibility Zoom is never touched by this reset (0.8.1). Manual recovery: `node scripts/zoom-probe.js private 1`, or restart `npm start`. Signatures are the community-documented ones (stackoverflow.com/questions/13100589; the SLS zoom symbols are listed in github.com/NUIKit/CGSInternal/issues/3); unverified on macOS 12.7.6.
- **`keys`**: the Zoom keyboard shortcuts ("Use keyboard shortcuts to zoom", `closeViewHotkeysEnabled`). The shortcuts are read from `com.apple.symbolichotkeys` `AppleSymbolicHotKeys` 17 (zoom in) and 19 (zoom out) `parameters` [char, keyCode, modifiers] (defaults ⌥⌘= 0x18 / ⌥⌘- 0x1B). One action = modifiers down once, N key down / up pairs back to back (`keyGapMs`, default 0), modifiers up, all from the HID-state source at the HID tap. N = pos(target) - pos(from) with 1X = 0, 2X = `keySteps2x` (4), 4X = `keySteps4x` (8); 1X zooms out at least `keyOut` (20) presses (clamps at 1x).
- **`scroll`**: the 0.7.2 gesture: modifier key-down (`flagsChanged`, `kVK_Control` 0x3B), ONE scroll-wheel event of pos(target) - pos(from) (2X = `units2x`, 4X = `units4x`, 1X at least `-out`) carrying the modifier flags, modifier key-up, `leadMs` apart. Experiment switches: `scrollTap` `hid` (default) or `session`, `continuous` (sets `kCGScrollWheelEventIsContinuous`), `suppress` (source `localEventsSuppressionInterval = 0` and local events permitted during suppression).
- **`auto`** (default): `private` if the symbols exist and the last private set verified, else `keys` unless `closeViewHotkeysEnabled` is false, else `scroll`. `zoom.state` reports the method used, `requested`, `fallback`, `presses` / `delta` / `privateFactor`, and the Accessibility Zoom factor before / after (`settleMs` later). `zoom.get` reports `privateAPI`, `privateSymbols`, `hotkeys`, `hotkeyIn`, `scrollZoom`.
- Relative methods use the last level set: the server tracks it (`from`, assumed 1X at server start) and presses are never dropped helper-side; **1X is a reset** for them (it overshoots toward zoom-out). Pressing the current 2X / 4X again does nothing for `keys` / `scroll`.
- Runs on a helper serial queue, never the main thread. Stuck-modifier safety: the zoom tracks the modifier keys it holds; if it has not finished within the exit timeout (0.5 s) the exit path posts their key-ups with nil-source events before `exit(0)`. A `zoom.set` counts as keyboard use, so the next release runs the modifier key-up sweep. No AppleScript.
- Owner only with implicit engage (it changes the Mac screen); the server forwards only `zoom.set`, never a pointer scroll. Release does not send `zoom.set`. The helper's release leaves a private 1X / 2X / 4X button level in place (Accessibility Zoom set through `keys` / `scroll` is not touched either). The server keeps the zoom state (`level` = last level set) and pushes it to every phone; the phone highlights the level (pending while a press is in flight). Capability: helper `ready {zoom:true}` (0.7.0+); with an older helper the buttons are disabled, `zoom.set` gets `error unsupported`, and the server logs a rebuild hint once. The server logs the methods the helper sees (once per helper version), every press (from → to, method, what was posted, factor), once when the keyboard shortcuts are off while `keys` is used, and once when the scroll gesture is off while `scroll` is used.
- **Zoom during motion** (phone Preferences `zoomMotion`: 1, 1.25 or 1.5, default 1; sent in `prefs.set`, applied live; helper 0.8.2+, `ready {zoomMotion:true}`). Timed on the server, per owner: an owner `pointer.move` that actually moves, with `zoomMotion` above the button level (`zoomAssumed`), sends `zoom.motion {on: true, level, smooth}` once per burst and arms one idle timer (re-armed for whatever is left, like the cursor timeout); `zoom.motionIdleMs` (config, default 500, 100 to 10000; also in `welcome.settings.zoomMotionIdleMs`) after the last move it sends `zoom.motion {on: false}`. Helper (private API only): on = Set(max(level, button level)) around the pointer, verified with Get, silent if already there; off = Set(button level, 1.0 if none) and verify with Get. It replies `zoom.state {mode: motion, active, level, base?, factor, ok, error?}`; the server keeps the buttons on the button level and pushes `zoom.motion` (level or null) in its zoom state. Interplay with the buttons: motion only zooms in beyond the button level and returns to it (1.25X / 1.5X do nothing while a 2X / 4X button is set); a button at or above the motion level replaces the motion zoom; a lower one is recorded and applied when motion ends. 1X (or a level not above the button) while motion-zoomed sends `{on: false}` right away. Giving up the trackpad drops only a motion zoom, back to the button level (`zoom.state` mode `motion`, `reason: release`, `level` = that button level). The helper does that on `session.release` (this phone is alone, or it is the last one) and on `session.releasePointer` (another phone is still connected). A private button level stays, and the server keeps tracking it. Helper exit still resets the private zoom to 1x, and a killed helper's zoom is reset on the next start (`zoom.resetPrivate`).
- **Zoom timeout** (phone Preferences, under Zoom during motion: **Off** / **300s** / **60s** / **15s**, default 60). Confirmed on this Mac. A 2X or 4X button level returns to 1X this long after the phone stops being used. Pointer moves, keys, text, decks, and the other actions that restart the session idle timer (`_touchIdle`) start this wait again from the beginning. A heartbeat `ping` does not. `session.release` (Release, disconnect, idle, the phone locking) leaves the button level and the wait in place. Off (`0`) leaves the level until 1X is pressed. Changing `zoomTimeout` while a 2X or 4X level is up re-arms immediately, so 15 s is not stuck behind a 300 s wait. The server times it and, when it elapses, sends `zoom.set {level: 1, from}` without engaging a session. If a motion zoom is on screen, that ends first, then the button level goes to 1X. `zoomTimeout` is in `prefs.set` (0, 15, 60, or 300; `null` resets to 60).
- Try each method with `node scripts/zoom-probe.js method <auto|private|keys|scroll> <1|2|4> [--from N]` (§12).

---

## 7. Wire protocol (v1 draft)

JSON text frames, one message per frame. Every message has `t` (type). Client→server input messages may be batched as `{t:"batch", m:[…]}` per animation frame.

### Client → server
| `t` | Payload | Notes |
|---|---|---|
| `hello` | `{token, device, clientVersion}` | must be first; ≤2 s |
| `ping` | `{}` | every 3 s |
| `session.engage` | `{}` | claims the trackpad when it is free; grows the cursor |
| `session.release` | `{}` | releases what this phone holds (§4) |
| `session.takeover` | `{}` | takes the trackpad from another phone; their keys and notes stay |
| `pointer.move` | `{dx, dy}` | relative, device px |
| `pointer.button` | `{button: "left"\|"right"\|"middle", down: bool}` | |
| `pointer.click` | `{button, count}` | convenience for tap / double-tap |
| `pointer.scroll` | `{dx, dy}` | pixel deltas |
| `key.text` | `{text}` | posted as Unicode |
| `key.press` | `{key, mods: [...]}` | named key with modifiers, e.g. `{key:"tab", mods:["cmd"]}` |
| `key.down` / `key.up` | `{key, mods?}` | holds. `mods` are the modifiers that went down with the key. A `key.up` from any phone releases that hold |
| `mods.set` | `{cmd?, opt?, ctrl?, shift?, fn?}` | only included keys change (`off`/`latched`/`locked`). The server sends the helper every key |
| `deck.run` | `{deckId, buttonId, confirmed?}` | server resolves to action; `confirmed: true` required for sleep / shutdown / `confirm:true` buttons |
| `midi.note` / `midi.cc` / `midi.pitch` / `midi.transport` | see §6.3 | |
| `apps.list` | `{}` | running apps (§6.4.1) |
| `apps.activate` / `apps.hide` / `apps.quit` | `{bundleId}` | any phone; one phone alone still engages |
| `apps.catalog` | `{}` | Applications folder listing (§6.4.3) |
| `apps.open` | `{path?, bundleId?}` | launch or activate; `path` must end in `.app`; any phone |
| `dock.list` | `{}` | Dock items (§6.4.2) |
| `dock.open` | `{id, path?, url?}` | open a Dock item by id (re-resolved helper-side); any phone |
| `icon.get` | `{iconIds: [...]}` | ≤ 64 ids per message (client sends ≤ 24) |
| `menu.get` | `{path?: [...]}` | top level when omitted; any client |
| `zoom.set` | `{level: 1\|2\|4}` | zoom preset (§6.8); owner only (implicit engage); `bad_request` for other levels, `unsupported` with a pre-0.7.0 helper |
| `prefs.set` | `{pointerSpeed?, acceleration?, cursorScale?, cursorTimeout?, autoKeys?, invertScroll?}` | per-phone Preferences (§6.7), any client; sent after every `welcome`; `null` resets a field to the server default |
| `menu.invoke` | `{path: [...], id?, pid?}` | AXPress only; never AppleScript or coordinates; any phone; `pid` from the `menu` reply; refused with `stale_menu` if another app is now in front |
| `library.list` | `{kind: "deck"\|"midi"}` | names saved on this Mac |
| `library.load` | `{kind, name, apply?: "copy"\|"refresh"}` | one layout. `copy` is a private load. `refresh` is for a layout this phone keeps synchronized |
| `library.save` | `{kind, name, previousName?, replace?, screen?, layout}` | writes `library/<kind>/<name>.json`. `replace` confirms a name that is already there. Channel, octave and ids are left out |

### Server → client
| `t` | Payload |
|---|---|
| `welcome` | `{serverVersion, owner: bool, settings}` |
| `pong` | `{}` |
| `session.state` | `{engaged, owner, ownerDevice, reason?, keysReleased?}` | `keysReleased` means this phone's keys were lifted. Deck toggles clear. Modifier chips follow `mods` |
| `session.revoked` | `{byDevice}` |
| `mods` | `{cmd, opt, ctrl, shift, fn}` | shared chips, each `off`/`latched`/`locked`. Sent on connect and whenever they change |
| `holds` | `{keys}` | keys held with `key.down`. Sent on connect and whenever the set changes |
| `midi` | `{notes?, cc?, pitch?, transport?, ccs?, pitches?}` | sounding notes `{ch, note}[]`, one CC or pitch, or the last transport command. A full list arrives on connect |
| `library.list` | `{kind, items: [{name, updatedAt, rev, created}]}` |
| `library.layout` | `{kind, name, previousName?, updatedAt, rev, created, layout, apply?}` | a save reaches every phone. A load reaches the phone that asked |
| `focus.changed` | `{bundleId, name, pid}` |
| `apps` | `{apps: [{bundleId, name, pid, active, hidden, path, iconId}]}`: reply to `apps.list`, and broadcast on launch/quit/hide/activate |
| `dock` | `{apps: [DockItem], others: [DockItem], recents: [DockItem], showRecents}` with `DockItem = {id, kind: "app"\|"folder"\|"file"\|"url", name, bundleId?, path?, url?, iconId?, running?, missing?}` |
| `apps.catalog` | `{apps: [{bundleId, name, path, folder?, iconId, running}]}` (alphabetical) |
| `apps.opened` | `{ok, path, bundleId?}`: app launched/activated |
| `dock.opened` | `{ok, id, kind}`: folder/file/link opened |
| `icon` | `{iconId, png}` (base64 PNG, or `null`) |
| `menu` | `{path, pid, app, bundleId, items: [{id, title, enabled, checked, mark?, shortcut?, hasSubmenu, apple?}]}`: `shortcut` is a display string with modifiers (e.g. `⇧⌘N`); `apple: true` on the first top-level item |
| `menu.invoked` | `{ok, path}`: AXPress succeeded (errors: `not_found`, `disabled`, `invoke_failed`, `no_accessibility`) |
| `decks` | `{system, app}` |
| `error` | `{code, message, ref?}` |
| `permission` | `{accessibility: bool}`: drives the onboarding screen |
| `prefs` | `{pointerSpeed, acceleration, cursorScale, cursorTimeout, autoKeys, invertScroll, cursorScaleEffective, cursorTimeoutMsEffective, textFocusState}`: ack of `prefs.set` as applied |
| `zoom.state` | `{supported, level, pending, factor, mode}`: zoom presets (§6.8), on connect, on helper status changes and whenever it changes; `level` is the last level set (1 at server start), `mode` the method used |
| `textfocus` | `{editable, role?, subrole?, pid?}`: Mac focus entered / left a text input; only to phones with `autoKeys` on (§6.7) |
| `textfocus.status` | `{state: ok\|unsupported\|unavailable\|no_accessibility, helperVersion}`: shown under the auto-Keys toggle |
| `helper.status` | `{available, running, reason}`: helper process state (dry-run, restarting, missing) |

Close codes: `4401` bad token, missing `hello` or `hello` timeout; `1001` server shutdown. A takeover does not close the socket (the previous owner gets `session.revoked`), and helper problems are reported with `helper.status` / `permission`, not by closing.

Server ↔ helper uses the same message shapes over stdin/stdout (newline-delimited JSON), so the later Swift app can absorb both roles without changing the client. Helper-only messages:

| Direction | `t` | Payload |
|---|---|---|
| helper → server | `ready` | `{version, accessibility, cursorScale, midi, midiSourceName, textFocus, zoom, zoomMotion, notify}`: capability flags drive one-time "rebuild the helper" logs. `notify` is helper 0.9.0+ |
| server → helper | `session.engage` / `cursor.scale` | `{cursorScale}` / `{scale}` (§5, §6.7) |
| server → helper | `session.release` / `session.releasePointer` | full release when this phone is alone or last (§4) / pointer only while another phone is connected (mouse buttons, cursor, motion zoom). Both drop a motion zoom back to the button level. Only `session.release` sends all-notes-off |
| server → helper | `textfocus.watch` | `{on}`; helper replies `textfocus.status {ok, watching, reason?}` and emits `textfocus` (§6.7) |
| server → helper | `zoom.set` / `zoom.get` | `{level, from, method, keySteps2x, keySteps4x, keyOut, keyGapMs, units2x, units4x, out, scrollUnits, scrollTap, continuous, suppress, smooth, settleMs, invert, holdModifier}` / `{}` (§6.8) |
| server → helper | `zoom.motion` | `{on: true, level: 1\|1.25\|1.5, smooth}` / `{on: false, smooth}`: Zoom during motion (§6.8); helper replies `zoom.state {mode: motion, active, level, base?, factor, ok, error?}` when it changed the zoom |
| server → helper | `zoom.resetPrivate` | `{}`: on helper (re)start when `zoom.get` reported `privateFactor > 1.001`; helper resets the private zoom to 1x and replies `zoom.state {mode: reset, method: private, level: 1, before, privateFactor?}` (only if it reset something) |
| probe → helper | `zoom.once` | `{method: scroll\|keys\|private, delta \| factor, ...tuning}`: one scroll gesture of `delta`, `delta` shortcut presses, or private zoom `factor` (never sent by the server) |
| helper → server | `zoom.state` | `{level, from?, method, requested?, fallback?, presses?, delta?, privateFactor?, before?, factor?, ok, error?, scrollZoom?, ms?}`; `zoom.get` reply: `mode: read` + `privateAPI, privateSymbols?, privateWorks?, privateFactor?, hotkeys (pref, else the shortcut's enabled flag, else null), hotkeyIn, scrollZoom?` |
| server → helper | `system.sleep` / `system.shutdown` | `{confirmed: true}` required (§6.2) |
| server → helper | `pair.show` | `{code, timeoutMs?}`: 6-digit pairing code (helper 0.9.0). Default wait 120 s, clamped to 1–120 s |
| helper → server | `pair.shown` | `{id: "pair", code}`: the pairing banner is up |
| server → helper | `confirm.show` | `{id, title, body, timeoutMs?}`: Accept / Reject banner (helper 0.9.0). Default wait 30 s, clamped to 1–120 s |
| helper → server | `confirm.shown` | `{id}`: the confirm banner is up |
| helper → server | `confirm.result` | `{id, accepted}`: Accept (`true`) or Reject (`false`). Closing the banner is Reject |

Errors from the helper carry `ref: <request type>`.

### Chrome extension notifications (helper 0.9.0)

The Chrome extension is a separate client in [ground-control-app-hooks](https://github.com/yacfish/ground-control-app-hooks) (`chrome/HELPER.md`, protocol `ext/1`, final for v1 on 2026-10-13). It connects to the Node server on `/ws` with `device: "chrome"`. The Swift helper does not speak to the extension. It posts the two macOS notifications below and already reports the frontmost app. The server sends `pair.show` and `confirm.show`, keeps the latest focus, rules, and mappings, and forwards a preset version change so the phone can refresh the deck. It does not fetch or check presets. A chrome client never takes the pointer, and its pointer, key, and other phone commands are dropped. `ready.notify` is true from helper 0.9.0. An older helper answers `confirm_unavailable`, and the action does not run.

**Pairing code.** `npm run ext:pair`, or a phone message `ext.pair.request`, asks the running server. The server makes a 6-digit code, valid for 2 minutes, single use, and five wrong tries require a new code. It prints the code and sends `pair.show`. The phone's reply does not include the code. The extension submits it with `ext.pair`. On success the server stores only a SHA-256 hash in `ext-clients.json` (mode 0600) and closes that socket. `npm run ext:revoke -- <name>` removes the token. The phone token is never an extension token. One live socket per extension id; a new connection closes the old one. The origin must be `chrome-extension://<id>`. With an empty `extension.allowedIds` list, only a connection from this Mac is accepted. One pairing banner is up at a time. A new code replaces it. The helper removes it when the wait elapses and sends no further message.

**Confirm.** Adding or editing a mapping, and running an action the extension has marked sensitive, asks on the Mac. The helper posts one notification through `UNUserNotificationCenter`, in a category with Accept and Reject. The title is the control name. The body is the control type and the site, for example `button on youtube.com`. The phone shows a status line. It does not show a confirm sheet.

The server sends `confirm.show`. A mapping confirm waits 30 s. A sensitive action waits 15 s. The helper answers `confirm.shown` once the banner is up, then `confirm.result` with `accepted: true` for Accept and `accepted: false` for Reject. Closing the notification is Reject. When the wait elapses, the helper removes the notification and sends nothing, and the server reports `timeout`. When notifications are disabled, or the system refuses the banner, the helper answers `error` `confirm_unavailable` and the action does not run. A sensitive action is sent to the extension only after Accept, with `confirmed: true`.

`ext.state` is the live playback position (`currentTime`, `duration`, `paused`, `volume`, `muted`, `playbackRate`). The server forwards it to phones and does not store or log it. A slider whose chain contains `{kind: "media", op: "position"}` seeks in seconds. `ext.focus` `state.media` no longer carries `currentTime`. An optional `icon` on a mapping or a rule is a `data:image/png;base64,` URL of at most 4096 characters, stored and shown, never used for matching.

`UNUserNotificationCenter` is what the extension note names. macOS 12 refuses that call for this ad-hoc helper (`Notifications are not allowed for this application`), so the helper posts the same banner with `NSUserNotification`, including Accept and Reject. The binary is installed as `helper/bin/gc-helper.app` (bundle id `com.yacfish.ground-control.helper`) because a bare executable cannot show a banner. `bin/gc-helper` execs that app. A banner the system does not deliver is answered with `confirm_unavailable`.

### App hooks (hook/1)

Blender, OBS, VLC, GIMP, Inkscape, Godot, and Spotify connect as `device: "hook"` with an `app` id. The note is `HOOKS.md` in the same repo, protocol `hook/1`. Blender's add-on still speaks `blender/1`: the same messages with no `app` field, and the server answers in that shape. A hook is not a phone. It never takes the pointer, and its pointer, key, and other phone commands are dropped.

`npm run hook:pair` and `npm run hook:revoke -- <name>` use the phone token. The 6-digit code is printed by the server and sent with `pair.show`. The phone's reply does not include it. Tokens are SHA-256 hashes in `hook-clients.json` (mode 0600). A token works for one app only. Connections are from this Mac unless `hooks.allowRemote` is set. An http or extension Origin is refused.

The Deck tab shows that app's deck while it is frontmost and connected, or Chrome's deck while Chrome is front and a page deck is available. Pin keeps an app deck up. App deck returns to the shortcut deck until the focused app changes. `hook.state` (playback and levels) is forwarded to phones and not stored or logged. A position slider seeks in seconds. Panels (`hook.panel`) open from the phone and stay in memory only while a phone is showing them. A sensitive control uses the same Accept / Reject banner as Chrome. The body is the control type and the deck title, for example `button in Scene`.

A deck text button may set `secret: true`. The deck file stores `secretRef` and, when the user recorded them, `expectBundle` and `expectName`. It does not store the text. The phone types it with `key.text` and `secret: true`. The server forwards that once and does not log it. `deck.run` does not expand a secret. A hook text control or panel text field with `secret: true` is the same idea: the value is left out of the cached panel, and the phone sends it on `hook.action`. This page keeps the value in memory for the visit. The later Capacitor wrap stores it in the Keychain or Keystore and asks for a biometric, or the device passcode, before reveal or send. Copy clears the clipboard after 20 seconds. iOS may ignore that clear. The LAN WebSocket is the existing connection and is not additionally encrypted. Customer-facing terms should say that a secret is typed into the focused field and is not written into the deck. Counsel should review that wording.

---

## 8. macOS permissions and setup

- **Accessibility** (System Settings → Privacy & Security → Accessibility) is required for posting `CGEvent`s and for reading/pressing menus. macOS attributes the helper to the app that launched the server: **Cursor** when `npm start` runs from Cursor, and the terminal app (Terminal, iTerm, …) when you launch it yourself. That app's grant covers the helper. Adding `gc-helper` to the list does not satisfy the prompt. The helper checks `AXIsProcessTrusted()` on start; if false it prompts once and the server reports `permission {accessibility:false}` so the phone shows clear instructions instead of silently doing nothing. (Later the menu-bar app holds this grant itself.)
- **Automation** (Apple Events → System Events) is **not required** for v1 menus; invocation is Accessibility-only.
- **Apps surface needs no extra permission**: listing running apps, reading the Dock's preferences (`com.apple.dock`, readable by the logged-in user's non-sandboxed processes), scanning the Applications folders and launching apps are all unprivileged. No Full Disk Access, no Automation.
- No Screen Recording permission: there is no screen capture. Zoom presets need no extra permission either (they read `com.apple.universalaccess` / `com.apple.symbolichotkeys`, post events under the Accessibility grant, or call the private WindowServer zoom).
- **Notifications** (System Settings → Notifications → Ground Control) are for Chrome and app-hook pairing codes and for their Accept / Reject confirms (§7, helper 0.9.0). The first pairing or confirm asks. When they are turned off, the helper answers `confirm_unavailable` and the action does not run.
- Must run inside the logged-in graphical session (not over SSH), like ClipBridge.
- Known SSH-only limitation: outside a GUI login, creating events from a `CGEventSource(stateID: .hidSystemState)` can block for more than 3 s, while nil-source events are created in ~12 ms. The zoom exit fallback and the release modifier sweep use nil-source events; the zoom key / scroll events and the rest of `InputController` (pointer, keys, text) use the HID-state source, which works in the GUI session (the zoom creates it lazily on its own queue, never on the main thread). CoreMIDI client creation also fails without a GUI session (e.g. `268435459`).

---

## 9. Configuration

`~/.config/ground-control/`
- `token`: auth token
- `config.json`: port, sensitivity, acceleration, cursor mode (`system`/`off`) and scale, cursor timeout, idle timeout, MIDI source name, `zoom` tuning (§6.8)
- `decks/*.json`: user decks (override / extend built-ins by bundle ID)
- `cursor-restore.json`: pre-session cursor scale (crash recovery)
- `ext-clients.json`, `hook-clients.json`: SHA-256 token hashes for the Chrome extension and for app hooks (mode 0600). `extension.allowRemote` and `hooks.allowRemote` default to off.
- `library/decks/<name>.json`, `library/midi/<name>.json`: saved layouts, one JSON file per deck or MIDI layout. The filename is the unique layout name. The phone saves and loads them from the share sheet. See [docs/multi-client.md](docs/multi-client.md). Distinct from `decks/`.

---

## 10. Roadmap

**v0: docs.** SPEC + README. ✅

**v1: Node + Swift helper** (current build, helper 0.9.0)
1. Node server: static client, token, WebSocket, heartbeat, ownership, release-everything path. ✅
2. Swift helper: pointer / click / scroll / text / keys with CGEvent; Accessibility check. ✅
3. Client: trackpad + keyboard with sticky modifiers, echo, auto-repeat. ✅
4. Cursor grow/restore (✅, with per-phone size and timeout).
5. Apps tab: Dock, running apps (switcher), Applications folder; focus tracking. ✅
6. Shortcut decks: system popup plus the focused app’s editable deck, including window controls and key toggle mode. ✅
7. Menu tree via Accessibility with AXPress invocation, Apple menu always present. ✅
8. Virtual MIDI. ✅
9. Top menu, theme, Preferences, auto Keys on Mac text focus. ✅
10. Trackpad zoom: 1X / 2X / 4X buttons, pinch or a slider from 1X to 6X, Zoom during motion, and zoom timeout (Off / 300s / 60s / 15s, default 60). ✅ private method and zoom timeout confirmed on this Mac (helper 0.8.2; 0.8.0 builds on Xcode 14.2 and the private symbols resolve). `keys` / `scroll` remain the fallbacks; Accessibility Zoom ignored synthesized Control + scroll on 12.7.6.

**Final product** (§2.1). Not built yet.
- Menu-bar Mac app, notarized, off the store: server, input, Accessibility, QR code, login item, command gate.
- iPhone app on the App Store: free trackpad and keys, Pro in-app purchase for the other surfaces. The Mac accepts Pro commands only with Apple's signed transaction.

**v1 remains** the Node server, the helper and the browser page until that app exists. The one-pointer rule (§4) stays.

**Later / maybe**
- Apps tab polish: A–Z index, favourites row, Dock live refresh (watch `com.apple.dock`), Trash tile, Dock folder contents (stacks) on the phone.
- MIDI feedback (virtual destination → phone LEDs).
- Gesture shortcuts (three-finger swipe ⇒ Mission Control / switch Spaces).
- Bonjour discovery for the QR/bookmark flow.
- Chrome extension (`ext/1`) and app hooks (`hook/1`), specified in ground-control-app-hooks. The server accepts both on `/ws`, pairs them with a hashed token, and sends `pair.show` and `confirm.show`. The Deck tab shows the live deck. The helper does not fetch presets.

---

## 11. Decisions log

| Decision | Status |
|---|---|
| Name: **Ground Control** (ex-PointerBridge) | Locked |
| Current build: Node on Mac + React/Vite phone page, token-in-bookmark, LAN/Tailscale, no account, no cloud | Locked (v1) |
| Port ≠ 8787 (default 8788) | Locked |
| WebSocket stream for input, not HTTP | Locked |
| Phone is a controller only: **no screen stream** | Locked |
| Swift helper posts CGEvent; Accessibility permission | Locked |
| Cursor grows on engage, restores on release / socket drop. Idle restores it only when the Trackpad tab is not holding the pointer | Locked |
| Menus invoked via **Accessibility (AXPress) only**, never AppleScript and never by clicking menu-bar coordinates | **Locked** |
| Apple menu always present in the menu tree | Locked |
| Apps tab = **Dock · Running · Apps** sections (no new bottom tab); running-apps switcher kept | Locked |
| Dock read from its preferences (`com.apple.dock` via CFPreferences), read-only; **no AppleScript / System Events / Dock UI scripting**; AX on the Dock process only as a future fallback | Locked |
| Launching via `NSWorkspace.openApplication` / `NSWorkspace.open`; app actions require pointer ownership | Locked |
| Icons: helper-rendered 96 px PNG by versioned `iconId`, cached client-side in IndexedDB | Locked |
| Zoom presets: one action per press via `zoom.method` (auto: private WindowServer zoom if available and verified, else Zoom keyboard shortcuts, else one modifier + scroll event); relative methods track the last level and 1X overshoots as a reset; private zoom is placed around the pointer when set and then pans only at the magnified edge; a 1X / 2X / 4X button level stays across release and returns to 1X on the zoom timeout (default 60 s) or when the helper exits; Zoom during motion and the zoom timeout are server-timed; never AppleScript | Locked |
| No hosted CI (GitHub Actions removed): `npm run check` and, on a Mac, `npm run check:helper` before pushing | Locked |
| Final product (§2.1): free notarized Mac menu-bar app off the store + App Store iPhone app (free trackpad and keys, Pro in-app purchase). The Mac gates Pro commands after a signed StoreKit transaction. The browser page stays the free remote | Locked |
| Editable grids: MIDI chooses its column and row counts. The Deck uses fixed 50×50 cells, separate width and height, and pages. Both are in this build (§6.2, §6.3). Deck buttons still cannot send every helper command the plan lists | Locked |
| Chrome extension confirms and the pairing code are macOS notifications from the helper (`pair.show`, `confirm.show` / `confirm.result`, Accept and Reject). Protocol `ext/1` in ground-control-app-hooks. The server sends those messages, drops chrome input, forwards preset versions, and forwards `ext.state` without storing it. The helper does not talk to the extension and does not fetch presets | Server wired |
| App hooks (`hook/1`, Blender still `blender/1`) pair the same way. The server forwards decks, panels, and `hook.state` without storing playback state. The Deck tab shows the hook while that app is in front | Server wired |

---

## 12. Development and verification

- `npm test`: end-to-end protocol smoke test (`scripts/smoke-test.js`) against `scripts/mock-helper.js`; runs on Linux, no Mac needed. Also unit-checks `src/keysModel.js` (echo, diff, repeat timing), `src/menuModel.js`, the Chrome extension client (`scripts/extension-test.js`, `scripts/mock-extension.js`), and app hooks (`scripts/hook-test.js`, `scripts/mock-hook.js`).
- `npm run check`: `npm test` then `npm run build`. Run before every push; there is no hosted CI.
- `npm run check:helper` (`scripts/check-helper.sh`, macOS only, skips elsewhere): builds the helper with `scripts/build-helper.sh` and checks `ready`, refusal of unconfirmed sleep / shut down, `textfocus.watch` / `cursor.scale`, the zoom capability with a read-only `zoom.get`, helper 0.9.0 with `notify`, a bad `confirm.show` / `pair.show` and an answer to a real one, prompt exit on SIGTERM, the Apps probe, MIDI with all-notes-off, and `menu.get` / `menu.invoke bad_path`. GUI-session-only steps print `WARN` and are skipped instead of failing when there is no GUI session (e.g. over SSH): MIDI when CoreMIDI client creation fails, and the Apps probe when `launchctl managername` is not `Aqua` (or MIDI already showed there is no GUI session).
- `node scripts/helper-probe.js --verbose`: real helper lists Dock / Applications / running apps and one icon (read-only).
- `node scripts/zoom-probe.js`: prints the `closeView*` keys and which zoom methods the helper sees (read-only). `method <auto|private|keys|scroll> <1|2|4> [--from N]` does what a button does with that method; `private <factor>` sets the private zoom (kept 5 s, `--hold N`, then the helper exits and resets it); `keys <n>` posts n shortcut presses (negative = out); `once <delta>` posts one modifier + scroll event. Tuning flags match config.json `zoom`. Run it in the GUI session.
- The helper must stay buildable with Swift 5.7 / Xcode 14.2 on macOS 12.
