# CLCD display compositor, scaler and enhancement

CLCD at **`0x38900000`** combines **two RGB layers and one video layer**,
scales/rotates the video, applies processing tables and sends the result to
[the LCD interface](lcd-controller.md). OSOS clock ID `0x3a`; interrupt
**IRQ14, kind 1** (`mem:0x08295652`). The enhancement region starts at CLCD
`+0x1b00` and is an aperture of this block.

The page covers all 111 reconstructed source accesses, including unknown masks
and dynamic array accesses. The routines named are original-OSOS reconstructions,
not an execution trace.

## Global, RGB and output registers

32-bit MMIO offsets from `0x38900000`:

| Offset | Observed use |
|---|---|
| `000` | bit 0 enable; bit 7 set in initialization. Diagnostics: reset `100` (poll bit 8 clear), then `84` (bit 7 and bit 2); enable `\|= 1` (`diag Clcd.pe+0x434`, `+0x4f8`) |
| `004` | bit 3 video enable; bits 4/5 RGB0/1 enable; bits 1/8 processing controls |
| `010` | background color (OSOS writes `231`). Diagnostics use it as three 4-bit per-layer format fields: bits 3:0 video (code 2), 7:4 RGB0 (5), 11:8 RGB1 (8) (`diag Clcd.pe+0x540..0x5a8`) |
| `014` | video format low byte plus software field shifted 17, old bits masked by `fffeff00` |
| `020+20*i` | RGB layer i control: format, fields 8/16/17/20, constant bit 21, rotation 22, direction 23 |
| `024+20*i` | RGB physical address (bit 31 stripped) |
| `028+20*i` | RGB pitch |
| `02c+20*i` | RGB extra field cleared to 0 |
| `030+20*i` | RGB width<<16 \| height |
| `034+20*i` | RGB x<<16 \| y |
| `03c`, `05c` | init `(old&fffff000)+120`; lower-field meaning unknown. Diagnostics write `00020020` to both (`diag Clcd.pe+0x434`) |
| `064`, `068`, `06c` | three color registers; OSOS writes the same object field to all three |
| `2e0`, `2e4` | video pitch<<17, preserving old `c000ffff` |
| `300` | bus configuration: bit 31 always set; fields 29, 28, 25:24, 21:20, 17:16, 0 selected by arguments |
| `304` | timing arguments `a<<16 \| b<<8 \| c`; init `00080808` |
| `400` | processing-enable bits 0/1/2 |
| `408`, `40c` | table address/select and table data |

RGB orientation:

- 0 and 2 keep the axes.
- 1 swaps x/y and sets direction.
- 3 computes `x=screen_width-height-y`, `y=old_x`, direction 0.
- Other values return.

The direction variable survives between layer iterations in OSOS. Pixel-layout
codes are firmware-defined; unused codes are not decoded.

## Video layer and scaler

The source's `VIDEO(o)` means **CLCD+100+o**; the offsets below are already
resolved from the CLCD base.

| Offset | Observed use |
|---|---|
| `114` | cold init preserves masks and installs `00200020`; fields incomplete. Diagnostics write `00020020` (`diag Clcd.pe+0x434`) |
| `118` | bit 0 rotated, bit 1 direction, bit 2 selects filter-axis assignment; cold init clears bit 2 and sets low 2 bits = 3 |
| `11c`, `120` | luma/chroma base |
| `124`, `128` | luma/chroma base+pitch, paired-line addresses |
| `130` | source crop x<<16 \| y |
| `134` | source crop width<<16 \| height |
| `138` | destination x<<16 \| y |
| `13c` | output width low 12 bits<<16 \| height low 12 bits |
| `140..1cc` | 36-word filter bank |
| `1d0..214` | 18-word filter bank |
| `220..264` | 18-word filter bank |
| `270..2b4` | 18-word filter bank |
| `2c0`, `2c4` | horizontal/vertical 16.16 sample step, masked to 20 bits |

`mem:0x0829cdc4` does fit/crop/aspect selection:

- Rotation swaps source/destination axes.
- Initial step is `(source_dimension<<16)/destination`; a zero denominator becomes 1, a computed zero step becomes `10000`.
- Nine-phase filter tables are chosen by unsigned threshold arithmetic (threshold literals and wrap behavior are in the clean scaler implementation).
- The two source-origin arguments are unused in this image.
- Fitting and cropping use signed divisions on wrapping 32-bit products.

All 324 wide-filter words and 162 narrow-filter words are verified against
firmware (wide table at `osos mem:0x086b4ebc`, narrow table at
`osos mem:0x086b53cc`).

## Gamma and enhancement programming

Table upload `mem:0x0824c9d0` preserves `408` outside mask `1f0ff`:

1. 225 entries using selector `17000`.
2. 256 values to each of banks 11-13.
3. Panel-dependent 256-entry banks 14-16.

Each write sets `408` before `40c`. The tables are at
`osos mem:0x086b2338` (`gamma_initial`, 225 words), `0x086b26bc` (`gamma_common`,
256 words) and three 256-word channel banks per panel class: `c4` at `0x086b2abc`,
`b3` at `0x086b36bc`, other at `0x086b42bc` (stride `0x400` per channel). All were
compared word for word against the image.

Enhancement offsets below are relative to **`0x38901b00`**:

| Offset/range | Observed use |
|---|---|
| `10` | bit 0 toggled; init OR `10080`. Diagnostics write `10082`; enable `\|= 1` with Stall 100 µs, disable clears bit 0 (`diag Clcd.pe+0x718`, `+0x4f8`) |
| `14`, `1c` | init 2, 2 (diagnostics the same) |
| `18` | init 0 |
| `20` | init `10110`; fields unknown |
| `24` | `(screen_width-1)<<16 \| (screen_height-1)` (diagnostics mask each half with `3ff`) |
| `28`, `2c` | diagnostics write `0` and `0xf` (`diag Clcd.pe+0x718`); not used by OSOS |
| `30` | independent bits 0/1; bit 5 ORed; roles unknown |
| `34` | bit 20 busy; low 16 bits sampled for enhancement/backlight policy |
| `38` | strength low 8 bits; bit 8 separate enable; default strength `80` |
| `3c` | low 4 bits mode |
| `50`, `54` | three 11-bit offset values packed 16/0, then 16 |
| `58`, `5c` | three 9-bit offset values packed similarly |
| `60` | three 8-bit channels packed 16/8/0 |
| `70`, `74` | secondary window start/end, 10-bit coordinates |
| `78`, `7c` | primary window start/end, 10-bit coordinates |
| `88+4*(i/2)` | eleven table fields: 8 bits for i<8, 11 bits for i>=8, alternating high/low 16 |
| `a0+4*(i/2)` | eleven 16-bit fields, alternating high/low 16 |
| `b8+12*index` | eleven bytes packed into three big-endian-order words; third word leaves low byte 0 |

Raw tables in the OSOS image (compared byte for byte): kernel, 33 words at
`osos mem:0x08465a28`; curve, 22 words at `0x084659d0`; offsets, 3 words at
`0x08465aac`; thresholds, 512 words at `0x084651d0`. Processing formulas
contain unexplained constants such as `fffff777` and `44480`.

## Initialization, frame ownership and power transitions

`mem:0x081844e8`:

1. Install lower priority fields `120` at `03c`/`05c`.
2. Change SDRAM arbitration: mask `300000` at `0x3d700080`, plus offsets `0,4,8,c,10,14,18,1c,3c` set to `0,200000,200000,200000,100000,0,0,0,200000` respectively.
3. Enable gate `3a`; disable enhancement and CLCD; load gamma; enable processing 4; set background 231.
4. Bus config `0,0,010101,1`; timing 8/8/8; set control bit 7; disable all layers.
5. Enhancement init literals as above.
6. Enable CLCD, create release/suspend events, bind IRQ14.
7. Configure LCD `78=000a000a`, `74=width/height`, `70=1`, and the panel window.

Submission retains pending buffers. On IRQ `mem:0x0829cd02` the routine checks
OS tick delta > 10, waits for GRAM completion when updating, and signals a
release worker that releases/requeues references. An IRQ alone does not make
still-displayed memory reusable.

Panel GRAM begin `mem:0x082ae27c(force)`:

- force=0 (IRQ path via `0x082a8144`): returns -1 at once if LCD `8c` low 2 bits are non-zero; the frame stays pending.
- force=1 (`0x082a8090`): spins (`0x083b6550`) until LCD `8c` low 2 bits are clear, before and after the sequence.
- Sequence: LCD `80=1`, panel window (orientation index derived from CLCD rotation 0/0x5a/0xb4/0x10e), LCD `80=0`.

Pending/complete/suspended flags are software fields.

Save `mem:0x08155db0` reads all `0x400` CLCD words (offsets `000..ffc`), then
`0x30` enhancement words (`000..bc`), and disables enhancement. Restore
`mem:0x0814bcfc` reinitializes, writes the saved regions back, reloads gamma
and marks pending. Firmware therefore touches words in those entire ranges that
are not in the register tables above.

## Diagnostic CLCD driver (`diag Clcd.pe`)

Initialization order (`diag Clcd.pe+0x434`, `+0x5b4..0x640`, `+0x688`):

1. CLCD `+04 = 0`; Stall 50 µs.
2. Clear `0x3c500048` bit 14 and `0x3c500068` bits 14, 11, 10, 9 (mask `0xb1ff`).
3. GPIO port 1 PCON (`0x3cf00020`) nibble 6 (bits 27:24) = 3 (PreEfi nibble is 0); Stall 1 µs.
4. DisplayPlatform info; CLCD reset `+00 = 0x100`, poll until bit 8 clears (20 µs steps).
5. LCD/CLCD setup ([lcd-controller.md](lcd-controller.md)); enhancement values as in the register tables; Stall 2 µs.
6. DisplayPlatform method 1 (width, height).

Layer enable bits in `+04`: bit 3 video, bit 4 RGB0, bit 5 RGB1 (same as OSOS). Video-layer setup (`+0x76c`):

| Offset | Diagnostics write |
|---|---|
| `2e0`, `2e4` | `0x50`, then pitch<<17 under mask `3fff0000` |
| `130`, `134` | `0`; `pitch<<16 \| height` (mask `0fff0fff`) |
| `11c..128` | base, base + h*pitch, base + pitch, base + h*pitch + pitch |
| `118` | `3` |
| `2c0`, `2c4` | `(dim<<16)/dst`, masked `fffff`; horizontal step > `0x80000` or vertical step > `0x40000` returns EFI invalid parameter |
| `13c`, `138` | `(W&fff)<<16 \| H&fff`; `0` |

Filter tables: 9 wide sets of 36 words at `Clcd.pe+0xb08`, 9 narrow sets of 18 words at `Clcd.pe+0x1018`. Class selection
(`+0x76c`) uses 7 wide and 8 narrow classes with the threshold literals `0x3ffff 0x15555 0xaaa9 0x6666 0x4443 0x30c2 0x2491`
applied to the step plus an offset (not decoded further). The tables equal the OSOS tables except wide word 192
(`0x00020ff8`, OSOS `0x00020ff9`), wide words 288..323 (set 8; a real filter row set, zero in OSOS), narrow word 118
(`0x0ffb005d`, OSOS `0x0ffd005b`) and narrow word 143 (`0x00460ffa`, OSOS `0x00460ffb`).

The [scaler block](scaler.md) at `0x39000000` repeats this filter-bank layout.

## Unknowns

- Diagnostic per-layer nibbles in `010` versus the OSOS background use of the same offset: not reconciled.
- Diagnostic classes of the filter threshold chain: not decoded.
- TV output and alternate CLCD modes are not covered here.
- Some processing fields are named by bit position only.
- Pixel-clock limits, filter precision and tearing timing are not known.
- 3,664 ordered setup writes for six cases were captured by dynamic analysis of the osos image; composed RGB pixels were not captured.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Register packing and buffer ownership | clean LCD library, original VAs above | observed; finite differential tests |
| Filter/gamma/enhancement table values | addresses given above | observed (byte comparison) |
| Unknown writes and save/restore ranges | the 111-row source-access inventory | listed, meaning unknown |
| Diagnostic init values, layer nibbles, video-layer setup | `diag Clcd.pe+0x434`, `+0x540..0x5a8`, `+0x718`, `+0x76c` | observed |
| Diagnostic filter tables differ in 4 places | `diag Clcd.pe+0xb08`, `+0x1018` vs OSOS tables | observed (byte comparison) |
| Optical/electrical specification | not derivable from current coverage | not decoded |
