# 03. The decompressor and the Graphics folio's font

Session 2. The first boot stopped at `launchme`'s 556th OS call,
`ResetCurrentFont`: the Graphics folio's built-in font, which no game on
the kit had used. The font is the folio's own data -- a `Font`, its CCB and
PLUT, 49 characters' images -- and GRAPHIX is compressed on the disc. So
the runtime first had to unpack a System image itself, then lay GRAPHIX's
image in the OS's memory, and only then answer the font's calls.

## The decompressor, in C++

Every compressed AIF image of the three discs ends in the same 0x188 bytes
of code (session 1: 28 images, one hash once its four parameter words are
set aside). Read in GRAPHIX 20.45 at 0x5c90:

| | |
|---|---|
| 0x5c90 | `b` over four words: +0x04 is 1 on every image; +0x08 the decompressor's own offset, where the packed data stops; +0x0c where the unpacked words end (0x100 + 4 x their count; not read by the code); +0x10 how far past the decompressor the data is moved |
| 0x5ca4 | the code from 0x5cec copied to base + [+0x08] + [+0x10], and run there |
| 0x5cf4 | the packed data (0x100 up to the decompressor) moved up to end where the copied code begins, a word at a time from its last |
| 0x5d10 | its first word is the count of words to unpack |
| 0x5d50 | a table of 0x300 bytes on the stack: 0x60 records, each a flag byte and the bytes whose flag bit is 0 (a 1 is a zero byte) |
| 0x5da0 | each word: a control byte, four 2-bit codes; code 0 is a byte of the data, codes 1-3 the byte at the previous byte's index in the table's row 0-2 (256 bytes a row) -- a model of the byte that follows a byte |
| 0x5d3c | the NOP into the BL's place at 0x00, back to the header at 0x04 |

`runtime/pf_aif.cpp` is its transliteration: the same operations in the
same order on the same bytes in the same places (the move, the copy, the
unpacking over the moved data), refusing any decompressor whose CRC-32
(parameters zeroed) is not this one's. `pfboot FILE --unpack OUT` writes
what it makes.

**Checked** against `aif.py`, which runs the image's own code in `armemu`,
on all 28 images (Doctor Hauzer's six, Crash 'n Burn's six, Immercenary's
sixteen): **27 the same, byte for byte** -- armemu's memory from the base
over the whole unpacked length, the `ro + rw` that
`python -m 3dokit.aif --decompress` writes, and the relocation list read
from each.

**The 28th, Immercenary's `System/Drivers/LANGUAGES/ja.language`, does not
unpack, on either side.** Its +0x10 is 0xcc, less than the decompressor's
length: the copy at 0x5cd0 runs upwards into the part it has yet to copy,
so the code that runs at the destination is the first 0x70 bytes over and
over, not the decoder. armemu runs off memory on it (a read at
0xfffffffc); the C++ says so and stops. No kernel of the three discs has a
decompressor of its own (the unpacked kernels hold none of the decoder's
instructions), so the console would fail on it the same way: a file an
English console never loads.

## GRAPHIX's image in the OS's memory

The runtime now lays the disc's GRAPHIX where the loader would leave it:
unpacked, its relocation list applied for its address, its
zero-initialised data cleared, at **0x4E0000**, the top 128 KB of the OS's
1 MB (`PF_OS_IMAGES`). The OS's own allocations stop below it; none of them
moved (Crash 'n Burn's and Immercenary's traces are the same). None of
GRAPHIX's code runs. The two words its code reads KernelBase and GrafBase
from (0x63c8, 0x6978) are written as its start writes them.

## The font (GRAPHIX 20.45)

The characters are `FontEntry`s (`graphics.h`) in a binary search tree by
character: a head node whose greater branch is the root, every missing
branch a "butt" node, both in the folio's data (`gf_FontEntryHead`,
`gf_FontEntryButt`).

| | what the code does |
|---|---|
| the start (0x3f8c, from 0x15d4) | the GrafFolio's font fields (`gf_CurrentFontStream` at gf+0xd0 to `gf_fileFontCacheUSed` at gf+0x11c: the cache size 0x4000, the head 0x69f0, the butt 0x6a1c, the built-in `Font` 0x6a9c current); the tree emptied (0x3800: the butt's branches to itself, the head's value -1, `InitList` of the LRU list, "FontLeastRecentlyUsedList"); then (0x407c) 49 records of 12 bytes at 0x7ef8 -- `A`-`Z`, `0`-`9`, space and `! * ( ) - + = : , . ? /`, each 8 pixels wide, 0x60 bytes of image -- each a 0x2c-byte `FontEntry` from the kernel's allocator (0x4014) put in the tree (0x3c44, signed); the root kept at 0x699c and as the `Font`'s `font_FontEntries` |
| `ResetCurrentFont` (-44, user code 0x3970) | SWI 14 (0x3920: the tree emptied, the built-in `Font` current), then `GRAFERR_NO_FONT` if 0x699c is 0, else SWI 27 on the built-in `Font` (0x3bd8: its tree hung from the head and relinked (0x3a18), the `Font` copied to the folio's own at 0x69e4, its CCB (0x44 bytes) to 0x69a0 with the PLUT (0x40 bytes) at 0x8268 (0x38c4, 0x386c), the copy current, `FONT_FILEBASED` cleared) |
| `GetCurrentFont` (-144) | SWI 37 (0x3a08): `gf_CurrentFont` -- 0x4E69E4 in the runtime |
| `SetCurrentFontCCB` (-140) | SWI 36 (0x39e4 -> 0x386c): the caller's CCB as the current font's, with its PLUT |
| `DrawChar` (-128) | SWI 31 (0x3d84): with `FONT_ASCII` and `FONT_ASCII_UPPERCASE`, a-z made A-Z; the character looked for in the tree (unsigned); the current CCB at the pen (16.16), its source the image, the pen on by the width (or the height with `FONT_VERTICAL`); then `DrawCels` (SWI 39, 0x1954) |
| `DrawText8` (-148), `DrawText16` (-36) | SWIs 38 and 29: `DrawChar` per character up to a 0 or an error |

The built-in font's flags are `FONT_ASCII | FONT_ASCII_UPPERCASE`, height
8: an upper-case font. The title's `PUSH "P" BUTTON!` (the Phoenix
screenshot) has quotes, which the 49 characters lack; whether that line is
this font is for when the run reaches it.

A character not in the tree: the code takes the *word at* 0x7e90 as the
CCB's source -- 0x800001c1 in 20.45's data, which the relocation list does
not name: no address of the folio's. The runtime stops there and says so
rather than drawing from it.

Only 20.45's addresses are read (`kFonts` in `runtime/pf_font.cpp`); on a
disc with another GRAPHIX the font is not laid out, and a font call stops
("its addresses not read yet"). Crash 'n Burn's (20.31) and Immercenary's
(23.10) programs make none.

## Checked on the folio's own code

`python -m 3dokit.pfcheck OS_CODE DIR --graphix GRAPHIX [--font-start]` on
a 20.45 snapshot runs the folio's code **where the runtime laid it** (the
image is in the snapshot's OS memory), over the memory before. No kernel
code runs: the folio's kernel calls (`InitList`, `memcpy`, `AddTail`,
`RemNode`, `AllocMemFromMemLists`) are stand-ins made the runtime's way --
the 20.21 kernel's own `InitList` is not the 1993 one's code, and is not
read yet. `--font-start` first rebuilds the image as the loader leaves it,
runs the start's font part (0x3f8c) and compares the whole memory with
the snapshot's memory before: valid on any snapshot before the first font
call.

| snapshot | result |
|---|---|
| 556 `ResetCurrentFont`, with `--font-start` | the start: 0 bytes differ; the call: 0, and 0 bytes differ |
| 557 `GetCurrentFont` | 0x4E69E4 both; 0 bytes differ |
| 558 `SetCurrentFontCCB` (the game's CCB at 0x371ac) | 0 both; 0 bytes differ |

## Where the run is now

`launchme` takes the font, sets its own CCB on it, opens its own Japanese
font (`$boot/OrgData/font/fontNew.bin`, read in pieces), dims the 3DO logo
and at its **9,015th OS call** opens `/nvram/RH_HAUZERJ` -- the save
-- which the runtime does not have (`D556F101`), and asks the kernel for
the error's text: `GetSysErr` (Kernel -88), not implemented. The box with
text (0x1fa34, `DrawText8`) has not been reached yet.
