# What this port gave 3dokit

3dokit (`3dokit/`, a submodule from
[vs-sr-dev/3dokit](https://github.com/vs-sr-dev/3dokit)) was started by
Immercenary's port and grew on Crash 'n Burn's; Doctor Hauzer is its third
game, and the first on Portfolio 20.21. Each entry is a 3dokit commit and
what this game asked of it. The submodule was taken at **eb96a85**; it is at **0903c17**.

## Session 1

Two commits, made with the user's word; every port's submodule then moved
to 9a38b90:

| 3dokit | What |
|---|---|
| 81c0e85 | `aif`: an image with a NOP at 0x04 does not relocate itself and has no relocation list (`AIF.fixed`). Only the kernels linked at their own base have one: 1993's `os_code` and `misc_code`, 1994's `os_code` (20.21), all at 0x10000. This disc's kernel would not unpack (`aif --decompress`: "relocation list runs off the end"); Crash 'n Burn's had unpacked only because a stray 0xffffffff lay after it in the decompressor's memory. Its unpacked bytes are unchanged |
| 9a38b90 | `runtime`: `pf_system_version(path)` -- a System image's own version from its 3DO header, `PF_VERSION(v, r)` -- and `CreateScreenGroup`'s buffer table and `bm_SysMalloc` decided by GRAPHIX's version (`>= 20.45`) instead of the kernel's release (`>= 23`): this disc's GRAPHIX 20.45 does both, as 23.10's (`02-the-os-version.md`); and the Graphics and audio folios' node sizes from each folio's own node database, by its version |

Checked before and after, the old side built from the submodule's clean
eb96a85 runtime, the new from the working tree (the generated C++ is the
same; only the runtime differs):

* the battery (pc-crashnburn's `battery.sh`: 51 files over Immercenary,
  OMF2097 and Crash 'n Burn) and this disc's own (22 files): **byte for
  byte**;
* the self-test (pc-crashnburn's optest and `launchme`): 0 failures;
* `pfcheck`: the six memory runs (0 results, 0 bytes differ) and the
  sixteen Graphics snapshots (16 of 16, 0 bytes differ);
* the three traces (Crash 'n Burn's `launchme` to call 234: 529 lines;
  Immercenary's `p`: 539,774 lines; OMF2097's `LaunchMe` to its stop at
  call 36: 107 lines): **the same**;
* the discs booted whole: Crash 'n Burn `--boot --max-calls 60000 --pad
  a@1300x1` (124,396 lines) and Immercenary `--boot --max-calls 300000`
  with its replay's presses (810,152 lines, four `CreateScreenGroup`s on
  the 23.10 path): **the same**;
* Crash 'n Burn's frames (`frames.sh`): 1,131 to field 3216 and 2,974 of
  fields 8,000-14,000, **0 differ**.

On this disc the second change moves every later allocation of
`launchme`'s down by 0x10 (the 16-byte table given back at call 21), and
the run stops where it stopped before, at call 556.

The scripts: `regress.sh`, `buildnew.sh`, `buildold.sh`, `hbattery.sh` in
session 1's scratchpad (`21eda0d0-.../scratchpad`), over Immercenary's
session 24 scripts (`traces.sh`, `frames.sh`, `pfcheck3.sh`).

**The node sizes** (the second half of 9a38b90, added after the user's
word) moved nothing on Crash 'n Burn (20.31 and 20.19 are 1993's sizes:
the same checks as above, again byte for byte) nor OMF2097's trace. On
Immercenary (23.10's sizes now) the traces differ **only in OS
addresses**: 0 lines once every 0x004xxxxx word is masked (`osmask.py`),
over `p`'s 539,774 lines and a `--boot` replay's 810,152; that replay to
1,500,000 calls gives the same 294 frames and the same 471,859,208 bytes
of sound. Recorded in PC-Immercenary's TODO (its traces' new baseline) and
pc-crashnburn's `docs/10-3dokit.md`; both ports pushed with the submodule
at 9a38b90.

## Session 2

One commit, **1f46573**, made with the user's word; every port's submodule
then moved to it (pc-crashnburn 4efb2ae, PC-Immercenary 1e9f7f5, both
pushed). What it adds:

| file | What |
|---|---|
| `runtime/pf_aif.cpp` | the compressed images' decompressor (GRAPHIX 0x5c90), transliterated: `pf_aif_unpack`, and `pfboot FILE --unpack OUT`. Against armemu on the 28 compressed images of the three discs: 27 the same byte for byte; Immercenary's `ja.language` unpacks on neither (`docs/03`) |
| `runtime/pf_font.cpp` | GRAPHIX's image laid relocated in the OS's memory at `PF_OS_IMAGES` (0x4E0000; `pf_os_alloc` now stops below it), and the folio's built-in font from 20.45's code: the start's 49 `FontEntry`s, `ResetCurrentFont`, `GetCurrentFont`, `SetCurrentFontCCB`, `DrawChar`, `DrawText8`, `DrawText16`; `pf_draw_cels` for `DrawChar`'s `DrawCels`. A GRAPHIX without its addresses (20.31, 23.10) lays nothing, and a font call there stops |
| `pfcheck.py` | GRAPHIX 20.45: a snapshot's font call run on the folio's code where the runtime laid it (`FontCall`), its kernel calls stood in the runtime's way; `--font-start` |
| `README.md` | the above in the runtime's, `aif --decompress`'s and `pfcheck`'s rows |

Checked, the old side built from this port's submodule (9a38b90), the new
from the working tree (session 2's `buildold.sh`, `buildnew.sh`,
`regress.sh`):

* the three traces: Crash 'n Burn's `launchme` to call 234 (529 lines),
  Immercenary's `p` (539,774 lines), OMF2097's `LaunchMe` to its stop at
  call 36 (107 lines, from builds of its own: the `regress.sh` line has
  had no OMF module since session 1) -- **the same**;
* the discs booted whole: Crash 'n Burn `--boot --max-calls 60000 --pad
  a@1300x1` (124,396 lines), Immercenary `--boot --max-calls 300000` with
  its replay's presses (810,152 lines) -- **the same**;
* the self-test (Crash 'n Burn's optest and `launchme`): 0 failures; this
  port's: 1,187 functions, 15,164 vectors, 0 failures;
* `pfcheck`, the old and the new: the six memory runs (0 results, 0 bytes
  differ) and the sixteen 1993 Graphics snapshots (16 of 16, 0 bytes
  differ);
* Crash 'n Burn's frames: 1,131 to field 3216 and 2,974 of fields
  8,000-14,000, **0 differ**.

On this disc: `pfcheck` on 20.45, the font's start and the three calls
`launchme` makes (snapshots 556-558), 0 bytes differ; the run goes on from
call 556 to call 9,015 (`GetSysErr`).

### aceddb7: `GetSysErr`

Committed and pushed with the user's word; every port's submodule moved
(pc-crashnburn 432c12c, PC-Immercenary 1cebb1b, both pushed).

| file | What |
|---|---|
| `runtime/pf_err.cpp` | Kernel -88 `GetSysErr` as the 20.21 kernel's 0x1aacc does it, every table and string read from the disc's own images: `os_code`'s three, unpacked by `pf_aif` -- the kernel's tables at its version's addresses, the File folio 20.30's ErrorText (its tag list at 0xf8c, "File errors", object `FFS`). Another kernel version, or an extended error of an object whose ErrorText is not read, stops the run (`docs/04`) |

Regressed as 1f46573 was (Crash 'n Burn, Immercenary, OMF2097: traces,
whole boots, self-test, `pfcheck`, frames): nothing moves.

## Session 3

The save (`docs/05`). One commit, **0903c17**, made and pushed with the
user's word; every port's submodule then moved to it (pc-crashnburn
77b8315, PC-Immercenary 975aeb8, both pushed, each noting the new
baseline of its traces). What it adds:

| file | What |
|---|---|
| `runtime/pf_nvram.cpp` (new) | the Operator 20.18's `ram` device (0x214f0: CMD_WRITE 0x21278, CMD_READ 0x21020, CMD_STATUS 0x2140c), unit 3 the NVRAM -- 32 KB at 0x03140000, a write only from a privileged task or the OS's own request --; the other units (ROM, memory) stop the run. The NVRAM outlives each program's boot; `pfboot --nvram DIR` keeps it in `DIR/nvram.bin` |
| `runtime/pf_file.cpp` | the File folio 20.30's linked-memory filesystem: the mount (0x1e18; `MountFileSystem`, and the folio's start mounting the NVRAM, 0x48b8), `DismountFileSystem` (0x25d8), `CreateFile` (0x403c), `DeleteFile` (0x4190), the walk through its entries (`READENTRY`, `ADDENTRY`, 0x2b1c), and its requests run step by step as 0x513c takes them, the device's state and buffers kept as the folio keeps them; `CMD_STATUS`'s copy by the File folio's own version (20.30 on: the buffer's length, 0x28 at most; 1993's 20.19: 0x28); the shell runs `System/Programs`' programs when the build has their modules, with their command lines |
| `runtime/pf_task.cpp` | `pf_image_header`: a 3DO header's signature (its length must end the file, else 0xD57B9112; the RSA check itself not made), privilege (a signed image with flag 2 makes a privileged task) and priority, as 23.10's `CreateTask` (0x6cc0) takes them -- for a task's image, and for the program the shell starts |
| `runtime/pf_os.cpp`, `pf_main.cpp` | `pf_run` with the file's size (the header above) and a command line at the stack's top, as `CreateTask` leaves it; `pfboot --nvram DIR` |
| `runtime/pf_err.cpp` | `pf_os_code_version(i)`: `os_code`'s images' own versions (kernel, Operator, File folio) |
| `README.md` | the above in the runtime's row; what is not done yet |

Checked, the old side built from this port's submodule (aceddb7), the new
from the working tree, and a third from the working tree without the `ram`
device's creation (session 3's `regbuild.sh`, `regrun.sh`, `itemmask.py`):

* without the device, the five traces are **the same byte for byte**:
  Crash 'n Burn's `launchme` to call 234 (529 lines) and its `--boot
  --max-calls 60000 --pad a@1300x1` (124,396 lines), Immercenary's `p`
  (539,774 lines) and its `--boot --max-calls 300000` replay (810,154
  lines), OMF2097's `LaunchMe` (107 lines): every other change is neutral
  for them (no NVRAM, no linked-memory filesystem, `lmadm` passed over
  without its module, Immercenary's statuses all 0x28 bytes or longer);
* with it, the device is one more item and 0x70 bytes more of the OS's
  memory: OMF2097's trace is the same; the other four differ only in item
  numbers one greater and in OS addresses -- **0 lines** otherwise;
* the self-test (Crash 'n Burn's optest and `launchme`): 0 failures;
  `pfcheck`: the six memory runs (0 results, 0 bytes differ), the sixteen
  Graphics snapshots (16 of 16);
* Crash 'n Burn's frames: 1,131 to field 3216 and 2,974 of fields
  8,000-14,000, **0 differ**.

So the other ports' traces will move by the item when their submodules
move: a new baseline, as 9a38b90's node sizes made one for Immercenary.
