# Next session

The top of this file is the next session's work; the history of sessions
is at the bottom. Read the top, not the history.

## Where the port stands

| | |
|---|---|
| the disc on the kit | 366 files, all the pipeline's to the hash; battery clean (`docs/01`) |
| the OS | Portfolio 20.21; GRAPHIX 20.45, AUDIOFOLIO 20.27, File folio 20.30 (`docs/02`) |
| `launchme` recompiled | 1,109 functions, 45,525 instructions, six seeds; self-test 0 failures |
| `sramtools` recompiled | 199 functions, 14,047 instructions; self-test 0 failures |
| `pfboot build/disc --boot` | the shell's scripts, then `launchme` to its **556th OS call** |
| on the screen | the 3DO logo (`OrgData/etc/3DOlogo.cel`, 8-bit coded, at 120,40), faded in over 33 fields |
| the kit | 9a38b90: this disc's two changes committed, pushed, in every port (`docs/10`) |

```sh
PYTHONIOENCODING=utf-8 python -m 3dokit.recomp --out build/recomp --optest \
  "launchme=build/disc/launchme+310d0,311f0,31310,31438,31558,31680" \
  "sramtools=build/disc/OrgData/program/sramtools"   # from D:/Homebrew6 (paths adjusted) while the kit has uncommitted work
export PATH=/c/msys64/mingw64/bin:$PATH                                # for cmake/ninja/clang only: its python has no capstone
cmake -S build/recomp -B build/recomp-build -G Ninja -DCMAKE_CXX_COMPILER=clang++ && ninja -C build/recomp-build
build/recomp-build/selftest build/recomp/selftest/optest.txt build/recomp/selftest/launchme.txt build/recomp/selftest/sramtools.txt
build/recomp-build/pfboot build/disc --boot --trace 1 --max-calls 5000            # to the stop
build/recomp-build/pfboot build/disc --boot --trace 0 --max-calls 5000 --frames DIR
```

## The work, in order

### 1. The Graphics folio's font (the stop at call 556)

`launchme` at 0x1f9f0: `ResetCurrentFont()`, `GetCurrentFont()`, keeps the
font, takes the font CCB's source pointer (`font->font_CCB->ccb_SourcePtr`)
into its own CCB at 0x371ac, and `SetCurrentFontCCB(0x371ac)`. Later a box
with text (0x1fa34: `SetFGPen`, `MoveTo`, `DrawTo`, `FillRect`,
`DrawText8` four times), reached from 0x1f00c and 0x1f598. `--lenient` does
not get past: `GetCurrentFont` returning 0 is dereferenced.

What GRAPHIX 20.45 does (`build/os/graphix.bin`, unpacked; the same
machinery in 1993's 20.31 and 23.10, at other addresses -- `docs/02`):

* **At the folio's start** (0x407c): 49 characters from a table of 12-byte
  records at 0x7ef8 in its data (0x4014 makes each a 0x2c-byte `FontEntry`
  with the kernel's allocator, 0x3c44 puts it in the tree), then
  `[0x699c]` and the default `Font` at 0x6a9c (+8, `font_FontEntries`)
  point at the tree's root (gf+0xe0's +0x28).
* **`ResetCurrentFont`** (-44, user code 0x3970): SWI 14 (0x3920: gf+0xd0 =
  0, the lists reset by 0x3800 -- `InitList` of gf+0xe8,
  "FontLeastRecentlyUsedList" --, `gf_CurrentFont` (gf+0x114) = 0x6a9c,
  gf+0x108 bit 0 cleared); then, if `[0x699c]` is 0, `GRAFERR_NO_FONT`
  (0xd55b9122), else SWI 27 on 0x6a9c (0x3bd8: the tree re-linked
  (0x3a18), the font copied to the folio's static `Font` at 0x69e4 and its
  CCB to the static CCB at 0x69a0 with the static PLUT at 0x8268 (0x38c4,
  0x386c), `gf_CurrentFont` = 0x69e4, `FONT_FILEBASED` cleared).
* **`GetCurrentFont`** (-144) = SWI 37 (0x3a08): `gf_CurrentFont`.
* **`SetCurrentFontCCB`** (-140) = SWI 36 (0x39e4 -> 0x386c): 0x44 bytes of
  the caller's CCB to 0x69a0, its next pointer to itself, its PLUT pointer
  to 0x8268, and 0x40 bytes of the caller's PLUT there when it has one.
* **`DrawText8`** (-148) = SWI 38 (0x3998): `DrawChar` per byte until 0 or
  an error. **`DrawChar`** (-128) = SWI 31 (0x3d84): to read -- the tree's
  lookup, the CCB, the cel engine. **`DrawText16`** (-36) = SWI 29 (0x3f34).
* The `GrafFolio` fields: the devkit's `graphics.h` names them
  (`gf_CurrentFontStream` ... `gf_CurrentFont`, `gf_CharArrayOffset`), at
  gf+0xd0..0x11c in 20.45; check them against the 1993 folio before the
  runtime writes them.

**What the runtime needs that it has never had: the folio's own data in
guest memory.** The game reads pointers into it (the static `Font`, its
CCB, the glyphs) and the cel engine draws from it. The glyphs are in
GRAPHIX's read-write data, and GRAPHIX is compressed on the disc. The
proposal, for the user's word:

1. **The images' decompressor in C++** (`runtime/`), transliterated from its
   code (GRAPHIX 0x5c90: the table builder 0x5d50, the decoder 0x5da0;
   the same 0x188 bytes in all 28 compressed images of the three discs),
   checked against `python -m 3dokit.aif --decompress` on every one of
   them, byte for byte.
2. **GRAPHIX's image relocated into the OS's memory** where its code would
   be (its relocation list follows the unpacked image: `aif.relocated`),
   so that its data -- the default font, the static `Font`/CCB/PLUT, the
   49 glyphs -- lies at real addresses; nothing of it runs. Where: above the
   folios' pages, outside `g_os_free`'s range, so that no other port's
   memory moves; on all three discs, since all three folios build the
   font when they start (the font's `FontEntry`s come from the kernel's
   allocator: check what that does to Crash 'n Burn's and Immercenary's
   traces and `pfcheck` snapshots, and say so in the kit's log).
3. **The folio's start and the six calls** as above, each version's
   addresses from its own code (a small table per GRAPHIX version, like
   pfcheck's `G1993`), `DrawChar` through `pf_cel`.
4. **`pfcheck --graphix` on 20.45**: a `G2045` table (vector end 0x6698,
   SWIs at 0x6504, 52 of them, the glue words) so that the font calls are
   replayed on this folio's own code -- with this kernel's allocator, which
   means reading the 20.21 kernel's addresses (`build/os/os_code_0_10000.bin`,
   linked at 0x10000).

### 2. On from there, one call at a time

What the surface promises (`docs/01`): the File folio's `CreateFile` and
`DeleteFile` and NVRAM (`/nvram/RH_HAUZERJ`, `/nvram/another`; the disc's
`startopera` runs `$c/lmadm -a ram 3 0 nvram`, which the runtime's shell
passes over); `$boot/OrgData/program/sramtools` started by `launchme`;
`ControlMem`; the films through the game's own DataStream (`OrgData/stream`,
eight streams); the music (`OrgData/music`, AIFC SDX2) and the effects; the
3D rooms (many small cels a frame, `MapCel`, the projector's edge cases;
729 8-bit cels in the rooms); Japanese text in the game's own font
(`OrgData/font/fontNew.bin`). Each difference between 1993 and 23.10 the
game reaches is read on 20.21's code (`docs/02`, "The runtime's other
1993/23.10 places").

### 3. The reference: Phoenix

The user's Phoenix screenshots of the opening (session 1, in
`D:\Tools\phoenix28\ph-win64\3DO\hauzer@panafz10@gio ottobre 8 2026
18-56-*.jpg`), in order:

1. the 3DO logo, on its grey panel, fading in -- the runtime draws it the
   same (33 fields, at 120,40);
2. the title: "Doctor Hauzer" in green, "(C)1994 Riverhill Soft Inc.",
   "(C)1994 Matsushita Electric Industrial Co., Ltd." and **`PUSH "P"
   BUTTON!`** in yellow capitals -- capitals, quotes and an exclamation
   mark: likely the folio's own 49-character font (`FONT_ASCII_UPPERCASE`)
   through `DrawText8`, the stop's reason; check when the font is in;
3. Riverhill Soft's logo: white squares appearing on a blue field, then
   the fourth, red, turning, on black, "RIVERHILL SOFT" under it (a film:
   `GODS`, 260x200, 131 frames, no sound, is the candidate);
4. "in 1952" in red italics and "1952年" in a white box (Japanese text:
   the game's own font);
5. a newspaper, "Archeologists & Historians" (a film, the intro).

## Answered (session 1)

* The kit's changes: committed and pushed, the submodules pulled.
* This repository is published once the game is playable.
* The font: the next session is the C++ decompressor, then the font.
* The node sizes: each folio's own, by its version (done, `docs/02`).

## Questions for the user

* None open.

## Keep in mind

* **Chat in Italian, every file in English.** No game data, no OS bytes in
  any repository; hashes, sizes and addresses are fine.
* **`PYTHONIOENCODING=utf-8`** for every Python tool (Shift-JIS names), and
  Python with the normal PATH (msys's has no capstone); mingw64 on PATH
  only for cmake, ninja and clang.
* C++ or Python with `\n` or `\\` goes through Write or Edit, never a
  shell heredoc.
* Always `--max-calls`; never `--trace 1` a long run into a file without
  `grep`/`tail`.
* The OS images, unpacked (session 1, `build/os/`): `os_code_0_10000.bin`
  (kernel 20.21, at 0x10000), `os_code_1_20000.bin` (Operator 20.18),
  `os_code_2_0.bin` (File folio 20.30), `graphix.bin`, `audiofolio.bin`,
  `eventbroker.bin`, `shell.bin`, `misc_code.bin`, `operamath.bin` (not
  compressed); the same for Crash 'n Burn in `build/os/cnb/` and
  Immercenary in `build/os/im/`, and `graphix.dis` of each. Session 1's
  scratchpad (`21eda0d0-.../scratchpad`): `odis.py IMAGE BASE ADDR [N]`
  (disassembly with literals and strings; `--tables` finds the SWI and
  vector tables), `gvec.py` (a vector table with the SDK's slot names),
  `unpack_os.py` (every image of an `os_code`), `aifs_in.py`, `word4.py`,
  `decomp_same.py`, `sha1cmp.py`; `hbattery.sh` (this disc's battery),
  `buildold.sh`/`buildnew.sh`/`regress.sh` (the kit against eb96a85 on the
  other two games).
* The SDK's headers: `D:/Homebrew6/refs/3do-devkit/include/3do/`
  (`graphics.h` for the font and `GrafFolio`).
* The documentation pipeline's chapter 05 reads `TRDS`'s short `FILL` at a
  block's end as a pressing defect; it is a block's slack (`docs/01`).
  Worth a correction there.

# History

## Session 1 (2026-10-08) -- the repository, the disc on the kit, 20.21, the first boot

* **The repository**: git, a BYOA `.gitignore` (pc-crashnburn's, which keeps
  `PROMPT.md` out), MIT, the README, 3dokit as a submodule at its published
  URL (eb96a85). Not published.
* **The disc on the kit** (`docs/01`): 366 files extracted, every one
  identical to the pipeline's hash list; `--verify` clean; the kit's
  battery over it -- 36 AIF images (0 failing), 56 DSP instruments, 99 cel
  frames all decoded, 8 streams and 61 AIFF files read to their ends, 0
  disagreements with capstone over both programs, discovery with 0
  problems. One kit question: `os_code` would not unpack (`aif` looked for
  a relocation list after an image that has none) -- corrected.
* **The OS** (`docs/02`): the release reads 20 (kernel folio 20.21) and the
  reading is right; the ROM tag still says 0.16. `os_code` is three images
  (kernel and Operator at fixed addresses as in 1993, the File folio
  relocatable); each System image carries its own version. GRAPHIX 20.45
  read against 1993's and 23.10's: 52 SWIs, 49 vectors with three of
  23.10's filled in, 23.10's node sizes, and **`CreateScreenGroup`'s user
  and supervisor halves 23.10's** -- the one place the runtime tested the
  release, which sent this disc the 1993 way. The kit now asks the folio's
  own version (`pf_system_version`).
* **The recompiler**: `launchme` 1,109 functions with six seeds (a dispatch
  table into hand-written 3D math past `ro`), `sramtools` 199; self-test
  0 failures (1,187 functions, 15,164 vectors).
* **The first boot**: the shell carries out `startopera` and `AppStartup`
  (passing over `sleep`, `lmadm`, `minmem`), and `launchme` configures
  itself with the event broker, opens Graphics (three screens), SPORT, the
  timer, the audio folio (`mixer4x2`) and Operamath, reads `3DOlogo.cel`,
  fades the 3DO logo in, and stops at call 556, `ResetCurrentFont`: the
  Graphics folio's built-in font, which the runtime has never needed.
* **The kit** (81c0e85, 9a38b90): regressed on Crash 'n Burn, Immercenary
  and OMF2097 (battery, self-test, pfcheck, traces, two whole boots,
  frames, a replayed game's sound); committed and pushed with the user's
  word, every port's submodule moved. With the user's word too, the
  Graphics and audio folios' node sizes by version (Immercenary's traces
  move in OS addresses only).
* **Phoenix screenshots** of the opening from the user (`3.` above).
