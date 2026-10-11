# Several clients at once

This is a plan, not the current behavior. SPEC §4 still says one client controls the Mac. A second phone connects, sees "In use by …", and has to take over, which drops the first phone's held keys, mouse buttons, and MIDI notes.

The split below is the one to build. One phone alone should keep working as it does today.

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
- A hold of about half a second locks it until any phone taps it again. ⌘E from the keys phone and then a click from the trackpad phone both see that ⌘, and both phones show it on.
- The server remembers which phone turned each modifier on. That phone leaving, locking, going idle, or disconnecting lifts only what it turned on. The other phones' chips turn off with it.

Toggle-mode keys on Keys and on a deck are the same kind of hold. An arrow held on one phone shows held on the others, and any phone can release it.

## No mouse actions on the deck

Controls is off the button list for now, so this plan never has to move the pointer. The sheet, `window.controls`, `window.press`, and `window.set` stay in the code. A button already saved as Controls on a phone still opens. A new button cannot.

The one list of everything Accessibility returns was clunky, and some of those actions fall back to a real click. That click is what is out. The other modes stay the long-term goal: press a button, set a slider, set a field, and step a value through Accessibility, then a few widgets built from that data one kind at a time. None of those move the pointer. The trackpad phone stays the only client that clicks.

## MIDI

Every phone may play at once. Note on, note off, CC, pitch, and transport all go to the same virtual source.

The sync that matters is the pad, not only the bytes. If two phones have a pad for the same sounding note, pressing it on one turns that pad on on the other, and releasing it turns both off together. The pad stays on while any phone is still holding that note, and goes off when the last finger lifts. A note that exists on only one phone just plays. A layout kept in sync, below, also lights the same widget, even when one phone's octave offset makes that widget sound a different note.

A phone that leaves must silence only the notes it turned on, and only those pads turn off. Today one release sends all-notes-off for the whole port, which would cut the other players and darken their pads. The server has to remember which phone holds which notes.

## Layouts on the Mac

Decks and MIDI layouts can be saved on the computer, beside the JSON file a deck can already download or upload. The store is one SQLite file on that Mac. The Node server uses it now. The menu-bar app opens the same file later. There is no account and no database server. Each saved layout is the JSON the phone already stores, plus a name and a time. The copy on the Mac is the unshifted layout.

**Load** copies it onto the phone. That copy is private until the phone opts in.

**Keep synchronized** is that option, per layout. The grid on every phone that opted in is the same grid. Add a pad on one and it appears on the others. The same widget turns on and off together. Two phones editing at once: the later save is the one stored.

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

Idle follows the same split. The trackpad phone going quiet gives the pointer back and shrinks the cursor. It does not lift a chord locked on the keys phone. A heartbeat still does not count as activity.

## Doubts

- **Opening the Trackpad tab claims the pointer**, even before a finger is down. A phone left on that tab holds it until idle, background, or Release. Background already ends a session today. The same rule should give the trackpad up, or the other phones have to take over from a phone nobody is touching.
- **A shared ⌘ is one chord, not two.** The chips match because they are one control. If the keys phone latches ⌘ and another phone immediately sends a shortcut, the latch is spent and every chip goes off. Two people who wanted independent modifiers will step on each other. That is the cost of turning on and off together.
- **A locked modifier's owner is the phone that locked it.** Another phone sees it and can tap to release it. If the owner disconnects, the lock lifts even though the other phones were still showing it. The alternative is to keep the lock until an explicit tap, which leaves ⌘ down after a dead phone.
- **Menu and Apps actions stay live** while someone is on the trackpad. They do not click, but they do change the frontmost app, so the trackpad phone's target moves. The focus broadcast already tells that phone. Graying them was considered and left out.
- **A saved Controls button can still click.** Taking it off the list stops new ones. A phone that already has one can still open it, and the helper still falls back to a real click when Accessibility cannot press the control. That click moves the pointer. Leaving the path in place is what makes it easy to restore. It is also the one deck action that does not fit "nothing on the deck needs the mouse."
- **Zoom and pointer speed stay with the trackpad phone.** The keys phone's cursor size and zoom preference do nothing until that phone takes the trackpad. Two zoom gestures at once are not a goal.
- **The Mac's own mouse still ignores a phone's ⌘.** Shared modifiers are on keys and on clicks Ground Control posts. A click from the Mac's trackpad is stamped by the physical keyboard. Command-click in Max keeps working from the phone's trackpad, and keeps failing from the Mac's mouse. This plan does not change that.
- **Text drafts stay on the phone.** A synced deck or MIDI layout shares the grid. An unsent text draft does not.
- **The offset and a hand edit should not both apply.** Adding an octave on top of a channel someone already set by hand is easy to misread. Each synced layout on a phone is either shifted by that phone's offset or set by hand.
- **Same widget and same note can disagree** once an octave offset is on. The synced layout lights the widget. The note rule lights whatever pad is actually sounding that note. A phone shifted up an octave will light its C5 pad when the other phone plays C5, and will also light the same pad the other phone pressed, which on the shifted phone may be sounding C6.
- **Faders and knobs are not on/off.** On a synced layout the same fader exists on every phone, so its position can move together. That is part of keeping the layout synchronized. Two private layouts still have no twin fader to move.

## Milestones

1. **Split the session.** The server distinguishes the trackpad holder from everyone else. Keys, text, deck buttons that do not click, menus, app actions, and MIDI are accepted from any connected phone. Take over applies to the trackpad only. Release is per phone, as above. One connected phone behaves as it does now. SPEC §4 is updated in this milestone so the code and the spec match.

2. **Mirror the modifier chips.** One on/off value per modifier, pushed to every client, so a press on one phone turns that chip on everywhere and a release turns it off everywhere. Latched clears on the next key from any phone or the next mouse-up from the trackpad phone. Locked stays until a tap. Toggle-held keys use the same on/off.

3. **MIDI pads for the same note.** Pressing a note on one phone turns that pad on on every phone that has it, and the last release turns them all off. Notes are remembered per phone so one disconnect silences and darkens only that phone's notes. Transport from any phone is last-writer-wins. Faders and knobs on private layouts wait.

4. **Layouts on the Mac.** SQLite on the computer. Save and load from the share sheet, for decks and MIDI layouts. Keep synchronized is optional per layout. Channel and octave stay on the phone: set by hand per layout, or by one offset applied to every MIDI layout that phone keeps synchronized. Neither is written into the stored copy.
