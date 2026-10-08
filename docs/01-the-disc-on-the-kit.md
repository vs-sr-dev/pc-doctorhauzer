# 01. The disc on the kit

What 3dokit reads on Doctor Hauzer's disc, tool by tool, and what it does
not. The disc is measured in its own right by the documentation pipeline
([3do-doctorhauzer-doc](https://github.com/vs-sr-dev/3do-doctorhauzer-doc));
this is the kit's reading of the same image, the port's starting point.
Every number below was produced in session 1 with 3dokit at eb96a85 (plus
the one `aif` change of `10-3dokit.md`), the commands as in the README,
`PYTHONIOENCODING=utf-8` throughout. The battery's outputs are kept in
`build/battery/`.

## The disc

```
hauzer.cue: raw 2352, 128000 blocks of 2048 (image 128300), label 'CD-ROM',
  root 1 block(s) x7 copies; 366 files, 32 directories, 153.9 MiB
  rom tag 0f.0d  v0.0   -> System/Kernel/boot_code (5050 bytes, tag says 5168)
  rom tag 0f.07  v0.16  -> System/Kernel/os_code (79692 bytes)
  rom tag 0f.0c  v0.0   offset 0xa9d42b59 size 0x0
  rom tag 0f.02  v0.0   -> launchme (246232 bytes, tag counts blocks from block 0)
```

* **366 files, every one the pipeline's.** `--extract build/disc` and a
  hash of each file against the pipeline's `notes/sha1-all.txt`: 366 listed,
  366 extracted, 366 identical. The three Shift-JIS names come through.
* **`--verify`**: the root's seven copies identical; 112 entries stored more
  than once, 295 extra copies -- 288 identical, 6 directory copies that
  differ only in the case of names (the pipeline's chapter 09: copy 0 holds
  the upper case, 190 of 190 differing bytes a case bit), 1 `rom_tags` copy
  with its own relative offsets, 0 different. 0 entries past the image's end.
* **The ROM tags.** The launcher's tag counts from block 0, as Immercenary's
  does. The os_code tag still says **v0.16** -- the 1993 kernel's tag -- while
  the kernel's own 3DO header says 20.21 (`02-the-os-version.md`): the tag
  is not the release. `boot_code`'s tag says 118 bytes more than the file,
  as on the three 1994-95 discs the pipeline compared.

## The executables

`aif --scan`: **36 AIF images, 4 compressed, 10 signed, 0 failing**. Two are
the game's, the rest the System tree's:

| | `/launchme` | `/OrgData/program/sramtools` |
|---|---|---|
| file | 246,232 | 68,108 |
| read-only | 0x302cc | 0xec5c |
| read-write | 0xb694 | 0x350 |
| zero-init | 0x446e4 (280 KB) | 0x72a0 |
| relocations | 495 | 1,640 |
| 3DO binary header | none, as on the 1993 disc | none |
| stack (header) | 0x9c40 | 0x1000 |
| `arm60 --check` | 48,464 words agree with capstone, 0 disagree | 15,029 agree, 0 disagree |
| discovery | 1,103 functions (1,109 with six seeds), 15 switches, 0 problems | 199 functions, 3 switches, 0 problems |
| embedded names | 60, all library | 0 |

* **`sramtools`'s relocation stub** is where its BL at 0x04 says, 4 bytes
  past `ro + rw` (the pipeline's "exception"): `aif` reads the BL, as it
  learnt on Crash 'n Burn, and the image checks clean.
* **The compiler's names are the libraries', not the game's**: 60 names,
  every one a function, and every one from the SDK's linked libraries -- the
  DataStream (`DataAcqThread`, `NewDataAcq`...), its audio and control
  subscribers (`SAudioSubscriberThread`, `CtrlSubscriberThread`...), the
  Cinepak decoder (`Decompresx`, `ExpandCodeBook`...), the item and memory
  pools, `OpenSoundFileS`. **The films are played by the game's own linked
  code**, recompiled like the rest, as on Crash 'n Burn and Immercenary.
  The game's own functions carry no names.
* **Hand-written code in the read-write area**: `launchme`'s code runs on
  past `ro` (0x302cc) to 0x32054 -- fixed-point 3D math (dot products through
  Operamath's `MulSF16` glue, a multiply of its own at 0x31798), with **a
  dispatch table** of eight words at 0x310a4 read by two dispatchers
  (`ldr pc, [r5, r4]` at 0x31084 and 0x310a0), whose targets jump past the
  prologue of six routines (0x310c4, 0x311e4, 0x31304, 0x3142c, 0x3154c,
  0x31674, each entered at +0xc, the dispatcher's own frame standing in for
  theirs). Discovery finds the routines but not the entries the table
  names: **the six seeds** `310d0, 311f0, 31310, 31438, 31558, 31680`.
  Without them the self-test stops at 0x31558 ("a call to an address that
  is no function's entry"). The game's 3D code calls the first dispatcher
  from six places (in 0x1256c, 0x12648, 0x13528..., the functions that also
  call `MulVec3Mat33_F16`).
* **The OS surface** (`portfolio --sites`, `launchme`): 433 SWI sites to 43
  entry points (Kernel 20, File 6, audio 13, Operamath 4) and 119 vector
  sites, every one attributed: Graphics 40 slots, Kernel 20, audio 45,
  Operamath 8, File 6. These are lib3DO's glue tables, a superset of what
  the game reaches. Worth noting among them: the File folio's `CreateFile`
  and `DeleteFile` (the save), `ControlMem`, `AbortIO`, the Graphics folio's
  **text**: `ResetCurrentFont`, `GetCurrentFont`, `SetCurrentFontCCB`,
  `DrawChar`, `DrawText8`, `DrawText16`; and a device named **"mac"** looked
  for and not found (the development station's, absent on a console).
* **`sramtools`** names no DSP instrument and is run by nothing in the
  disc's scripts: `launchme` names it (`$boot/OrgData/program/sramtools`,
  at 0x6936, beside `/nvram/RH_HAUZERJ` and `/nvram/another`), so the game
  starts it -- the save-game utility (the pipeline's chapter 13).
  Recompiled and self-tested with `launchme`.

## The OS images

The four compressed System images (`AUDIOFOLIO` 20.27, `GRAPHIX` 20.45,
`eventbroker` 20.31, `shell` 20.33) unpack with `aif --decompress`, each by
its own decompressor. **`os_code` did not**, and that was a kit question:
the kernel, like the 1993 one, is linked at 0x10000 with a NOP at 0x04 (it
does not relocate itself), and `aif` looked for a relocation list after
it, which on Crash 'n Burn's kernel it had found only by accident.
Corrected in the kit (`10-3dokit.md`): an image with a NOP at 0x04 has no
list. Over four discs, only the 1993 and 1994 kernels (and Crash 'n Burn's
`misc_code`) have one.

`os_code` holds **three** AIF images, as Immercenary's does: the kernel
(20.21, at 0x10000; 47,904 bytes unpacked), the Operator (20.18, at
0x20000; 28,484) and the File folio (20.30, linked at 0, relocatable as
23.10's; 29,580). So does Crash 'n Burn's (kernel, Operator 20.15, File
folio 20.19) -- which that port's notes read in the console ROM instead.
`misc_code` is one more, at 0x1e000, with no 3DO header.

**One decompressor for all**: the code appended to every compressed image
on the three discs on the kit (28 images, `os_code`'s and `misc_code`'s
among them) is the same 0x188 bytes once its four parameter words are set
aside -- a 0x60-entry table of pairs, then two bits per output byte choosing
a literal or a dictionary byte keyed by the byte before (GRAPHIX 0x5c90,
0x5d50, 0x5da0). Worth knowing: the runtime will need it in C++ for the
font (`TODO.md`).

## Pictures, films, sound

* **`cel --survey`/`--check`: 87 files, 99 frames, 99 decoded**: 77 `IMAG`
  (the pipeline's 77, its pixel order measured on 77 of 77), 19 16-bit
  uncoded frames (the four menu `ANIM`s' four frames each, `bg`, `cursor`
  and `colcel`) and 3 8-bit **coded** ones -- `3DOlogo.cel`,
  `NowLoading.cel` and `pen.cel`, each with a 32-colour `PLUT` chunk and
  PRE0's UNCODED bit clear. The pipeline counts 1,157 cels, 729 of
  them 8-bit, and calls none coded: it counts `CCB ` structures wherever
  they lie (1,066 inside the 33 rooms' sections, 48 in `HumanTex`, 33 in
  `ItemTex`, 10 in containers), and it reads "coded" from the CCB's PLUT
  pointer, which is 0 in every file (the loader fills it). The kit reads
  files that are chunk files from their first byte and no game container;
  the rooms and the texture banks are this game's formats, to be read in
  this port when the recompiled game is checked against them.
* **`stream --scan`: 8 streams**, block 0x8000, 4 buffers, every one read
  to its end: 18 Cinepak films (260x200 to 320x240, 7,176 frames), SDX2
  stereo at 22,050 Hz in seven of the eight streams (not `GODS`),
  `CTRL`/`SYNC`. The pipeline's
  chapter 05 calls a four-byte step in `TRDS` "a defect on a retail
  pressing"; it is not one: the bare `FILL` tag sits at 7,995,388, four
  bytes before block 244's end (7,995,392 = 244 x 0x8000), where no chunk
  header fits, and a DataStream reader moves on to the next block. The kit
  does exactly that ("fewer than eight bytes left").
* **`audio --scan`: 61 AIFF files, 0 failing**: 48 effects in
  `/OrgData/effect` (8-bit mono, 11,025 Hz; `SAKEBI.AIFF` 11,127.3 Hz; the
  directory's other two files, `Geffect.list` and `Geffect.list_no`, are
  text lists of effect paths), 12
  music tracks in `/OrgData/music` as AIFC SDX2 stereo at 22,050 Hz (33 s
  to 144 s), and the System's `sinewave.aiff` with its sustain loop.
* **`dsp --verify`: 56 instruments, 201 knobs, 601 relocations**, every
  file walked to its last byte. `dsp --used launchme` finds 19 of them
  named in the game (and three the disc does not carry:
  `adpcmhalfmono`, `adpcmhalfstereo`, `adpcmstereo`) -- names in the linked
  libraries' tables, more than the game loads. The first run loads
  `mixer4x2`.

## The recompiler

```
launchme   1109 functions    45525 instructions   (six seeds)
sramtools   199 functions    14047 instructions
self-test: optest 891 functions 10,580 vectors; launchme 247 functions 3,853 vectors;
           sramtools 49 functions 731 vectors -- 0 failures
```

## What the kit does not read here

* The game's own formats: the rooms (`/OrgData/room`, a three-word offset
  table over four sections), the polygon and texture banks
  (`HumanPol`/`HumanTex`, `ItemPol`/`ItemTex`, `RGB.bin`), `/OrgData/macro`,
  `/OrgData/isearch`, the font (`/OrgData/font/fontNew.bin`) and the
  messages (`/OrgData/mes`, Shift-JIS). They belong to this port, and the
  recompiled game reads them itself.
* `forsramtools` (7,201 bytes), the save tool's data.
* NVRAM: the disc's `startopera` runs `$c/lmadm -a ram 3 0 nvram` ("auto-
  maintain nvram"), and the game saves to `/nvram/RH_HAUZERJ`. The runtime
  has no NVRAM device yet.
