# 02. The OS version: Portfolio 20.21

3dokit's Portfolio runtime has met two OSes so far: Crash 'n Burn's of
1993 (the one its code follows, address by address) and Immercenary's
23.10 (where it differs, the runtime took 23.10's way when no 1993 program
could tell, or tested `pf_os_release() >= 23`). Doctor Hauzer's disc
carries a third, from between them. This is what session 1 read of it, and
what it means for the runtime.

## The release, read

The runtime takes the OS release from `System/Kernel/os_code`: the AIF
after the 16-byte boot header, its 3DO header at +0x80, the version byte
at +0x14 -- file offset 0xa4 (`3dokit/runtime/pf_file.cpp`,
`pf_os_release`).

| disc | os_code 0x90..0xa5 | kernel's 3DO header | ROM tag 0x07 |
|---|---|---|---|
| Crash 'n Burn (1993-09) | all zero | none | v0.16 |
| **Doctor Hauzer (1994-04)** | node 0x0104, **0x14 0x15** | folio, **v20.21** | **v0.16** |
| Immercenary (1995) | node 0x0104, 0x17 0x0a | folio v23.10, "Sherry" | v23.10 |

So the release reads **20** here, and the reading is right: the kernel's
own header says folio 20.21. The ROM tag still says 0.16, the 1993 value:
the tag is not the release on this disc.

**`os_code` is three images** (`01-the-disc-on-the-kit.md`), as on the
other two discs:

| | Crash 'n Burn | Doctor Hauzer | Immercenary |
|---|---|---|---|
| kernel | at 0x10000, NOP at 0x04, no header | **at 0x10000, NOP at 0x04**, v20.21 | linked at 0, relocates itself, v23.10 |
| Operator | at 0x20000, v20.15 | **at 0x20000**, v20.18 | at 0, v23.10 |
| File folio | at 0, v20.19 | at 0, v20.30 | at 0, v23.10 ("FileSystem") |

In its kernel's form the 1994 OS is the 1993 one: linked at fixed
addresses, not relocated. Every address the runtime cites in the 1993
kernel will want reading again in this one (they moved: the kernel is
47,904 bytes unpacked against 49,968).

## Each image has its own version

The System images carry their own versions, and they are not the
kernel's:

| | Crash 'n Burn | Doctor Hauzer | Immercenary |
|---|---|---|---|
| kernel | (none) | 20.21 | 23.10 |
| GRAPHIX | 20.31 | **20.45** | 23.10 |
| AUDIOFOLIO | 20.19 | 20.27 | 23.10 |
| OPERAMATH | 20.27 | 20.53 | 23.10 |
| eventbroker | 20.21 | 20.31 | 23.10 |
| shell | 20.26 | 20.33 | 23.10 |
| `run`, `LAUNCH` | 20.18 | 20.29 | 23.10 |

(`aif --scan`; the 1993 numbers are the same series, so "20" alone does
not tell 1993 from 1994.) **A folio's own version is what says which way
it went**, and that is how the kit now asks (`pf_system_version`, below).

## GRAPHIX 20.45, read against 20.31 and 23.10

The three folios unpacked by their own decompressor (`aif --decompress`),
their tables found where the runs of words pointing into the code lie --
the SWI table (SWI 0 last) and the vector table right after it (slot -4
last) -- and laid side by side:

| | 1993 (20.31) | 1994 (20.45) | 23.10 |
|---|---|---|---|
| SWIs | 51, from 0x52a0 | **52**, from 0x6504 | 52, from 0x7490 |
| vector slots | 49 (-4 to -196) | **49** (-4 to -196) | 51 (-4 to -204) |
| the node database | 0x5430 | 0x6698 | 0x762c |

* **Its SWIs**: SWI 51 at the table's head (0x4d0c; 23.10's 0x53e8), then
  SWI 50, `CreateScreenGroup`'s supervisor half, at 0x2d94 (1993's 0x27a0,
  23.10's 0x2edc).
* **Its vectors**: as many as 1993's, but `DeleteScreenGroup` (-92,
  0x4504), `GetFirstDisplayInfo` (-184, 0x1604) and `ModifyVDL` (-188,
  0x562c) are filled in, as in 23.10, where 1993 has the folio's "not
  implemented" stub; `QueryGraphics` and `QueryGraphicsList` (-192, -196)
  are still the stub, as in 1993 (23.10 has them, and two slots more).
* **Its node database** (after the vector table, 4 bytes a type): ScreenGroup
  0x74, Screen 0x7c, Bitmap 0x84, VDL 0x44 and a fifth type of 0x1c --
  23.10's sizes; 1993's are 0x54, 0x7c, 0x84, 0x34 and no fifth. The
  runtime made every node at 1993's size on every disc; it now makes each
  at its folio's size by GRAPHIX's version (`pf_graphics.cpp`,
  `node_size`), cleared, the 1993 fields filled. No game had stopped on
  it: the gain is `n_Size` and the OS's memory as the folio lays it out.
* **AUDIOFOLIO's node database** the same way (20.19 at 0xc04c, 20.27 at
  0xbf94, 23.10 at 0xb9e8): 20.27 makes an instrument of 0x64 bytes and
  type 6 of 0x78 (1993: 0x58, 0x74); 23.10 also a template of 0x70 and a
  sample of 0x9c (0x54, 0x98). The runtime follows AUDIOFOLIO's version
  (`pf_audio.cpp`, `node_size`).
* **`CreateScreenGroup`** (-48, 0x4104 here): the user half is 23.10's
  (0x4320 there) instruction for instruction: it takes twelve tags, and
  when the caller gives no buffers it allocates them and a table of them,
  **gives the table back** once the group is made (0x44c4..0x44f4), and its
  supervisor half **marks each buffer the folio's own** (`bm_SysMalloc`,
  0x351c; 23.10's 0x3684), which `DeleteScreenGroup` gives back. The 1993
  folio does neither. The runtime chose by `pf_os_release() >= 23`, which
  sent this disc the 1993 way; it now asks GRAPHIX's own version,
  `>= 20.45` (the versions between 20.31 and 20.45 are unread). With it,
  the game's first `CreateScreenGroup` gives its 16-byte table back and
  every later allocation of the program moves down by 0x10, as on the
  console.
* **The font** (`ResetCurrentFont` -44, `GetCurrentFont` -144,
  `SetCurrentFontCCB` -140, `DrawChar` -128, `DrawText8` -148,
  `DrawText16` -36): all three folios have the same machinery -- a font of
  49 characters built into the folio's own data, made into a tree of
  `FontEntry` nodes when the folio starts (here 0x407c, its table at
  0x7ef8; the loop's `cmp r5, #0x31` at 1993's 0x3e04, this one's 0x40c0,
  23.10's 0x42dc), `ResetCurrentFont` = SWI 14 then, if the default font
  was built, SWI 27 on it. No game on the kit had called any of it; this
  one does at its 556th call (`TODO.md`).

## The runtime's other 1993/23.10 places

Only `CreateScreenGroup` tested the release. Elsewhere the runtime took
23.10's way unconditionally where a 1993 program could not tell the
difference. Each is a place to read on 20.21's own code **when the game
reaches it**, not before:

| where | the runtime does | 1993 | to read here |
|---|---|---|---|
| `pf_os.cpp`, `arm_swi` | a folio-0 SWI from 0x100 up is kernel SWI (n - 0x100) & 0xff | sent elsewhere (0x3d8) | the 20.21 kernel's dispatcher |
| `pf_task.cpp`, `kTaskExit` | a task with an image returns to 0x410 | (no such tasks) | where 20.21's CreateTask points lr; this kernel is at 0x10000 |
| `pf_task.cpp`, CreateThread | tag 24 `_ALLOCDTHREADSP` accepted | BADTAG | 20.21's tag callback |
| `pf_task.cpp`, Kernel -148 | the C library's exit | -- | whether 20.21 has the slot |
| `pf_audio.cpp`, `same_name` | names without case, to the end | strncmp, 32 characters | AUDIOFOLIO 20.27 |
| `pf_audio.cpp`, `MakeSample`, `ScanSample` | 1993's and 23.10's orders noted | | AUDIOFOLIO 20.27 |
| `pf_file.cpp`, `LoadCode` | 23.10's File folio | -- | File folio 20.30 (the third image of os_code) |

## The kit change

`pf_system_version(path)`: a System image's version from its 3DO header,
`PF_VERSION(20, 45)`, 0 when the disc has no such file; `pf_os_release()`
stays for the kernel. `CreateScreenGroup` now tests
`pf_system_version("/System/Folios/GRAPHIX") >= PF_VERSION(20, 45)`:
Crash 'n Burn's 20.31 and Immercenary's 23.10 go the way they went before
(their traces, frames, `pfcheck` runs unchanged: `10-3dokit.md`). The
Graphics and audio folios' node sizes follow their own versions the same
way.
