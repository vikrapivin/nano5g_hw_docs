# AVC/H.264 decoder engine

The decoder is at **`0x38f00000`**, **IRQ35**, OSOS clock ID `0x41`.
Constructor `osos mem:0x082d9a0c` allocates scratch regions of `0x4000` and
`0x8000` bytes; its reset ORs bits in and preserves the rest. Clock-table entry
`mem:0x08551d98` has words `[41,0,0,0,4,40000]`: clear `0x3c500068` bit 2 and
`0x3c50006c` bit 18 to enable.

## Descriptor and input interface

32-bit MMIO offsets relative to `0x38f00000`:

| Offset / formula | Observed use |
|---|---|
| `1200+4*id` | descriptor page address (`physical>>10`) |
| descriptor 0 | input stream page |
| descriptors 1/2 (`1204`, `1208`) | scratch areas |
| IDs 3..22 (`120c..1258`) | 20 luma descriptors |
| IDs 23..42 (`125c..12a8`) | 20 chroma descriptors |
| IDs 43..62 (`12ac..12f8`) | 20 motion descriptors |
| `180c` | input start byte offset within a 1 KiB page; preserve upper 9 bits |
| `1810` | start offset + NAL byte length; preserve upper 9 bits |
| `1814`, `1818` | preserve upper 2 bits; set low fields to 32 and 8 |
| `1804` | input enable bit 0, followed by a **readback** |
| `1600` | preserve bits above 11, set low field `801` to launch the frontend |
| `1628` | clear low 9 bits before launch; bit 8 is NAL ready; low fields include NAL type/ref flags |
| `1400+4*n` | read consumes `n` bitstream bits; sampled jobs use chunks of 1..31 |
| `1508` | cleanup writes 0 |

Input initialization is `mem:0x081f0a24`. Decompilation shows the offsets and
masks but omits the unused `1804` readback; the
executed original trace captures it at
`mem:0x081f0ab4`. The frontend removes emulation-prevention bytes. Header
consumption counts include the eight-bit NAL header; the remaining header bits
are consumed before slice launch.

Plane allocations are independent, each 1 KiB aligned. Allocator
`mem:0x08300fcc` rounds luma height to 64, chroma half-height to 64 and motion
quarter-height to 64. Clearing virtual address bit 31 is specific to this
mapping.

## Reset, scalar configuration and start

| Offset | Meaning / programming |
|---|---|
| `1000` | decode start 1 |
| `1004` | full reset: OR `7ff`, preserve upper bits; slice/frontend reset `(old&~f)\|c`; poll bit 0 clear. A third form at `mem:0x082ee0f8` stores exactly `3e0` (bits 9:5) and polls the whole register until zero (bounded); which sub-block it resets is not decided |
| `1008`, `100c`, `1010`, `1014`, `1018` | sampled Baseline jobs write 0, 1, 0, 0, 0; wider mode meanings not decoded |
| `101c` | reference-picture flag |
| `1020` | sampled Baseline jobs write 0; wider meaning unknown |
| `1024` | two chroma QP offsets, five bits each |
| `1028`, `102c` | slice QP and slice type |
| `1030`, `1034` | width/height in 16×16 macroblocks |
| `1038`, `103c` | first macroblock x/y |
| `1040` | list 0 active count - 1 |
| `1044` | list 1 active count - 1 (original `mem:0x082f7df4` stores `[slice+0x48]-1`); sampled Baseline I/P jobs write `ffffffff` = empty list 1 |
| `1048` | Baseline setup clears low 6 bits |
| `104c`, `1050`, `1054`, `1058` | sampled Baseline jobs write 0; broader syntax mappings not established |
| `105c`, `1060`, `1064` | deblocking mode, alpha, beta |
| `1068` | output motion descriptor OR `02010000` |
| `106c` | packed output chroma descriptor<<8 and luma descriptor |
| `1074` | completion status; sampled jobs see 0/1, the read status is written back |
| `1078` | clear 0 immediately before launch |
| `10c0` | IRQ acknowledge 0 |
| `10c4` | setup 1; semantics unknown |
| `10cc` | reference-list preparation 0 |
| `10d0` | setup `ff`; semantics unknown |
| `10d4` | Baseline setup 2 |
| `10d8` | low 2 bits pulse 3 then 1 during setup/cleanup |
| `1808` | setup 0; semantics unknown |

The original scalar routine `mem:0x082f7d1c` accepts more syntax than the
Baseline jobs sampled; the entries above are not a map of all firmware modes.

Reference list entry `+100+(list<<7)+4*i` (list 1 at `+180`) packs
`(23+slot)<<8 | (3+slot)`, plus `0x8080` when the original's per-entry type
byte equals 2 (`mem:0x08303f84..0x08303fae`; byte meaning not decoded). Sampled
Baseline jobs use list 0 with the flag clear. Reverse map
`+4*((3+slot)&31)` contains the list index. Original `mem:0x08303f00` iterates
in reverse, so the lowest duplicate list index wins. Slice start is
`mem:0x082eddf0`; completion/acknowledge is `0x082df3c8`/`0x082f812c`.
Shared bus register `0x3e000004` bit 0 is enabled around operation; restore
only the owned bit so other engines' state is kept.

## Constraints and completion

The descriptor API has 20
frame-plane slots.

On completion:

1. Keep frame/reference ownership.
2. Read `1074`, acknowledge `10c0`, clear `1508`, pulse `10d8`.
3. Invalidate output caches before CPU access.

On error or close: disable IRQ35, acknowledge, reset and gate before freeing
DMA memory.

Not known: reset/status bit meanings, motion-plane semantics, MPEG-4 hardware
use, interlaced/B-frame modes, completion timing.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Base, gates, reset preservation, NAL readback | `osos mem:0x081f0a24` decompilation | observed (readback seen in trace and disassembly) |
| Descriptor IDs and allocation | `mem:0x082e5180`, `0x081bd952`, `0x08300fcc` | observed |
| Scalar/header/reference agreement | 75 frames / 1725 comparisons in the executed trace of the osos image | dynamic analysis, finite subset |
| Physical format limits, full field map | not established | not decoded |
