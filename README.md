# pc-doctorhauzer

Toward a native PC port of **Doctor Hauzer** (Riverhill Soft, Japan, 3DO,
pressed 1994-04-15), a survival-horror adventure in real-time 3D rooms,
often compared with Alone in the Dark: 33 rooms on the disc, eight Cinepak
films, and its text in Japanese. It was released only on the 3DO, only in
Japan. The goal is the game
running natively on PC by **static recompilation**: the game's own ARM60
code turned into C++ and run on a reimplementation of the 3DO's OS,
booting the disc as the console boots it, in a window in real time, with
its sound.

This repository holds the documentation and the tooling for the port. The
disc itself is measured in the documentation pipeline,
[3do-doctorhauzer-doc](https://github.com/vs-sr-dev/3do-doctorhauzer-doc).
It is the third port built on
[3dokit](https://github.com/vs-sr-dev/3dokit), the game-agnostic 3DO
toolkit -- after [pc-immercenary](https://github.com/vs-sr-dev/pc-immercenary)
(Portfolio 23.10) and [pc-crashnburn](https://github.com/vs-sr-dev/pc-crashnburn)
(the 1993 launch OS) -- and the first on Portfolio 20.21, the OS between
the two. 3dokit is taken here as a submodule: clone with `--recursive`, or
run `git submodule update --init`.

## BYOA — Bring Your Own Assets

This repository contains **documentation and tools only**. No game data, no
executables, no assets, no OS code. You need your own original disc. The
work is done on the Japanese release, one data track, as a .cue/.bin set
(raw MODE1/2352, 128,300 sectors). Hashes, sizes and addresses are written
down here; bytes of the product are not.

## Layout

    docs/            what the kit reads on this disc, the OS version, the sessions
    3dokit/          game-agnostic 3DO toolkit (submodule)
    build/           your disc and everything derived from it (ignored by git)

## The route

The same route two ports have already walked to the end
(pc-crashnburn's `docs/00-sessions.md`, PC-Immercenary's
`docs/30-pipeline-pivot.md`):

1. **Recompile**: `3dokit.recomp` turns `/launchme` (the whole game, one
   program) into C++, a function at a time, checked by the recompiler's
   self-test against an ARM60 interpreter.
2. **Boot it on the Portfolio runtime** (`pfboot`), one OS call at a time:
   each call the runtime does not answer yet stops the run; it is read in
   *this disc's* own OS code (the kernel, the folios, the shell are all on
   the disc) and implemented in the kit as that code does it, with its
   address in a comment, and checked against the other games on the kit.
3. **Compare** with the console and the Phoenix emulator; then play, record
   the presses, and replay them to guard every later change.

## Tools

The Python tools need Python 3.8+, and capstone for the code; run them from
the repository root with `PYTHONIOENCODING=utf-8` (three file names on the
disc are Shift-JIS). Building the C++ needs CMake, Ninja, clang and SDL3
(MSYS2's mingw64).

```sh
D=path/to/hauzer.cue
export PYTHONIOENCODING=utf-8

# the disc: volume, ROM tags, every copy compared; extract it
python -m 3dokit.disc "$D"
python -m 3dokit.disc "$D" --verify
python -m 3dokit.disc "$D" --extract build/disc

# the executables, the OS images, the pictures, the sound
python -m 3dokit.aif --scan build/disc
python -m 3dokit.aif --decompress build/disc/System/Folios/GRAPHIX build/os/graphix.bin
python -m 3dokit.cel --check build/disc
python -m 3dokit.stream --scan build/disc
python -m 3dokit.audio --scan build/disc
python -m 3dokit.dsp build/disc/System/Audio/dsp --verify

# the code
python -m 3dokit.arm build/disc/launchme --names
python -m 3dokit.portfolio build/disc/launchme --sites
python -m 3dokit.recomp.discover build/disc/launchme --report

# the recompiler: C++ for the whole program (seven seeds: docs/01, docs/05), the
# save-game utility, and the disc's own NVRAM tools (docs/05); and the self-test
python -m 3dokit.recomp --out build/recomp --optest \
  "launchme=build/disc/launchme+310d0,311f0,31310,31438,31558,31680,2ced4" \
  "sramtools=build/disc/OrgData/program/sramtools" \
  "lmadm=build/disc/System/Programs/LMADM" "format=build/disc/System/Programs/FORMAT"
python -m 3dokit.recomp.selftest --image launchme=build/disc/launchme+310d0,311f0,31310,31438,31558,31680 \
  --auto --out build/recomp/selftest/launchme.txt
cmake -S build/recomp -B build/recomp-build -G Ninja -DCMAKE_CXX_COMPILER=clang++
ninja -C build/recomp-build
build/recomp-build/selftest build/recomp/selftest/*.txt

# the disc as the console starts it, every OS call traced; the NVRAM kept in
# build/nvram/nvram.bin between runs (without --nvram it is blank each time)
build/recomp-build/pfboot build/disc --boot --trace 1 --max-calls 5000 --nvram build/nvram
```

## Status

**Session 1**: the repository, the disc read by the kit (366 files, every
one the documentation pipeline's to the hash), the kit's battery over it,
the OS version read (Portfolio 20.21, and each folio's own version: what
the runtime must follow where 1993 and 23.10 differ), `launchme`
recompiled (1,109 functions, 45,525 instructions) and `sramtools` too, the
self-test at 0 failures, and the first boot: the shell carries out the
disc's scripts, `launchme` opens its folios, its screens, SPORT, the timer
and the audio folio, loads its `3DOlogo.cel` and draws it with the cel
engine, and stops at its 556th OS call, `ResetCurrentFont` -- the Graphics
folio's built-in font, which no game on the kit had used.

**Session 2**: the System images' decompressor in C++, checked byte for
byte against the images' own code on the three discs; GRAPHIX's image laid
in the runtime's OS memory, and the folio's built-in font answered from
its own data and code, checked on it. `launchme` goes on to read its own
Japanese font and to look for its save in NVRAM; the kernel's `GetSysErr`
follows, its texts read from the disc's own kernel and File folio, and the
run stops at its 9,058th call, `CreateFile`: the save.

**Session 3**: the save. The Operator's `ram` device and the console's
NVRAM behind it (kept on the host with `pfboot --nvram DIR`), and the File
folio 20.30's linked-memory filesystem on it, read in the disc's own
`os_code`; the disc's own `LMADM` and `FORMAT` recompiled, run by the shell
as `startopera` names them: on a blank NVRAM they format it as a fresh
console's. The game creates its save, writes it and reads it back on the
next start; it shows its title, `PUSH "P" BUTTON!`, and on Start plays its
opening film on its own DataStream, to its 196,582nd call.

**Session 4**: the opening, the menu and the first room. The stop at call
196,582 was a read the console never makes: lib3DO's DataStream "kabongs"
the drive only on a File folio of version 0.0, and the runtime's folio
nodes had no versions; they now carry the ones the 20.21 kernel gives
them. With four more calls read on the disc's own OPERAMATH 20.53,
AUDIOFOLIO 20.27, Operator 20.18 and kernel, the whole opening plays, the
menu comes up, the attract demos run in the 3D rooms, and Start at the
menu leads through the prologue to the first room, where the pad moves the
visitor about. Five million calls on, nothing stops the run. Played in
the window: the three cameras, the menu, a save; the rooms' music, after two
fixes in the kit (an AIFF's rate, the kernel's quantum), plays right. See
`TODO.md`.

## Documentation

| | |
|---|---|
| [TODO](TODO.md) | the next session's work, and the history of sessions |
| [01-the-disc-on-the-kit](docs/01-the-disc-on-the-kit.md) | what 3dokit reads on this disc, and what it does not |
| [02-the-os-version](docs/02-the-os-version.md) | Portfolio 20.21: the kernel, the folios, and the runtime's 1993/23.10 switches |
| [03-the-decompressor-and-the-font](docs/03-the-decompressor-and-the-font.md) | the images' decompressor in C++, GRAPHIX in the OS's memory, the folio's font |
| [04-the-error-texts-and-the-save](docs/04-the-error-texts-and-the-save.md) | `GetSysErr` on the 20.21 kernel, and what the save asks for |
| [05-the-save](docs/05-the-save.md) | the NVRAM, the File folio's linked-memory filesystem, the disc's own LMADM and FORMAT |
| [06-the-opening-and-the-first-room](docs/06-the-opening-and-the-first-room.md) | the folios' versions and the kabong read, OPERAMATH 20.53, `SleepAudioTicks`, the timer's microseconds; the run to the first room |
| [10-3dokit](docs/10-3dokit.md) | what this port gave 3dokit |

## Licence

MIT -- see [LICENSE](LICENSE). Doctor Hauzer is © 1994 Riverhill Soft;
this repository contains none of it.
