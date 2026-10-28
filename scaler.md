# Scaler and colour-space converter

**MMIO `0x39000000`**, 32-bit accesses. It converts and scales planar camera frames for the display. Protocol GUID
7133b1b6-7f9e-4b60-89ab-d915b3826206; only the diagnostics module `diag Scaler.pe` drives it, and its only client is
`diag IspDriver.pe`. The camera IRQ 57 handler starts it ([camera-interface.md](camera-interface.md)). Clock gate:
ClockAndReset id `0x40` (argument 1 = on) at `diag Scaler.pe+0xf24`. The filter-bank layout repeats the CLCD video
scaler ([display-compositor.md](display-compositor.md)): the 36-word and 18-word polyphase banks sit at the same offsets
(`140..1cc`, `1d0..214`).

## Registers

| Offset | Use | Evidence |
|---|---|---|
| `00` | status; bit 1 set = idle/done (polarity inferred). The completion method returns `0x80000014` while bit 1 is clear | `Scaler.pe+0x466`, `+0xe94` |
| `04` | bit 0 start (set last); bit 8 and bit 9 set from two descriptor flags (descriptor `+0x60` -> bit 8, `+0x5c` -> bit 9; default bit 9 on) | `+0xe84`, `+0x472`, `+0x44e` |
| `0c` | bit 0 written before start when `00` bit 1 is clear; meaning unknown (inferred abort/flush) | `+0x440`, `+0xef4` |
| `10` | source format, bits 2:0; accepted 0, 2, 3, 4, 5, 6, 7 (1 clears the field only) | `+0x9b0` |
| `14`, `18` | source plane 0 / plane 1 address | `+0xa46`, `+0xa0a` |
| `1c` | source pitch pair: low half (descriptor `+0x18`), high half (`+0x1c`) | `+0xa4e`, `+0xa12` |
| `20` | source origin `(desc+0x08)<<16 \| (desc+0x0c)&ffff` (pairing by name only) | `+0xa26` |
| `24` | source size `(width&1fff)<<16 \| (height&1fff)` | `+0xa34` |
| `30` | destination format, bits 2:0 (same codes) | `+0x8f8` |
| `34`, `38` | destination plane 0 / plane 1 address | `+0x984`, `+0x952` |
| `3c` | destination pitch pair: low half, high half | `+0x98c`, `+0x95a` |
| `40` | destination size `width<<16 \| (height&1ffff)` | `+0x970` |
| `50` | horizontal ratio `(src_width<<16) / dst_width` | `+0x9a4`, `+0x1510` |
| `54` | vertical ratio `(src_height<<16) / dst_height` | `+0xa64` |
| `60..6c`, `70..7c`, `80..8c`, `90..9c`, `a0..ac`, `b0..bc`, `c0..cc`, `d0..dc`, `e0..ec` | bank B wide filter, 36 words, each `(c_hi&3ff)<<16 \| (c_lo&3ff)` | `+0xa70..0xcd0` |
| `f0..134` | bank B narrow filter, 18 words, same packing | `+0xd1c..0xe5c` |
| `140..1cc` | bank A wide filter, 36 words | `+0x48c..0x6ec` |
| `1d0..214` | bank A narrow filter, 18 words | `+0x790..0x8d0` |
| `220..240` | 3x3 colour matrix, 9 x 12-bit entries (`value & fff`), row-major: `220 224 228 / 22c 230 234 / 238 23c 240` | `+0x738`, `+0x754`, `+0x770` |

## Filter banks

Each filter word holds two 10-bit two's-complement taps; the firmware tables are sign-extended and masked on write.
Wide banks are 9 phases x 8 taps (sum 128), narrow banks 9 phases x 4 taps (sum 128). Seven wide sets
(`Scaler.pe+0x17c8`, stride `0x200`, 72 ints used) and eight narrow sets (`+0x25c8`, stride `0x100`, 36 ints used) are
selected by the scale-ratio class. Ratio `r` (16.16): 0 if `r < 0x10000`; 1 if `r == 0x10000`; 2 if `<= 0x12492`;
3 if `<= 0x15555`; 4 if `<= 0x19999`; 5 if `<= 0x20000`; 6 if `<= 0x2aaaa`; 7 if `<= 0x40000`; otherwise 1
(`+0xf9c`, `+0x105c`). The coefficient rows equal the OSOS CLCD filters (wide 0..6 here = OSOS sets 7..1, narrow 0..7 =
OSOS 7..0).

## Colour matrices (`Scaler.pe+0x2dc8`, four sets of nine)

| Set | Use | Entries |
|---|---|---|
| 0 | limited-range YCbCr -> RGB | `254 000 331 / 254 f38 e60 / 254 409 000` |
| 1 | RGB -> limited-range YCbCr | `084 102 032 / fb4 f6b 0e1 / 0e1 f44 fdc` |
| 2 | full-range YCbCr -> RGB | `200 000 2be / 200 f54 e9b / 200 377 000` |
| 3 | full-range RGB -> YCbCr | `099 12d 03a / fa8 f52 106 / 106 f25 fd6` |

Entries are 12-bit two's complement with 9 fractional bits (`0x254` = 1.164). The set is chosen from the source and
destination format class ({0, 2, 3} versus {4, 5, 6, 7}) and a descriptor flag (`+0x1124`, `+0x1304`).

## Sequence (protocol method 1, `+0xef4`)

1. Write source format, plane addresses, pitches, origin and size (`+0x158e`).
2. Write destination format, plane addresses, pitches and size (`+0x1492`).
3. Compute the two ratios, write `50`/`54`, select and load both filter banks (`+0x1510`).
4. Set `04` bits 8/9 and load the colour matrix (`+0x1124`).
5. If `00` bit 1 is clear, write `0c` bit 0.
6. Set `04` bit 0.

Method 0 (`+0xf24`) enables the clock and fills a default descriptor of 640x480. IspDriver feeds it a 640x480
destination (pitches `0x280`, chroma plane `+0x4b000`) from the camera IRQ.

## Unknown

Status bit 1 polarity and the role of `0c` bit 0 (inferred from the completion method and the start routine); the
descriptor pairing of the source origin (`20`); no frame was run through the block; the ClockAndReset id `0x40` is not
mapped to a PWRCON bit.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Register offsets and field packing | `diag Scaler.pe` functions above | observed |
| Colour matrices, filter classes | `Scaler.pe+0x2dc8`, `+0x17c8`, `+0x25c8` | observed (byte read) |
| Camera IRQ starts the block | `diag IspDriver.pe+0x5e8` | observed |
| Bit meanings of `00`, `0c`, `04` bits 8/9 | usage only | inferred |
