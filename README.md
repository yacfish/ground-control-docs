> **Note:** This is a read-only documentation mirror for **Ground Control**.
> The full source code and active development happen in a private repository.

---

# Ground Control

Turn your phone into a trackpad, keyboard, shortcut deck, MIDI controller, app launcher (Dock · running apps · Applications) and menu bar for your Mac: from the couch, across the room, or over Tailscale. No account, no cloud.

The current build is a phone browser and a Node server. The plan for what ships is in **[ground-control-research](https://github.com/yacfish/ground-control-research)** (`analysis/mobile-plan.md`, `analysis/monetization.md`, `analysis/porting-windows-linux.md`). In short:

- The host stays a Mac. Menus and the focused app are why: Windows has no menu bar a remote can read, and the focused app cannot be followed there. A Windows or Linux port of the full product was considered and dropped.
- The phone apps start as a Capacitor wrap of this page, on the App Store and on Google Play. Small native plugins, Swift on iOS and Kotlin on Android, cover purchases, haptics, and Bluetooth or USB MIDI. A full native rewrite of a platform waits until that platform has 1,000 paying users.
- Pro is an in-app purchase on each store, so an iPhone and an Android phone are two purchases. Free is the trackpad, the keyboard, and text. Pro, about $15–20, is decks, menu buttons, the app switcher, and a small set of general decks. One Creative Suite package, about $29–49 once, covers the big creative apps together. Decks people build and share stay free. Those prices are placeholders.
- A MIDI spin-off can be its own controller over Bluetooth or USB, with nothing installed on the computer. The helper here stays optional for layouts, Tailscale, and several devices at once.
- The Mac app is still a free menu-bar app, notarized, downloaded from the developer, outside the Mac App Store. It posts the clicks and keystrokes and checks each store purchase on the Mac. The browser page stays the free remote.
- Near term, refine the Keys tab layout first. It is the smaller piece of work, and the tab is due for it. After that, the next big priority for the first release is a strong set of decks.

**[SPEC.md](SPEC.md)** §2.1 is the earlier shape: an iPhone app only, and one Pro unlock for Text, Deck, MIDI, Apps, Menus, and zoom. The research notes replace that.

> Status: **v1 prototype.** Trackpad (pinch, slider, or 1X / 2X / 4X, plus a zoom timeout), keyboard (live echo, auto-repeat), a Text composer, an editable Deck for the focused app, a system popup, **Virtual MIDI** (CoreMIDI source “Ground Control”: pads / CC / pitch / transport), the **Apps** tab (**Dock · Running · Apps**), the **Menus** tab and per-phone **Preferences** work on a Mac. Helper **0.9.0**. What is still open is in **DECK-PLAN.md (only available in the main repo)** and **[SPEC.md](SPEC.md)** §10.

Ground Control is a sibling of [ClipBridge](https://github.com/yacfish/clipbridge) and shares its shape:

- a small **Node** server on the Mac (plus a tiny **Swift helper** that posts input events),
- a **React (Vite)** page on the phone, opened from a bookmark that carries a token,
- **LAN or Tailscale** only, default port **8788** (ClipBridge keeps 8787),
- a single **WebSocket** stream for input: the phone is a controller, it never shows the Mac's screen.

Surfaces on the phone (bottom tabs **Trackpad · Keys · Text · Deck · MIDI · Apps · Menus**), all over the same socket:

1. **Trackpad** with Left / Drag lock / Right buttons and zoom (pinch or a slider from 1X to 6X, or **1X · 2X · 4X** buttons)
2. **Keys**: phone keyboard to Mac, special keys, sticky modifiers
3. **Text**: compose a whole entry, then send it in one go, with a history you can resend, edit, or delete. An unsent draft stays when you leave the tab
4. **Shortcut decks**: the Deck tab is an editable grid for the focused app. **Ground Control** in the status bar opens the system grid on every tab
5. **Virtual MIDI** controller: layouts you create and select. Each one is a grid of notes, continuous controllers and system widgets, with one channel for the grid or a channel on each control
6. **Apps**: **Dock** (what's in your Mac's Dock), **Running** (app switcher: activate / hide / quit), **Apps** (Applications folder with search)
7. **Menus**: the frontmost app's menu tree, read and invoked through macOS Accessibility (never by clicking menu-bar coordinates)

While a session is engaged the Mac cursor grows so it can be seen from the couch, and shrinks back when you let go (or after a timeout you pick in Preferences).

This repository is the current build. The research notes above are how it ships.

*Ground Control was briefly called PointerBridge during design; that name is historical only.*

See **[SPEC.md](SPEC.md)** for architecture, protocol, surfaces, and permissions, and **[ground-control-research](https://github.com/yacfish/ground-control-research)** for the product plan.

---

## Run it on a Mac

Requirements: macOS 12+, Node 18+, Xcode Command Line Tools (`xcode-select --install`) for `swift` (Swift 5.7 / Xcode 14.2 is enough).

```bash
git clone https://github.com/yacfish/ground-control.git
cd ground-control
npm install
npm run build:helper   # swift build -c release → helper/bin/gc-helper
npm run build          # React client → public/
npm start              # server on :8788
```

`npm start` prints a clickable link for this Mac (`http://127.0.0.1:8788/?k=<token>`), then a QR code for each LAN address and another for the Tailscale address when the Mac is on a tailnet. The line above each code names the interface and the address (`192.168.1.20` and so on). Click the link to open the page in the browser on this Mac. Scan a code with the phone camera; the code opens `http://<address>:8788/?k=<token>`. The MagicDNS address stays as text: `http://<mac>.<tailnet>.ts.net:8788/?k=<token>`. Open one on your phone. In the browser the page leaves `?k=` in the address and shows a banner: add it to the Home Screen before dismissing, so the shortcut keeps the pairing link. The Home Screen web app hides that banner and removes `?k=` from the visit; the icon's saved address stays as it was. The token is stored in `~/.config/ground-control/token` (mode 0600); delete that file to rotate it.

After `git pull`: run `npm run build` (client) and, when helper sources changed, `npm run build:helper`, then restart `npm start`. The server logs the helper version on start and logs once when a feature needs a newer helper.

### Accessibility permission (required)
macOS only lets processes with **Accessibility** access post mouse/keyboard events. The prompt is for the app that launched the server, and that grant covers `gc-helper`. Adding `gc-helper` itself does not satisfy the prompt.

- Started from Cursor: authorize **Cursor**.
- Started from Terminal, iTerm, Ghostty, …: authorize that terminal app.

System Settings → Privacy & Security → Accessibility, then restart `npm start`. Until it's granted the phone shows **no AX** and a hint, and the server logs *helper reports Accessibility NOT granted* once.

**Auto Keys not switching?** Enable it in Preferences (Keys tab visible), then watch the `npm start` output: you should see `[gc] auto Keys: text focus polling on (helper 0.8.2, ...)`, `[gc] auto Keys: helper confirmed text focus polling on`, and `[helper] gc-helper: textfocus editable=true role=AXTextField ...` when you click into a text field. A line saying the helper *cannot watch text focus* means it is older than 0.6.0 (`npm run build:helper`, then restart); *Accessibility NOT granted* means Cursor or the terminal app above is not enabled. Chrome, Brave, Edge, Arc and Electron apps are asked to expose their web content (`AXManualAccessibility`), which can take a poll or two after the app is first focused.

### Development
```bash
npm run dev     # node --watch server + Vite on :5173 (proxies /api and /ws to :8788)
npm test        # end-to-end protocol test with a mock helper (works on Linux, no Mac needed)
npm run check   # local pre-push check: npm test && npm run build
npm run check:helper   # macOS only: builds the helper and runs its protocol checks (sleep refusal, text focus, zoom.get, SIGTERM exit, Apps, MIDI, menus; GUI-session-only steps such as CoreMIDI print WARN and are skipped over SSH)
node scripts/helper-probe.js --verbose   # on a Mac: real helper lists Dock / Applications / running apps + one icon (read-only)
node scripts/zoom-probe.js                     # on a Mac: zoom prefs + which zoom methods the helper sees (read-only)
node scripts/zoom-probe.js method <m> <1|2|4>  # on a Mac: what a zoom button does with method auto | private | keys | scroll (--from N)
node scripts/zoom-probe.js private <factor>    # on a Mac: private zoom straight to <factor>; keys <n>: n shortcut presses; once <delta>: one Control + scroll
```
There is no hosted CI (GitHub Actions were removed): run `npm run check` before pushing, and `npm run check:helper` on a Mac when helper sources change.

Open `http://<mac-ip>:5173/?k=<token>` on the phone during dev. Without a helper binary (or on Linux) the server runs in **dry-run** mode: everything works except events aren't posted; `GC_DEBUG=1` logs what would be sent. `GC_HELPER=/path/to/binary` overrides the helper location (e.g. `GC_HELPER=scripts/mock-helper.js`).

### Configuration
Optional `~/.config/ground-control/config.json`:
```json
{ "port": 8788, "sensitivity": 1.0, "acceleration": 0.6, "cursorMode": "system", "cursorScale": 3, "cursorTimeoutMs": 0, "idleMs": 0,
  "midiSourceName": "Ground Control",
  "zoom": { "method": "auto", "keySteps2x": 4, "keySteps4x": 8, "keyOut": 20, "units2x": 1000, "units4x": 2000, "out": 4000, "motionIdleMs": 500 } }
```
- `PORT` env overrides the port.
- `sensitivity` is a relative multiplier on top of a built-in 2.5x base pointer gain (`POINTER_BASE_GAIN` in `server/session.js`), so `1.0` is the default speed. `acceleration` is the default curve (0 = linear); each phone can override both in Preferences.
- `cursorMode`: `system` (grow the real cursor while engaged) or `off`. `cursorScale` is the default engage size (1 to 4). `cursorTimeoutMs` is the default cursor timeout (0 = Until I stop driving).
- `idleMs`: 0. A one-phone session started from keys, text, a deck, or MIDI releases as soon as that action is sent. It does not take the trackpad away while that tab is open.
- `midiSourceName`: name of the CoreMIDI virtual source.
- `zoom` picks and tunes the Trackpad 1X / 2X / 4X method (see below): `method` (`auto` / `private` / `keys` / `scroll`), `keySteps2x` / `keySteps4x` / `keyOut` / `keyGapMs` for `keys`, `units2x` / `units4x` / `out` / `scrollUnits` / `scrollTap` / `continuous` / `suppress` for `scroll`, `smooth` for `private`, `motionIdleMs` (default 500) for Zoom during motion. Step counts are guesses until tuned on a Mac.

### Trackpad tab
- One-finger drag moves the pointer (host-side acceleration), tap = left click, double-tap = double click, two-finger tap = right click, two-finger drag = scroll with momentum, hold then drag = select / drag.
- **Left** / **Drag lock** / **Right** buttons under the pad.
- Zoom under the pad. Preferences → **Zoom entry method** picks **Pinch** (default), **Slider**, or **Buttons**. Pinch and the slider set anywhere from 1X to 6X. Buttons are the discrete **1X · 2X · 4X** presses. Switching method returns the zoom to 1X. Helper 0.8.2, `zoom.method` in config.json, default `auto`. macOS has no public API to set the Zoom level, and on macOS 12.7.6 Accessibility Zoom ignored synthesized Control + scroll (helpers 0.7.0 to 0.7.2), so there are three methods:
  - **`private`**: the WindowServer's own zoom (`CGSSetZoomParameters`, the private call behind screen zoom), looked up at runtime so a missing symbol just means "not available". **Absolute and exact**: 1X = 1.0, 2X = 2.0, 4X = 4.0, checked by reading it back. It is separate from Accessibility Zoom. The zoom is placed around the pointer when it is set. The view stays there until the pointer reaches an edge of the magnified area, then pans only enough to keep the pointer on that edge (the factor stays 2 or 4). Helper 0.8.2 removed the 0.8.0 / 0.8.1 pointer-follow, which re-centered on every move. A 1X / 2X / 4X choice stays until you press another zoom button, or until the zoom timeout below returns it to 1X. Release, the phone locking, disconnect, the idle timeout, and sending text leave that button level in place. The private zoom goes back to 1x **when the helper exits** (stopping `npm start` unzooms). If the helper was killed or crashed while zoomed, the server resets the private zoom to 1x the next time it starts the helper (and the buttons go back to 1X); Accessibility Zoom is never touched by that reset.
  - **`keys`**: the Zoom keyboard shortcuts (needs **System Preferences > Accessibility > Zoom > Use keyboard shortcuts to zoom**). The helper reads the actual shortcuts from your keyboard settings (default ⌥⌘= / ⌥⌘-) and posts N presses back to back in one action: N = position(target) - position(last level), with 1X = 0, 2X = `keySteps2x` (4), 4X = `keySteps4x` (8); 1X zooms out at least `keyOut` (20) presses, which just stops at 1x.
  - **`scroll`**: the 0.7.2 gesture (Control down, ONE scroll event of position(target) - position(last), Control up; 2X = `units2x`, 4X = `units4x`, 1X at least `-out`), with experiment switches `scrollTap` (`hid` / `session`), `continuous` and `suppress`. Needs "Use scroll gesture with modifier keys to zoom".
  - **`auto`** (default): `private` if the call exists and its last use read back correctly, else `keys` unless the keyboard shortcuts are known to be off, else `scroll`.

  The server remembers the last level (1X when `npm start` starts) for the relative methods; press **1X** to re-sync if the screen was zoomed by other means. Older helpers: buttons disabled, one server log line. The `npm start` log shows which methods the helper sees at start (`[gc] zoom buttons: method auto; private zoom API found ...`) and every press (`[gc] zoom: 1x -> 4x, private: ...`).

**Trying the zoom methods** (on the Mac, in Terminal in the GUI session, after `npm run build:helper`; start unzoomed):
1. `node scripts/zoom-probe.js`: shows which methods the helper sees (private API found or not, keyboard shortcuts on / off, scroll gesture on / off).
2. `node scripts/zoom-probe.js method private 2`, then `method private 4`, then `method private 1`: exact zoom through the private call. The probe keeps the helper alive 5 s (`--hold N`) because the helper resets private zoom when it exits.
3. `node scripts/zoom-probe.js method keys 2`, then `method keys 4 --from 2`, then `method keys 1 --from 4`: the keyboard shortcuts. Find how many presses reach 2x and 4x with `node scripts/zoom-probe.js keys 1`, `keys 4`, `keys -20`, then set `keySteps2x` / `keySteps4x`. If nothing zooms, try `--keyGapMs 30`.
4. `node scripts/zoom-probe.js method scroll 2` (then `--scrollTap session`, `--continuous`, `--suppress`): the scroll gesture experiments.
5. Put the method that works in `~/.config/ground-control/config.json` (`"zoom": { "method": "private" }`, or `"keys"` with `keySteps2x` / `keySteps4x`), restart `npm start`, press 1X once, then try 2X / 4X.

**Zoom during motion** (Preferences, under Cursor timeout: **1X** / **1.25X** / **1.5X**, 1X by default, saved on the phone and sent with the other preferences): while the phone moves the pointer, the Mac screen zooms to that level with the private zoom, placed around the pointer, and goes back **500 ms after the pointer stops** (`zoom.motionIdleMs` in config.json). The server times it: the first move sends `zoom.motion {on}`, the idle timer sends `zoom.motion {off}`, and the helper sets the private zoom back (factor 1.0, or the button level) and checks it with `CGSGetZoomParameters`. 1X means moving the pointer never zooms in further. With the 1X / 2X / 4X buttons:
  - a button level is kept when it is already higher: motion only zooms in **beyond** it (1.5X during motion with the buttons at 1X) and returns to it; with 1.25X or 1.5X during motion and a 2X or 4X button, moving does nothing;
  - pressing a button while motion-zoomed: a level at or above the motion level replaces the motion zoom; a lower one is remembered and applied when motion ends (no zoom out while you move);
  - Release, disconnect, the idle timeout and sending text drop only this extra zoom, back to the 1X / 2X / 4X button level. Stopping `npm start` puts the private zoom back to 1x. Needs helper 0.8.2 and the private zoom call (with an older helper the setting does nothing and the server logs a rebuild hint once).

**Zoom timeout** (Preferences, under Zoom during motion: **Off** / **300s** / **60s** / **15s**, 60s by default). Confirmed on this Mac: a 2X or 4X button level returns to 1X that long after the phone sits still. Moving the pointer, keys, text, decks, and the other actions that keep the session alive start the wait again from the beginning. Changing the duration applies to a wait that is already running, so picking 15s is not stuck behind a 300s wait. Leaving the Trackpad tab, the phone locking, and disconnect leave both the zoom and the wait in place. Off keeps the zoom until you press 1X. The server sends that return to 1X on its own, without taking the pointer. If a zoom-during-motion level is on screen when the wait ends, that extra zoom ends first and the screen then goes to 1X. A heartbeat ping does not restart the wait.

**Stuck zoom?** The private zoom is not Accessibility Zoom, so ⌥⌘8 / Control-scroll will not undo it. Run `node scripts/zoom-probe.js private 1` on the Mac (sets it back to 1x right away), or restart `npm start` (the server resets a leftover private zoom when it starts the helper).

### Keys tab
- A hidden text field takes the phone keyboard; what you type is sent to the Mac as Unicode text, so any layout, emoji, dictation and IME work. Autocorrect / IME replacements are sent as ⌫ + the new text.
- **Return** on the phone keyboard sends a real **Return** keypress to the Mac (never a newline in the field), including soft keyboards that only report a line break.
- **Echo**: the strip shows what you just typed as spaced tokens, with named keys as pills: **ENTER**, **SPACE**, **TAB**, **BACKSPACE** (Backspace adds a BACKSPACE token instead of erasing the echo). The echo and the field clear **1 s after the last keystroke** (not during an IME composition); nothing is sent to the Mac when it clears. **clear** empties it at once.
- Special keys, in this order: **esc ↑ ⏎ ⌫ ⌦ ← ↓ → ⇥ home end pg↑ pg↓ space**, then media keys and an **F1–12** toggle. **⌫ and the four arrows auto-repeat while held**: one press on touch-down, repeating from 500 ms at 300 ms intervals that speed up to 100 ms over 1.5 s. The phone keyboard's own Backspace keeps its native repeat.
- Sticky modifiers **⌘ ⌥ ⌃ ⇧ fn** are shared by every connected phone. A tap latches the chip for the next key from any phone, or for the next mouse-up on the trackpad phone. Holding a chip for 0.8 s locks it. The chip fills blue and grows 3px on every side while the finger is still down, then returns to its normal size when the finger lifts. The chips around it stay put. A chip that is already locked turns off when the finger lifts. The same chips on the other phones show latched or locked. A deck toggle shows held on the other phones, and any phone can release it. A held arrow on Keys keeps the modifiers it started with (for example ⇧ to extend a selection) and auto-repeats as key presses.
- Preferences can switch to Keys automatically when a Mac text field is focused, and raise the phone keyboard when you tap the Keys tab (below). With the phone keyboard down, the round button at the bottom right of Keys brings it back.

### Top menu and Preferences
Tap **⋮** in the top bar: **Light mode / Dark mode**, **Preferences**, **Disconnect** (then **Connect** to come back, no reload). Preferences live on the phone (`localStorage`) and the per-session ones are sent to the server on every connect:

- **Tabs**: hide tabs you don't use (at least one stays).
- **Switch to Keys automatically when typing is needed** (off by default): when a text field is focused on the Mac, the phone jumps to Keys and returns to the previous tab when focus leaves. Moving from one text field to another triggers it again. The helper only polls the focused element (every ~300 ms) while a phone has this on. iOS won't raise the phone keyboard by itself; tap the Keys field (or the Keys tab, with the next option on) to type. Problems (old helper, no Accessibility) show under the toggle.
- **Open keyboard when Keys tab is selected** (off by default, stays on the phone): tapping the Keys tab, even when Keys is already showing, focuses the type field so the phone keyboard comes up ready to type. Automatic switches to Keys also try, which works on Android but not on iOS (Safari only raises the keyboard for a focus made inside a tap).
- **Pointer and scroll speed** (0.25x to 3x on top of the built-in pointer speed; the same multiplier scales two-finger scrolling and its momentum, 1.00x scrolls exactly as before), **Acceleration** (pointer only), **Invert scroll direction**.
- **Cursor size** while engaged: Off / Small / Medium / Large (default = `cursorScale` from config, normally Medium).
- **Cursor timeout**: how long the enlarged cursor stays big after the last pointer activity, 0.5 s to 10 s, or *Until I stop driving* (big until you leave the Trackpad tab, lock the phone, or disconnect). The server shrinks it with `cursor.scale 1` and re-grows it on the next move or touch (helper 0.6.0+). Default = `cursorTimeoutMs` from config (0 = Until I stop driving).

### Shortcut decks
The **Deck** tab shows the deck for the app in front on a fixed 50×50 grid. **Edit** moves, adds, and removes buttons; drag one off the right edge to open the next page, and swipe between pages when there is more than one. Until that deck is edited on the phone, the built-in buttons are laid out two across. An app with no deck shows an empty state. **Ground Control** in the top-left of the status bar opens the system grid on every tab; tap the title again, or tap outside the grid, to close it. Built-in decks in [`decks/`](decks/):

| Deck | `bundleId` | Buttons |
|---|---|---|
| System | (always, title popup) | Desktop (Mission Control, App Exposé, Show Desktop, Spotlight, Screenshot, Lock Screen), Clipboard (Cut, Copy, Paste), Windows (Previous, Next, Close), Power (Sleep, Shut Down) |
| Safari | `com.apple.Safari` | tabs, reload, back / forward, full screen, Reader, bookmarks, find, private window |
| Chrome | `com.google.Chrome` | tabs (new, close, reopen, previous / next), windows (previous / next), reload, back / forward, Incognito, full screen, find, downloads, bookmarks, home |
| Finder | `com.apple.finder` | File (New Window, New Folder, Get Info, Duplicate, Quick Look, Move to Trash); Go (Home, Downloads, Desktop, Applications, Enclosing Folder); View (as Icons, as List, as Columns, as Gallery, Path Bar, Preview Pane) |
| Max | `com.cycling74.Max` | File (New Patcher, Open, Save); View (Edit / Lock, Presentation, Sidebar, Zoom In, Zoom Out); Objects (New Object, Message, Comment, Toggle, Button); Audio (Overdrive, Audio Status, MIDI Setup, Max Console) |
| Spotify | `com.spotify.client` | play / pause, next / previous, shuffle, repeat, volume, full screen, like, search |
| VLC | `org.videolan.vlc` | play, pause, stop, next / previous, full screen, volume, mute, subtitles |
| TV | `com.apple.TV` | play / pause, full screen, skip back / forward, volume, mute, home |
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

Drop overrides in `~/.config/ground-control/decks/*.json`: a user file replaces a built-in with the same `id` or `bundleId`. See [`decks/README.md`](decks/README.md) for the JSON shape (`key`, `text`, `activate`, `menu`, `sequence`). Menu buttons use helper `menu.invoke` via **AXPress only** (SPEC §6.5); no AppleScript.

The **System** popup's Mission Control / App Exposé / Show Desktop / Screenshot (⌘⇧3, instant full screen) buttons send the standard macOS shortcuts, so they need those shortcuts **enabled in System Settings → Keyboard → Keyboard Shortcuts → Mission Control / Screenshots**. Cut, Copy, Paste, Previous (⌘⇧`), Next (⌘`), and Close are the same keystrokes in every app and apply to whatever is in front. **Sleep** and **Shut Down** always ask for confirmation on the phone first (OK → runs immediately); see [`decks/README.md`](decks/README.md#confirmation-dangerous-buttons). **Controls** is off the button list for now, so a deck does not have to click. Pressing a control, setting a slider or a field, and stepping a value through Accessibility stay the long-term goal. **Key** and **Extra Keys** can use **Toggle mode**: the first tap holds the key, characters repeat after a short wait, and the second tap releases it. Modifiers stay down and do not repeat. A click, a scroll step, and screen zoom stay on the Trackpad. An app’s own zoom is a menu action when that app offers one. Hide, quit, launch, and Dock items stay on the Apps tab. MIDI stays on the MIDI tab.

### Menus tab
The **Menus** tab shows the frontmost app's menu bar as a drill-down list: the **Apple menu () first**, then the app's menus. Tap a menu to open it, the **‹** button or a breadcrumb segment to go back, **↻** to re-read. Each item shows its checkmark, keyboard-shortcut hint (e.g. `⇧⌘N`) and is dimmed when disabled. Submenus are read lazily, one level per tap, so dynamic menus (Open Recent, Window…) are always current; the list resets to the top when another app comes to the front.

Tapping a leaf item presses it through the helper's Accessibility **AXPress** on that exact `AXMenuItem` (addressed by path + id), never AppleScript / System Events and never clicks on menu-bar coordinates (SPEC §6.5). Any connected phone may browse and press. The trackpad is the surface another phone can hold. Its tab icon has a green dot on the phone that has it and a red dot on the others. *Quit …*, *Sleep*, *Restart*, *Shut Down* and *Log Out* items ask for confirmation on the phone first. Needs the same Accessibility grant as everything else.

### Over Tailscale
Use the Mac's `100.x` address or MagicDNS name with `:8788`. For HTTPS (`wss://`): `tailscale serve --bg 8788` and bookmark `https://<mac>.<tailnet>.ts.net/?k=<token>`.

## What works in this build

| Area | Status |
|---|---|
| Server: static client, `/api/health`, `/api/status` (bearer), token file, LAN/Tailscale QR codes, MagicDNS URL | ✅ |
| WebSocket `/ws`: `hello` auth within 2 s (or `Authorization: Bearer` on upgrade), close `4401` | ✅ |
| Heartbeat (`ping`/`pong` 3 s, drop after 10 s), reconnect with backoff | ✅ |
| One pointer owner, `session.engage` / `release` / `takeover` / `revoked`. The Trackpad tab keeps the pointer until that phone leaves it. Release on disconnect / tab hidden / shutdown. Idle is 0 s: a one-phone key or MIDI session releases as soon as the action is sent. A modifier that is still down stays down until a key, a trackpad mouse-up, or a tap releases it | ✅ |
| Helper spawn + JSON-lines protocol, auto-restart with backoff, dry-run when missing, capability flags (`textFocus`, `zoom`) with one-time rebuild hints | ✅ |
| Trackpad: move (2.5x base gain + host-side acceleration), tap / double-tap / two-finger tap, two-finger scroll + momentum, hold-to-drag, drag lock, L/R buttons | ✅ |
| Trackpad zoom buttons 1X / 2X / 4X via the private WindowServer zoom (helper 0.8.2), Zoom during motion, and zoom timeout (Off / 300s / 60s / 15s, default 60s; a 2X / 4X level returns to 1X after the phone sits still) | ✅ private method and zoom timeout confirmed on this Mac. `keys` / `scroll` stay the fallbacks (`scroll` was ignored by Accessibility Zoom on 12.7.6) |
| Keyboard: Unicode text, Return → Mac Return, echo with ENTER / SPACE / TAB / BACKSPACE tokens and 1 s idle clear, special keys (⌫ and arrows auto-repeat while held), F-keys, media keys, sticky modifiers (latched / locked, shared across phones), layout-aware shortcuts (AZERTY etc.) | ✅ |
| Release-everything path (buttons, keys, modifiers, cursor, MIDI notes); signal / stdin-EOF handling off the helper main thread | ✅ |
| Cursor grow/restore via private `CGSSetCursorScale` (dlsym, crash-restore file), per-phone size and timeout | ✅ |
| Focus tracking (`focus.changed`) drives decks | ✅ |
| Apps tab, **Dock**: Finder + pinned apps, folders & links, recent apps from `com.apple.dock` prefs; tap opens (`dock.list` / `dock.open`) | ✅ (helper sources; verify on Mac) |
| Apps tab, **Running**: live list, tap activates, long-press Hide / Quit (`apps.list` / `activate` / `hide` / `quit`, pushed on launch/quit) | ✅ (helper sources; verify on Mac) |
| Apps tab, **Apps**: `/Applications`, `/System/Applications`, `~/Applications` (+1 sub-folder level), alphabetical + search; tap launches (`apps.catalog` / `apps.open`) | ✅ (helper sources; verify on Mac) |
| App icons: 96 px PNG per versioned `iconId`, cached in IndexedDB on the phone (`icon.get`) | ✅ |
| Shortcut decks: system popup (grouped, dimmed scrim), editable app deck (50×50 cells, pages, phone storage), window controls, built-ins for Safari, Chrome, Finder, Max, Spotify, VLC, TV, QuickTime Player, Preview, Photoshop, Illustrator, Premiere Pro, After Effects, DaVinci Resolve, Blender, Ableton Live, Keynote, Zoom, Slack, confirm for Sleep / Shut Down | ✅ (menu actions via AXPress). Extra button actions still open: DECK-PLAN.md (only available in the main repo) |
| Helper `menu.get` / `menu.invoke` (AXPress only, no AppleScript): enabled / checked / shortcut with modifiers, Apple menu flagged first, disabled items refused | ✅ (helper sources; verify on Mac) |
| Menus tab: drill-down tree + breadcrumb, refresh on focus change / drill-down / ↻, invoke by path + id (any phone, stale-app guard), confirm for Quit / Sleep / Restart / Shut Down / Log Out | ✅ |
| Virtual MIDI: CoreMIDI source (`midiSourceName`, default “Ground Control”), `midi.note` / `midi.cc` / `midi.pitch` / `midi.transport`, all-notes-off on release. A sounding note, CC, pitch bend or transport command shows on every phone that is sending that same address | ✅ (helper sources; verify on Mac / in a DAW) |
| MIDI tab: layouts (create, select, rename, delete). Each layout has a grid size, a background, and either one channel or a channel per control. The minimum cell size in Preferences sets how many columns and rows fit, and the grid fills the screen. Edit replaces the octave stepper with **+**. A one-second press opens a widget’s settings | ✅ |
| Top bar ⋮ menu: light / dark mode, Preferences, Disconnect / Connect (no reload) | ✅ |
| Preferences: visible tabs, auto Keys on Mac text focus (Chromium / Electron web content included), open keyboard on Keys, MIDI stage mode, pointer and scroll speed, acceleration, invert scroll, cursor size, cursor timeout, zoom during motion, zoom timeout, delete all sent text (`prefs.set`, per phone, resent on connect) | ✅ (auto Keys + cursor: helper sources; zoom timeout confirmed on this Mac) |
| Local checks: `npm run check` (smoke tests + build), `npm run check:helper` (macOS) | ✅ |

### Try MIDI in a DAW (Mac)

1. `npm run build:helper && npm run build && npm start`
2. On the Mac open Logic / Ableton / MainStage / GarageBand / any CoreMIDI host and look for input **Ground Control** (rename via `midiSourceName` in `~/.config/ground-control/config.json`).
3. Phone → bookmark → **MIDI** tab: choose a layout or make a new one, set the grid and the channel, then tap pads and drag faders. **Edit** changes the controls. Notes / CCs should appear in the DAW’s MIDI monitor. Ending the session (leave the Trackpad tab, lock the phone, disconnect) sends all-notes-off when that phone is the last one.

Extra messages beyond SPEC §7:
- server → client: `helper.status {available, running, reason}`, `menu.invoked {ok, path}`, `textfocus.status {state, helperVersion}`.
- server → helper: `session.engage {cursorScale}`, `cursor.scale {scale}`, `textfocus.watch {on}`, `zoom.set {level, from, method, keySteps2x, keySteps4x, keyOut, keyGapMs, units2x, units4x, out, scrollUnits, scrollTap, continuous, suppress, smooth, settleMs, invert, holdModifier}`, `zoom.get`, `zoom.resetPrivate`, `zoom.motion {on, level?, smooth}`, `zoom.once {method, delta | factor, ...}` (probe only).
- helper → server: `ready {version, accessibility, cursorScale, midi, midiSourceName, textFocus, zoom, zoomMotion}`, `textfocus {editable, role?, subrole?, pid?}`, `textfocus.status {ok, watching, reason?}`, `zoom.state {level, from?, method, requested?, fallback?, presses?, delta?, privateFactor?, before?, factor?, ok, error?, hotkeys?, privateAPI?, scrollZoom?, motion?, active?, reason?, ms?}` (`mode: motion` for Zoom during motion, `mode: reset` when a private zoom was reset).
- client → helper: `midi.transport {cmd: start|stop|continue}` (system realtime).

Helper 0.9.0 posts the Chrome extension banners. The note is `chrome/HELPER.md` in [ground-control-app-hooks](https://github.com/yacfish/ground-control-app-hooks), protocol `ext/1`. The extension connects to the Node server on `/ws` as device `chrome`. The server sends these. The helper only posts macOS notifications (`ready.notify`):

- `pair.show {code, timeoutMs?}` shows a 6-digit pairing code. `npm run ext:pair` asks the running server for it. The server prints the same code and sends `pair.show`. The helper answers `pair.shown` and removes the banner when the wait elapses (2 minutes unless `timeoutMs` says otherwise). A successful pair stores a hashed token in `ext-clients.json`. `npm run ext:revoke -- <name>` removes it.
- `confirm.show {id, title, body, timeoutMs?}` posts one notification with Accept and Reject. The title is the control name. The body is the type and the site, for example `button on youtube.com`. A new or edited mapping waits 30 s. A sensitive action waits 15 s. The helper answers `confirm.shown` when the banner is up. macOS 12 will not let this ad-hoc helper use UserNotifications, so the banner is posted with the older notification center, from `helper/bin/gc-helper.app`.
- `confirm.result {id, accepted}` is the answer. Closing the notification is Reject. On timeout the helper removes the banner and sends nothing. When notifications are off, the helper answers `error` `confirm_unavailable` and the action does not run.
- The helper does not fetch presets, and it needs no new message for which app is in front. The first banner asks for notification permission.
- App hooks use the same banners. `HOOKS.md` in that repo is protocol `hook/1`. `npm run hook:pair` prints a code and the server sends `pair.show`. `npm run hook:revoke -- <name>` removes the hashed token in `hook-clients.json`. Blender's add-on still speaks `blender/1` (no `app` field). The Deck tab shows the hook while that app is in front, and Chrome's deck while Chrome is in front. `ext.state` and `hook.state` are forwarded to the phone and not stored. A secret text field on a deck or a panel is masked. The deck file keeps a reference, not the text. The phone sends it once, as `key.text` with `secret: true` or as a hook action, and the server does not log it.

Errors that come back from the helper carry `ref: <request type>` (e.g. `menu.get`, `menu.invoke`, `deck.run`).

App icons fall back to a letter tile when the helper returns no PNG; an `.icns`-file fallback is deferred.

## Layout
```
server/      Node: index.js, session.js (sessions, prefs, routing, cursor timeout, zoom state), extension.js (ext/1 chrome client), hooks.js (hook/1 app hooks, Blender alias), helper.js, config.js, decks.js (load + resolve deck.run)
src/         React client: App.jsx (tabs, top menu, auto Keys), connection.js, Trackpad.jsx, Keys.jsx + keysModel.js, Deck.jsx, LiveDeck.jsx (Chrome and app-hook decks), Midi.jsx,
             Apps.jsx (+ icons.js cache), Menus.jsx + menuModel.js, Preferences.jsx + prefs.js, styles.css
decks/       Built-in shortcut deck JSON (system + per-app)
helper/      Swift package gc-helper: Input, Keys, Cursor, Menu (AXPress), Apps, MIDI (CoreMIDI virtual source), System (sleep / shut down),
             TextFocus (auto Keys), Zoom (zoom presets: private API, keys, scroll), Confirm (pairing code and Accept / Reject, SPEC §7), IO, main.
scripts/     build-helper.sh, check-helper.sh, mock-helper.js, smoke-test.js, helper-probe.js, zoom-probe.js
```
