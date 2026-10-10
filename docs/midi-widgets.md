# MIDI widgets

This is the widget list the MIDI page builds. SPEC §6.3 points here. A saved fader or horizontal knob opens as a 1D slider or a square rotary knob.

## Adding a widget

In edit mode, the octave stepper is replaced by a **+** button sitting immediately left of **Done**. Leaving edit puts the octave stepper back, when the layout has show-octave on. Stage mode still has no Edit, so it has no **+** either.

**+** opens a popup in three steps:

1. Pick a group: **Note**, **Continuous Controller**, or **System**.
2. Pick a widget from that group.
3. A settings panel for that widget. Confirming it places the widget on the first free cells. If the grid is full, it grows downward, up to the rows that fit. A shape that does not fit is refused.

A widget already on the grid opens that same settings page, and only that page, after a one-second press that stays within 3 pixels. A short tap selects the widget and does not move it or open its settings. While it is selected, a small + inset at its bottom-right corner copies it into the next free cells when they exist. Edit mode still does not send MIDI. The grid fills the MIDI area and does not scroll. How many columns and rows fit comes from the minimum cell size in Preferences.

Every widget shares these rules:

- It occupies a rectangle on the layout grid and cannot overlap another widget.
- It has a label, where a label makes sense. A widget Label may be left empty. Other text and number fields can be cleared while they are being edited; leaving one empty puts back that field’s default, and a number is brought inside its range only then.
- In **each control** channel mode, it has its own channel (1–16). In **whole grid** mode it uses the layout channel.
- Note widgets that follow the octave use the layout octave. The root is note 60, labeled C4.



## Note



### Pad

One button, one note. Press sends note on, release sends note off. Velocity comes from touch force, or from where the finger is vertically on the pad when the phone reports no force. The pad shows its label, or the note name when the label is empty. It does not show the note number or the channel. The face has no border. While it is held, the whole pad fills with the theme blue.

This is the current pad.


| Setting | Values                                                                            |
| ------- | --------------------------------------------------------------------------------- |
| Label   | Any length. Empty shows the note name.                                   |
| Size    | 1×1 or 2×2.                                                                       |
| Note    | Follows the layout octave, with an offset from −64 to +64, or a fixed note 0–127. |
| Channel | Only in each-control mode.                                                        |




### Keyboard

One widget that spans the full width of the grid and is 1, 2, or 3 rows tall. The keys inside that box do not line up with the grid columns. Playing a key sends that note the same way a pad does (note on while held, velocity from force or from Y on the key).

White keys share the width of the box equally. Black keys sit across the boundary of two white keys, in the usual piano pattern, and cover the top half of the box. Note names sit at the bottom of the white keys, clear of the black keys. The keys run left to right from a starting note, including the sharps. Following the octave means the strip starts at the layout’s C, plus an optional offset.

The widget occupies every column, so it fits only when that band of rows is free across the whole grid.


| Setting    | Values                                                                      |
| ---------- | --------------------------------------------------------------------------- |
| Label      | Any length, shown above or beside the keys if there is room.       |
| Height     | 1, 2, or 3 rows. The width is always the grid’s columns.                    |
| White keys | How many white keys divide the box.                                         |
| Note names | Alphabet (C4), Solfège (Do4), or None.                                      |
| Note       | Follows the layout octave, with an offset, or starts at a fixed note 0–127. |
| Channel    | Only in each-control mode.                                                  |




### Pad grid

One widget made of equal pads, for a drum rack or a block of notes. Each pad is a note. Press and release work like a single pad. Moving or deleting the widget moves or deletes the whole block.

Notes fill left to right, and the top row is the higher notes, matching the current default layout.


| Setting                | Values                                                                                                                                   |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Label                  | Any length, for the block.                                                                                                        |
| Pad names              | Alphabet (the note name), Index (1 at the top left, then across and down), XY (1-1 at the top left, column then row), YX (1-1 at the top left, row then column), Custom, or None. Custom types a comma-separated list in that same box, under the menu. Names fill left to right from the top left. |
| Pads across, pads down | How the widget is subdivided. The widget’s footprint on the layout is the same count: 4 across and 4 down occupies a 4×4 cell rectangle. |
| Note                   | Follows the layout octave from a root offset, or starts at a fixed note. Pads are chromatic from that root.                              |
| Channel                | Only in each-control mode. One channel for the whole block.                                                                              |




## Continuous controller

A continuous controller sends `midi.cc` with a value from 0 to 127. Nothing in this group sends notes.

### 1D slider

One axis, one CC. This replaces the current fader. The finger sets the value along the track. Releasing leaves the value where it is. The number is not drawn on the slider. The track fills the widget. The label is drawn over the top-left of the track and does not shorten it.

The footprint comes from the direction, the length, and the width. Length runs along the travel. Width is the thickness, across the travel. Vertical travel uses rows for the length and columns for the width. Horizontal travel swaps those.

Length is 1, 2, 3, 4, or fill. Fill uses the whole grid on the travel axis. On a 6×8 grid a vertical slider fills 8 rows, and a horizontal slider fills 6 columns. A fixed length that is longer than that axis does not fit. Width is 1 or 2, so a vertical slider on that grid with fill and width 2 occupies 2×8.

With pan mode off, the fill grows from the low end of the track up to the current value. The control starts at Default. With pan mode on, Default sits at the centre of the track. One half of the track runs from Default down to 0, the other half from Default up to 127, and the fill grows from the centre toward the finger.


| Setting   | Values                                                                                      |
| --------- | ------------------------------------------------------------------------------------------- |
| Label     | Any length.                                                                        |
| CC        | 0–127.                                                                                      |
| Direction | Vertical or horizontal.                                                                     |
| Length    | 1, 2, 3, 4, or fill. Fill is the grid rows when vertical, the grid columns when horizontal. |
| Width     | 1 or 2.                                                                                     |
| Default   | 0–127. The value before the first touch.                                                    |
| Pan mode  | On or off. On makes Default the centre, and the fill grows either way from there.           |
| Channel   | Only in each-control mode.                                                                  |




### 1D slider array

Several 1D sliders in one widget, for a row of sends or camera faders. Each slider has its own value and keeps it on release. The sliders fill the widget, with a small gap between them. A slider shows its name over the track when one is set, and does not show its value. The group label is drawn over the top-left of the sliders and does not shorten them. Length, width, default, and pan mode are the 1D slider’s settings, shared by every slider in the array.

The first slider uses the CC you set. The next uses the next CC number, and so on.

A row of vertical sliders is `(count × width)` columns wide and `length` rows tall. A stack of horizontal sliders is `length` columns wide and `(count × width)` rows tall. Fill still follows the travel axis: on a 6×8 grid a vertical array fills 8 rows, and a horizontal array fills 6 columns.


| Setting   | Values                                                                      |
| --------- | --------------------------------------------------------------------------- |
| Label     | Any length, for the group. Each slider can have its own name. |
| Count     | How many sliders.                                                           |
| First CC  | 0–127. Later sliders are first CC + 1, + 2, and so on.                      |
| Direction | A row of vertical sliders, or a stack of horizontal sliders.                |
| Length    | 1, 2, 3, 4, or fill, along the travel, same rule as the 1D slider.          |
| Width     | 1 or 2, the thickness of each slider.                                       |
| Default   | 0–127. Every slider starts here.                                            |
| Pan mode  | On or off, applied to every slider.                                         |
| Channel   | Only in each-control mode. One channel for every slider in the array.       |




### 2D slider

A rectangular pad. Horizontal position sends one CC, vertical position sends another. Both messages go out while the finger moves. Releasing leaves both values where they are.

Width is how many columns the pad spans, and length is how many rows. Each is 1, 2, 3, 4, or fill. Fill width uses every column, and fill length uses every row. On a 6×8 grid, both set to fill, the pad occupies 6×8.

The bottom-left corner is 0 on both axes and the far edges are 127. Each axis starts at its default.


| Setting              | Values                                                          |
| -------------------- | --------------------------------------------------------------- |
| Label                | Any length.                                            |
| CC X, CC Y           | Each 0–127.                                                     |
| Length               | 1, 2, 3, 4, or fill. The vertical side. Fill is every row.      |
| Width                | 1, 2, 3, 4, or fill. The horizontal side. Fill is every column. |
| Default X, Default Y | Each 0–127. The value on that axis before the first touch.      |
| Channel              | Only in each-control mode.                                      |




### Rotary knob

A square widget with a circular slider, one CC. This is not the current knob. The finger drags around the ring, and the value stays on release. The ring is a band centred in the square. Its outer size is a percentage of the square’s shorter side. The value arc and the rest of the track share that band’s inner and outer edge, and the band does not reach the centre. The label is centred on the widget, drawn over the ring. The value is not shown.

Default and pan mode work as on the 1D slider. Pan mode off draws the arc from the low end of the ring up to the current value, starting at Default. Pan mode on puts Default at the top of the ring. One way around the ring runs from Default down to 0, the other way from Default up to 127, and the arc grows from the top toward the finger.


| Setting  | Values                                                                           |
| -------- | -------------------------------------------------------------------------------- |
| Label    | Any length.                                                             |
| CC       | 0–127.                                                                           |
| Size     | 1x1, 2x2, 3x3, 4x4                                                               |
| Default  | 0–127. The value before the first touch.                                         |
| Pan mode | On or off. On puts Default at the top of the ring, and the arc grows either way. |
| Channel  | Only in each-control mode.                                                       |




### Toggle

A pad that stays on or off. Tapping swaps the state. Off sends the off value, on sends the on value, both on the same CC. It does not spring back.

The face has no border. A rounded square, two-thirds of the widget’s width and height, sits inset in the middle. That square is blue while the toggle is on. A label is drawn over the square. With no label, the square is the only sign of the state.


| Setting             | Values                              |
| ------------------- | ----------------------------------- |
| Label               | Any length. Empty shows no text.    |
| CC                  | 0–127.                              |
| Off value, on value | Each 0–127. Defaults are 0 and 127. |
| Size                | 1×1 or 2×2.                         |
| Channel             | Only in each-control mode.          |




### Toggle group

A block of toggles, laid out like a pad grid. Each stays on or off on its own, with the same inset square as a single toggle. The squares are separated by the same gap as the pads in a pad grid. The first toggle, at the top left, uses the CC you set. The next, to its right, uses the next CC number, and so on, wrapping to the next row. Off and on values are shared.

Across and down are how the block is subdivided, and they are also the widget’s footprint on the layout. Four across and four down occupies a 4×4 cell rectangle. Names work like a pad grid: Index, XY, YX, Custom, or None. Custom types a comma-separated list in that same box, under the menu. Names fill left to right from the top left. None shows no text. The group label is drawn over the top-left and does not change the size of the toggles.


| Setting             | Values                                              |
| ------------------- | --------------------------------------------------- |
| Label               | Any length, for the group. |
| Across, down        | How the block is subdivided, 1 to 8 each. The footprint matches that count. |
| Toggle names        | Index, XY, YX, Custom, or None, the same choices as a pad grid except Alphabet. Custom types a comma-separated list in that same box, under the menu. |
| First CC            | 0–127. Later toggles are first CC + 1, + 2, and so on, left to right, then down. |
| Off value, on value | Each 0–127, shared. Defaults are 0 and 127.         |
| Channel             | Only in each-control mode. One channel for every toggle in the group. |




### Radio button

A row of options, one of which is selected. All options share one CC. Tapping an option selects it and sends that option’s value. The previous option turns off without sending a second message.


| Setting | Values                                                                     |
| ------- | -------------------------------------------------------------------------- |
| Label   | Any length, for the group. Each option has its own name and value. |
| CC      | 0–127.                                                                     |
| Options | Name and value (0–127) for each option. At least two.                      |
| Size    | One row tall, one column per option.                                       |
| Channel | Only in each-control mode.                                                 |




## System



### Pitch

Pitch bend. A horizontal drag sends `midi.pitch` from 0 to 16383. Releasing springs back to centre, 8192. This is the current pitch control.


| Setting | Values                     |
| ------- | -------------------------- |
| Label   | Any length.       |
| Size    | 2×1 or 4×1.                |
| Channel | Only in each-control mode. |




### Transport

Start, continue, and stop. Each button sends `midi.transport` with that command. This is the current transport control, grouped into one widget so the three commands sit together. A button you turn off is hidden, and the widget narrows to the buttons that remain.


| Setting | Values                                                                  |
| ------- | ----------------------------------------------------------------------- |
| Label   | Any length.                                                    |
| Buttons | Start, continue, and stop, each shown or hidden. At least one stays on. |
| Size    | One row tall, one column per visible button.                            |
| Channel | Transport is a system message, so it has no channel.                    |




### Label

Text on the grid. It sends nothing. It is a heading, such as “Drums” or “Cameras”, and it can be dragged like any other widget.


| Setting    | Values                                                                                          |
| ---------- | ----------------------------------------------------------------------------------------------- |
| Label      | Any length. Empty stays empty.                                                        |
| Text colour | A checker swatch of the current colour, or the word Theme on the right when none is set. Opened, the layout colour picker plus alpha. Theme, between Cancel and Apply, keeps the theme text at once. Apply keeps a picked colour. Cancel discards it. |
| Border     | Same swatch and picker, plus alpha. Theme keeps the theme border at once.                      |
| Background | Same swatch and picker, plus alpha. Theme stays clear at once, so the layout background shows through. |
| Align H    | Left, center, or right.                                                                         |
| Align V    | Top, center, or bottom.                                                                         |
| Size       | 1, 2, 3, 4, or fill. Fill is every column, height 1.                                           |
| Font size  | 1 to 5. 1 is the current text size. 5 is five times that.                                      |

Text aligned to an edge keeps a small margin, so it stays inside the widget’s rounded corner.




### Tab capsule

A box on the layout that holds several mini layouts. A row of tabs across the top of the box chooses which mini layout is showing. The open tab shares its face with the layout under it. The capsule sends no MIDI. It inherits the layout’s octave and channel mode, and nothing else: not the layout grid, and not the layout background.

The box’s width and height on the layout grid are each 2, 3, 4, or fill. Fill width is every column of the layout. Fill height is every row. A fill-by-4 capsule on a 6×8 layout occupies 6 columns and 4 rows.

Inside the box, the tabs are only as tall as the labels need. They sit on the top edge of the mini layout and do not take a full layout-cell of height. The remaining area is the mini layout, divided into that tab’s own columns and rows. Those inner counts ignore the layout grid and ignore the capsule’s width and height. One tab can be 3×4, another 8×2, another 5×5, all inside the same 6×4 box. More inner cells means smaller cells. Widgets inside a tab sit on that inner grid, so a pad of 1×1 is one inner cell, and a slider set to fill uses the inner grid.

The capsule starts with one tab. In edit mode, a **+** at the end of the tab row adds another. Every new tab is empty. Each tab has its own label, its own background colour, and its own inner grid. The colour fills that tab and the background of its mini layout, so the open tab and the layout read as one piece.

Tapping a label shows that tab, while playing and while editing. In edit mode the labels stay selectable, and a one-second press on a tab’s button opens the capsule’s settings, which still works when the mini layout is full. The selected tab is where a new widget lands. Tapping the layout outside the capsule clears that, so the next widget lands on the layout grid. The trash icon in those settings deletes the current tab and closes the settings, and deletes the capsule only when that tab is the last one.


| Setting       | Values                                                                    |
| ------------- | ------------------------------------------------------------------------- |
| Width, height | Each is 2, 3, 4, or fill, on the layout grid.                             |
| Tabs          | One to start. in edit mode, **+** on the tab row adds an empty tab.     |
| Label         | Any length, per tab.                                             |
| Colour        | Per tab. The open tab and its mini layout share it. |
| Inner grid    | Columns and rows for that tab, dividing the area under the tabs.     |




## What this replaces


| Today     | Becomes                                                                                                          |
| --------- | ---------------------------------------------------------------------------------------------------------------- |
| Pad       | Note → Pad                                                                                                       |
| Fader     | Continuous Controller → 1D slider, vertical                                                                      |
| Knob      | Continuous Controller → Rotary knob. The saved control becomes the square ring; the old horizontal drag is gone. |
| Pitch     | System → Pitch                                                                                                   |
| Transport | System → Transport                                                                                               |


A layout saved with those five kinds still opens. Each old widget is read as the matching kind above.