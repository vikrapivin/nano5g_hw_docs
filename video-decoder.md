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
| `1008` | `field_pic_flag` (inferred from the slice field) |
| `100c` | 1 when `bottom_field_flag` is 0 (inferred) |
| `1010` | `direct_spatial_mv_pred_flag` (inferred) |
| `1014` | MBAFF: `mb_adaptive_frame_field_flag` and not a field picture (inferred) |
| `1018` | `constrained_intra_pred_flag` |
| `101c` | reference-picture flag |
| `1020` | `entropy_coding_mode_flag`: 0 CAVLC, 1 CABAC |
| `1024` | `chroma_qp_index_offset` in bits 4:0, `second_chroma_qp_index_offset` in bits 9:5 |
| `1028`, `102c` | slice QP and slice type |
| `1030`, `1034` | width/height in 16×16 macroblocks |
| `1038`, `103c` | first macroblock of the slice: `1038` row (`first_mb / width_mbs`), `103c` column (`first_mb % width_mbs`); `mem:0x082eddf0` computes both with one divide |
| `1040` | list 0 active count - 1 |
| `1044` | list 1 active count - 1 (original `mem:0x082f7df4` stores `[slice+0x48]-1`); sampled Baseline I/P jobs write `ffffffff` = empty list 1 |
| `1048` | low 6 bits replaced, upper bits kept: co-located picture of a B slice (below); 0 for I/P |
| `104c` | `weighted_pred_flag` |
| `1050` | `weighted_bipred_idc` (0 none, 1 explicit, 2 implicit) |
| `1054`, `1058` | slice `+0xbc/+0xc0`, the luma/chroma log2 weight denominators (inferred; 5 for implicit weights) |
| `105c`, `1060`, `1064` | deblocking mode, alpha, beta |
| `1068` | bits 7:0 output motion descriptor (`43+slot`); bits 15:8 motion descriptor of list-1 entry 0 for a B slice, else 0; bits 31:16 `0x0201` in every slice observed |
| `106c` | packed output chroma descriptor<<8 and luma descriptor |
| `1074` | completion status; sampled jobs see 0/1, the read status is written back |
| `1078` | clear 0 immediately before launch; during header parsing, Exp-Golomb command (below) |
| `107c` | Exp-Golomb result (below) |
| `10c0` | IRQ acknowledge 0 |
| `10c4` | setup 1; semantics unknown |
| `10cc` | `cabac_init_idc` (Baseline jobs: 0) |
| `10d0` | setup `ff`; semantics unknown |
| `10d4` | bit 0 `profile_idc == 100`; bits 2:1 `chroma_format_idc` (Baseline: 1, so the register reads 2); bit 3 `transform_8x8_mode_flag` |
| `10d8` | low 2 bits pulse 3 then 1 during setup/cleanup |
| `1808` | setup 0; semantics unknown |

The scalar routine `mem:0x082f7d1c` writes every register above for every
slice; the syntax-element names come from tracing each source field back to
the header parsers. The weighting condition in that routine (explicit weights
when `weighted_pred_flag` is set on a P/SP slice, or `weighted_bipred_idc` is 1
on a B slice; implicit set-up when it is 2 on a B slice) matches the H.264
`pred_weight_table` rule and confirms the `104c`/`1050` assignment and slice
type 1 = B. That the silicon decodes CABAC, B, interlaced or High-profile
streams correctly is not tested.

### Exp-Golomb and table windows

| Window | Use |
|---|---|
| `1078 = 1` / `1078 = 3` | decode one `ue(v)` / `se(v)` code (`mem:0x0820d37c`, `0x081fb944`): poll `107c` bit 7; `107c[5:0]` is the code length; with bit 6 clear the value is `107c >> 8`, with bit 6 set the code is long and the value is read through the raw-bit window `1400 + 4*(len+1)` |
| `0400..07ff` | prediction weights (`mem:0x08303144`), layout below |
| `3400..37ff` | weight flag words, same indexing as the weights |
| `0200 + 4*i` | temporal-direct scale factor per list-0 entry (12 bits) |
| `2800`, `2900` | 8x8 scaling lists, Intra Y and Inter Y, 64 words |
| `2a00 + 0x40*k` | 4x4 scaling lists `k` = Intra Y, Cb, Cr, Inter Y, Cb, Cr, 16 words |

The scaling lists are written by the slice start `mem:0x082eddf0` only when the
SPS `profile_idc` is at least 100 (fallback rule `mem:0x082f7fd4`).

### Header limits enforced in software

The SPS, PPS and slice-header parsers (`mem:0x082f7910`, `0x082f75d4`,
`0x082f86f0`/`0x082f816c`, dispatch `0x082ee4c0`) are code not defined as functions
in the analysed image; the limits below were read from their bytes. Each
violation sets a bit in an error word; any set bit makes `mem:0x081f0a24`
return failure.

| Element | Accepted |
|---|---|
| `profile_idc` | 66, 77, 88, 100; 110, 122, 144 parsed but flagged |
| `level_idc` | 10..13, 20..22, 30..32, 40..51 |
| `seq_parameter_set_id` | 0, 1 |
| `log2_max_frame_num_minus4`, `log2_max_pic_order_cnt_lsb_minus4` | <= 12 |
| `pic_order_cnt_type` | 0..2 |
| `num_ref_frames` | <= 16 |
| `pic_parameter_set_id` | 0..15 |
| slice groups | rejected only together with profile 77 |
| `num_ref_idx_*_active` | <= 32 (field) / 16 (frame) |
| `cabac_init_idc` | 0..2 |
| slice QP | 0..51 |
| slice type | 0..9 (all five types) |

Not rejected by the parsers: CABAC, B slices, interlace (`frame_mbs_only_flag
= 0`, MBAFF), 8x8 transform, scaling matrices, weighted prediction, picture
size. The DPB size is derived from the level and the frame size
(`mem:0x082f7c34`, table `mem:0x08467c94`) and capped at 16 frames. The playback
capability list that the device reports (`mem:0x0800784c`) is narrower: H.264
profile Baseline level 3.0 up to 720x480 (640x480 for the low-complexity
entry), and MPEG-4 profile-level 3 up to 720x576. No MPEG-4 Part 2 decode path
was found (bounded search: codec strings, MPEG-4 start codes, IDCT constants);
the MP4 reader accepts `mp4v` sample entries with object type `0x20`
(`mem:0x081edc78`).

## Main and High profile programming

The rules below come from the register layer of the iPod OS, read in its
decompilation and confirmed by executing that code (dynamic analysis of the
image) on 33 test streams covering CABAC, several slices per picture, B
slices with spatial and temporal direct prediction, reference B pictures,
explicit and implicit weights, the 8x8 transform and custom scaling lists
(1,866 slices). The silicon's decoding of these modes is not tested.

**Sequence.** Every slice is a separate job: parser reset (`1004`), NAL window,
header bits, scalar block, `1078=0`, `1000=1`, one IRQ and one completion
(`mem:0x082e22b4`). All slices of a picture use the same output slot and the
same `1068`/`106c`. The order within the slice start `mem:0x082eddf0` is:
scaling lists, `1038`/`103c`, weights, `10cc`, lists, `1008..1064` in
ascending order, `1068`, `106c`, `10d4`. Scaling lists are written on the
first slice of each picture, only when the SPS `profile_idc` is at least 100;
lists, reverse map, scale factors and `1048` are rewritten on the first slice
or when the slice modifies its lists. Writing every block on every slice is a
superset of this behaviour.

**Reference lists** (`mem:0x08303f00`). Entry `+100+(list<<7)+4*i` (list 1 at
`+180`) packs `(23+slot)<<8 | (3+slot)` for the final list after
initialisation and modification, duplicates kept. `0x8080` is added when the
picture's structure byte equals 2 (`mem:0x08303f84..0x08303fae`); it is 0 for
every frame reference observed (the bottom-field reading is inferred). The
reverse map `+4*((3+slot)&31)` holds the lowest list-0 index of each slot; it
is built from list 0 only, also for B slices.

**Temporal-direct scale factor.** For each list-0 entry of a B slice,
`0200+4*i` holds `clip(-1024, 1023, (tb*tx + 32) >> 6) & 0xfff`, with
`td = clip8(POC(list1[0]) - POC(list0[i]))`, `tb = clip8(POC(cur) -
POC(list0[i]))` and `tx = (16384 + |td/2|) / td`. When `td` is 0 or the list-0
picture is long-term the word is `0x400`. The engine receives no picture order
counts: they reach it only through this window and the implicit weights.

**Co-located picture** (`1048` bits 5:0, B slices). Bit 0 co-located picture is
frame coded, bit 1 co-located MBAFF, bit 2 co-located parity (which field POC is
nearer the current picture), bit 3 co-located picture is short-term, bit 4
`direct_8x8_inference_flag`, bit 5 `frame_mbs_only_flag`. Observed `0x3d` for
every progressive B slice; bits other than 2 and 3 are inferred from their
sources. The co-located motion is read through the list-1 entry 0 motion
descriptor named in `1068` bits 15:8.

**Weights.**

| Case | Words |
|---|---|
| P, `weighted_pred_flag` | luma `0400+4i = w \| o<<8`; chroma `0480+4i = wCb \| oCb<<8 \| wCr<<16 \| oCr<<24` (low bytes of the signed values); no flag words |
| B, `weighted_bipred_idc = 1` | the same packing for list 0 at `+4i` and list 1 at `+0x100+4i`; flag words `3400+4(64l+i)` bit 0 = luma weight is 128, `3480+4(64l+i)` bit 0 / bit 1 = Cb / Cr weight is 128 |
| B, `weighted_bipred_idc = 2` | `w0` only, four list-0 entries per word: byte `l0&3` of `0400+4((l1<<3) + (l0>>2))`; `w1 = 64 - w0` is implied; flag word at `3400` with the same index, bit k = `w0` of entry `4g+k` is 128. `w0 = 64 - (DistScaleFactor>>2)`, or 32 when `td` is 0, either picture is long-term or the value is outside -64..128 |

`1054`/`1058` carry the luma/chroma log2 denominators for explicit weights and
5/5 for implicit weights. A weight of 128 (the default with denominator 7) is
stored as byte `0x80` together with its flag.

**Scaling lists.** One matrix entry per 32-bit word in raster order (already
de-zigzagged), after the fall-back rules are resolved; flat 16s are uploaded
for High-profile streams without matrices.

**Other per-slice values.** `1010` direct spatial flag (B), `101c` 1 for every
reference picture including reference B pictures, `1044` list-1 count minus 1
(B) or `ffffffff`, `10cc` `cabac_init_idc` (0 for CAVLC and I slices). The
slice header is consumed to its last bit; CABAC alignment bits are left to the
engine. The PPS High-profile tail is parsed only while `1508` reports more
RBSP data. `10e0`/`10e4` are written only when the iPod OS requests a second
output copy of the picture.

Slice start is `mem:0x082eddf0`; completion/acknowledge is
`0x082df3c8`/`0x082f812c`.
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
use, whether the CABAC, B-slice, interlace and High-profile modes the register
layer programs work on this silicon, completion timing.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Base, gates, reset preservation, NAL readback | `osos mem:0x081f0a24` decompilation | observed (readback seen in trace and disassembly) |
| Descriptor IDs and allocation | `mem:0x082e5180`, `0x081bd952`, `0x08300fcc` | observed |
| Scalar/header/reference agreement | 75 frames / 1725 comparisons in the executed trace of the osos image | dynamic analysis, finite subset |
| Mode registers `1008..1024`, `104c..1058`, `10cc`, `10d4` | `mem:0x082f7d1c` writes; field origins in the parsers | observed writes; `1008..1014`, `1054/1058` inferred |
| Exp-Golomb command `1078`/`107c` | `mem:0x0820d37c`, `0x081fb944` | observed |
| Weight and scaling-list windows | `mem:0x08303144`, `0x082eddf0`, `0x082f7fd4` | observed |
| Header limits; CABAC/B/interlace/High not rejected | `mem:0x082f7910`, `0x082f75d4`, `0x082f86f0` | read from bytes of undefined code |
| Capability list narrower than parser limits | `mem:0x0800784c` | observed |
| `1038` is the first-macroblock row, `103c` the column | `mem:0x082eddf0` divide; executed with `first_mb` 37, width 20: `1038=1`, `103c=17` | observed, dynamic analysis |
| Per-slice sequence; lists, scale factors, weights, scaling-list and `1048`/`1068` layouts | `mem:0x082e22b4`, `0x082eddf0`, `0x08303f00`, `0x08303144`; 33 streams, 1,866 slices executed | observed, dynamic analysis; `1048` bits 0, 1, 4, 5 inferred |
| Physical format limits on silicon | not established | not tested |
