# 06. The opening, the menu and the first room

Session 4. Session 3 left the run at its 196,582nd call: with Start pressed
at the title, the opening film played on the game's own DataStream until
the streamer read one block from block 0 of `.` -- the CD's root
directory -- and the runtime, whose disc is a host directory with no
blocks, stopped. The read turned out to be one the console never makes.
Five more calls later the whole opening plays, the menu comes up, the
attract demos run in the 3D rooms, and Start at the menu takes the game to
its first room, where the pad moves the doctor's visitor about. Five
million calls on, nothing has stopped the run.

## The "kabong" read, and why the console does not make it

The functions at 0x19880 and 0x19760 of `launchme` are lib3DO's
DataStream "kabonging": it opens a file (`.` when given none), asks its
`CMD_STATUS` for the block size (0x800 when it says 0 or less), and makes
"jamming" reads of its block 0 into a buffer of its own -- a read to move
the CD drive's head, its bytes thrown away. Each of them first asks
0x19a50, which looks at the File folio's node (`FindNamedItem` of
`MKNODEID(KERNELNODE, FOLIONODE)` "File", 0x288d4) and answers yes only
when its `n_Version` and `n_Revision` (+0x14, +0x15) are both 0: the
kabong is for a File folio of version 0.0.

The runtime made every folio's node with 0.0 there. On the console:

* the 20.21 kernel's `CreateItem`, for a folio, a driver or a device
  (node types 4, 13 and 15; the switch at 0x122b4), after the type's own
  routine (a folio's 0x171cc), calls **0x12254**: a node whose version
  and revision are both 0 gets those of the task creating it. 23.10's
  kernel does the same (0x244c, called at 0x2534); the 1993 kernel does
  not (its switch at 0x12c3c tail-calls each type's routine);
* no folio on the three discs gives its own version in its
  `CreateItem` tags (the File folio's at 0xf24, GRAPHIX's at 0x66b0,
  AUDIOFOLIO's at 0xbfb8, OPERAMATH's at 0x2db0: no `TAG_ITEM_VERSION` or
  `TAG_ITEM_REVISION`; 23.10's alike), and each creates its folio from its
  own task, whose version the kernel's `CreateTask` takes from the image's
  3DO header (0x168ac);
* so on this disc the folios are the File folio **20.30** (`os_code`'s
  third image), GRAPHIX **20.45**, AUDIOFOLIO **20.27**, OPERAMATH
  **20.53**; and KernelBase is **20.21**: the kernel's start copies its
  own header's version there (0x17818, from the header at 0x10080;
  1993's 0x1818c and 23.10's 0x7cb0 the same -- the 1993 kernel's header
  says 0.0).

The runtime now gives the nodes these versions (kit, `pf_kernel.cpp`), the
folios' only from a kernel of 20.21 on; with the File folio at 20.30 the
game never opens `.`, never reads it, and plays its opening through.

### What the File folio does with such a read, for the record

Read on the way, and kept because a directory's read will come back:
20.30's driver for an open file (0x109c) sends every command but
`CMD_STATUS` (0x134c) and `FILECMD_GETPATH` (0x1408) to its filesystem's
queue; a CD's (the table at 0x224c that the mount, 0x21f8, puts in the
`OptimizedDisk`) takes `CMD_READ` of whole blocks (else `BADPTR`) and
`FILECMD_OPENENTRY`, done at once (0x1448). The scheduler (0x1510) turns
the file's block into the disc's -- through `fi_Burst` and `fi_Gap` when
the file has a burst -- and of the file's avatars takes the lowest-ranked
(an avatar's top byte), then the nearest to where the drive last read: it
**does not check the read against the file's end**, a directory's or a
file's. The root directory's `File` at the mount (0x2410 on) is the
volume label's: `fi_BlockSize` 2048, `fi_BlockCount` 1, `fi_ByteCount`
2048, `fi_Burst` 1, its seven avatars (blocks 85, 19675, 38475, 57690,
79779, 100363, 109741 on this disc, all seven identical), flags 7 and
`FILE_BLOCKS_CACHED` (0x40) on a filesystem whose device's blocks are
0x800. The runtime's directories have no blocks (their `fi_BlockCount` and
`fi_ByteCount` are 0) and their bytes are not in the extracted tree: the
disc's image has them. (The folio's own `## KABONG ##` at 0x64c8 is
another thing: the name `ReadDirectory`, 0x6430, writes into the entry it
asks the filesystem for.)

## Operamath 20.53: `MulVec3Mat33_F16`, `Dot3_F16`

After the opening film the game loads its 3D rooms and its people
(`NowLoading.cel`, `HumanPol.bin`, `ItemPol.bin`, `room004.bin`,
`macroDemo1`) and calls SWIs the runtime had not met. This disc's
OPERAMATH is 20.53 (the runtime's addresses so far were 20.27's and
23.10's); it picks its SWI table at its start as 20.27 does (0x534: MADAM
revision 0, Red, 0x2cc8; revision 1, Green, not a wirewrap, 0x2d2c; else
the software routines, 0x2c64), the table read from its end (SWI 0 its
last word). On Green:

* **`MulVec3Mat33_F16`** (SWI 0, 0x1c50): the folio's semaphore, the
  matrix's columns into the matrix engine's rows, the vector, one 3x3
  product (control 2), the three outputs: what `MulManyVec3Mat33_F16`
  does for one vector (0x1ee4 goes to 0x1c50 for a count of 1);
* **`Dot3_F16`** (SWI 12, 0x18e0): the first vector into the engine's
  first row, the second as the vector, a 3x3 product, the first output:
  the 64-bit sum of the three products shifted down 16.

## `SleepAudioTicks` and a cue's deletion

AUDIOFOLIO 20.27's `SleepAudioTicks` (vector -20, 0x4408, in the
caller's task): `CreateItem` of a cue, its error the call's;
`SleepUntilTime` of it at the folio's time plus the ticks (0x43d0); the
cue deleted; `SleepUntilTime`'s result.

Deleting a cue (0x4818) depends on KernelBase's version: up to 0x13 its
signal is freed only when the deleting task owns it; above (20.21's 20),
it is freed in its owner's task whoever deletes it -- `LookupItem` of the
owner and the kernel's `FreeSignal` of that task's bits (20.21's 0x1910c;
the SWI is the same on the current task, 0x19140). The game meets the
difference at once: deleting one of its tasks deletes the cues the task
owns, from the deleting task.

## The timer's unit 1

Past the menu the game asks the timer's unit 1 -- microseconds -- for the
time (`CMD_READ`). Operator 20.18's driver (the `timer` device, commands
at 0x26b20; `CMD_READ` 0x21974) checks the buffer as for unit 0 and asks
the kernel (its user function -42 by the Operator's numbering, 20.21's
0x1432c; 1993's 0x14ed4 and 23.10's 0x46ac the same) for a timeval: three
of CLIO's counters in a cascade (read with interrupts off, 0x108a8), the
lowest loaded with 62,499 and stepping 62,500 times a second, the two
above it from 0xffff down at each of its turns (0x14388 sets them up, and
the shift at KernelBase +0xc8 to 4); the seconds are the upper two's turns,
the microseconds the lowest's steps into its turn times 16. The runtime
counts them from its own clock's 0.

## Where the run is

`pfboot build/disc --boot --pad start@400+6`: the 3DO logo, the title,
and on Start the opening -- Riverhill Soft's logo, the newspaper film,
the typewriter, the mansion in 3D, the title in its colours, the credits
over 3D scenes -- then the menu (`OPENING`, `START`, `CONTINUE`,
`OPERATION`). Left alone the game cycles through attract demos in the 3D
rooms and the opening again. With `--pad start@11500+6` as well, Start at
the menu: the prologue's scrolling text in Japanese, the mansion's door
with its subtitles in the game's own font, `now LOADING...`, the visitor
at the door, the close-up of his face, and the first room, the doctor's
visitor standing on the carpet waiting for the pad; with
`--pad down@18100+240 --pad right@18400+90 --pad up@18550+300` he turns,
walks, and the camera cuts to the room's next angle. To five million calls
nothing stops the run.

## The kit

Five changes, one commit (**c11e36d**, pushed with the user's word; every
port's submodule moved to it), all from the disc's own code as above:

| | |
|---|---|
| `pf_kernel.cpp` | KernelBase's version from the kernel's header; the folios' nodes their own images' versions from the 20.21 kernel on |
| `pf_math.cpp` | `MulVec3Mat33_F16`, `Dot3_F16` |
| `pf_audio.cpp` | `SleepAudioTicks`; a cue's deletion by KernelBase's version |
| `pf_task.cpp` | `pf_free_signal` of another task's bits (the kernel's own) |
| `pf_io.cpp` | the timer's unit 1, `CMD_READ` |

Regressed on Crash 'n Burn (its `launchme` to call 234, and booted whole
to 60,000 calls), Immercenary (`p`, and booted whole to 300,000 calls with
its replay's presses) and OMF2097 (to its stop): every trace the same,
byte for byte, as before the changes. On Crash 'n Burn's disc nothing can
move (its kernel is 0.0); on Immercenary's the folios and KernelBase are
now 23.10, and nothing in those runs reads them.
