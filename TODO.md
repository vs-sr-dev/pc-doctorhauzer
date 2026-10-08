# Next session

The top of this file is the next session's work; the history of sessions
is at the bottom. Read the top, not the history.

## Where the port stands

| | |
|---|---|
| the disc on the kit | 366 files, all the pipeline's to the hash; battery clean (`docs/01`) |
| the OS | Portfolio 20.21; GRAPHIX 20.45, AUDIOFOLIO 20.27, File folio 20.30, OPERAMATH 20.53 (`docs/02`, `docs/06`) |
| `launchme` recompiled | 1,109 functions, 45,525 instructions, seven seeds; self-test 0 failures |
| `sramtools`, `lmadm`, `format` recompiled | 199, 64 and 34 functions; self-test 0 failures |
| `pfboot build/disc --boot` | `lmadm` (and `format` on a blank NVRAM), then `launchme`: the save made, the title, waiting for Start |
| `... --pad start@400+6` | **the whole opening** (Riverhill's logo, the newspaper film, the mansion in 3D, the credits), **the menu**, then the attract demos in the 3D rooms, round again |
| `... --pad start@11500+6` too | Start at the menu: the prologue, the door, `now LOADING...`, **the first room**; the pad moves the visitor (`--pad down@18100+240 --pad right@18400+90 --pad up@18550+300`) |
| the stop | **none** to 5,000,000 calls |
| the NVRAM | `--nvram DIR` keeps it (`DIR/nvram.bin`); the save written, then read back on the next start (`docs/05`) |
| the kit | **075ad17** in every port (`docs/06`, `docs/10`): c11e36d (the folios' versions, OPERAMATH 20.53's two SWIs, `SleepAudioTicks` and a cue's deletion, `pf_free_signal` of a task, the timer's unit 1), then an AIFF's rate read right and the kernel's quantum (the rooms' music); committed and pushed |

```sh
cd D:/Homebrew6 && PYTHONIOENCODING=utf-8 python -m 3dokit.recomp --out pc-doctorhauzer/build/recomp --optest \
  "launchme=pc-doctorhauzer/build/disc/launchme+310d0,311f0,31310,31438,31558,31680,2ced4" \
  "sramtools=pc-doctorhauzer/build/disc/OrgData/program/sramtools" \
  "lmadm=pc-doctorhauzer/build/disc/System/Programs/LMADM" \
  "format=pc-doctorhauzer/build/disc/System/Programs/FORMAT"   # from D:/Homebrew6 whenever the kit has uncommitted work
export PATH=/c/msys64/mingw64/bin:$PATH                       # for cmake/ninja/clang only: its python has no capstone
cmake -S build/recomp -B build/recomp-build -G Ninja -DCMAKE_CXX_COMPILER=clang++ && ninja -C build/recomp-build
build/recomp-build/selftest build/recomp/selftest/*.txt
P="--pad start@400+6 --pad start@11500+6"                     # Start at the title, Start at the menu
build/recomp-build/pfboot build/disc --boot --trace 1 --max-calls 5000000 $P | tail   # 54 s; no stop
build/recomp-build/pfboot build/disc --boot --trace 0 --max-calls 5000000 $P --frames DIR --frames-at 11400-50000/60
build/recomp-build/pfboot build/disc --boot --window --trace 0 --nvram build/nvram    # to play it
```

Session 4's scratchpad (`e87dc26d-.../scratchpad`): `build.sh` (the
port's build from the kit's working tree), `regbuild.sh VARIANT KITPARENT`
and `regrun.sh VARIANT` (the other ports' builds and runs; session 3's
`reg/out-new` the baseline, 0903c17), `sheet.py DIR OUT.png FIRST LAST
STEP` (a contact sheet of `--frames`), the disassemblies with strings:
`ff2.dis` (File folio 20.30), `k.dis` (kernel 20.21), `kc.dis` (1993's
kernel), `ki.dis` (23.10's), `op.dis` (Operator 20.18), `om.dis`
(OPERAMATH 20.53), `au.dis` (AUDIOFOLIO 20.27). Session 3's (`a5bbcab7-`)
`plist.py`, `kp.py`, `lmdump.py`, `itemmask.py`; session 1's
(`21eda0d0-`) `odis.py IMAGE BASE ADDR [N]` and `gvec.py`.

## The work, in order

### 1. Play on, with the user

Published (session 4): the user played the first rooms in the window --
the three cameras (fixed, from above, first person), the menu, a save --
and, after the music's fix, confirmed the sound right. The repository is
public at `vs-sr-dev/pc-doctorhauzer`; push each session's commits there
on the user's word.

### 2. Play it: the window, and the next stop

The run no longer stops by itself. Next, with the user: the window
(`--window --nvram build/nvram`), playing from the first room on --
whether the sound is right in the rooms (the music `music004.aifc`, the
effects), the doors, the camera's cuts, the items, the menu's `CONTINUE`
(the save read back: `sramtools`?) and `OPERATION`. Each call the runtime
does not answer yet stops the run, as before; record the presses that
reach it (`--pad`) to replay it here.

### 3. The text in the folio's font, when the run reaches it

The font's calls are in (`docs/03`), checked on 20.45's own code. Still
unseen: `DrawChar`/`DrawText8` (the box at 0x1fa34, reached from 0x1f00c
and 0x1f598). The game's subtitles are its own font (`fontNew.bin`), not
the folio's. When it is reached: check its frame against Phoenix's, and
add DrawChar to `pfcheck`.

### 4. Open ends of session 4

* **The devices' versions.** The 20.21 kernel gives a driver's and a
  device's node its creator's version too (0x12254 for types 13 and 15):
  on the console the Operator's devices (`timer`, `SPORT`, `ram`) would be
  20.18, the File folio's 20.30. The runtime's devices are still 0.0; no
  program has read one yet.
* **A directory's blocks.** The runtime's directories have
  `fi_BlockCount` and `fi_ByteCount` 0 and no bytes; the console's root
  is 1 block of 2048 with seven avatars, its bytes the disc image's
  (`docs/06`). A read of a directory's blocks still stops the run.
* **The frame rate against Phoenix.** The user, in the window: the frame
  rate varies with what is on the screen, and the footsteps' sound with it
  (a step every so many animation frames), while the walking pace stays
  the same (the game scales its movement by the time passed -- likely what
  it reads the timer's microseconds for). As on the console; the runtime
  charges the ARM60's clocks but not the cel engine's time nor the DMA's
  share of the bus, so in a full scene it can only run faster than the
  console, never slower. To measure: the same room and camera in the
  window and in Phoenix, footsteps a second in an empty and a full scene.
* **The opening against Phoenix.** The user's screenshots (session 1)
  show "in 1952" in red italics and "1952年" in a white box between
  Riverhill's logo and the newspaper; the runtime's frames show the
  newspaper first, then "現在" in a white box. Compare frame by frame
  (the same presses) before deciding anything.

### 5. On from there

What the surface promises (`docs/01`): `/nvram/another` (a second file in
the NVRAM); `$boot/OrgData/program/sramtools` started by `launchme` -- the
save-game manager, which will likely want the File folio's directory
vectors (`OpenDirectoryItem` 0x6214, `OpenDirectoryPath` 0x641c,
`ReadDirectory` 0x6430, `CloseDirectory` 0x661c: not in the runtime yet);
`ControlMem`; the other streams (`OrgData/stream`, eight; OPDS and SLDS
play); the 33 rooms. Each difference between 1993 and 23.10 the game
reaches is read on 20.21's code (`docs/02`).

Open ends of session 2, for when they are needed: the font's addresses of
GRAPHIX 20.31 and 23.10 (`kFonts`); the 20.21 kernel's own `InitList` and
allocator for `pfcheck`.

### 6. The reference: Phoenix

The user's Phoenix screenshots of the opening (session 1, in
`D:\Tools\phoenix28\ph-win64\3DO\hauzer@panafz10@gio ottobre 8 2026
18-56-*.jpg`), in order: the 3DO logo (the runtime draws it the same);
the title with `PUSH "P" BUTTON!` (`CopyRight.img`, the same); Riverhill
Soft's logo (the runtime plays it, a blue field); "in 1952" and "1952年"
(see 4.); the newspaper film (the runtime plays it).

## Answered

* (Session 1) This repository is published once the game is playable.
* (Session 1) The node sizes: each folio's own, by its version (done, `docs/02`).
* (Session 2) The decompressor in C++, then the font: done (`docs/03`).
* (Session 2) `GetSysErr`: committed (aceddb7) and pushed, every port's
  submodule moved.
* (Session 2) **The NVRAM on the host**: `pfboot --nvram DIR` keeps its files
  in a host directory between runs; without it, an NVRAM in memory, empty
  at each boot as a fresh console's (runs stay replayable).
* (Session 2) The kit's work: committed (1f46573) and pushed, every port's
  submodule moved; this repository committed, not pushed (not yet public).
* (Session 3, the user's choice) The session dedicated to the save, by the
  route of item 1 of session 2's TODO: done (`docs/05`).

* (Session 3) The kit's work: committed (0903c17) and pushed; every port's
  submodule moved (pc-crashnburn 77b8315, PC-Immercenary 975aeb8, both
  pushed, each with the new baseline noted: traces one item greater).
  This repository committed locally.
* (Session 3) **The window** (`pfboot build/disc --boot --window --trace 0
  --nvram build/nvram`), tried by the user: the logo, the title, Start, the
  opening film to the kabong stop -- **the sound is there throughout, and
  right**.
* (Session 4) **Publishing**: the user's criterion met (the first rooms
  play in the window with their sound); published with the user's word.
* (Session 4) The kabong read (session 3's item 1): a read the console
  never makes -- the game kabongs only on a File folio 0.0, and the
  runtime's folio nodes had no versions (`docs/06`).

## Questions for the user

* None open.

## Keep in mind

* **Chat in Italian, every file in English.** No game data, no OS bytes in
  any repository; hashes, sizes and addresses are fine.
* **`PYTHONIOENCODING=utf-8`** for every Python tool (Shift-JIS names), and
  Python with the normal PATH (msys's has no capstone); mingw64 on PATH
  only for cmake, ninja and clang.
* C++ or Python with `\n` or `\\` goes through Write or Edit, never a
  shell heredoc -- **not even a quoted one** (session 2: a `'EOF'`
  heredoc into Python still turned `\\n` into a newline in the C++; and
  again in session 3, caught at once).
* The NVRAM: `pfboot ... --nvram DIR` to keep it; `lmdump.py
  DIR/nvram.bin` to read it. A run without `--nvram` starts blank and
  formats it each time (LMADM and FORMAT run before `LaunchMe`).
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
  vector tables), `gvec.py IMAGE START END FOLIO` (a vector table with the
  SDK's slot names; GRAPHIX 20.45: `65d4 6698 Graphics`), `unpack_os.py`,
  `aifs_in.py`, `word4.py`, `decomp_same.py`, `sha1cmp.py`, `hbattery.sh`.
* Session 2's scratchpad (`60f587c0-.../scratchpad`): `decparams.py` (every
  compressed image's decompressor parameters; `list.txt` the 28),
  `unpackcmp.py PFBOOT LIST OUT` (the C++ decompressor against armemu);
  `buildold.sh` (the kit at this port's submodule), `buildnew.sh` (the
  working tree), `regress.sh`. **Its OMF2097 trace line is empty** (the
  Immercenary build has no OMF module: "no recompiled module", both
  sides, in session 1 too); OMF is checked with `reg/omf-old` and
  `reg/omf-new` (`recomp --optest "LaunchMe=.../omf/LaunchMe"`), 107 lines.
* The SDK's headers: `D:/Homebrew6/refs/3do-devkit/include/3do/`
  (`graphics.h` for the font and `GrafFolio`).
* The documentation pipeline's chapter 05 reads `TRDS`'s short `FILL` at a
  block's end as a pressing defect; it is a block's slack (`docs/01`).
  Worth a correction there.

# History

## Session 4 (2026-10-08/09) -- the opening, the menu, the first room

* **The kabong read** (`docs/06`): lib3DO's DataStream jams the drive
  only on a File folio of version 0.0 (`launchme` 0x19a50). The 20.21
  kernel gives a folio's node its creating task's version (0x12254; 23.10
  too, not 1993), each folio's its own image's: File 20.30, GRAPHIX 20.45,
  AUDIOFOLIO 20.27, OPERAMATH 20.53; KernelBase the kernel's header's
  (0x17818). The runtime now does the same; the game never opens `.`.
  On the way, 20.30's CD read path (0x1448, the scheduler 0x1510: the
  nearest avatar, no check of the file's end) and the root's `File` at the
  mount (1 block of 2048, seven avatars), kept for a directory's read.
* **OPERAMATH 20.53**: its tables (Red 0x2cc8, Green 0x2d2c, software
  0x2c64); `MulVec3Mat33_F16` (0x1c50) and `Dot3_F16` (0x18e0).
* **AUDIOFOLIO 20.27**: `SleepAudioTicks` (0x4408); a cue's deletion by
  KernelBase's version (0x4818), met at once: a task deleted with its
  cues; `pf_free_signal` of another task (the kernel's 0x1910c).
* **The timer's unit 1** (`CMD_READ`, Operator 20.18's 0x21974; the
  kernel's 0x1432c): a timeval from CLIO's three counters, 16 us steps.
* **The run**: the whole opening, the menu, the attract demos in the 3D
  rooms; with Start at the menu the prologue, the door, the first room,
  the pad moving the visitor; no stop to 5,000,000 calls.
* **The kit**: five changes, regressed (the other ports' traces the same
  byte for byte); c11e36d, pushed with the user's word, every port's
  submodule moved (pc-crashnburn f9643f6, PC-Immercenary 89ae6ef).
* **The window**, played by the user: the cameras, the menu, a save; the
  rooms' music repeating. Two kit causes (`docs/06`): an AIFF's rate read
  with unaligned word loads (the game picked the full-rate player for its
  22,050 Hz music), and no kernel quantum (the music thread, at the main
  task's priority, starved). 075ad17, pushed; the user confirmed the sound
  right.
* **Published**: `vs-sr-dev/pc-doctorhauzer`, with the user's word.

## Session 3 (2026-10-08) -- the save: the NVRAM, its filesystem, LMADM

* **The NVRAM** (`docs/05`): the Operator 20.18's `ram` device read, its
  unit 3 the NVRAM (32 KB at 0x03140000, a byte a word, writes only from a
  privileged task or the OS's own request); `runtime/pf_nvram.cpp`, and
  `pfboot --nvram DIR` keeping it in `DIR/nvram.bin`.
* **The File folio 20.30** read for its linked-memory filesystem: the SWI
  and vector table (0x721c), the mount (0x1e18) and the folio's start
  mounting every unit with a filesystem (0x48b8), the walk's entries
  (`READENTRY`, `ADDENTRY`), `CreateFile`, `DeleteFile`,
  `DismountFileSystem`, the open file's driver (0x109c) and the
  filesystem's requests as the 29 steps of 0x513c; all of it in
  `runtime/pf_file.cpp`, the device's state and buffers kept as the folio
  keeps them. `CMD_STATUS`'s copy is now by the File folio's version
  (20.30 on: the buffer's length; 1993's: 0x28).
* **LMADM and FORMAT**, the disc's own, signed and privileged:
  `pf_image_header` (23.10's `CreateTask` rules); the shell runs
  `System/Programs`' programs when the build has them, with their command
  lines. On a blank NVRAM LMADM finds "Not a flat linked-memory
  filesystem", runs FORMAT (label, anchor at 132, free space at 152) and
  mounts it; on a kept one it validates it, "CLEAN".
* **The save**: `CreateFile`, `ALLOCBLOCKS` of 0xC30, `CMD_WRITE` at block
  216; read back on the next start. Then the title (`CopyRight.img`), and
  with Start the opening film (0x2ced4 a seventh seed) to call 196,582,
  the DataStream's "kabong" read of `.`.
* **The kit**: regressed on Crash 'n Burn, Immercenary and OMF2097 (old,
  new, and new without the `ram` device): without it every trace is the
  same byte for byte; with it only item numbers and OS addresses move;
  self-test, `pfcheck`, frames unchanged. Committed (0903c17) and pushed
  with the user's word, every port's submodule moved; the window tried by
  the user, the sound right throughout.

## Session 2 (2026-10-08) -- the decompressor in C++, GRAPHIX in the OS's memory, the font

* **The decompressor** (`docs/03`): GRAPHIX's 0x5c90 read and
  transliterated (`runtime/pf_aif.cpp`, `pfboot FILE --unpack OUT`);
  against armemu on the 28 compressed images of the three discs, 27 the
  same byte for byte (whole length, `ro + rw`, relocations). Immercenary's
  `ja.language` unpacks on neither: its decompressor's copy of itself
  overruns itself (+0x10 = 0xcc).
* **GRAPHIX laid in the OS's memory**: unpacked, relocated, at 0x4E0000
  (`PF_OS_IMAGES`, the OS's top 128 KB; its allocations stop below it), its
  KernelBase and GrafBase words written; no code of it runs.
* **The font** (`runtime/pf_font.cpp`): the folio's start builds the 49
  `FontEntry`s' tree; `ResetCurrentFont`, `GetCurrentFont`,
  `SetCurrentFontCCB`, `DrawChar`, `DrawText8`, `DrawText16`, from 20.45's
  code. `pfcheck --graphix` on 20.45 runs the folio's own code where the
  runtime laid it: the start and the three calls `launchme` makes, 0 bytes
  differ.
* **The boot** goes on to call 9,015: the game's Japanese font file read,
  the logo dimmed, the save looked for in `/nvram` and `GetSysErr` asked
  for the error's text.
* **`GetSysErr`** (`docs/04`, `runtime/pf_err.cpp`): the 20.21 kernel's
  0x1aacc, every table and string read from the disc's own kernel and
  File folio ("FFS-Severe-System-extended-No such file"); the run goes on
  to call 9,058, `CreateFile`: the save.
* **The kit**: regressed (Crash 'n Burn, Immercenary, OMF2097: traces,
  whole boots, self-test, `pfcheck`, frames), nothing moves; 1f46573,
  pushed with the user's word, in every port.

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
