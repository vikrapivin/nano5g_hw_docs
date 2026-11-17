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
| `3400..37ff` | weight flag words, same indexing as the weights: bit 0 marks a luma weight of +128, bits 0/1 a Cb/Cr weight of +128 (the weight byte `0x80` alone is -128). The iPod OS writes them for explicit B only (`mem:0x08303144`); the decoder honours them for explicit P as well (device) |
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
entry), and MPEG-4 profile-level 3 up to 720x576. MPEG-4 Part 2 and H.263 are
decoded by a separate engine at `0x39600000`, described below; the MP4 reader
accepts `mp4v` sample entries with object type `0x20` (`mem:0x081edc78`).

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
RBSP data. `10e0` is written on the first slice of every picture, before the
plane descriptors: 0 without a second output copy, otherwise the second-output
value (`osos mem:0x082ee40c`, called from `mem:0x082e22b4`); `10e4` is written
only with a second copy.

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

Completion status `1074` as the iPod OS driver reads it (`osos mem:0x082f812c`; error messages at
`osos mem:0x0845be68..0x0845c014`; observed):

| `1074` | Meaning | iPod OS action |
|---|---|---|
| 0 or 1 | slice done | next slice |
| bit 1 | VLC error | core reset (`1004 = 0x3e0`, poll until 0, at most 10,000 polls, `osos mem:0x082ee0f8`), then the next slice of the same picture |
| bit 2 | macroblock type error | same |
| bit 3 | sub-macroblock type error | same |
| bit 4 | coded block pattern error | same |
| bit 5 | "idf" error | same |
| bit 6 | delta QP error | same |
| bit 7 | time-out | core reset, the stream stops |

If the `3e0` reset does not read back 0 within the poll limit, the driver ends the picture and skips to
the next IDR. On the device a four-slice picture now and then ends with one of bits 1..6 set; that the `3e0`
reset followed by the next slice recovers the picture is not yet device-tested.

Not known: the sub-blocks the reset bits 9:5 select, motion-plane semantics, whether the
interlace (field and MBAFF) modes the register layer programs work on this
silicon, completion timing. CABAC, B slices, weighted prediction, the 8x8
transform and scaling lists decode exactly on the device (finding ledger).

## MPEG-4 Part 2, H.263 and JPEG decoder (`0x39600000`)

A second decoding engine sits at **`0x39600000`**, **IRQ45**, OSOS clock ID
**`0x40`** (clock-table entry `mem:0x08551d80`: `[40,0,0,0,8,0]`, i.e. clear
`0x3c500068` bit 3 to enable). It is separate from the AVC engine at
`0x38f00000`. The iPod OS drives it from two clients:

- the MPEG-4 Part 2 / H.263 video decoder (open `mem:0x082d954c`, per-picture
  decode `mem:0x082d96c8`);
- the JPEG picture decoder (feeder `mem:0x080cb838`, decode loop
  `mem:0x080e44b0`), used for still pictures.

Both clients enable clock ID `0x40` and clear bit 0 of `0x3e000004` before use
(`mem:0x082d954c`, `mem:0x080cb838`); the AVC path sets that bit (see "Slice
start" above), so bit 0 selects between the two engines (inferred from the
opposite use). Both register IRQ object `0x2d`; the video client's handler is
`mem:0x082df430`, the JPEG client's `mem:0x080f585c`. The video handler
acknowledges VIC1 with `0x38e0200c = 0x2000` (bit 13 = source 45).

The register file has six windows. Offsets are relative to `0x39600000`; every
entry is a 32-bit access seen in the iPod OS code named in the last column.

| Offset | Use | Evidence |
|---|---|---|
| `00000` | top status: bit 1 block unit, bit 6 DMA unit, bit 8 VLD unit; written all-ones to clear | ISR `0x082df430` |
| `00004` | interrupt mask: `142` (video), `40` (JPEG) | `0x082dfc1c`, `0x080e44b0` |
| `0000c` | control: 0 at reset | `0x082699e4`, `0x080e44b0` |
| `00010` | configuration: `80000000` at stream set-up, then OR `(mb_width-1)<<16 \| (mb_height-1)<<8`; JPEG writes `10100` | `0x082dfc1c`, `0x082e08f0`, `0x080e44b0` |
| `0001c` | all-ones at reset | `0x082699e4` |
| `0002c` | 2 at reset | `0x082699e4` |
| `10000` | VLD status, all-ones to clear; 4 written before a picture | `0x082df7c6`, ISR |
| `10004` | VLD interrupt mask `3f` | `0x082dfc1c` |
| `1000c` | VLD start: `c` decodes the picture | `0x082d96c8` via `0x082df9e4` |
| `10010` | picture parameters: bit 0 coding type, bits 2+ and 5+ two fcode fields, bit 11 one header field, bit 17 a VOL flag, bit 10 set for short-header (H.263) pictures | `0x082df7c6` |
| `10014` | quantiser / short-header fields (MPEG-4: one 16-bit header field; H.263: two further fields packed at bits 6 and 15) | `0x082df7c6` |
| `1001c` | 1 at stream set-up | `0x082dfc1c` |
| `10034` | 0 per picture | `0x082df7c6` |
| `1009c` | `field<<8 \| bits(mb_width*mb_height)<<4 \| field` (time-increment fields, inferred) | `0x082df7c6` |
| `100a0`..`100a8` | one header byte, 0, 0 | `0x082df7c6` |
| `30004` | each read drops one bit of the input (bit position mod 8 reads) | `0x082df7c6` |
| `30020` | each read drops one byte (start address mod 4 reads, `mem:0x082df8e0`) | `0x082df7c6` |
| `30100` | block-unit status, all-ones to clear; `20` before a picture | ISR, `0x082df7c6` |
| `30104` | block-unit interrupt mask `14` | `0x082dfc1c` |
| `3010c` | JPEG block hand-off `n<<3 \| 31` | `0x080f5114` |
| `30110` | `2` (MPEG-4), `102` (H.263), `800` (JPEG reset) | `0x082df7c6`, `0x080e44b0` |
| `41200 + 4*i` | 64-word matrix: MPEG-4 default intra matrix (video), JPEG luminance table (JPEG) | `0x082dfc1c`, `0x080e44b0` |
| `41300 + 4*i` | 64-word matrix: MPEG-4 default inter matrix (video), JPEG chrominance table | as above |
| `41800`, `41808` | JPEG block command / busy bit 1 | `0x080f5114` |
| `41804` | `quant_type<<8 \| short_header<<7 \| 20` (video); `40` (JPEG) | `0x082df7c6`, `0x080e44b0` |
| `50000` | motion-compensation status, all-ones to clear | `0x082ea720` |
| `50010` | `c02 + 4*rounding` (video); `10` (JPEG) | `0x082df7c6` |
| `60000` | DMA status, all-ones to clear; bit 1 = picture done (signals the waiting thread) | ISR |
| `60004` | DMA interrupt mask `12` (video), `2` (JPEG) | `0x082dfc1c`, `0x080e44b0` |
| `6000c` | DMA start: 3 starts the input fetch | `0x082df7c6` |
| `60010` | `200` (video), `182` (JPEG) | `0x082e08f0`, `0x080e44b0` |
| `60018`, `6001c` | input start / end bus address; end = start + length + `103` | `0x082df7c6` |
| `60020`, `60024` | input buffer start / end | `0x082dfc1c` |
| `6002c`/`60034` | reference picture luma / chroma, bus address with bit 31 cleared (`lsls #1; lsrs #1`); `60030`/`60038` written 0 | `0x082dfcda` |
| `6003c`/`60044` | current picture luma / chroma, bus address with bit 31 cleared; `60040`/`60048` written 0 | `0x082dfcda` |
| `6006c` | `mb_width<<16 \| mb_height` | `0x082e08f0` |
| `60078`, `6007c`, `60080` | two buffer addresses and 0 at stream set-up | `0x082dfc1c` |

Bus addresses are written with bit 31 cleared, as for the AVC engine.

### Picture sequence (video)

`mem:0x082d954c` opens the decoder. For codec byte `0x0b` it searches the
decoder configuration for the visual-object-sequence start code `0x000001b0`
and a video-object-layer start code `0x00000120..0x0000012f`, then parses the
VOL (`mem:0x082eae9c`); for the other codec type it takes the picture size from
two big-endian 16-bit words of the configuration and marks the stream as short
header (H.263). Width and height are rounded up to macroblocks.

`mem:0x082d96c8` decodes one access unit of at most `0x40000` bytes:

1. Find the VOP start code `0x000001b6` (MPEG-4) and parse the VOP header in
   software (`mem:0x082e13f8`; short header `mem:0x082dfabc`).
2. If the picture is not coded, copy the reference picture (luma `w*h`,
   chroma `w*h/2` bytes, `mem:0x081916cc`) and finish.
3. Otherwise: reset (`mem:0x082699e4`), stream set-up with the MPEG-4 default
   matrices (`mem:0x082dfc1c`; tables at `mem:0x084a5a68`, `mem:0x084a5b68`),
   geometry (`mem:0x082e08f0`), reference and current frame stores
   (`mem:0x082dfcda`, two stores that alternate), picture registers
   (`mem:0x082df7c6`), DMA start, then the byte- and bit-skip reads that align
   the engine to the first macroblock bit, then VLD start `c`.
4. Wait for the DMA-done event with a timeout of `0x42` ticks; on timeout the
   engine is reset.

The chroma plane of a frame store is one plane of `w*h/2` bytes (inferred:
interleaved Cb/Cr). Only I and P pictures fit two frame stores; B-VOP handling
was not found.

### JPEG

The JPEG client scans markers in software (`mem:0x08205218`, `FF 00`
stuffing, restart markers modulo 8), loads the quantisation tables into the
two matrix windows (default tables at `mem:0x08466c18`, `mem:0x08466d18`) and
hands decoded blocks to the block unit (`mem:0x080f5114`); the engine writes
the output through its DMA window. Entropy decoding is done by the CPU in this
client (inferred from the software bit reader); dequantisation, inverse
transform and output are done by the engine (inferred).

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
| Explicit P with a +128 weight (default at denominator 7) decodes exactly only with the `3400`/`3480` flag words set; without them every such picture and those predicted from it differ | device: four weighted clips, every picture compared, 0 mismatches with the flags, 47..78 without | device |
| Engine at `0x39600000`, IRQ45, clock ID `0x40` = `0x3c500068` bit 3 | `mem:0x082d954c`, `0x080e44b0`; clock table `mem:0x08551d80`; ISR `0x082df430` VIC1 bit 13 | observed |
| `0x3e000004` bit 0 cleared for this engine, set for AVC | `mem:0x082d954c`, `0x080cb838` | observed; selector meaning inferred |
| MPEG-4 Part 2 and H.263 (short header) decode path | `mem:0x082d954c`, `0x082d96c8`; start codes `1b0`, `120..12f`, `1b6` as call arguments | observed (code not defined as functions in the analysed image; read with a read-only disassembly) |
| Register windows `00000`/`10000`/`30000`/`41200`/`50000`/`60000` and their writes | literal pools `mem:0x080e48d8..0x080e4908`, `0x082df46c..0x082df484`, `0x082df904..0x082df91c`, `0x082dfcb4..0x082dfcd0`, `0x082e0920`, `0x082ea73c..0x082ea748` | observed |
| Bit/byte skip by reading `30004`/`30020` | read loop `mem:0x082df8e0` | observed |
| MPEG-4 default intra/inter matrices loaded into `41200`/`41300` | `mem:0x082dfc1c`, tables `mem:0x084a5a68`/`0x084a5b68` | observed |
| JPEG still decoding uses the same engine | `mem:0x080e44b0`, `0x080f5114`, ISR `0x08106794`; JPEG tables `mem:0x08466c18` | observed; split of work inferred |
| Frame-store addresses written as the bus address with bit 31 cleared | `mem:0x082dfcda` (`lsls #1; lsrs #1`) | observed |
| No B-VOP path | bounded: the two-store sequence in `mem:0x082d96c8` | inferred |
| Engine responds on silicon | not established | not tested |
| No reference to `0x396xxxxx` in the restore OS, bootloader or diagnostics images | bounded: aligned 32-bit literal scan of the disk, bootloader and diag images (the diag modules inside their firmware volume were searched only where they are stored uncompressed) | observed negative, bounded |
| Completion status `1074`: bits 1..6 are slice-syntax errors (VLC, macroblock type, sub-macroblock type, pattern, "idf", delta QP) after which the driver resets the core with `1004 = 0x3e0` and continues with the next slice; bit 7 is a time-out | `osos mem:0x082e24a4..0x082e24f2` (one bit test per message), `mem:0x082ee0f8` | observed |
| CABAC, B slices (spatial and temporal direct, pyramid), explicit/implicit weights, 8x8 transform, custom scaling lists, up to four slices per picture and pictures up to 1280x720 decode exactly with the programming above | device: 21 clips, every picture compared with a software decoder, 0 mismatches; four-slice pictures intermittently end with an error status | device |
