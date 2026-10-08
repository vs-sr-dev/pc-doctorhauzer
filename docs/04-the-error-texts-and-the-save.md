# 04. The error texts, and what the save asks for

Session 2, after the font. At its 9,015th call `launchme` opened
`/nvram/RH_HAUZERJ`, its save, got `FERR_NOFILE` (0xD556F101 -- what a
console whose NVRAM holds no save answers too), and asked the kernel for
the error's text.

## `GetSysErr` (Kernel -88), the 20.21 kernel's 0x1aacc

Found by its strings: the 1993 kernel's `GetSysErr` (slot -88, 0x1b4a8)
holds " no text for -1 error" 0x78 bytes in; the 20.21 kernel has the
string at 0x1ab28, so its function starts at 0x1aacc. (The 20.21 kernel's
code is not the 1993 one's moved: its list functions, `InitList` and
`AddHead` among them, match none of 1993's bytes, and its vector table is
not yet found.)

`GetSysErr(char* buf, int32 n, Err err)`, as the code does it:

* a 0x80-byte buffer on its stack, cleared (`memset` and `memcpy` reached
  through a table of the kernel's at [0x10df8]);
* -1, an Err without its top bit, and number 0 each give a text of the
  kernel's own (0x1ab28, 0x1ab5c, 0x1ab84);
* otherwise the object's three characters (bits 25-30, 19-24, 13-18, each
  through a 64-character map at 0x1b9b8), then the names of the severity
  (bits 11-12, 0x1b9f8), the environment (9-10, 0x1ba08) and the kind
  (bit 8, standard or extended, 0x1ba18);
* for a standard error the kernel's text (19 of them at 0x1ba20); for an
  extended one the first ErrorText item of that object on the kernel's
  list (0x1bad0; node +0x24 the id, +0x28 the count as a byte, +0x2c the
  table); a number the table has is its text, copied after the names to at
  most 0x7f bytes from them; else the number in three digits between two
  strings of the kernel's (0x1ad6c, 0x1ad74);
* the text and its 0 into `buf`, cut to `n` bytes with a 0 last; the count
  returned.

The ErrorText items are made by the folios at their start. On this disc's
`os_code` only one image makes one: the **File folio 20.30**, at 0x9d0,
`CreateItem(0x111, tags at 0xf8c)` -- name "File errors", object 0x2aab7
(`FFS`), 14 errors, its table at 0x727c, longest 0x20.

`runtime/pf_err.cpp` does this with every table and string read from the
disc's own images: `os_code`'s three images unpacked by `pf_aif`, the
kernel's tables at its version's addresses, the File folio's tag list at
its own. An extended error of an object whose ErrorText is not read stops
the run. For the save's error it gives

    FFS-Severe-System-extended-No such file

(40 bytes with its 0), which the game's `PrintfSysErr` (lib3DO, 0x2b2d0)
prints through `printf` -- the trace's `kprintf`s, a character each.

## What the save asks for (the stop at call 9,058)

The game's routine at 0x18000 writes a file (`path`, `buffer`, `size`):

1. `OpenDiskFile(path)`; if that fails, `CreateFile(path)` (File SWI 9,
   0x30009) and `OpenDiskFile` again; a failure is printed and returned;
2. an IOReq on the file; `CMD_STATUS` (2) into a 0x24-byte status;
3. if the status's block size times its block count is less than `size`,
   `FILECMD_ALLOCBLOCKS` (6) with `ioi_Offset` the blocks wanting
   (rounded up);
4. `CMD_WRITE` (0) of `size` bytes from `buffer`; closed.

For the save, `size` is 0xC30 (3,120 bytes) from 0x9f310. Run with
`--lenient` (`CreateFile` returning 0 but making nothing), the game
deletes the file (`DeleteFile`, SWI 0x3000a) and shows
`OrgData/images/NoMemory.img` -- likely its "not enough memory to save"
screen. So the save is on the way to the title.

What the runtime lacks for it: an NVRAM -- on the console the linked-memory
filesystem of the `ram` device's unit 3, which the disc's `startopera`
keeps with `$c/lmadm -a ram 3 0 nvram` (the runtime's shell passes over
it) -- and the File folio's write side: `CreateFile`, `DeleteFile`,
`FILECMD_ALLOCBLOCKS`, `CMD_WRITE`, and `CMD_STATUS` as a new, empty file
answers it. Each is to be read on the disc's File folio 20.30 (the third
image of `os_code`) and, for the filesystem's own layout and free space,
on the code that keeps it (`LMADM`, `LMFS` in `System/Programs`).
