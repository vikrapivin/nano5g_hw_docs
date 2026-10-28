# Video encoder engine

The encoder is at **`0x3e400000`**, **IRQ59**, OSOS clock ID `0x42`. It is a
separate engine from the AVC decoder and the JPEG encoder. Before recording it
runs an embedded `0x4760`-byte program uploaded by DMA. That program is ARM code
for a processor inside the encoder (see [Embedded firmware](#embedded-firmware)).
The commands and register packing are listed here; some configuration
fields are not decoded.

Offsets below are 32-bit MMIO relative to the encoder base. Addresses such as
`086bdbf4` are OS event objects in RAM, not registers. The
tables cover all 66 reconstructed MMIO expressions of the register layer,
including fields of unknown meaning.

## DMA, command, reset and completion registers

| Offset | Observed use / values |
|---|---|
| `000` | write 1 to start internal DMA/program transfer |
| `00c` | DMA transfer length in 32-bit words (`bytes>>2`) |
| `014` | DMA source address |
| `018`, `01c` | output ring start and end (`start+bytes`) |
| `020` | DMA setup writes 0; meaning unknown |
| `024`, `034` | previous/current output cursor; getter synchronizes `024` to `034` |
| `044`, `048` | reset writes 1 to each (order below); bit meanings incomplete |
| `108`, `134` | enable writes 1; disable writes 0 |
| `10c` | idle request 1; after idle event write 0 |
| `110` | set 1 around program/DMA load, clear after event |
| `114` | command trigger 1 |
| `118`, `11c` | actual width/height, without macroblock padding |
| `130` | disable pulses 1 then 0 |
| `138`, `600` | reset writes 0; function unknown |
| `500` | IRQ/reset initialization writes 0 |
| `508` | IRQ acknowledge/reset writes 1 |
| `514` | IRQ status: bit 5 idle, bit 7 DMA event, bit 8 command event |
| `518` | reset writes `1a2`; fields not fully decoded |
| `a00` | two 16-bit partition counts |
| `a04` | status result; not fully decoded |
| `a08` | two 16-bit header lengths |
| `a0c` | encoded output byte count |
| `c0c` | `(stream<<8) \| command`; command 1 header, 2 frame submission |

ISR `mem:0x08133018` reads `514`, writes `508=1`, then signals only the
**first** matching event, in priority 5, 7, 8. The three RAM events are
`086bdbec`, `086bdbf0`, `086bdbf4` respectively. IRQ installation is in
`mem:0x0812d21c`.

Reset `mem:0x0812d014` writes, in order:
`048=1`, `500=0`, `508=1`, `518=1a2`, `138=0`, `044=1`, `600=0`.

Program load `mem:0x08120bf0`:

1. IRQ reset.
2. Start DMA with `110=1`; wait for its event.
3. Clear `110`, then reset.

Source is RAM `086bf000`, length read from `086bdbe4` (`4760`).

## Configuration and buffer fields

Arguments keep the original packing; unnamed ones are not decoded.

| Offset/range | Written value / interpretation |
|---|---|
| `300`, `308` | `(coding<<2)\|mode`; `308` value is independent |
| `310` | `(b<<2)\|a` |
| `314` | `(motion_a<<7)\|(motion_b<<2)\|mode`; negative motion values use a five-bit complement-plus-one conversion |
| `31c`, `320`, `324`, `328`, `344` | five cached buffer/configuration values; meanings incomplete |
| `32c` | `(a<<8)\|(b<<5)\|(c<<4)\|(d<<1)\|e` |
| `330` | `(enable<<31)\|(value<<8)` |
| `334`, `338` | reference-write plane 0 and plane 0 + padded_width*padded_height |
| `33c`, `340` | reference-read planes in the same layout |
| `348` | metadata address OR `80000000` (bit 31 is an enable, not an alias-removal bit) |
| `34c..3c8` | 32 matrix words, 4-byte stride |
| `3cc`, `3d0` | `(enable<<31)\|value` |
| `3d4` | `(repeat<<1)\|mode` |
| `800`, `804` | input planes |
| `810`, `814` | cached `810` value; scratch address at `814` |
| `900` | slice packing `(a<<9)\|(b<<8)\|c`; QP setter replaces low 6 bits without masking the caller QP |
| `904` | configured value OR `10000` |
| `908`, `910` | cached slice values; fields incomplete |
| `90c` | `(max_qp<<8)\|min_qp`; invalid bounds use `3301`; valid when 0 <= min <= max < 52 |
| `914` | `(a<<3)\|(b<<2)\|(c<<1)\|d` |
| `918` | read low 6 bits for last QP |
| `c00` | low 2 bits mode; bits 16 and 24 have independent setters |
| `c04` | four byte fields set independently, byte 3 at bits 31:24 |

- Dimensions are cached padded as `(n+15)&~15`; only actual dimensions go to `118`/`11c`.
- Two alternating read/write references are selected by repeat bit 0.
- The encoder object holds the reference plane array at RAM object `+88` and the IRQ object at `+110` (not MMIO offsets).
- Config application takes a `0x164`-byte software record and maps its fields to the config offsets above.

Enable: `108=1`, then `134=1`.

Command: write `c0c`, write `114=1`, wait on the command event. The output
getter returns `024` minus `018`, plus `a0c`, advancing `024` if needed.

Disable:

1. `10c=1`, wait for idle, clear `10c`.
2. `134=0`.
3. Pulse `130` = 1 then 0.
4. `108=0`.

Buffers stay owned until completion/idle, not just until submission returns.

Clock entry `mem:0x08551db0` enables masks `80` at `0x3c500058` and `30000` at
`0x3c500068`, active-low.

## Unknowns

The preset dispatcher, six-slot delayed pipeline and session lifetime are
host software above the engine. Physical throughput, fault-bit meanings and the
engine fields listed as unknown in the firmware sections are not known.

## Embedded firmware

The uploaded program (`osos mem:0x086bf000`, `0x4760` bytes) is not microcode
in a private instruction set: it is a 32-bit ARM image (ARMv4/v5 instruction
encodings, no Thumb), built with the ARM RealView C library, that runs on an
ARM processor inside the encoder block. Its first nine words are the ARM
exception vector table (`b` to reset, undefined, SWI, prefetch abort, data
abort, a `b .` in the reserved slot, IRQ, FIQ). The undefined, SWI, abort,
IRQ and FIQ handlers all return immediately, so the firmware runs without
interrupts. The C-library semihosting call (`swi 0x123456`) in the start-up code
is never reached on hardware.

Layout and memory map as seen by the embedded processor (observed in the
firmware image; names are ours):

| Range | Use |
|---|---|
| `0x00000000..0x0000475f` | code and literal pools (the uploaded image, executed at address 0) |
| `0x00004760..0x00005fdc` | zero-initialised data and the context block (`0x2ec` bytes allocated at start-up), stack above |
| `0x00200000` | mode/motion engine output: read per macroblock (`+00..+3c`), only `+a0` written (role inferred; details below) |
| `0x00500000..0x00500c0c` | the host-visible register file: the same offsets the CPU writes at `0x3e400000+off` appear here at `0x00500000+off` |
| `0x00520000` | bitstream writer and entropy coder: raw-bit, Exp-Golomb and kick ports, `+34` polled until it reads 1, `+44` bit counter, `+78` escape-byte count (details below) |
| `0x00550000` | picture/macroblock engine: `+00` picture size from host `600>>1`, `+18/+24` from host `120/124`, `+1c=4`, `+20=1`, `+4c` from host `810`, `+140..+190` per-frame parameters |
| `0x00560000` | reference and prediction set-up (role inferred; details below) |
| `0x00600000` | table RAM: running bit count, a 32-entry per-slice statistics table and vector rows (inferred; details below) |

Host registers from the firmware's side (all reads of `0x005xxxxx` in the
firmware are reads of the host-written words at the same offset):

| Host offset | Firmware use |
|---|---|
| `000`, `008` | written 1 / 3 then 7 during start-up and at picture start (engine enable and mode, inferred) |
| `010` | `((bitstream_engine+a8 & 0x1f) + 7) >> 3` at start-up |
| `03c` | polled until below 8 at start-up (ready) |
| `10c` | abort request: non-zero forces command 4 |
| `114` | doorbell: the command loop reads `c0c` only while this is non-zero |
| `118/11c`, `120/124` | per-picture parameters read at command time; `120/124` copied to the picture engine `+18/+24` |
| `300` | bits `[1:0]` profile (0 Baseline `0x42`, 1 Main `0x4d`, 2 High `0x64`), bits above = `level_idc` (SPS writer `0x68c`) |
| `314` | deblocking: `[1:0]` disable_deblocking_filter_idc, `[6:2]` beta offset, `[11:7]` alpha offset |
| `3d4` | bit 0 force-picture-type enable, `[2:1]` forced type |
| `31c..330`, `344`, `348`, `3cc/3d0` | slice and rate parameters copied into the context at each frame command: `324` minus `0x780` clamped at 0; `3cc/3d0/348` as sign bit plus 31-bit magnitude; `330` bit 31 and bits `[3:0]` (rate path below) |
| `50c`, `510` | written 1 after the header command / at start-up (completion flags, inferred) |
| `600` | picture size source (`>>1` into the picture engine) |
| `810` | copied to the picture engine `+4c` |
| `900` | bits `[5:0]` and bit 8 (QP and a flag) |
| `904`, `908`, `910` | read at frame start and copied to the picture engine `+144/+148/+150` |
| `90c` | bits `[5:0]` = QP; clamped against the min/max QP fields of the context |
| `918` | written with `picture_engine+158 & 0x3f` (QP used) |
| `a00` | frame result (CAVLC): intra macroblock count in the upper half, non-skipped inter count in the lower half (`0xe64`); markers `0x87301018` at start-up, `0xdeaddead` after the second start-up stage, `0xbad0cafe` after command 5 |
| `a08` | after a header command: `(bits_a >> 3) << 16 | (bits_b >> 3)`, two header lengths in bytes |
| `a0c` | after header and frame commands: total bits so far `>> 3` (output byte count); `a10` the same minus a running value |
| `c08` | bit 31 and bits `[30:0]` copied into the context at each frame |
| `c0c` | command word: bits `[15:8]` stream index, bits `[7:0]` command |

Command loop (firmware `0x1f98`, wait routine `0x1f08`): the processor spins
until `114` is non-zero, then takes `c0c`; if `10c` is non-zero the command
becomes 4 regardless. The loop accepts stream indices 0..8.

| Command | Handler | What it does |
|---|---|---|
| 0, 3 | none | ignored |
| 1 | `0x1ef8` | header: resets the bitstream engine (`+30 = 0x32`, `+44/+78/+b4 = 0`), writes the sequence and picture headers (two bit counts reported in `a08`), optionally runs the table set-up routines, reports the byte count in `a0c`, sets `50c = 1` |
| 2 | `0x1ee4` | encode one picture: frame set-up (`0x1834`: copies the host configuration, clamps QP, selects reference slot 0/1/2), then one pass per slice (`0x1550`, slice header `0x352c`, the entropy-coder handler selected through a function pointer: CAVLC `0x1ca8` or CABAC `0x1d88`, which loops over the macroblocks; `0x1104`), then rate-control bookkeeping (`0xf3c`, writes `a0c/a10`), and the per-stream rate-control state (21 words at `context[0] + stream*0x54`, restored at the next frame) |
| 4 | `0x1e9c`, `0x1e8c` | stop: tear down, then an endless loop (the host resets the engine) |
| 5 | `0x1ebc` | per-stream parameter update (`0x248`), then `a00 = 0xbad0cafe` |

This matches the host driver, which uses command 1 for the header and
command 2 per frame (`osos mem:0x0812cd80`, `0x0812d080`) with the stream index
in bits `15:8` of `c0c` and `114 = 1` as the doorbell.

The image carries no strings of its own beyond the C library's run-time
error messages. The engine blocks, header writer, macroblock loop and rate
control follow.

## Firmware engine blocks, header writer, macroblock loop and rate control

Everything below comes from the 112-function disassembly of the uploaded program. Addresses are firmware addresses
(`firmware 0xADDR`). Value expressions are **observed** unless marked *inferred*. Register roles are inferred from
the access pattern, because the firmware contains no names.

Two entry points are stubs. `firmware 0x1ef8` (header) loads the context and jumps to the body at `firmware 0xbec`;
`firmware 0x1ee4` (frame) jumps to the body at `firmware 0x18c4`. Decompiler listings of these two merge stub and body.

### Firmware state used below

| Name | Where | Meaning |
|---|---|---|
| context (ctx) | block allocated at `firmware 0x1f78`, size `0x2ec` | base pointer; `ctx+0x00` points to the result records |
| P | `ctx+0x27c` = `ctx+0x18c`, `0xe8` bytes | per-stream parameters, filled from host registers by `0x924` and `0x3b88` |
| S | `ctx+0x278` = `ctx+0xac`, `0xe0` bytes | per-macroblock decoded fields, filled by `0x3688` from block `0x200000` |
| rate state | `ctx+0x20..+0xa8` | rate-control variables, saved per stream in the result record (see below) |
| stream slots | `ctx+0xa8`, eight slots of `0x58` bytes | P is copied in by `0x1674` and out by `0x368`; slots 4..7 hold the pre-frame backup (`stream+4`, `firmware 0x18e4`) |

| Frame type `ctx+0x298` | Meaning | Evidence |
|---|---|---|
| 2 | IDR picture (NAL `0x65`) | `0x1340` |
| 0 | I picture, not IDR (NAL `0x61`, I slice) | `0x1340` |
| 1 | P picture (NAL `0x61`, P slice) | `0x1340` |

Type choice (`firmware 0x1610`): the first frame and every `period+1`-th frame is type 2, otherwise 1, where
`period = P+0x18` = host `308`. Host `3d4` overrides it: bit 0 enables, bits 2:1 give the forced type 0, 1 or 2
(`0x3b88` `P+0x48/+0x4c`). The frame counter since the last IDR is `P+0xb8`, the total frame counter `P+0xbc`;
both advance in `0xd10`.

## Register blocks

### Block `0x00520000` : bitstream writer and entropy coder (inferred)

Every header and slice-header bit goes through it. Raw bits use `+3c` (count), `+40` (value), `+38` (strobe) after
`+34` reads non-zero (`firmware 0x4374`). Exp-Golomb codes use `+0c` after `+34` reads exactly 1: `value<<16`
is `ue(v)` (`0x3894`), `value<<16 | 1` is `se(v)` (`0x3874`). The use sites confirm both (ue for sizes and
macroblock type, se for QP deltas).

| Offset | Access | Value / meaning | Routine | Stage |
|---|---|---|---|---|
| `+00` | W | `4` raw-bit mode; `0x10` CABAC slice init (after `+48`, `+4c`); `1` frame start | `0x68c`, `0x5c0`, `0x352c`, `0x4084` | header, slice, frame set-up |
| `+04` | W | kick: `4` macroblock residual/syntax (waits `+20`); `0x800` motion data of a P macroblock (waits `+50`); `0x80` intra prediction modes (waits `+50`); `0x10000` CABAC macroblock (waits `+20`) and end of slice | `0x3d04`, `0x38b0`, `0x1d88` | per macroblock |
| `+08` | W | entropy mode: `4` when `P+0x24 = 0` (CAVLC), `0x80` when 1 (CABAC) | `0x4084` | frame set-up |
| `+0c` | W | Golomb data port: `v<<16` ue, `v<<16\|1` se, `v<<16\|2` / `v<<16\|3` coded-block-pattern code (3 for inter, 2 for intra macroblocks; *inferred* me(v) tables) | `0x3894`, `0x3874`, `0x3d04`, `0x1340` | header, slice, macroblock |
| `+10` | R/W | macroblock address `ctx+0x2ac` written each macroblock; low 3 bits cleared at frame start | `0x4060`, `0x4084` | per macroblock, frame set-up |
| `+14` | W | `P+0x08 \| P+0x0c<<16` (padded width and height in pixels) | `0x4084` | frame set-up |
| `+20` | R | macroblock done (polled non-zero) | `0x1d88`, `0x3d04` | per macroblock |
| `+30` | R/W | writer control: `0x32` at header start; `&0x1f` before flush and padding; `\|0x20` after final flush | `0xbec`, `0x25c`, `0xf3c`, `0xe8c` | header, end of frame |
| `+34` | R | ready flag for the data ports | `0x4374`, `0x3894` | all |
| `+44` | R/W | **total bits written** (*inferred*: `&7` is the bit position in `0x528` and `0x1104`, `>>3` gives bytes in `0x19d4`); zeroed at each header NAL and at frame end | `0x68c`, `0x5c0`, `0xf3c`, `0x1104` | all |
| `+48` | W | CABAC slice init: `P+0x40<<5 \| P+0x24<<3 \| mb_row<<15 \| mb_col<<8`; CAVLC macroblock: `skip<<7 \| cbp<<1 \| flag` | `0x352c`, `0x3d04` | slice, macroblock |
| `+4c` | W | CABAC slice init: `P+0x54<<12 \| frame_type<<8 \| P+0x44<<6 \| slice_qp \| 0x800000`; CAVLC macroblock: macroblock type `S+0x0c` | `0x352c`, `0x3d04` | slice, macroblock |
| `+50` | R | motion/intra data kick done | `0x3d04`, `0x38b0` | per macroblock |
| `+54` | W | `0`, then `1` before the final `0x10000` kick; `0` after (end-of-slice request) | `0x1d88` | end of slice |
| `+58` | W | `P+0x34` (macroblock count of the picture) | `0x4084` | frame set-up |
| `+5c` | W | `1` at frame set-up, `0` at frame end (busy) | `0x1834`, `0xd10` | frame |
| `+68`, `+6c` | R | bit alignment state `8 / 0x10 / 0x18`; byte count that decides how many `00 00 03` units the tail needs | `0x25c`, `0xe8c` | end of frame |
| `+70` | W | `P+0x5c` (active reference count) | `0x4084` | frame set-up |
| `+78` | R/W | **escape-byte count**: bit total is `+44 + +78*8` everywhere (*inferred* emulation-prevention bytes); clamped to 1 in `0x1340`; zeroed at each header and at frame end | `0x68c`, `0x1104`, `0xf3c` | all |
| `+b4` | W | `0` | `0xbec`, `0xf3c` | header, end of frame |
| `+c8` | W/R | handshake: write `1`, poll until `0`. Per macroblock fetch in `0x4060`; also at the last macroblock in `0x1104` | `0x4060`, `0x1104` | per macroblock, end of frame |
| `+a8`, `+dc`, `+f0` | R | start-up and flush-done status (`+a8&0x1f` feeds host-side offset `010`; `+dc` and `+f0` polled in the final flush) | `0x2018`, `0x25c` | start-up, end of frame |

Unknown: bit meanings of `+20`, `+50`, `+30` low five bits, `+dc`, `+f0`.

### Block `0x00550000` : picture / macroblock pipeline parameters (inferred)

Frame-level configuration and rate-control feedback. The upper half (`+140..+190`) mirrors host `900..944`
with a fixed offset of `0x7c0` for the host copy done in `0x4084` (`900->140`, `904->144`, `908->148`, `90c->14c`,
`910->150`, `914->154`, `91c->15c` … `944->184`).

| Offset | Access | Value / meaning | Routine | Stage |
|---|---|---|---|---|
| `+00` | W | host `600>>1` (also kept in `ctx+0x294`; non-zero selects the tiled reference layout) | `0x1f78` | start-up |
| `+14` | W | slice limit: `P+0x80` (mode 0), table entry `host[34c+4*slice]` (mode 2), else `0xfff` | `0x12a4` | per macroblock (from `0x352c`) |
| `+18`, `+24` | W | host `120`, `124` | `0x1f78` | start-up |
| `+1c`, `+20` | W | constants `4`, `1` | `0x1f78` | start-up |
| `+30`, `+34` | W | host `800`, `804` (input planes) | `0x4084` | frame set-up |
| `+3c..+48` | W | four reference/reconstruction plane addresses: `base`, `base+N`, `base+2N`, `base+2.5N` with `base = P+0x60` (host `80c`) and `N = P+0x08 * P+0x0c`; a tiled variant is written when host `600>>1` is non-zero | `0x3f0c` | frame set-up |
| `+4c` | W | host `810` | `0x1f78`, `0xbec` | start-up, header |
| `+80` | W | deblocking: `P+0x64 \| (P+0x6c&0xf)<<8 \| (P+0x70&0xf)<<3` | `0x352c` | per slice |
| `+84` | W | host `32c` | `0x4084` | frame set-up |
| `+88` | W | `P+0x58<<31 \| P+0x50<<8 \| P+0x5c` (reference configuration) | `0x3f0c` | frame set-up |
| `+8c..+98` | W | host `334`, `338`, `33c`, `340` when `P+0x58` is set | `0x3f0c` | frame set-up |
| `+140` | W/R | host `900`; low 6 bits read back as frame start QP | `0x4084`, `0x1834` | frame set-up |
| `+144`, `+148`, `+150` | W | host `904`, `908`, `910` | `0x4084`, `0x2104` | frame set-up, header |
| `+14c` | W | QP window `min \| max<<8` (host `90c` raw, or the computed window when rate control A is on) | `0x4084`, `0x1834` | frame set-up |
| `+154` | R/W | host `914` (`0xf` when `P+0xdc` is set); written back to host `914` | `0x4084` | frame set-up |
| `+158` | R | last QP, low 6 bits, copied to host `918` | `0xf3c` | end of frame |
| `+15c` | W | `bits_per_mb (16 bit, capped 0xffff) \| weight<<16` | `0x1834`, `0x2580` | frame set-up |
| `+160` | W | virtual buffer level in bits (`ctx+0x20`) | `0x2104`, `0x2628`, `0x27d4` | header, end of frame |
| `+164` | W | buffer size in bits (`ctx+0x28`) | `0x2104`, `0x27d4` | header |
| `+168` | W | `ctx+0x14<<28 \| ctx+0x2c<<16 \| ctx+0x48` (default `0x80460046`) | `0x2580`, `0x1834`, `0x26f8` | header, frame, end of frame |
| `+16c` | W | `ctx+0x54 \| ctx+0x4c<<16` (default `0x00640032`) | `0x2580`, `0x26f8` | header, end of frame |
| `+170`, `+17c`, `+180`, `+184` | W | defaults `0x0020003a`, `0x00015a30`, `0x00000090`, `0x00006478` | `0x2580` | header |
| `+174` | R | frame statistic, divided by the macroblock count in `0x26f8` | `0x26f8` | end of frame |
| `+178` | W | `ctx+0x10<<8 \| rate_flag<<7 \| ctx+0x5c` (default `0x000002b0`) | `0x2580`, `0x1834` | header, frame |
| `+190` | W | host `814` | `0x4084` | frame set-up |

Unknown: all bit meanings of `+168..+184`; what `+174` accumulates.

### Block `0x00560000` : reference and prediction set-up (inferred)

Written at frame start and per slice; the only value the firmware reads back is `+0c`.

| Offset | Access | Value / meaning | Routine | Stage |
|---|---|---|---|---|
| `+00` | W | `P+0x08 \| P+0x0c<<11` (padded size) | `0x4084` | frame set-up |
| `+04` | W | `is_P \| profile<<4` (`is_P = ctx+0x2a0`, profile `P+0x1c`) | `0x4084`, `0x352c` | frame, slice |
| `+08` | W | `0x1000000` when the frame counter `P+0xbc` is even, `1` when odd (two alternating reference sets) | `0x4084` | frame set-up |
| `+0c` | W, R | `is_P`; per slice `(old&0xf)`, `\|0x10` when the refresh flag `ctx+0x2b8` is set on a P picture without references | `0x4084`, `0x352c` | frame, slice |
| `+14` | W | `3` at frame start; `2` per slice when slice mode is on and the macroblock is not the first | `0x4084`, `0x352c` | frame, slice |

### Block `0x00200000` : mode and motion engine output (inferred)

The firmware never writes `+00..+3c`. It only reads them; the one write is `+a0`. The earlier page note of
"22 bulk writes" is **not found** in the image (searched: all immediate loads of `0x200000`, all literal-pool words,
whole `0x4760` image). The read pattern is a per-macroblock result record:

| Offset | Bits | Decoded into | Meaning (inferred) | Routine |
|---|---|---|---|---|
| `+00` | `5:0` | `S+0x0c` | macroblock type (inter 0..4; intra types follow H.264 numbering) | `0x3688` |
| `+00` | `6` | `S+0x04` | intra flag | `0x3688` |
| `+00` | `9:7` | `S+0x10` | type class / partition mode | `0x3688` |
| `+00` | `10` | `S+0xd4` | skip flag | `0x3688` |
| `+00` | `13:12` | `S+0xd0` | Intra16x16 prediction mode | `0x3688`, `0x38b0` |
| `+00` | `15:14` | `S+0xcc` | intra chroma prediction mode, written with `ue` after the intra kick | `0x3688`, `0x38b0` |
| `+00` | `== 0xffffffff` | `ctx+0x2c8 = 1` | end-of-slice marker, honoured only when slice mode `P+0x74` is on | `0x1d88`, `0x1ca8` |
| `+04` | `19:14` | `S+0x24` | `mb_qp_delta` (6-bit, sign-extended before `se`) | `0x3688`, `0x3d04` |
| `+04` | `25:20` | `S[0]` | coded block pattern | `0x3688` |
| `+04` | `31:26` | `P+0xcc` | running value; difference goes into the statistics table | `0x1d88`, `0x1104` |
| `+08` | all | stats table word | unknown quantity, copied to table | `0x1104` |
| `+20..+2c` | `9:0` used | `S+0x2c,+0x34,+0x3c,+0x44` | per-partition vector component, 1/2/2/4 partitions for types 0/1/2/3-4 | `0x3688`, `0x1b5c` |
| `+30..+3c` | `8:0` used | `S+0x30,+0x38,+0x40,+0x48` | partner component per partition | `0x3688`, `0x1b5c` |
| `+a0` | W | `1` when slice mode, `P+0x78 == 1` and bytes written exceed `P+0x7c` (before the last five macroblocks); `0` at the end marker | `0x19d4`, `0x1d88` | per macroblock |

So the block is the motion/mode decision hardware: it hands the firmware the decisions, and the firmware only formats
them. `+a0` is a stop/degrade request toward it.

### Table RAM `0x00600000` (inferred)

| Address | Content | Routine |
|---|---|---|
| `0x600000` | header bit total `ctx+0x2d8 + ctx+0x2dc` (written when host `348` bit 31 is set) | `0xf3c` |
| `0x600004 + 0x10*slice` | per slice: macroblock count `ctx+0x2a8`; `200004` word plus delta of its bits 31:26; bits used `ctx+0x2e4 + ctx+0x2d4`; `200008` word. Index is the slice counter `ctx+0x2a4`, at most 32 entries | `0x1104` |
| `0x6002a0 + 0x10*(mb % 40)` | four vector words per macroblock (see below) | `0x1b5c` |

Vector words (`0x1b5c`): word 0 `(type&7)<<28 \| (v20&0x3ff)<<16 \| v30&0x1ff`; words 1..3 the same pair for partitions 1..3.
Intra and skipped macroblocks get `intra<<31 \| skip<<15` instead (P pictures only); other types get 0.
`0x1c4c` ships a full row (`0x280` bytes = 40 macroblocks x 16) through the `0x500000` DMA registers to
`host c08[30:0] + row*0x280`, enabled by host `c08` bit 31. The table in `0x600004..` goes out at end of frame through
`0xd60` to the address in host `348[30:0]`.

## Header writer

The writer is called by the header handler `firmware 0xbec` in the order `0x858`, `0x68c`, `0x5c0`.
Each NAL starts with a 32-bit start code `00 00 00 01` written as `(1, 32 bits)`, then the NAL byte.

### `0x858` : picture geometry

| Result | Formula |
|---|---|
| width in macroblocks `P+0x38` | `(W>>4)`, plus 1 when `W & 15` |
| height in macroblocks `P+0x3c` | `(H>>4)`, plus 1 when `H & 15` |
| padded size `P+0x08`, `P+0x0c` | `mbw*16`, `mbh*16` |
| macroblock count `P+0x34` | `mbw * mbh` |
| crop `P+0x10` / `P+0x14` | `(mbw*16 - W) >> 1` / `(mbh*16 - H) >> 1` (2-pixel units); flag `P+0x8c = 1` when either dimension is not a multiple of 16 |

### `0x68c` : sequence parameter set, NAL byte `0x67`

| Syntax element | Emitted as | Source |
|---|---|---|
| `profile_idc` and constraint byte | 16 bits: `0x4200` Baseline, `0x4d00` Main, `0x6400` High (constraint flags all 0) | `P+0x1c` = host `300[1:0]` (0, 1, 2) |
| `level_idc` | 8 bits | `P+0x20` = host `300>>2` (raw level value) |
| `seq_parameter_set_id` | `1` bit (ue 0) | constant |
| High-profile extras | 7 bits `0x2c`: chroma 4:2:0, 8-bit depth, no bypass, no scaling matrix | profile 2 only |
| `log2_max_frame_num_minus4` + `pic_order_cnt_type` | 10 bits `0x6b`: ue(12) then ue(2); frame number is 16 bits; POC type 2 | constant |
| `max_num_ref_frames` | ue(`P+0x50 + 1`) | host `330[11:8] + 1` |
| gaps flag | 1 bit 0 | constant |
| width, height | ue(`mbw-1`), ue(`mbh-1`) | `P+0x38`, `P+0x3c` |
| `frame_mbs_only`, `direct_8x8` | 2 bits `11` | constant |
| cropping | flag 0, or flag 1 with ue(0), ue(`P+0x10`), ue(0), ue(`P+0x14`) | `P+0x8c` |
| VUI | 1 bit 0 | constant |
| trailing | `0x528`, see below | |

### `0x5c0` : picture parameter set, NAL byte `0x68`

| Syntax element | Emitted as | Source |
|---|---|---|
| ids, entropy mode, field flags, slice groups | 5 bits `0x19` (CAVLC) or `0x1d` (CABAC): pps id 0, sps id 0, entropy flag, bottom-field flag 0, ue(0) | `P+0x24` = host `310` bit 0 |
| `num_ref_idx_l0_default_active_minus1` | ue(`P+0x54`) | host `330[11:8]` |
| l1 default, weighted prediction | 4 bits `1000` | constant |
| `pic_init_qp_minus26` | se(`P+0x30 - 26`); also copies `P+0x30` to `P+0x2c` | host `900[5:0]` |
| QS, chroma offset, deblocking control, constrained intra, redundant count | 5 bits `11100` | constant |
| trailing | `0x528` | |

### `0x528` : trailing bits

It reads `+44 & 7` and writes `8 - pos` bits whose value has a single `1` at the current position (`0x80 >> pos`):
the RBSP stop bit followed by zero alignment (*inferred*), then waits for the writer. It does not write an
explicit stop bit itself, so every header ends byte aligned.

### `0xe8c` : CABAC zero words (end of frame, CABAC only)

Computes `q = (+6c + 2) / 3`, writes `q` units of `00 00 03` (24 bits each, with leading zero byte fix-up for
alignment states `0x10` and `0x18`) and returns `q`; `0xf3c` adds `q*0x18` to the bit totals (*inferred*:
`cabac_zero_words`).

### The two counts in host `a08`

`0xbec` zeroes `+44`, `+78`, `ctx+0x2d8`, `ctx+0x2dc`; `0x68c` and `0x5c0` each zero `+44`/`+78` again, write their NAL
(including start code) and finish with `0x528`. After each, the total is `+44 + +78*8` bits:
`ctx+0x2d8` after the SPS, `ctx+0x2dc` after the PPS. Then `a08 = (ctx+0x2d8 >> 3) << 16 | (ctx+0x2dc >> 3)`
(`firmware 0xc5c..0xc74`): upper 16 bits = SPS byte length, lower 16 bits = PPS byte length, each including start code,
NAL byte and escape bytes. Host `a0c` after the header is `(ctx+0x2d8 + ctx+0x2dc) >> 3`.

### Slice header (`0x1340`, per slice from `0x352c`)

| Element | Emitted as |
|---|---|
| start code, NAL byte | `1` in 32 bits; `0x65` (type 2) or `0x61` (types 0 and 1) |
| `first_mb_in_slice` | ue(`ctx+0x2ac`) |
| `slice_type` + `pps_id` | I: 4 bits `0111`; P: 2 bits `11` |
| `frame_num` | 16 bits `P+0xb8` (IDR also writes ue(`P+0xbc % 2`) as `idr_pic_id` and 2 zero bits for the marking flags) |
| P only: `num_ref_idx_active_override` | `0` when `P+0xb8 > P+0x50`, else `1` plus ue(`P+0x54`); then 1 zero bit (list modification); 1 zero bit (marking) |
| non-IDR I: marking | 1 zero bit |
| `cabac_init_idc` | ue(`P+0x44`) = host `310[3:2]`, CABAC and P only |
| `slice_qp_delta` | se(`P+0x2c - P+0x30`) |
| `disable_deblocking_filter_idc` | ue(`P+0x64`) = host `314[1:0]` |
| alpha / beta offsets | se(`P+0x6c`), se(`P+0x70`) when idc is not 1; 5-bit signed from host `314[11:7]` and `314[6:2]` |

## Macroblock loop (`firmware 0x18c4`, entered from `0x1ee4`)

Per frame: `0x16c` selects the stream slot, `0x1834` frame set-up, `0x368` backs up P to slot `stream+4`, then the loop
below, then `0xf3c`, `0xd10`, `0x368` (store P to slot `stream`), and `0x28fc` (store the result record) when rate
control is on and the bypass flag `ctx+0x288` is 0.

One iteration of the outer loop is one **slice**, not one macroblock:

| Step | Routine | Action |
|---|---|---|
| 1 | `0x1550` | refine picture type for this slice. With slice mode off (`P+0x74 = 0`) it only sets `ctx+0x2a0 = (type == 1)`. With slice mode on: bit `slice` of host `328` forces type 0 unless IDR; bit of host `344` forces a refresh (`ctx+0x2b8 = 1`) of a P slice when `ctx+0x2c0 = 0`; the bit-limit flag `ctx+0x2c4` from host `3cc` takes part. |
| 2 | `0x352c` | slice set-up: clears `ctx+0x2a8/+0x2bc/+0x2c8`, writes slice header (`0x1340`), `0x550080`, CABAC init words (`0x520048/4c`), `0x560004/0c/14`, calls `0x12a4` |
| 3 | handler through the pointer | codes every macroblock of the slice until `ctx+0x2c8` is set |
| 4 | `0x1104` | slice close: zero-aligns (CAVLC: stop bit and zeros through `0x528`; CABAC: zeros only), updates bit totals `ctx+0x2d0/+0x2e0/+0x2e4`, sets the over-limit flag `ctx+0x2c4 = (P+0xa4 < bits)`, stores the statistics record, increments the slice counter `ctx+0x2a4`. At the last macroblock (`ctx+0x2cc`) it runs the `+c8` handshake first. |

**Function-pointer dispatch** (`0x1ee4` body, `firmware 0x1904..0x1928`): the pointer pair at `ctx+0x280/+0x284` is a
C++ pointer-to-member (low bit of the adjust word selects a virtual lookup). `0x4084` loads it from the table at
`firmware 0x45c8`: `{0x1ca8, 0}` when `P+0x24 = 0`, `{0x1d88, 0}` when 1. It selects the **entropy coder**,
CAVLC handler `0x1ca8` or CABAC handler `0x1d88`, not intra, inter or skip. Intra, inter and skip come per
macroblock from the `0x200000` record (`S+0x04`, `S+0xd4`, `S+0x0c`).

**Per-macroblock work inside the handlers** (same skeleton in both): `0x4060` (macroblock kick: `+c8` handshake, `+10` =
address), end-marker test on `200000`, last-macroblock test (`P+0x34 == ctx+0x2ac+1` sets `ctx+0x2c8` and `ctx+0x2cc`),
`0x3688` (decode the `0x200000` record into S), optional `0x1c4c` (vector dump, host `c08` bit 31), then
CAVLC: `0x3d04` (syntax, below); CABAC: kick `0x520004 = 0x10000` and wait `+20`. Finally `0x19d4`:
advance `ctx+0x2ac` (index), `ctx+0x2a8` (count), column `ctx+0x2b0` (row `ctx+0x2b4` on wrap at `P+0x38`), clear the skip run
`ctx+0x2bc` unless the macroblock was a skipped P macroblock, and raise `0x2000a0` as described above.

`0x3d04` (CAVLC macroblock layer): skipped macroblock: only `+48`/`+04` kick. Otherwise `+4c = mb_type`, `0x38b0`
(skip run as ue when the picture is P, macroblock type as ue (I16x16 type = `1 + pred_mode + 4*chroma_cbp + 12 if luma_cbp`),
sub-macroblock types `1111`, reference indices: one inverted bit when `P+0x54 = 1`, else ue), motion kick `0x800`, coded-block-pattern
code through `+0c` modes 2/3, `mb_qp_delta` se, residual kick `4`. At slice end a pending skip run is flushed with ue (`firmware 0x1d80`).

Per-macroblock register **reads** from `0x550000` and `0x560000`: none, except the `0x56000c` read-modify-write in
`0x352c`. All other reads of those blocks are per frame (`0x550140`, `0x550158`, `0x550174`). All per-macroblock
information comes from `0x200000` and the `0x520000` status words.

## Rate control

Rate control is split: the firmware sets targets and windows per frame and updates its state after the frame, while
the per-macroblock QP adaptation runs in hardware (`0x550000`). Two enables come from host `900` through `0x924`:
bit 9 is `P+0xc4` (firmware rate control A: `0x2104`, `0x2628`, QP window), bit 8 is `P+0xc8` (engine rate flag,
`0x550178` bit 7). `0x3b88` sets both when bit 9 is set without bit 8, and sets `P+0xdc` (forces `900` bit 8 and
`0x550154 = 0xf` in `0x4084`). Command 5 with a non-zero stream byte sets the bypass flag `ctx+0x288`
(`firmware 0x248`): the firmware then skips its own rate control, copies host `91c..944` into
`0x55015c..0x550184` unchanged (with `928` and `938` ANDed against fixed masks), and answers `a00 = 0xbad0cafe`.

### Set-up at header time (`0x2104`, only when A or the engine flag is set)

With `R904`, `R908`, `R910` the host words and `period = P+0x18`:

| Result | Formula (*inferred* reading) |
|---|---|
| bits per frame | `R908 / R904` (at least 1); treat `R908` as bit rate and `R904` as frame rate |
| I-frame target `ctx+0x84` | `(R908/R904) * (period+1) / (1 + 60*period/160)`; the fixed ratio P : I = 0.375 comes from the constants `0x3c` and `0xa0` |
| P-frame target `ctx+0x88` | `((R908/R904)*(period+1) - I target) / period`; `I target*6/9` when `period = 0` |
| per-macroblock targets `ctx+0x7c`, `+0x80` | target `/ P+0x34`; `R908/(R904*8)` form when the target is below one frame of bytes |
| weights `ctx+0x34`, `+0x38` | constants 6 (I), 9 (P) |
| QP floor margin `ctx+0x8c` | `0x18 - 1000000-scaled log` of the frame size for small frames, else 4 |
| buffer size `ctx+0x28` | `(period+1)*(R908/R904 >> 1) + (R908/R904 * R910) >> 5` |
| initial buffer level `ctx+0x20` | scaled form of the buffer size, then `>>1` |
| buffer ceiling / floor | `ctx+0x94 = 3 * P target`, `ctx+0x98 = P target` |

`0x2580` first loads the engine defaults, `ctx+0x6c/+0x70` = host `90c` bits 13:8 / 5:0 (max / min QP).

### Per frame (`0x1834`)

| Step | Action |
|---|---|
| 1 | `0x1674` loads P from the stream slot; `0x27d4` restores the rate state record; `0x3b88` re-reads the host words; `0x1610` picks the type; `0x4084` writes the engine |
| 2 | type 0 or 2: `ctx+0x40 = ctx+0x7c`, target `ctx+0x78 = ctx+0x84`, weight `ctx+0x30 = ctx+0x34`, `ctx+0xa0 = 0`; type 1: the `+0x80`, `+0x88`, `+0x38` set and `ctx+0xa0` counts P frames since the last I |
| 3 | from the second frame: `QP = clamp(host 900[5:0], ctx+0x70, ctx+0x6c)` stored in `P+0x2c` (it goes into `slice_qp_delta`) |
| 4 | new floor: `lo = min(ctx+0x6c, 4)` if `QP < 6`, else `lo = min(ctx+0x6c, ctx+0x8c, QP-1)`; then `lo = max(lo, host 90c[5:0])`; stored in `ctx+0x70` |
| 5 | `0x55014c = lo \| ctx+0x6c<<8`; `0x55015c = min(ctx+0x40, 0xffff) \| weight<<16` |

So the QP update the firmware performs is a window update, not a per-macroblock rule; the actual start value is
`host 900[5:0]` (set by the host after reading `918`) and the hardware moves QP per macroblock inside the window,
steered by `0x550160` (buffer level) and `0x55015c` (bits per macroblock). Whether the hardware itself clamps the
unmodified `0x550140` start value into the window is unknown.

### After the frame (`0xf3c`, `0x2628`, `0x26f8`)

| Step | Action |
|---|---|
| `918` | `0x550158 & 0x3f` (last QP) |
| padding | host `3d0` bit 31 enables; frame is padded with zero bytes up to `3d0[30:0] >> 3` bytes |
| `a0c` | total frame bytes `ctx+0x2e0 >> 3`, padding included; `a10` = the same minus the padding bytes |
| `a00` | `0xe64`: `P+0xd8 \| P+0xd4<<16` for CAVLC (counts of intra and of non-skipped inter macroblocks, from `0x38b0`), else 0 |
| buffer (`0x2628`, A only) | `d = bits - target`. I frames with overshoot bank `ctx+0x9c = d>>3`. Otherwise `level = max(level + d, 0)`, plus `ctx+0x9c` once while the P-frame counter is in `[period>>2, period>>2 + 8)`; then clamp to the floor `ctx+0x98` (shortfall accumulates in `ctx+0x90`) and ceiling `ctx+0x94`; written to `0x550160` |
| thresholds (`0x26f8`) | `avg = 0x550174 / P+0x34`, floor `0x10`, cap `0x3ff` into `ctx+0x2c`, `ctx+0x48`; `ctx+0x4c = ctx+0x54 = 0x3000 / (2 * Qstep(QP))` clamped to 0..`0x1ff` (`0x19` when `avg = 0x10`); `Qstep` from the double table at `firmware 0x45d8` (0.625, 0.6875, 0.8125, 0.875, 1.0, 1.125, times 2^(QP/6)); written to `0x550168`, `0x55016c` |
| completion | `0xd10` clears `0x52005c`, waits for host `514` bit 1 to clear, writes `50c = 1`, increments `P+0xb8`, `P+0xbc` |

### Host registers in the rate path

| Host reg | Role (firmware `P` field) |
|---|---|
| `900` | `[5:0]` start QP (`P+0x2c`, `P+0x30`, `0x550140`); bit 8 `P+0xc8`; bit 9 `P+0xc4` |
| `90c` | max `[13:8]` / min `[5:0]` QP: `ctx+0x6c/+0x70`, `0x55014c`; lower bound of the floor update |
| `904`, `908`, `910` | `0x550144/148/150`; frame-rate-like, bit-rate-like and buffer-ratio-like inputs of `0x2104` (*inferred* roles) |
| `918` | output: last QP |
| `324` | `P+0x80` slice limit; minus `0x780` (floor 0) is `P+0x7c`, byte threshold for the early-stop request |
| `328`, `344` | `P+0x84`, `P+0x88`: per-slice bit masks, force I / force refresh (`0x1550`) |
| `31c`, `320` | `P+0x74` slice mode on/off; `P+0x78` limit mode 0, 1 or 2 (`0x12a4`, `0x19d4`) |
| `34c..3c8` | 32 per-slice limits read by `0x12a4` in mode 2 |
| `3cc` | bit 31 enable, `[30:0]` bit limit: sets the over-limit flag `ctx+0x2c4` per slice (`0x1104`) |
| `3d0` | bit 31 enable, `[30:0]` minimum frame size in bits (padding) |
| `348` | bit 31 enable, `[30:0]` address for the per-slice statistics table |
| `c08` | bit 31 enable, `[30:0]` address for the vector rows |

## Result record (21 words at `ctx[0] + stream*0x54`)

Written by `0x28fc` at the end of a frame, restored by `0x27d4` at the start of the next: it is the **persistent
rate-control state of the stream**, not a report to the host. The host reads its numbers from `a00`, `a0c`, `a10`, `918`.

| Word | Offset | Source | Quantity |
|---|---|---|---|
| 0 | `+00` | `ctx+0x20` | virtual buffer level, bits |
| 1 | `+04` | `ctx+0x90` | accumulated shortfall below the floor |
| 2 | `+08` | `ctx+0x94` | buffer ceiling (3 x P target) |
| 3 | `+0c` | `ctx+0x98` | buffer floor (P target) |
| 4 | `+10` | `ctx+0x84` | I-frame target bits |
| 5 | `+14` | `ctx+0x88` | P-frame target bits |
| 6 | `+18` | `ctx+0x7c` | I bits per macroblock |
| 7 | `+1c` | `ctx+0x80` | P bits per macroblock |
| 8 | `+20` | `ctx+0x9c` | banked I-frame overshoot / 8 |
| 9 | `+24` | `ctx+0xa0` | P frames since the last I |
| 10 | `+28` | `ctx+0x24` | copy of host `910` |
| 11 | `+2c` | `ctx+0x28` | buffer size, bits |
| 12 | `+30` | `ctx+0x34` | I weight (6) |
| 13 | `+34` | `ctx+0x38` | P weight (9) |
| 14 | `+38` | `ctx+0x4c` | Qstep-based threshold |
| 15 | `+3c` | `ctx+0x54` | same threshold, second copy |
| 16 | `+40` | `ctx+0x2c` | mean macroblock statistic |
| 17 | `+44` | `ctx+0x48` | same statistic, second copy |
| 18 | `+48` | `ctx+0x8c` | QP floor margin |
| 19 | `+4c` | `ctx+0x6c` | maximum QP |
| 20 | `+50` | `ctx+0x70` | minimum QP (current floor) |

The record is only used when rate control is on (`P+0xc4` or `P+0xc8`) and the bypass flag `ctx+0x288` is 0.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Register offsets, command sequencing, events | original VAs, the 66-row source-access inventory | observed; finite original/clean comparisons |
| Embedded program upload | `mem:0x08120bf0`, RAM `086bf000`/`086bdbe4` | observed |
| Embedded program is ARM firmware with a command loop (commands 1 header, 2 frame, 4 stop, 5 update) | firmware image at `osos mem:0x086bf000`, vectors at offset 0, loop at `0x1f98`, wait at `0x1f08` | observed; engine register roles inferred |
| Host register file mirrored at `0x00500000` inside the encoder | firmware reads of `0x500114/0x500c0c/0x500300..` match the host writes at `0x3e400114/0x3e400c0c/0x3e400300..` | observed (same offsets), mirror inferred |
| Unexplained literals and fields | tables above | observed values, meaning unknown |
| SPS/PPS/slice-header fields and sources | `firmware 0x68c`, `0x5c0`, `0x1340`, `0x858`, `0x924`, `0x3b88` | observed (bit patterns decoded against H.264 syntax) |
| `a08`, `a0c` formation | `firmware 0xc5c..0xc74`, `0xcf0`, `0xf3c` | observed |
| `+44` is a bit counter | `&7` in `0x528`, `>>3` in `0x19d4` | inferred |
| Dispatch selects entropy coder | table `firmware 0x45c8`, `0x4084`, `0x1904..0x1928` | observed |
| `0x200000` is read-only mode/motion output | `0x3688`, `0x1104`, `0x1b5c`, `0x1d88` | inferred role, observed reads |
| Block roles `0x550000`, `0x560000` | access patterns above | inferred |
| Rate-control formulas | `0x2104`, `0x1834`, `0x2628`, `0x26f8`, `0xf3c` | observed arithmetic, inferred names |
| Result record is persistent state | `0x27d4`, `0x28fc`, `0x1834`, `0x18c4` | observed |
| Slice mode (`31c`, `320`, `328`, `344`, `34c..3c8`) | `0x12a4`, `0x1550`, `0x19d4`, `0x1d88` | inferred |
