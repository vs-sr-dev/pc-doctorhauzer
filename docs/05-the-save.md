# 05. The save: the NVRAM, its filesystem, the disc's own LMADM

Session 3. Session 2 left the boot at its 9,058th call, `CreateFile
("/nvram/RH_HAUZERJ")`: the game's save routine (0x18000, `docs/04`) making
its save in the console's NVRAM. Now the save is made, kept and read back,
every step of it the disc's own code or the runtime doing what that code
does; and the boot goes on to the title and its `PUSH "P" BUTTON!`.

## The NVRAM: the `ram` device's unit 3

None of the kernel's or the folios' images names the NVRAM: it is unit 3 of
the Operator's `ram` device (Operator 20.18, the second image of
`os_code`, linked at 0x20000; the 1993 and 23.10 Operators make one of the
same name). The device is made at 0x214f0; its driver has three commands
(table at 0x26a38: `CMD_WRITE` 0x21278, `CMD_READ` 0x21020, `CMD_STATUS`
0x2140c), and its six units are spans of the console's address space in
blocks of one byte, set up at 0x20e20 in a table at 0x269d0 (block count,
bytes, base, block size, 16 bytes a unit):

| unit | base | bytes | |
|---|---|---|---|
| 0 | 0x03701000 | 0x2f000 (0 on some machines) | memory |
| 1 | 0x03028000 | 0xd8000 | the ROM's filesystem part, read-only |
| 2 | 0x03000000 | 0x100000 | the whole ROM, no filesystem |
| **3** | **0x03140000** | **0x8000** | **the NVRAM** |
| 4 | -- | 0x200000 when present | a second ROM, read-only |
| 5 | 0x03000000 | 0x100000 | the ROM again, read-only |

The NVRAM is a byte at each word: the driver reads and writes it a byte a
word (0x211a4, 0x2135c). A write to it is taken only from a privileged
task (`TASK_SUPER` in the current task's flags) or by a request with a
callback -- the OS's own -- else `NOTPRIV`; a read sets `io_Actual`, a
write to the NVRAM leaves it as it was. `CMD_STATUS` says block size 1,
0x8000 blocks, `DS_USAGE_FILESYSTEM`, and the unit's base at +0x24.

`runtime/pf_nvram.cpp` is this device, unit 3 alone (the others are the
console's ROM and memory, which the runtime has not: a request for one
stops the run). The NVRAM's 32 KB outlive each program's boot: the shell's
programs each boot a fresh OS, and the NVRAM is the console's.
**`pfboot --nvram DIR`** keeps them in `DIR/nvram.bin` (read at the first
use, written after each change: the device's 32,768 bytes in order);
without it the NVRAM is blank at each start, as a fresh console's, and the
disc's own programs prepare it.

## The filesystem on it

The NVRAM holds a linked-memory filesystem. Its blocks lie in a ring, each
beginning with a header of 0x40 bytes:

| at | | |
|---|---|---|
| +0x00 | fingerprint | `0xBE4F32A6` a file's, `0x7AA565BD` free, `0x855A02B6` the anchor |
| +0x04 | next block | |
| +0x08 | previous block | |
| +0x0c | the block's length in blocks | its header's included |
| +0x10 | its header's blocks | |
| +0x14 | the file's bytes | a file's only, to +0x40 |
| +0x18 | an identifier | |
| +0x1c | a type | |
| +0x20 | its name, 32 characters | |

A label (a `DiscLabel` of 0x84 bytes, version 2) is at the first block;
it names the volume, and the root directory's one avatar: the anchor
block. **`FORMAT`** (`System/Programs`, `format DEVICENAME UNIT OFFSET
FSNAME`) lays a fresh one out from the device's status: the label needs
`(0x84 + bs - 1) / bs` blocks (132), the linked-memory structures
`(0x14 + bs - 1) / bs` each (20); the anchor at block 132, its next and
previous the free block at 152, which runs to the end and whose next and
previous are the anchor. The label's other fields: volume identifier -1,
block size 1, block count 0x8000, root identifier -2, root block size 1,
root block count 0. (Its commentary and avatars 1-7 are whatever FORMAT's
stack held: FORMAT writes one byte of the one and none of the others.)

## The File folio 20.30

The third image of `os_code`, linked at 0. Its 14 SWIs and 10 vectors are
one table at 0x721c (in RW, the SWIs backwards):

| SWI | | vector | |
|---|---|---|---|
| 0 `OpenDiskFile` | 0x3ddc | -4 `OpenDiskStream` | 0x59f0 |
| 1 `CloseDiskFile` | 0x3edc | -8 `ReadDiskStream` | 0x5cb0 |
| 2 | 0x4370 | -12 `SeekDiskStream` | 0x619c |
| 3 | 0xef0 | -16 `CloseDiskStream` | 0x5c38 |
| 4 `MountFileSystem` | 0x2770 | -20 `LoadProgram` | 0x6afc |
| 5 `OpenDiskFileInDir` | 0x3e20 | -24 `LoadProgramPrio` | 0x6684 |
| 6 `MountMacFileSystem` | 0x4ad4 | -28 `OpenDirectoryItem` | 0x6214 |
| 7 `ChangeDirectory` | 0x3f4c | -32 `OpenDirectoryPath` | 0x641c |
| 8 `GetDirectory` | 0x3fc4 | -36 `ReadDirectory` | 0x6430 |
| 9 `CreateFile` | 0x403c | -40 `CloseDirectory` | 0x661c |
| 10 `DeleteFile` | 0x4190 | | |
| 11 `CreateAlias` | 0x4094 | | |
| 12 `LoadOverlay` | 0x4324 | | |
| 13 `DismountFileSystem` | 0x27b4 | | |

What the save reaches, as `runtime/pf_file.cpp` now does it:

* **The mount** (0x1e18, behind `MountFileSystem`): an IOReq on the
  device; its status; nothing for 0xe1 blocks or fewer; a label looked for
  at the first block, at 0xe1, then every 0x8012 blocks, nine places in
  all -- record type 1, five sync bytes 0x5a, version 1 or 2, at most eight
  root avatars; none is -1, a name already mounted `DuplicateFile`.
  Version 2 is a linked-memory filesystem: a `LinkedMemDisk` device (0x2cc
  bytes) with its five functions (the table at 0x23dc: queue 0x4b30,
  schedule 0x4c70, start 0x4ef4, abort 0x4f90, the transfers' end action
  0x4ff8), the `FileSystem`, and the root directory's `File` (flags 0x2d:
  a directory, the filesystem's, scannable, with entries).
* **The folio's start** (its daemon, 0x48b8) asks every device's every
  unit for its status and mounts each one that says it holds a filesystem:
  of the `ram` device's, units 0, 1, 3, 4, 5. So an NVRAM once formatted
  is mounted at every boot before any program runs; the runtime does that
  for unit 3.
* **The walk** (0x2b1c): at the root a mounted filesystem's name; in a
  directory that has entries, the folio's own `File` if it has one (its
  list, 0x2f68), else `FILECMD_READENTRY` to the directory and a `File`
  made of the `DirectoryEntry` (0x31a4). In its creating mode (1, from
  `CreateFile`), a last name that is not there is `FILECMD_ADDENTRY`ed and
  read again; a name that is there is `DuplicateFile` (0x39d8).
* **`CreateFile`** (0x403c), **`DeleteFile`** (0x4190: `BUSY` when the file
  is open, `READONLY` in a CD's directory, else `FILECMD_DELETEENTRY` to the
  directory) and **`DismountFileSystem`** (0x25d8: every unused `File` of
  it deleted, `BUSY` if any other is in use).
* **The open file's driver** (0x109c): `CMD_STATUS` answered by the folio
  (0x134c), the rest to the filesystem's queue (0x4b30: `CMD_READ` and
  `CMD_WRITE` of whole blocks inside the file, else `BADPTR`).
* **A request** is a run of steps (0x513c), each one transfer through the
  `ram` device -- a block's header into or out of one of two buffers, or
  file data -- after which the end action (0x4ff8) takes the next. The
  steps (29 states): grow a file in place into the free block after it
  (`ALLOCBLOCKS`); else find the first free block long enough from the
  anchor round, copy the file there 0x100 bytes at a time and free the old
  block; cut a block to what is wanted when 0x60 blocks or more are left
  over, mending the neighbours' links; free a block and join it to the free
  blocks before and after it; walk the entries for `READDIR` (by index,
  remembering where the last one was), `READENTRY` and `DELETEENTRY` (by
  name); `SETEOF`, `SETTYPE`. On the console the folio's daemon sends each
  transfer; here they are done one after the other at once (the NVRAM is
  memory its driver copies), the request complete when `SendIO` returns.
  The device's state between requests and its buffers are kept as the folio
  keeps them: a header written back is written whole, with what an earlier
  request left in it.

Two of the folio's own ways show in what it writes: `SETEOF` updates the
byte count of the `File` the *previous* request was on (0x4eac does not set
it); and `CMD_WRITE` to the NVRAM completes with `io_Actual` 0 (the `ram`
driver does not set it for that unit, and the folio passes on what the
driver said).

**A version difference**, the first the save met: `CMD_STATUS` of an open
file copies its 0x28-byte `FileStatus` to the caller's buffer as the
buffer's length, 0x28 at most, in 20.30 (0x134c) and 23.10 (0x1198); the
1993 folio (20.19, Crash 'n Burn's `os_code` too) copies 0x28 bytes
whenever the buffer is shorter -- the game's status is 0x24 bytes, so 1993's
way would write a word past it. The runtime now copies by the File
folio's own version (`pf_os_code_version(2)`, `os_code`'s third image).

## LMADM, FORMAT: signed, privileged, run by the shell

`startopera` runs `$c/lmadm -a ram 3 0 nvram #`. Both `LMADM` and `FORMAT`
carry a 3DO header with a 64-byte signature (+0x34; +0x30 the signed
bytes, the two the file's length) and flag 2 (+0x24): privileged. 23.10's
`CreateTask` (0x6cc0) takes a signature that does not end the file as
0xD57B9112; checks it against the 3DO Company's key (0xacd0) -- which the
runtime does not: a disc's image is taken as the console takes the disc's
own, signed --; makes a task privileged only for a signed image with flag 2
(the creator's privilege is not inherited); and lets a signed or privileged
task have any priority from its header. `pf_image_header` (pf_task.cpp)
does this for a task's image and for the program the shell starts.

The shell now runs the System directory's programs (`System/Programs`)
when the build has their modules, and passes over them as before when it
has not; a program named with arguments is given its line ("$c/lmadm -a
ram 3 0 nvram", at its stack's top for its startup to read, as
`CreateTask` leaves it). `LMADM -a` (auto-maintain, 0x71c): when `/nvram`
is mounted, dismount it; validate the filesystem (0x16a4) or, when it is
one, check and repair it (0x1000); when it is none, `LoadProgram`
`$boot/System/Programs/format ram 3 0 nvram` and wait for the task to end
(`SIGF_DEADTASK`); then mount it.

## What a run shows

The port builds `lmadm` and `format` as modules of their own (2,899 and
1,024 instructions; self-test 0 failures, 23 functions, 352 vectors):

    python -m 3dokit.recomp --out build/recomp --optest \
      "launchme=build/disc/launchme+310d0,311f0,31310,31438,31558,31680,2ced4" \
      "sramtools=build/disc/OrgData/program/sramtools" \
      "lmadm=build/disc/System/Programs/LMADM" "format=build/disc/System/Programs/FORMAT"

**A blank NVRAM** (`pfboot build/disc --boot`):

    $c/lmadm: VALIDATING (ram, 3, 0)
    $c/lmadm: Not a flat linked-memory filesystem ram
    Device ram unit 3 has 32768 blocks of 1 bytes each
    Label requires 132 blocks
    Linked-memory structures require 20 blocks each
    Writing anchor to absolute block 132: ok.
    Writing free-space to absolute block 152: ok.
    Writing label to absolute block 0: ok.
    Filesystem initialization complete.

then LMADM mounts it, and `LaunchMe`'s boot (a fresh OS) mounts it again
at the folio's start. The game: `OpenDiskFile` fails (`NOFILE`),
`CreateFile` (`READENTRY` fails, `ADDENTRY`, `READENTRY`), `OpenDiskFile`,
`CMD_STATUS` (no blocks), `FILECMD_ALLOCBLOCKS` of 0xC30, `CMD_WRITE` of
3,120 bytes -- at block 216, the file's block 152 and its 0x40 of header --,
`CloseDiskFile`. The NVRAM after it:

    label: version 2, "nvram", block size 1, 32768 blocks, root avatar 132
      132 ANCHOR  next  152  previous 3336  20 blocks
      152 FILE    next 3336  previous  132  3184 blocks, header 64, 0 bytes, "RH_HAUZERJ"
     3336 FREE    next  132  previous  152  29432 blocks

(0 bytes: the game sets no end of file; its blocks hold the save.)

**The same NVRAM again** (`--nvram DIR`): the folio's start mounts it;
LMADM dismounts it, validates it (`pass 1` to `pass 3`, "is CLEAN",
"Compressed by 0 blocks" -- rewriting the free block's header with a
header of 20 blocks, its own reckoning), mounts it; the game opens
`/nvram/RH_HAUZERJ` and reads its 3,120 bytes back.

Then the game shows `OrgData/images/CopyRight.img` -- the title: "Doctor
Hauzer", the two copyright lines and `PUSH "P" BUTTON!` in yellow, as in
Phoenix's second screenshot (`TODO.md`, 4.); an image, not the folio's
font -- and waits, reading the event broker's messages each field. With
`--pad start@400+6` it opens `OrgData/stream/OPDS` for the opening film and
stops at 0x2ced4: a call through a register to code the recompiler did not
find -- a leaf of three instructions that the game's DataStream reaches
through a table, just before `...StreamFile`'s embedded name. With 0x2ced4
a seventh seed of `launchme`'s, **the opening film plays**, on the game's
own DataStream and decoder: the newspapers ("Archeologists & Historians",
the Japanese captions in their boxes), the typewriter, the house, the
title, the credits ("Polygon Designer KOTARO MITOMA"), 157 different
frames between fields 400 and 10,720 (every 60th looked at). At call
196,582 it stops: the game reads block 0 of "." -- opened at call 12,278,
the current directory, the CD's root -- as a file, and the disc as a host
directory has no blocks for a directory (`TODO.md`, where the work goes
on).

## Not done, said so

* The `ram` device's units other than 3 (the ROM, memory): a request stops
  the run; the folio's start mounts only unit 3.
* The mount makes the `FileSystem` and the root's `File`; the
  `LinkedMemDisk` device and its IOReq the folio also makes are the
  runtime's own state, not items.
* `CreateFile` with an alias in its last name, or in a directory of the
  disc, and `DeleteFile` of a disc's file whose directory is not read-only,
  stop the run; so does a mount of anything but the `ram` device, and a
  label of version 1 on it.
* The folio's directory vectors (`OpenDirectoryItem`, `OpenDirectoryPath`,
  `ReadDirectory`, `CloseDirectory`) are not in the runtime yet: the
  save-game manager `sramtools` will likely want them.
* The RSA check of a signature.
