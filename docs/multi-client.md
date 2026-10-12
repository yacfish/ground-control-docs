# Several clients at once

Milestones 1, 2 and 3 are the current behavior. Only the trackpad is exclusive. Modifier chips are shared, and a deck key held down shows held on the other phones. A sounding MIDI note lights every control that is sending it, on every phone. A second phone can type, hold keys, run a deck button, press a menu, launch an app and play MIDI while someone else holds the pointer. Take over moves the trackpad and leaves everyone else's keys and notes down. One phone alone still engages on the first key or note. That session releases immediately, and also when the phone leaves, locks, or disconnects. A modifier or a toggled key that is still down stays down until it is released. The Trackpad tab does not time out. The layout library is still a plan.

## Who owns what

**One trackpad.** Move, scroll, buttons, drag lock, screen zoom, and the enlarged cursor belong to a single phone. That phone has the Trackpad tab open. A second phone that opens Trackpad sees who has it and can take it. Taking it releases that phone's mouse button and cursor. It does not drop anyone else's keys or MIDI notes.

**Keys and Text, freely.** Any other phone can type, hold a key, or send text while the trackpad is in use. Two people typing into the same document will interleave. That is acceptable.

**One deck is the normal case.** A third phone can run deck buttons that do not move the pointer: keys, text, menus, activate, sleep, shut down. A second deck at the same time is allowed and expected to stay rare. A deck can stay private on the phone, or be one of the layouts kept on the Mac.

**MIDI has no limit.** Every connected phone plays at once. A pad for the same note is the same control on every phone: it turns on and off together.

**Menus and Apps stay usable** from any phone. Browsing is already open to everyone. Pressing a menu item or launching an app does not need the trackpad, because those actions do not post a click.

## Modifier keys, synced

The chips are ⌘ ⌥ ⌃ ⇧ fn. These are the modifier keys. F1–12 are not part of this sync. They fire and finish.

Each chip is one control, the same way a MIDI note is one control. Press ⌘ on one phone and that chip turns on everywhere it appears: the Keys tab and the trackpad row, on every connected phone. Release it and every copy turns off at the same time. Latched and locked are both "on". Locked is the hold that stays until a tap. The screens do not disagree.

- A short tap latches the modifier. The next key from any phone, or the next mouse-up from the trackpad phone, releases it, and every chip goes off.
- A hold of 0.8 s locks it until any phone taps it again. The chip fills and grows 3px on every side as soon as that time is reached, while the finger is still down, then returns to its normal size when the finger lifts. The chips around it stay put. A chip that is already locked turns off when the finger lifts. ⌘E from the keys phone and then a click from the trackpad phone both see that ⌘, and both phones show it on.
- The server remembers which phone turned each modifier on. That phone leaving, locking, going idle, or disconnecting lifts only what it turned on. The other phones' chips turn off with it.

Toggle-mode keys on a deck are holds (`key.down` / `key.up`). An arrow held that way shows held on the other phones, including the matching arrow on Keys, and any phone can release it. Holding an arrow on the Keys tab auto-repeats as key presses. That press stays on the phone that is touching it.

## No mouse actions on the deck

Controls is off the button list for now, so this plan never has to move the pointer. The sheet, `window.controls`, `window.press`, and `window.set` stay in the code. A button already saved as Controls on a phone still opens. A new button cannot.

The one list of everything Accessibility returns was clunky, and some of those actions fall back to a real click. That click is what is out. The other modes stay the long-term goal: press a button, set a slider, set a field, and step a value through Accessibility, then a few widgets built from that data one kind at a time. None of those move the pointer. The trackpad phone stays the only client that clicks.

## MIDI

Every phone may play at once. Note on, note off, CC, pitch, and transport all go to the same virtual source.

The sync that matters is which note and channel a control is sending, not only the bytes. Each phone applies its channel shift and its octave offset first, then compares. Pressing a control turns on every control, on every phone, that is sending that same channel and note, and the last release turns them all off. On one phone a pad and every key sending that note light together, as when C4 is held and the pad and both keyboards show it. A note that no other control is sending just plays. A phone whose shift or offset moves a pad onto a different channel or note leaves that pad dark.

A phone that leaves must silence only the notes it turned on, and only those pads turn off. Today one release sends all-notes-off for the whole port, which would cut the other players and darken their pads. The server has to remember which phone holds which notes.

## Layouts on the Mac

Decks and MIDI layouts can be saved on the computer, beside the JSON file a deck can already download or upload. The store is a folder of JSON files on that Mac. The Node server reads and writes it now. The menu-bar app opens the same folder later. There is no account and no database server. Each saved layout is the JSON the phone already stores, plus a name and a time. The copy on the Mac is the unshifted layout.

The folder sits in the config directory. `$GC_CONFIG_DIR` replaces `~/.config/ground-control` when it is set.

```text
~/.config/ground-control/
  config.json
  token
  decks/                  # existing overrides of built-ins
  library/
    decks/<name>.json
    midi/<name>.json
```

`decks/` and `library/` do different jobs. A file in `decks/` replaces the built-in the server serves for that app. A file in `library/` is a named copy a phone loads.

The filename is the layout name. Two layouts in the same folder cannot share a name, and two MIDI layouts on one phone cannot either. A rename onto a taken name is refused. "Taken" ignores case, because the Mac volume treats `Keys.json` and `keys.json` as one file. A name containing `/`, or a name that is empty, `.`, or `..`, is refused. A rename writes the new file and removes the old one. A change of case only rewrites the name inside the file that is already there.

There is no id in the file. The layout id on the phone is local: it picks which layout is open. Share → Download already leaves button ids out of a deck file on purpose. A phone that loads a library file assigns its own layout id and does not read one from the file. Deck buttons and MIDI widgets get new ids on that phone, the same way Share → Upload works today.

Lighting does not use an id. Each phone applies its channel shift and octave offset, and every control that is then sending the same channel and note lights together, on that phone and on the others. A CC matches after the channel shift, on channel and CC number. A slider array and a toggle group count forward from the first CC, so each step has its own. An XY pad has one CC per axis. Pitch bend matches on the channel after the shift. Transport matches on the command. A label or a tab capsule sends nothing, so it has nothing to match and travels only as part of the saved grid. A synced update keeps a control's value by that sounding channel and note or CC, then the phone assigns local ids for editing.

```json
{
  "kind": "midi",
  "name": "Keys",
  "updatedAt": "2026-10-12T18:00:00.000Z",
  "rev": 3,
  "created": {
    "at": "2026-10-12T18:00:00.000Z",
    "device": "Studio iPhone",
    "screen": { "w": 390, "h": 844, "dpr": 3 }
  },
  "layout": {}
}
```

`kind` is `deck` or `midi`. `layout` is the phone's grid with channel and octave left out. The server stamps `updatedAt` and `rev` when it accepts a save. `created` is written on the first save and kept after that. `device` is the name of the phone that first saved the layout. The server reads it from the connection: the Bonjour name of that address on the LAN, which is the name set in Settings, or the Tailscale machine name when the phone is on the tailnet. When the lookup returns nothing, the server keeps the coarse label the phone sends with `hello` (`iPhone`, `iPad`, `Android`, `Mac browser`, or `Browser`). `screen` comes from the phone: the CSS pixel size of the page and the device pixel ratio, which is the space the grid was built in. The server writes `<name>.json.tmp` and renames that over the real file. The Node process is the only writer, and it handles one save at a time, so two phones editing one layout still end with one complete grid: the save that arrived second. Saving under a name the library already has warns that the name is in use, and replaces that layout only after the warning is confirmed. Cancelling leaves the file as it is. There is no index file. Listing the directory and reading each file is enough. A file that does not parse is skipped.

Pad lights, held modifiers, and which phone has opted in stay out of these files. Those are session state and phone state.

**Load** copies it onto the phone. That copy is private until the phone opts in.

**Keep synchronized** is that option, per layout. The grid on every phone that opted in is the same grid. Add a pad on one and it appears on the others. Two phones editing at once: the later save is the one stored. Lighting stays the sounding channel and note, after each phone's own shift and offset.

The grid is what syncs. Channel and octave belong to the phone, not to the copy on the Mac. Two phones can play the same pads and send different notes. Neither phone's channel or octave is written back.

There are two ways to set them:

- **By hand, on that phone.** Change the channel or the octave of that layout the way a private layout is changed. One layout can sit on channel 10 while another on the same phone stays on channel 1.
- **An offset for every synced layout.** When the phone loads from the Mac it can add a channel amount and an octave amount, then apply both to each MIDI layout it keeps synchronized. In per-control mode the channel amount is added to each widget. Channel stays inside 1–16 by wrapping. Notes that would fall outside 0–127 are clamped.

A phone uses one way or the other. The offset is the shortcut when every synced layout should move together. The hand edit is how a single layout takes its own channel or octave.

## Release

Today any exit — Release, idle, disconnect, takeover, shutdown — lifts every mouse button, every key and modifier, the cursor, and every MIDI note. That is right for one driver and wrong for two.

Each phone releases only what it holds:

- The trackpad phone: its mouse buttons, drag lock, and the cursor size.
- Whichever phone latched or locked a modifier, or is holding a toggled key: those keys.
- Whichever phone started a note: those notes.

Someone else's holds stay down. The last phone to leave still runs the full release, so nothing can stick after the room is empty.

Idle does not give the trackpad up. The phone that has the Trackpad tab open keeps the pointer, including its enlarged cursor, until it leaves that tab, locks, or disconnects. A one-phone session started from keys, text, a deck, or MIDI releases immediately (`idleMs` is 0). A modifier or a toggled key that is still down stays down, and that session releases once it is up. A heartbeat does not count as activity.

## Doubts

- **Opening the Trackpad tab claims the pointer**, even before a finger is down. A phone left on that tab holds it until it leaves the tab, the phone locks, or it disconnects. Idle does not give it up. Background already ends a session today.
- **A shared ⌘ is one chord, not two.** The chips match because they are one control. If the keys phone latches ⌘ and another phone immediately sends a shortcut, the latch is spent and every chip goes off. Two people who wanted independent modifiers will step on each other. That is the cost of turning on and off together.
- **A locked modifier's owner is the phone that locked it.** Another phone sees it and can tap to release it. If the owner disconnects, the lock lifts even though the other phones were still showing it. The alternative is to keep the lock until an explicit tap, which leaves ⌘ down after a dead phone.
- **Menu and Apps actions stay live** while someone is on the trackpad. They do not click, but they do change the frontmost app, so the trackpad phone's target moves. The focus broadcast already tells that phone. Graying them was considered and left out.
- **A saved Controls button can still click.** Taking it off the list stops new ones. A phone that already has one can still open it, and the helper still falls back to a real click when Accessibility cannot press the control. That click moves the pointer. Leaving the path in place is what makes it easy to restore. It is also the one deck action that does not fit "nothing on the deck needs the mouse."
- **Zoom and pointer speed stay with the trackpad phone.** The keys phone's cursor size and zoom preference do nothing until that phone takes the trackpad. Two zoom gestures at once are not a goal.
- **The Mac's own mouse still ignores a phone's ⌘.** Shared modifiers are on keys and on clicks Ground Control posts. A click from the Mac's trackpad is stamped by the physical keyboard. Command-click in Max keeps working from the phone's trackpad, and keeps failing from the Mac's mouse. This plan does not change that.
- **Text drafts stay on the phone.** A synced deck or MIDI layout shares the grid. An unsent text draft does not.
- **The offset and a hand edit should not both apply.** Adding an octave on top of a channel someone already set by hand is easy to misread. Each synced layout on a phone is either shifted by that phone's offset or set by hand.
- **A shifted phone lights the note it is sending.** Channel shift and octave offset are applied before the comparison. A phone at +1 octave does not light its C5 pad when another phone plays C4. It lights the control that is sending C4 on that channel. Two controls that send the same pair both light, including a pad and a key on another phone.
- **Faders and knobs are not on/off.** After the channel shift, the fader on that channel and CC moves together on every phone. Two private layouts still have no twin fader to move unless they are sending that same pair.

## Milestones

1. **Split the session.** Done. The server distinguishes the trackpad holder from everyone else. Keys, text, deck buttons that do not click, menus, app actions, and MIDI are accepted from any connected phone. Take over applies to the trackpad only. Release is per phone, as above. One connected phone behaves as it does now. SPEC §4 matches the code.

2. **Mirror the modifier chips.** Done. One value per modifier is pushed to every client. A press on one phone turns that chip on everywhere and a release turns it off everywhere. Latched clears on the next key from any phone or the next mouse-up from the trackpad phone. Locked stays until a tap. A deck toggle (`key.down`) shows held on the other phones, and any phone can release it. SPEC §6.1 matches the code.

3. **MIDI pads for the same sounding note.** Done. Each phone applies its channel shift and octave offset first. Pressing a control turns on every control, on every phone, that is sending that same channel and note, and the last release turns them all off. Notes are remembered per phone so one disconnect silences and darkens only that phone's notes. Transport from any phone is last-writer-wins. Faders and knobs follow the same rule after the channel shift: same channel and CC. Pitch bend matches on the channel. SPEC §6.3 matches the code.

4. **Layouts on the Mac.** A `library/` folder of JSON files under the config directory, each file named by the layout name, as above. Save and load from the share sheet, for decks and MIDI layouts. Keep synchronized is optional per layout. Channel and octave stay on the phone: set by hand per layout, or by one offset applied to every MIDI layout that phone keeps synchronized. Neither is written into the stored copy.
