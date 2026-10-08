# What this port gave 3dokit

3dokit (`3dokit/`, a submodule from
[vs-sr-dev/3dokit](https://github.com/vs-sr-dev/3dokit)) was started by
Immercenary's port and grew on Crash 'n Burn's; Doctor Hauzer is its third
game, and the first on Portfolio 20.21. Each entry is a 3dokit commit and
what this game asked of it. The submodule was taken at **eb96a85**.

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
