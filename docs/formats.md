# File formats

This page describes every file the SCream engine reads or writes, as the original
`scream.exe` defines it and as this port implements it. Where the 2006 engine differs from
the Director's Cut (DC), the difference is noted. Anything the port adds is marked
**(port)**. The function reference is in [`api.md`](api.md). The reverse-engineering
detail is in [`NOTES.md`](../NOTES.md), and addresses like `0x411370` point into the DC
update's `scream.exe`.

Unless stated otherwise, all integers are little-endian `i32`/`u32`, floats are IEEE `f32`,
and strings are Windows-1252. A `char[n]` field is NUL-padded, and a string that fills it
leaves no room for the NUL.

## Contents of an install

| File | Format | Role |
|---|---|---|
| `scream.cfg` | [text](#screamcfg) | engine settings, read at startup |
| `scream.sc` | [Lua bytecode](#scripts) | engine constants (`SC_*`, `SC_DEF_*`), run first |
| `scratches.sc` | [Lua bytecode](#scripts) | the game script (named by `SCRIPT` in `scream.cfg`) |
| `scream.res` | [archive](#res-archives) | engine resources |
| `scratches.res` | [archive](#res-archives) | game resources (named by `RESOURCE` / `SC_RES_FILE`) |
| `update.res` | [archive](#res-archives) | DC update: HD textures and new files, laid over `scratches.res` |
| `sdc-intro.srt` | [SubRip](#srt-subtitles) | DC only: the intro's subtitles |
| `scream.001`–`scream.011` | [binary](#saved-games-scream0nn) | saved games, written by the engine |
| `scream.log` | text | log, when `LOG` is on |

`tools/extract_game.py` collects exactly these input files from the original discs.

**Lookup.** Two kinds of file are looked up in different places:

- **Loose files** sit in the data directories: `scream.cfg`, the scripts, the archives
  and the `.srt`.
- **Assets** (images, sounds, videos, fonts) are looked up by name in the mounted
  archives. The newest mount wins: `update.res` is searched first, then the game archive,
  then `scream.res`.

The original installs the DC update by merging `update.res` into `scratches.res` by file
name (`scupdate.exe`). The port mounts `update.res` as an overlay instead, which gives the
same result. **(port)**: saved games and `scream.log` go to the save folder, not the game
folder. The save folder is SDL's preference path for `SCreamRedux/Scratches`, or
`--save-dir`.

---

## `.res` archives

A flat, uncompressed file store. `tools/scream_res.py` lists and extracts archives.
Nothing in this repository writes them yet.

```
offset  size        field
0x000   char[256]   magic "SCream resource file\0", then padding (uninitialized in the shipped files)
0x100   u32         entry count N
0x104   entry[N]    92 bytes each:
          +0   char[80]  name     NUL-terminated; bytes after the NUL are garbage
          +80  u32       type     1 = JPEG, 2 = TGA, 3 = Ogg (Vorbis or Theora)
          +84  u32       offset   absolute, from the start of the file
          +88  u32       size     bytes
```

- Names are flat file names with no directories, at most 79 characters. The port matches
  them exactly.
- The data is stored raw. Each entry is one complete `.jpg`, `.tga` or `.ogg` file.
- The type field is informational: the port picks a decoder from the file's content and
  never reads the type.
- Shipped sizes: 2006 `scratches.res` has 2829 entries. DC `scratches.res` has 3351
  (3350 in the single-disc release). `update.res` has 1623, of which 1575 replace existing
  files and 48 are new.

---

## Assets

### Images

Images are baseline JPEG or TGA (uncompressed or RLE: 8-bit grey, 24-bit or 32-bit
RGBA). The port decodes them with stb_image.

| Use | Name pattern | Format |
|---|---|---|
| Node faces | `<area>-<room>_n<N><f>.jpg`, `f` ∈ `f b l r u d` | JPEG, square: 1024² (2006, DC base) or 1280² (DC update) |
| Face variants | e.g. `_n1l1` … `_n1l12`, `_n3fs` | JPEG, alternate states of one face, swapped in with `scTextureUpdate` |
| Zooms | free | JPEG, 800×472, or 1024×640 in the DC update; drawn 800 wide at the image's own aspect |
| Masks | `<image>_msk.tga` | 8-bit grey TGA (RLE), typically 512² |
| Cursors | `cur-*.tga` | RGBA TGA, drawn as a 64-pixel quad |
| Inventory icons | `inv-*.tga` | RGBA TGA, `SC_INV_SIZE` (64) |
| Fonts | `font-*.tga` | see [Fonts](#fonts) |
| Menu | `menu-*.tga` | RGBA TGA. The logo is a 512² texture and the others 128², each quad stretching the whole texture (buttons are 112×60 images) |

The naming conventions are only conventions: scripts name every file explicitly. The one
name the engine derives itself is the **mask**: the image's name up to the first `.`, plus
`_msk.tga`. The engine looks for a mask for every node face and zoom image. Where a mask
is black, the backdrop (sky, fog or rain, from `scEnable`) shows through. Where it is
white, or where there is no mask, the photo is opaque. The panorama is composited as
`photo × (backdrop + mask²)`.

The engine also uses some files by fixed name: `gradient.tga` (sky gradient),
`rain_def.tga` (rain), the cloud textures, `sc_logo.tga` (the default `SC_SPLASH`) and
`black.jpg`. `black.jpg` is the image of the built-in `"black"` zoom. It is missing from
the archives and renders black.

### Fonts

A font is a 512×512 RGBA TGA: a 16×16 grid of 32-pixel cells, one per byte value in
Windows-1252 order (cell = code; row = code / 16, column = code % 16). There is no width
table. The engine draws every character as a 16×16 quad of its whole cell, with a fixed
10-pixel advance, so all text is monospace. **(port)**: for the inventory label only, the
port also measures each glyph's width from its alpha channel.

### Audio

Audio files are Ogg Vorbis, 44.1 kHz (a few at 48 kHz), mono or stereo.

- **Prefixes:** `fx-` effects, `dlg-`/`msg` voices, `trck-` music.
- **Surround music** is three files, `-F`, `-RL` and `-RR` (front, rear left, rear right).
  `scTrackSet3D` names all three, and they play together when `SURROUND` is on.

### Video

Videos (`vid-*`, `intro-*`, `outro-*`) are Ogg Theora, 4:2:0, 24 fps, at 640×352, 800×472
or 1024×640. They have **no audio stream**: scripts start the sound with `scFXPlay`
alongside `scCutscene`.

`scream --verify-media` decodes every asset. It walks each Ogg file page by page and
checks each page's CRC, because the decoders silently skip damaged pages.

---

## Scripts

`scream.sc` runs first, then the game script: `<SCRIPT>.sc`, or `<SCRIPT>.lua` if there is
no `.sc`. File names match ignoring case, as on Windows (the shipped cfg says
`SCRATCHES`). Each file is a single Lua chunk, run once. The game script builds the world
and then calls `scRun()` (see [api.md](api.md)).

- **DC:** Lua 5.1 bytecode with header little-endian, `int` 4, **`size_t` 4**,
  instruction 4, number 8 (double). Stock 64-bit Lua rejects `size_t = 4`, so the port's
  `lundump.c` is patched to accept it ([third_party](../third_party/)).
- **2006:** Lua 5.0 bytecode. `src/lua50.cpp` translates it to 5.1 at load.
- **Source text:** plain Lua 5.1 source loads too, under either extension. New games can
  ship readable scripts.
- **Contents:** `scream.sc` only assigns the `SC_DEF_*` defaults and `SC_*` constants
  (values in [`src/constants.hpp`](../src/constants.hpp)). Both shipped scripts carry
  debug info and decompile cleanly with unluac.

---

## `scream.cfg`

`scream.cfg` is text, but has no line structure: the parser (`0x40f4a0`) reads it as a
stream of whitespace-separated words.

- A word that exactly equals a key (case-sensitive) takes the next word as its value.
- Every other word is skipped. That covers the shipped `[SCREAM]` header and the comment
  text, which is why they work without any comment syntax.
- Numbers are read with `%d`. If a value is not a number, the key keeps the previous
  number read and the word is read again as a key. A value like `1.5` gives 1 and leaves
  `.5` to be read next.
- Nothing is range-checked, and if a key appears twice, the last value wins.

| Key | Default | Meaning |
|---|---|---|
| `SCRIPT` | `main` | game script, without extension |
| `RESOURCE` | `game.res` | game archive (`SC_RES_FILE`) |
| `PATH` | `.` | resource path; the engine appends `\` (`SC_RES_PATH`) |
| `WIDTH`, `HEIGHT` | 1024, 768 | video mode; every pixel layout in the engine follows `HEIGHT` |
| `DEPTH` | 32 | colour depth |
| `QUALITY` | 3 | only 0 matters: nearest-neighbour filtering everywhere |
| `FULLSCREEN` | 1 | |
| `SURROUND` | 1 | surround music (the shipped files say 0) |
| `LOG` | 0 | write `scream.log` |
| `VSYNC` | 1 | swap interval 1; 0 leaves it alone |
| `RATE`, `AUDIO_DEVICE`, `BUFFER_SIZE`, `NUM_BUFFERS` | 0, 0, 32768, 4 | OpenAL settings; `AUDIO_DEVICE` is used only with `DEBUG` |
| `DEBUG` | 0 | lists audio devices (DC only) |
| `LANGUAGE` | `en` | returned by `scGet(SC_LANGUAGE)` (not read by the 2006 retail exe) |
| `WIDESCREEN` | 0 | **(port)** widen node views to the window; `scream.exe` skips the word |

The DC's `scream.cfg` sets 1024×768. The 2006 one sets 800×600.

**(port)**: The setup launcher edits `scream.cfg` in place and keeps the comments and
layout. On its first save it keeps the shipped file as `scream.cfg.orig`. The engine also
re-reads `scream.cfg` while running and applies what can change live.

---

## `.srt` subtitles

`scLoadSubtitle(file)` reads a **loose file** from the data directory, not from an archive.
The file is SubRip-shaped:

```
1
00:00:02,000 --> 00:00:06,500
First line of the cue
second line

2
…
```

The player (`0x4180b0`) is much simpler than SubRip, and the port keeps its behaviour:

- The clock counts whole seconds since `scLoadSubtitle`, so cue times have one-second
  resolution in practice.
- Each cue is: an index, a start time, `-->`, an end time, the rest of that line, then text
  lines up to a blank line. The text lines are joined with spaces and shown as
  `scFeedback` text from the first tick at or after the start time. The text is cleared at
  the end time, and only then is the next cue read.
- **A cue numbered 13 ends the subtitles.** The player stops there, without showing it.
  `sdc-intro.srt` ends with an empty cue 13. A longer file stops at 13 too.
- Times are read as `HH:MM:SS,mmm` in fields of three characters. Two parsing bugs are
  kept as they are in `scream.exe`:
  - The hour counts as 20 minutes.
  - An end time read after the clock passes 100 seconds weights every field as an hour.
    That cue then stays up, in practice for good, and no later cue is shown.

  To stay clear of both, keep a subtitle file's cues within its first 100 seconds, as
  `sdc-intro.srt` does.
- The file is read in text mode, so CR LF and LF both work.

---

## Saved games: `scream.0NN`

`scream.001`–`scream.010` are the menu's ten slots, and `scream.011` is the F5
quicksave. The port writes `scream.exe`'s own binary format (writer `0x411370`, reader
`0x4107d0`), so the original's saves load in the port and the port's saves load in the
original.

The file is a dump of the engine's tables. Most tables are **1-based, in creation
order**: the order in which the script's `scCreate` calls made the rooms, nodes, zooms,
objects and items. Record 0 is unused, and is written too. A save therefore only fits the
script that built the world, and a script change that adds, removes or reorders creations
breaks old saves.

A table header is the **last index L**, followed by records 0..L (L + 1 records). The
timer and inventory tables differ, and their entries below say how.

Sizes below are DC / 2006. The 2006 layout is that of the **patched** 2006 exe. The
retail 2006 exe's format differs (the patch warns that it invalidates saves) and is not
covered here.

### 1. Description

```
char[80]   description   shown in the menu's slot list; the script's save(), or "quicksave"
```

### 2. Script globals

```
i32        count
count ×:   i32 len, char[len] line     "name=value\0", len includes the NUL
```

The file holds every global whose value is a number or a numeric string. On load, each
line **runs as Lua**. Only numbers survive a save: strings, booleans and tables are not
saved. **(port)**: before loading, the port first restores the start-of-game state, so
globals the file doesn't mention go back to their initial values. The original leaves
them as they were in memory.

### 3. Settings block

700 bytes / 676 bytes: a copy of the engine's settings area (`0x4df8b64`). Offsets for
the DC:

| Offset | Type | Field |
|---|---|---|
| 0x000–0x01c | i32 ×8 | WIDTH, HEIGHT, DEPTH, RATIOX, RATIOY, VIEWX, VIEWY, QUALITY |
| 0x020 | f32 | frame (1.0) |
| 0x024 | i32 | gamma 0–3 (F12) |
| 0x028 | f32 | field of view |
| 0x02c–0x04c | i32 ×9 | LOG, media (1), FULLSCREEN, resource (1), VSYNC, RATE, AUDIO_DEVICE, BUFFER_SIZE, NUM_BUFFERS |
| 0x050 | char[256] | RES_PATH |
| 0x150 | char[80] | RES_FILE |
| 0x1a0–0x1b8 | i32 ×7 | SURROUND, CAM_SPEED, TXT_SPEED, MOV_SPEED, SKY_SPEED, SHOW_HOTSPOTS, TEX_SIZE |
| 0x1bc | i32 ×5 | MSK_SIZE, SKY_SIZE, IMG_SIZE, INV_SIZE, CUR_SIZE |
| 0x1d0 | i32 | menu has been shown |
| 0x1d4 | i32 | HINTS |
| 0x1d8 | char[80] | SPLASH |
| 0x228 | char[80] | SCRIPT |
| 0x278 | i32 | slideshow mode |
| 0x27c | i32 | glow text (never set) |
| 0x280–0x2a0 | i32 ×9 | flags: WAIT, SKY, NIGHT, GLOW, RAIN, FOG, DREAM, CANDLE, LAMP |
| 0x2a4 | i32 | DEBUG |
| 0x2a8 | char[4] | LANGUAGE |
| 0x2ac | i32 | alternate control (`SC_FPS_STYLE`) |
| 0x2b0–0x2b8 | i32 ×3 | ambience presets: sky, cloud, screen colour |

**2006:** the 2006 block is the DC block without IMG_SIZE (+0x1c4) and DEBUG (+0x2a4), so
LANGUAGE sits at +0x2a0, and the block ends there. There is no alternate control and
there are no ambience presets. The original restores the whole block on load. The port
restores the gameplay fields (gamma, FOV, speeds, surround, hints, alternate control,
slideshow, flags, ambience) and keeps the player's video and audio configuration.

### 4. Journal (DC only)

```
i32        last line
i32        first line shown
i32[1000]  added flags, by scJournalAdd index
char[40]   lines[1000]
char[48]   font            current scFont
```

### 5. View and location

```
f32   yaw          radians
f32   look y       DC: the look vector's y (pitch, clamped ±1.75); 2006: the pitch angle
i32   state        4000 node, 0xfa2 zoom, 0xfa3 inventory, 0xfa8 journal
i32   previous state
i32   scBegin kind
i32   current room index
```

### 6. Global timers

```
i32        count C
record ×(C + 1), 68 bytes each, record 0 unused:
  i32 id, i32 seconds, i32 elapsed, i32 active
  char[48] handler
  i32 callback      address of the exe's call-by-name function: 0x404d80 / 0x404c20
```

### 7. Rooms

```
i32        last index L
record ×(L + 1):
  i32  index
  i32  current node index
  i32  entry (last-visited) node index
  i32  zoom-stack depth D          non-zero only for the current room
  i32  zoom stack[377 / 368]       [1..D] = zoom indices, [0] unused
  timer table                      the room's local timers, laid out as in section 6
```

### 8. Nodes

```
i32        last index L
record ×(L + 1):  i32 index, char[48] face[6], char[48] mask[6]
```

Faces are in the order front, back, left, right, up, down. The mask names are derived from
the face names; the port ignores them on load.

### 9. Zooms

```
i32        last index L
record ×(L + 1):  i32 index, char[48] image, char[48] mask
```

Zoom 1 is the engine's built-in `"black"` zoom (`black.jpg`), so the script's zooms start
at index 2.

### 10. Objects

```
i32        last index L
record ×(L + 1):  i32 index, i32 enabled
```

### 11. Items

```
i32        last index L
record ×(L + 1):  i32 index, char[48] icon, char[400 / 280] label, char[400 / 280] fail text,
                  char[48] fail voice (DC only)
```

Records are 900 / 612 bytes.

### 12. Inventory

```
i32        count C
i32        slot[L + 1]     L = the item table's last index; slots 1..C hold item indices, [0] unused
```

### 13. Sound channels

Channels 1–17, 60 bytes each:

```
char[32] file, i32 loaded, i32 playing, f32 volume, f32 current gain,
i32 fade-in, i32 fade-out, i32 loop
```

Channels 1–3 and 4–6 are the music banks, 8 is footsteps, 9–12 are the ambient loops,
13 is dialogue and 14 is effects. On load, every channel that was playing starts again
from the beginning, with its volume and fades, except footsteps and effects.

### 14. Playlist

A raw block of 196 / 172 bytes, the playlist state (`0x4df5b48`). No shipped script uses
the playlist. The port writes zeros, except a play counter of 1 at +0xb8 / +0xa0, and
skips the block on load.

### After loading

The engine enters the saved room, at its entry node, and runs the room's `scOnEnter`.
If the state was a zoom, it pushes the saved zoom stack. Then it restores the camera and
applies the ambience at once. "Game loaded" and "Game saved" show even if the file failed,
as in the original.

### Legacy port saves (port)

The port's first saves were text files named `slotN.sav`, starting with the line
`SCREAMREDUX-SAVE 1`. They hold the description and a line-based dump of what play can
change (`G` globals, `room`/`view`/`face` world state, `inv`, `jrn`, `timer`, …). They
still load, and saving to a slot deletes the slot's `.sav` file. New saves are always
`scream.0NN`.
