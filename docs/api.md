# sc* API reference

The game script talks to the engine through 87 global Lua functions, all registered
by the original `scream.exe` and all ported in [`src/api.cpp`](../src/api.cpp). This page
describes what each one does. For how the shipped scripts call them (every argument
pattern, with examples), see the generated [`api-usage-dc.md`](api-usage-dc.md) and
[`api-usage-2006.md`](api-usage-2006.md). Addresses like `0x4158d0` point into the DC
update's `scream.exe`. [`NOTES.md`](../NOTES.md) has the reverse-engineering detail, and
[`formats.md`](formats.md) the file formats.

**Usage** gives the number of calls in the Director's Cut script / the 2006 script.
Scratches calls 64 of the 87 functions. The **23 marked *Not used by Scratches*** are
never called by either release. They are still ported faithfully, so new games can use
them. Nine more are called **only by the Director's Cut**.

## Conventions

- **Arguments are loose.** Numbers and strings are read leniently, and `nil` reads as 0
  or `""`. The shipped scripts pass `nil` in a few places (the misspelt `SC_HITNS`, the
  never-defined `SC_STEREO_PATCH`), so the API never raises an error for a bad argument.
  Problems go to the log as warnings.
- **Constants** (`SC_*`) are defined by `scream.sc`; their values are in
  [`src/constants.hpp`](../src/constants.hpp).
- **Handlers** are Lua global functions named by string (`"exitHouse_dc"`). Clicks,
  reactions, timers, choices, room hooks and menu callbacks run them as coroutines. A
  runtime error ends that handler and the game carries on, as in `scream.exe`.
- **Blocking calls** suspend the running handler until something finishes:
  `scSleep`, `scLookAt(…, SC_SMOOTH)`, `scCutscene`, `scScriptGo` and
  `scFXPlayEx(…, wait=1)`. (`scream.exe` ran nested render loops instead. What scripts
  can observe is the same.)
- **Bindings.** Most builder functions act on the currently *bound* entity of a kind.
  Each kind (room, node, zoom, object, item, script, choice, track, overlay) has its own
  slot. `scCreate` and `scBind` set a slot. `scSwitch` also binds its target. A click
  binds the clicked object while its handler runs. Binding one kind leaves the other
  slots alone, and scripts rely on that.

### Object kinds

`SC_ROOM` 2000, `SC_NODE` 2001, `SC_ZOOM` 2002, `SC_OBJECT` 2003, `SC_OVERLAY` 2004,
`SC_ITEM` 2005, `SC_SCRIPT` 2006 (a dialogue), `SC_CHOICE` 2007, `SC_TRACK` 2008.

### Actions

`scOnClick`, `scReaction` and `scInventoryReaction` take an action `(kind, arg[, arg2])`:

| Kind | Effect |
|---|---|
| `SC_NODE, name` | walk to a node in the current room, playing its footstep |
| `SC_ROOM, name` | enter a room, at its last-visited node, playing that node's footstep |
| `SC_ZOOM, name` | open a zoom (pushed on the zoom stack) |
| `SC_OVERLAY, name` | show an overlay |
| `SC_ZOOMOUT` | pop one zoom |
| `SC_ZOOMBACK` | pop every zoom, back to the node |
| `SC_FEEDBACK, text` | show text |
| `SC_FEEDBACK_EX, text, voice` | show text and play a voice file |
| `SC_CUSTOM, handler` | run a Lua handler |

## Index

| Function | Usage (DC / 2006) | Section |
|---|---|---|
| [`scAbort`](#scabort) | 3 / 1 | Engine |
| [`scAttach`](#scattach) | *Not used by Scratches* | Engine |
| [`scAudio`](#scaudio) | 301 / 299 | World |
| [`scBegin`](#scbegin--scend) | 647 / 557 | World |
| [`scBind`](#scbind) | 549 / 512 | World |
| [`scChoiceAdd`](#scchoiceadd) | 24 / 20 | Dialogue |
| [`scChoiceClear`](#scchoiceclear) | *Not used by Scratches* | Dialogue |
| [`scChoiceGo`](#scchoicego) | 13 / 11 | Dialogue |
| [`scChoiceRemove`](#scchoiceremove) | *Not used by Scratches* | Dialogue |
| [`scCreate`](#sccreate) | 3104 / 2733 | World |
| [`scCursor`](#sccursor) | 11 / 11 | Presentation |
| [`scCutscene`](#sccutscene) | 145 / 114 | Presentation |
| [`scDisable`](#scenable--scdisable) | 229 / 218 | Engine |
| [`scEnable`](#scenable--scdisable) | 102 / 99 | Engine |
| [`scEnd`](#scbegin--scend) | 647 / 557 | World |
| [`scFXPlay`](#scfxplay) | 369 / 313 | Sound |
| [`scFXPlayEx`](#scfxplayex) | 275 / 249 | Sound |
| [`scFXSet`](#scfxset) | 134 / 117 | Sound |
| [`scFXStop`](#scfxstop) | 44 / 40 | Sound |
| [`scFeedback`](#scfeedback) | 414 / 382 | Text |
| [`scFeedbackEx`](#scfeedbackex) | 34 / — (DC only) | Text |
| [`scFont`](#scfont) | 3 / 1 | Presentation |
| [`scFootstep`](#scfootstep) | 201 / 142 | World |
| [`scGet`](#scget) | 43 / 42 | Engine |
| [`scGo`](#scwait--scgo) | 1 / — (DC only) | Control |
| [`scHints`](#schints) | 4 / 4 | Inventory |
| [`scHintsSet`](#schintsset) | 4 / 4 | Inventory |
| [`scInit`](#scinit) | 1 / 1 | Engine |
| [`scInventoryAdd`](#scinventoryadd) | 64 / 55 | Inventory |
| [`scInventoryReaction`](#scinventoryreaction) | 78 / 69 | Inventory |
| [`scInventoryRemove`](#scinventoryremove) | 33 / 30 | Inventory |
| [`scInventorySet`](#scinventoryset) | 69 / 61 | Inventory |
| [`scJournalAdd`](#scjournaladd) | 80 / — (DC only) | Journal |
| [`scJournalClear`](#scjournalclear) | 1 / — (DC only) | Journal |
| [`scJournalSet`](#scjournalset) | 1 / — (DC only) | Journal |
| [`scJournalShow`](#scjournalshow) | *Not used by Scratches* | Journal |
| [`scLoadSettings`](#scloadsettings) | *Not used by Scratches* | Engine |
| [`scLoadSubtitle`](#scloadsubtitle) | 1 / — (DC only) | Text |
| [`scLock`](#sclock--scunlock) | 51 / 45 | Control |
| [`scLookAt`](#sclookat) | 141 / 108 | Control |
| [`scMenuSet`](#scmenuset) | 1 / 1 | Menu |
| [`scMenuSetEx`](#scmenusetex) | 4 / 4 | Menu |
| [`scMenuSetSaveFunc`](#scmenusetsavefunc) | 1 / 1 | Menu |
| [`scMenuShow`](#scmenushow) | *Not used by Scratches* | Menu |
| [`scMotion`](#scmotion--scstill) | *Not used by Scratches* | Control |
| [`scMusicFadeIn`](#scmusic) | *Not used by Scratches* | Music |
| [`scMusicFadeOut`](#scmusic) | *Not used by Scratches* | Music |
| [`scMusicPlay`](#scmusic) | *Not used by Scratches* | Music |
| [`scMusicStop`](#scmusic) | *Not used by Scratches* | Music |
| [`scOnClick`](#sconclick) | 2297 / 2032 | World |
| [`scOnEnter`](#sconenter) | *Not used by Scratches* | World |
| [`scOnExit`](#sconexit) | *Not used by Scratches* | World |
| [`scOnHover`](#sconhover) | 2298 / 2033 | World |
| [`scOverlay`](#scoverlay) | *Not used by Scratches* | World |
| [`scPlace`](#scplace) | 2298 / 2033 | World |
| [`scPlaylistClear`](#scplaylistclear) | *Not used by Scratches* | Music |
| [`scPlaylistFirst`](#scplaylistnext--previous--first--last) | *Not used by Scratches* | Music |
| [`scPlaylistLast`](#scplaylistnext--previous--first--last) | *Not used by Scratches* | Music |
| [`scPlaylistNext`](#scplaylistnext--previous--first--last) | *Not used by Scratches* | Music |
| [`scPlaylistPrevious`](#scplaylistnext--previous--first--last) | *Not used by Scratches* | Music |
| [`scPlaylistQueue`](#scplaylistqueue) | *Not used by Scratches* | Music |
| [`scPlaylistStart`](#scplayliststart) | *Not used by Scratches* | Music |
| [`scPlaylistStop`](#scplayliststop) | *Not used by Scratches* | Music |
| [`scReaction`](#screaction) | 188 / 180 | World |
| [`scRun`](#scrun) | 1 / 1 | Engine |
| [`scScriptAdd`](#scscriptadd) | 395 / 395 | Dialogue |
| [`scScriptGo`](#scscriptgo) | 59 / 53 | Dialogue |
| [`scSet`](#scset) | 7 / 4 | Engine |
| [`scSetAmbience`](#scsetambience) | 17 / — (DC only) | Engine |
| [`scSetLastVisited`](#scsetlastvisited) | 36 / 30 | World |
| [`scSleep`](#scsleep) | 98 / 87 | Control |
| [`scStill`](#scmotion--scstill) | *Not used by Scratches* | Control |
| [`scSwitch`](#scswitch) | 409 / 339 | Control |
| [`scTerminate`](#scterminate) | 1 / 1 | Engine |
| [`scTexture`](#sctexture) | 1659 / 1282 | World |
| [`scTextureUpdate`](#sctextureupdate) | 466 / 455 | World |
| [`scTimerCreate`](#sctimercreate) | 3 / 3 | Timers |
| [`scTimerCreateGlobal`](#sctimercreateglobal) | 27 / 26 | Timers |
| [`scTrackFadeOut`](#sctrackfadeout) | 8 / 7 | Music |
| [`scTrackPlay`](#sctrackplay) | 35 / 29 | Music |
| [`scTrackSet`](#sctrackset) | 10 / 5 | Music |
| [`scTrackSet3D`](#sctrackset3d) | 28 / 27 | Music |
| [`scTrackSetEx`](#sctracksetex) | 108 / 90 | Music |
| [`scTrackStop`](#sctrackstop) | 11 / 9 | Music |
| [`scUnloadSubtitle`](#scunloadsubtitle) | 1 / — (DC only) | Text |
| [`scUnlock`](#sclock--scunlock) | 45 / 43 | Control |
| [`scWait`](#scwait--scgo) | 1 / — (DC only) | Control |

---

## Engine and settings

### scInit

`scInit()` · Usage 1 / 1

Starts the engine: applies the configured video mode, opens the resource archive
named by `SC_RES_PATH`/`SC_RES_FILE` and sets up rendering and audio. The boot sequence
is `scSet` (resource path and file) → `scInit()` → cursors, fonts, menu and hints → every
`create*()` world builder → `scRun()` → `scTerminate()`.

### scRun

`scRun()` · Usage 1 / 1

The main loop: the intro (`scMenuSetEx(SC_INTRO)`), then the menu and the game. It
returns when the player quits.

### scTerminate

`scTerminate()` · Usage 1 / 1

Called once `scRun` has returned. The port shuts down on its own, so this does nothing.

### scAbort

`scAbort()` · Usage 3 / 1

Ends the current game: stops the playlist, waits for a key or click and returns to the
main menu. The scripts call it at the endings.

### scAttach

`scAttach(...)` · *Not used by Scratches*

A no-op in `scream.exe` too: it reads its arguments and calls an empty function
(`0x414790`).

### scSet

`scSet(key, value)` · Usage 7 / 4

Sets an engine setting to a number or a string. The shipped scripts set `SC_RES_PATH`,
`SC_RES_FILE`, `SC_SPLASH`, `SC_TXT_SPEED` and `SC_MOV_SPEED`. For `SC_TXT_SPEED` and
`SC_CAM_SPEED`, the values `SC_LO`/`SC_MED`/`SC_HI` stand for 25/50/75, and any other
value is used as given. `SC_MOV_SPEED` is the crossfade speed (default 25). The intros
set it to 2 for slow fades of about 8 seconds.

### scGet

`scGet(key)` → string or number · Usage 43 / 42

| Key | Returns |
|---|---|
| `SC_ROOM`, `SC_NODE` | name of the current room or node |
| `SC_LANGUAGE` | the configured language (`"en"`) |
| `SC_TXT_SPEED`, `SC_MOV_SPEED`, `SC_HINTS` | the setting's number |
| anything else | 0 in the DC engine; nothing (`nil`) in the 2006 engine |

### scEnable / scDisable

`scEnable(key)`, `scDisable(key)` · Usage 102 / 99 and 229 / 218

With `SC_OBJECT`, these show or hide the **bound object** (only that one, even if other
objects share its name). With no object bound, `scream.exe` stops with a fatal error.
The port logs a warning instead.

Any other key is an effect flag:

| Key | Effect |
|---|---|
| `SC_SKY` | sky cube, gradient and cloud layers behind the panorama (on at start) |
| `SC_FOG`, `SC_RAIN` | fog layer; rain with storm dimming and lightning |
| `SC_NIGHT`, `SC_CANDLE`, `SC_LAMP`, `SC_DREAM` | lighting passes; the inventory won't open in a dream |
| `SC_WAIT` | allows the wait cursor during `scWait`/`scSleep`/smooth looks (on at start) |
| `SC_HINTS`, `SC_SHOW_HOTSPOTS`, `SC_SURROUND`, `SC_FULLSCREEN`, `SC_DEBUG` | set the setting of the same name to 1/0 |
| `SC_LOG` | turns the log file on or off |
| `SC_GLOW` | stored, but never read (also unused in `scream.exe`) |

### scSetAmbience

`scSetAmbience(target, preset, now)` · Usage 17 / — (**DC only**)

Picks a colour preset (`SC_DAWN`, `SC_MORNING`, `SC_DUSK`, `SC_STORM`) for
`SC_SKY_COLOR`, `SC_CLOUD_COLOR` or `SC_SCREEN_COLOR` (the screen tint). With `now = 1`
the colour changes at once. With 0 it fades slowly, and only on frames that draw the sky.

### scLoadSettings

`scLoadSettings(file)` · *Not used by Scratches*

Runs the `scream.cfg` parser on `file`. The format is whitespace-separated words: a
word that matches a key (case-sensitive) takes the next word as its value. Keys
include `SCRIPT`, `RESOURCE`, `PATH`, `WIDTH`, `HEIGHT`, `DEPTH`, `QUALITY`, `FULLSCREEN`,
`SURROUND`, `LOG`, `VSYNC`, `LANGUAGE` and `DEBUG`. The video-mode keys only take effect
when they are read before `scInit`. The engine reads `scream.cfg` itself at startup.

---

## Building the world

Worlds are declared up front, nested like this:

```lua
scCreate(SC_ROOM, "garden")  scBegin(SC_ROOM)
  scCreate(SC_NODE, "n1")    scBegin(SC_NODE)
    scTexture(SC_FRONT, "garden_n1f.jpg")  -- … one per face
    scFootstep("fx-foot_grass1.ogg")
    scCreate(SC_OBJECT, "to_n2")
      scPlace(400, 300, 200, 400, SC_FRONT)
      scOnHover(SC_FORWARD)
      scOnClick(SC_NODE, "n2")
    scCreate(SC_ZOOM, "mailbox")  scBegin(SC_ZOOM)
      scTexture(SC_ZOOM, "mailbox.jpg")
      scAudio(SC_ZOOM, "fx-mailbox_open.ogg")
      -- objects created here belong to the zoom
    scEnd(SC_ZOOM)
  scEnd(SC_NODE)
scEnd(SC_ROOM)
```

### scCreate

`scCreate(kind, name)` · Usage 3104 / 2733

Creates an entity and binds it. Nodes and zooms belong to the bound room, and objects
to the bound view (the zoom inside a zoom block, otherwise the node). Items, scripts,
choices, tracks and overlays are global. Object names may repeat: a hotspot that
crosses a cube edge is split into same-named objects, one per face.

### scBegin / scEnd

`scBegin(kind)`, `scEnd(kind)` · Usage 647 / 557 each

Bracket a room, node or zoom's contents. `scEnd(SC_ZOOM)` returns object creation to
the enclosing node, and `scEnd(SC_NODE)` leaves no view bound.

### scBind

`scBind(kind, name)` · Usage 549 / 512

Binds an existing entity, so later calls act on it:

- Nodes are looked up in the bound room.
- Zooms are looked up by name across all rooms, and the first one created wins.
- Objects are looked up in the bound node (node state) or the bound zoom (zoom state),
  and the first one of that name is bound. In the inventory, journal or other states,
  nothing is found.
- Items, scripts, choices, tracks and overlays are looked up by name.

When `scBind` fails, it leaves nothing bound. Using that empty slot is fatal in
`scream.exe`, so the port clears the slot and logs a warning.

### scTexture

`scTexture(face, file)` · Usage 1659 / 1282

Sets the image for one face of the bound node (`SC_FRONT`, `SC_BACK`, `SC_LEFT`,
`SC_RIGHT`, `SC_UP`, `SC_DOWN`), or with `SC_ZOOM`, the bound zoom's image. A
`<name>_msk.tga` next to a face image is its sky mask.

### scTextureUpdate

`scTextureUpdate(face, file, visible)` · Usage 466 / 455

Changes a texture during play (for example, a light switched on). `visible` is 1 when
the bound node is the one on screen. Textures load lazily, so the change shows on the
next frame either way.

### scPlace

`scPlace(x, y, w, h, face)` · Usage 2298 / 2033

Places the bound object as a hotspot rectangle. On a node, the coordinates are in a
logical 1024×1024 face, whatever the texture size. Rectangles that run past an edge are
clipped to the face. Inside a zoom, the coordinates are in a logical 800×472 space
covering the image, and `face` is ignored.

### scOnHover

`scOnHover(cursor)` · Usage 2298 / 2033

The cursor shown over the bound object: `SC_LOOK`, `SC_USE`, `SC_FORWARD`,
`SC_ZOOMOUT` (and the other `scCursor` keys).

### scOnClick

`scOnClick(kind, arg[, arg2])` · Usage 2297 / 2032

The [action](#actions) a click on the bound object performs. While a handler runs, the
clicked object, its view and its room are bound, so a bare `scDisable(SC_OBJECT)`
hides the clicked hotspot.

### scReaction

`scReaction(item, kind, arg[, arg2])` · Usage 188 / 180

Adds a reaction to the bound object: the [action](#actions) performed when the player
uses inventory item `item` on it. A held item over an object with a feedback or custom
click brightens, whether or not a reaction exists.

### scFootstep

`scFootstep(file)` · Usage 201 / 142

The sound played (channel `SC_STEP`) when the player walks to the bound node.

### scAudio

`scAudio(when, file)` · Usage 301 / 299

A sound for the bound zoom: `SC_ZOOM` plays on entering it, `SC_ZOOMOUT` on leaving it.

### scSetLastVisited

`scSetLastVisited(node)` · Usage 36 / 30

Rooms remember the last node the player stood on and return there when entered. This
overrides that memory for the bound room.

### scOnEnter

`scOnEnter(handler)` · *Not used by Scratches*

A handler for the bound room, run after `scSwitch(SC_ROOM)` enters it (and after loading
a saved game into it).

### scOnExit

`scOnExit(handler)` · *Not used by Scratches*

Stores an exit handler on the bound room. `scream.exe` never calls it, and neither does
the port.

### scOverlay

`scOverlay(image, enterSound, leaveSound)` · *Not used by Scratches*

Configures the bound overlay (made with `scCreate(SC_OVERLAY, name)`). An overlay shown
with `scSwitch(SC_OVERLAY, name)` draws `image` over the view and plays `enterSound`. A
click or Esc dismisses it with a crossfade and `leaveSound`.

---

## Movement, camera and control

### scSwitch

`scSwitch(kind, name)` · Usage 409 / 339

Moves the player and binds the target, with a crossfade:

| Kind | Effect |
|---|---|
| `SC_ROOM` | enters the room at its last-visited node and runs its `scOnEnter` handler; binds the room and node |
| `SC_NODE` | goes to a node of the current room |
| `SC_ZOOM` | opens a zoom, pushed on the zoom stack |
| `SC_ZOOMOUT` | pops one zoom (`name` is ignored); a parent zoom is bound again |
| `SC_ZOOMBACK` | pops every zoom, back to the node |
| `SC_OVERLAY` | shows an overlay |

Unlike an `SC_NODE`/`SC_ROOM` click, `scSwitch` plays no footstep. Every switch also
clears `scLock`.

### scLookAt

`scLookAt(face, transition)` · Usage 141 / 108

Turns the camera to face a cube side. `SC_INSTANT` jumps there. `SC_SMOOTH` animates
the turn with the camera locked and the wait cursor showing, and **blocks** until it
ends. As compiled in `scream.exe`, a smooth turn to `SC_DOWN` ends level, facing front.

### scSleep

`scSleep(seconds)` · Usage 98 / 87

**Blocks** for `seconds`, with the wait cursor. The DC engine also holds the camera
still, and Esc ends the sleep and stops every playing sound channel (this is how the logo
sequences are skipped). The 2006 engine leaves the camera free and has no Esc skip.

### scLock / scUnlock

`scLock()`, `scUnlock()` · Usage 51 / 45 and 45 / 43

Blocks or allows player input. It is a plain boolean, not a counter. Every view
transition (switches, journal or menu changes, camera keys, overlay dismissal) also
clears the lock, and the scripts rely on that: many `scLock()` branches end with
`scSwitch(SC_ZOOMOUT, "")` and no `scUnlock`.

### scWait / scGo

`scWait()`, `scGo()` · Usage 1 / — each (**DC only**)

Enter and leave wait mode: while input is locked, the wait cursor is drawn. This only
works while `SC_WAIT` is enabled.

### scMotion / scStill

`scMotion()`, `scStill()` · *Not used by Scratches*

Allow or freeze camera movement (mouse look and the edge zones). `scSleep` freezes the
camera itself and always allows motion again when it ends.

---

## Presentation

### scCursor

`scCursor(kind, image)` · Usage 11 / 11

Sets the image of one cursor: `SC_NORMAL`, `SC_FORWARD`, `SC_LOOK`, `SC_USE`, `SC_WAIT`,
`SC_ZOOMOUT`, `SC_MENU`, and the edge arrows `SC_LEFT`, `SC_RIGHT`, `SC_UP`, `SC_DOWN`.

### scFont

`scFont(atlas)` · Usage 3 / 1

Selects the font atlas for feedback, dialogue and menu text (a 512×512 RGBA TGA,
16×16 cells in Windows-1252 order, drawn monospace). The DC switches fonts between the
main game, *The Last Visit* and the menu.

### scCutscene

`scCutscene(video)` · Usage 145 / 114

Plays a Theora video and **blocks** until it ends. A click, Esc or Space skips it.
Videos are silent, so scripts start their sound with `scFXPlay` beforehand.

---

## Text

### scFeedback

`scFeedback(text)` · Usage 414 / 382

Shows text centred above the bottom bar, wrapped onto up to four lines. Text of 400 or
more characters (256 in the 2006 engine) is ignored. `|` is a non-breaking space. The
text stays for `100/SC_TXT_SPEED + len/16` seconds (texts of up to 16 characters
count as 32) and then fades out.
`scFeedback("")` clears it.

### scFeedbackEx

`scFeedbackEx(text, voice)` · Usage 34 / — (**DC only**)

`scFeedback`, plus a voice file played on the dialogue channel.

### scLoadSubtitle

`scLoadSubtitle(file)` · Usage 1 / — (**DC only**)

Loads an `.srt` file and starts showing its cues as feedback text, timed in whole
seconds from the moment of loading. Subtitles keep running during cutscenes, which is how
the DC intro is subtitled.

### scUnloadSubtitle

`scUnloadSubtitle()` · Usage 1 / — (**DC only**)

Stops the subtitles.

---

## Dialogue and choices

### scScriptAdd

`scScriptAdd(voice, text, hint)` · Usage 395 / 395

Appends a line to the bound script (made with `scCreate(SC_SCRIPT, name)`). Lines with
`hint = 1` are hints, played only while `SC_HINTS` is on.

### scScriptGo

`scScriptGo(name)` · Usage 59 / 53

Plays a script and **blocks** until it ends. Music is turned down by 0.25 for the whole
dialogue. Each line plays its voice (channel `SC_DIALOGUE`) and shows its text, and
the next line starts when the voice ends. Esc ends the current line.

### scChoiceAdd

`scChoiceAdd(text, handler)` · Usage 24 / 20

Adds an option to the bound choice (made with `scCreate(SC_CHOICE, name)`). A choice
holds at most 16 options.

### scChoiceGo

`scChoiceGo(name)` · Usage 13 / 11

Shows the choice's options at the bottom left. It does **not** block: scripts usually
call `scUnlock()` right after. Picking an option runs its handler as a new handler.

### scChoiceClear

`scChoiceClear()` · *Not used by Scratches*

Removes every option from the bound choice.

### scChoiceRemove

`scChoiceRemove()` · *Not used by Scratches*

Removes the option the player picked last from the bound choice, so questions already
asked disappear from the list.

---

## Inventory and hints

### scInventorySet

`scInventorySet(icon, label, failText[, failVoice])` · Usage 69 / 61

Describes the bound item. `failText` and `failVoice` play when the held item is used on
something with no reaction for it ("That doesn't work.").

### scInventoryAdd

`scInventoryAdd()` · Usage 64 / 55

Puts the bound item in the inventory (once).

### scInventoryRemove

`scInventoryRemove()` · Usage 33 / 30

Takes the bound item out of the inventory, and out of the player's hand if they are
holding it.

### scInventoryReaction

`scInventoryReaction(item, kind, arg[, arg2])` · Usage 78 / 69

Like [`scReaction`](#screaction), for the bound item: the [action](#actions) performed
when held item `item` is used on it in the inventory.

### scHintsSet

`scHintsSet(key, value)` · Usage 4 / 4

Configures the inventory tutorial. `SC_HINTS_OPEN`, `SC_HINTS_GRAB` and
`SC_HINTS_CLOSE` are the texts shown on opening the inventory, grabbing the tutorial
item and closing with it held. `SC_HINTS_ITEM` names that item.

### scHints

`scHints(state)` · Usage 4 / 4

`SC_HINTS_ACTIVE` turns the tutorial on (it shows only while the `SC_HINTS` setting is
also on). `SC_HINTS_INACTIVE`, `SC_HINTS_RESET` and any other value turn it off.

---

## Journal (Director's Cut)

### scJournalSet

`scJournalSet(image, font, openSound, turnSound, closeSound)` · Usage 1 / — (**DC only**)

Attaches the journal to the bound item. Using that item opens the book.

### scJournalAdd

`scJournalAdd(index, text)` · Usage 80 / — (**DC only**)

Appends an entry, once per `index`: adding the same index again does nothing. Lines
wrap at about 30 characters, and `|` is a non-breaking space.

### scJournalClear

`scJournalClear()` · Usage 1 / — (**DC only**)

Clears the journal's text. Indices already added stay used, so those entries can't be
added again.

### scJournalShow

`scJournalShow()` · *Not used by Scratches*

Opens the journal from script.

---

## Menu

### scMenuSet

`scMenuSet(logo, back, new, setup, load, save, quit, newGame1[, newGame2…])` · Usage 1 / 1

The main menu's seven images, then the name of one handler per game that "new game" can
start. With two handlers (the DC's `newGameSDC` and `newGameTLV`), the player is asked
to pick *Director's Cut* or *The Last Visit*.

### scMenuSetEx

`scMenuSetEx(key, value)` · Usage 4 / 4

| Key | Value |
|---|---|
| `SC_INTRO` | handler run before the menu first shows (logos) |
| `SC_OUTRO` | handler run on quitting (credits) |
| `SC_STEREO` | menu music file |
| `SC_IMAGE` | menu background image |

### scMenuSetSaveFunc

`scMenuSetSaveFunc()` · Usage 1 / 1

Turns saving on. The script's global `save()` returns the description for the save
slot. The engine itself writes every numeric global and its own state to the save.

### scMenuShow

`scMenuShow()` · *Not used by Scratches*

Opens the main menu from script, using the same routine as `scAbort` and Esc.

---

## Timers

Timers tick once a second and fire their handler once, on the Nth tick, then stay
inactive. Re-creating an id re-arms that timer. (Quirk kept from `scream.exe`: re-using
the id of the most recently created timer adds a duplicate instead, and id 0 never
runs.) Nothing cancels a timer. Up to 31 timers of each kind exist, and saved games keep
them all.

### scTimerCreate

`scTimerCreate(id, seconds, handler)` · Usage 3 / 3

A timer in the **bound room**. It counts only while the player is in that room (in any
state), so it pauses while they are elsewhere.

### scTimerCreateGlobal

`scTimerCreateGlobal(id, seconds, handler)` · Usage 27 / 26

A world timer. It counts only while the player is in node view, not in zooms, menus or
the inventory.

---

## Sound effects

Channels: `SC_STEP` 8 (footsteps), `SC_3DA`–`SC_3DD` 9–12 (ambient loops),
`SC_DIALOGUE` 13 (voices), `SC_FX` 14. Volumes run from 0 to 10. A fade value F is a
rate, meaning F seconds from full volume (in `scream.exe` this varied from PC to PC).

### scFXPlay

`scFXPlay(file)` · Usage 369 / 313

Plays a one-shot sound on no particular channel. Two calls in the shipped script pass a
channel first, in the `scFXPlayEx` form, so the port accepts that form too.

### scFXPlayEx

`scFXPlayEx(channel, file, wait, loop)` · Usage 275 / 249

Plays a file on a channel, replacing what it was playing, at that channel's volume.
`loop = 1` repeats it. `wait = 1` **blocks** until it ends, and Esc stops it.

### scFXSet

`scFXSet(channel, what, value)` · Usage 134 / 117

`SC_VOLUME` sets the channel's volume (0–10). `SC_FADEIN` fades it up from silence and
`SC_FADEOUT` down at rate `value`. A fade-out doesn't stop the sound.

### scFXStop

`scFXStop(channel)` · Usage 44 / 40

Stops a channel.

---

## Music: tracks

A track plays on channels 1–3: a front file, plus rear-left and rear-right files when
surround is on (with stereo output, all three are mixed). Starting a track always stops
the previous one. While the [playlist](#music-playlist) is playing, `scTrackPlay`,
`scTrackStop` and `scTrackFadeOut` are ignored.

### scTrackSet

`scTrackSet(file, loop)` · Usage 10 / 5

Gives the bound track (made with `scCreate(SC_TRACK, name)`) a single file.
`loop = 1` repeats it. In a playlist, `loop` is the number of plays instead (0 = for
ever).

### scTrackSet3D

`scTrackSet3D(front, rearLeft, rearRight, loop)` · Usage 28 / 27

The surround form of `scTrackSet`.

### scTrackSetEx

`scTrackSetEx(what, value)` · Usage 108 / 90

The bound track's `SC_VOLUME` (0–10), `SC_FADEIN` and `SC_FADEOUT` rates.

### scTrackPlay

`scTrackPlay(name)` · Usage 35 / 29

Starts a track from the beginning at its volume.

### scTrackStop

`scTrackStop()` · Usage 11 / 9

Stops the current track at once.

### scTrackFadeOut

`scTrackFadeOut(n)` · Usage 8 / 7

Fades the current track at rate `n`, using rate 1 when fewer than `n` seconds remain.
It doesn't stop the track: the scripts follow it with `scSleep(n)` and `scTrackStop()`.
With a negative `n`, the `SC_FADEOUT` of the last queued playlist track is used.

---

## Music: playlist

*Not used by Scratches.* A queue of up to 38 tracks that play in turn and crossfade
between two banks of channels (1–3 and 4–6). Each track plays `loop` times (0 = for
ever). `SC_FADEOUT` seconds before the end of its last play, the next track starts and
the old one fades out. `SC_FADEIN` applies only to a track's first play. The volume
drops by 0.25 during dialogue.

### scPlaylistClear

`scPlaylistClear()` · *Not used by Scratches*

Empties the queue.

### scPlaylistQueue

`scPlaylistQueue(track)` · *Not used by Scratches*

Appends a track (by name) to the queue.

### scPlaylistStart

`scPlaylistStart()` · *Not used by Scratches*

Starts the queue: from the first track the first time, otherwise from the current one.

### scPlaylistStop

`scPlaylistStop(n)` · *Not used by Scratches*

Fades out at rate `n` (rate 1 when fewer than `n` seconds remain) and cuts the track
after `n` seconds. With `nil` or 0, the current track plays out and the playlist stops.

### scPlaylistNext / Previous / First / Last

`scPlaylistNext()`, `scPlaylistPrevious()`, `scPlaylistFirst()`, `scPlaylistLast()` ·
*Not used by Scratches*

Jump to another queued track. The new track starts without fading out the old one, so
the two overlap.

---

## Music: legacy

### scMusic*

`scMusicPlay`, `scMusicStop`, `scMusicFadeIn`, `scMusicFadeOut` · *Not used by Scratches*

No-ops: in `scream.exe` these functions just return. Use tracks or the playlist instead.
