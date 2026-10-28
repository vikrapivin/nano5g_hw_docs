# JPEG encoder

The JPEG block is at **`0x39800000`**, **IRQ56**, OSOS clock-service ID `0x47`.
It is not the iPod Classic decoder base. OSOS's camera save layer supplies
pixels, output capacity, quality and format. The engine is independent from the
H.264 video encoder.

## Registers

32-bit MMIO offsets relative to the JPEG base. The table covers all 35
reconstructed accesses, including unexplained constants.

| Offset | Firmware operation / meaning |
|---|---|
| `00` | reset writes 8; configured format 1 writes 2, formats 0/2 write 1; bit 3 read by ISR |
| `04` | error read; bit map unknown |
| `08` | quality/table setup writes `143c`; fields unknown |
| `0c` | configure writes 0 |
| `10`, `14` | two requested dimensions; format-dependent axes not decoded |
| `18` | encoded length, compared to output limit |
| `1c` | configure and one ISR branch write 8 |
| `20` | status; ISR distinguishes bits 3 and 6 |
| `50`, `58` | input pixels / output address, bit 31 stripped |
| `5c`, `60`, `64` | setup constants `004d971e`, `002c5783`, `00836e13`; coefficient meaning not established |
| `68` | format 2 writes `41`, format 0/1 writes `21` |
| `6c` | enable 1; cleanup clears bit 0 |
| `70` | start 1 |
| `78` | reset OR 1, poll until whole register is 0 |
| `7c` | configure `ffffffff`; meaning unknown |
| `80` | DMA/error status, bit 31 acknowledged by ORing bit 31 |
| `98` | capacity `(capacity&00ffffe0)+01000000` |
| `9c` | nonzero means output-overflow path; acknowledge by OR 1 |
| `400..7fc` | four quantization slots, 64 bytes each, each byte in a 32-bit register |
| `800` onward | Huffman count/symbol windows, formulas below |

- Quant slot `q`: base `400+q*100`, stride 4, 64 entries.
- Huffman slot `s`: base `800+(s&1)*400+(s>>1)*80`. 16 count words, then symbols at base+40: 12 symbols for slots 0/1 (DC), 162 for slots 2/3 (AC). These are 32-bit registers, not packed byte arrays.

## Reset, quality and encode sequence

Reset `mem:0x08125d1c`:

1. Clear `6c` bit 0, sleep 10 OS units.
2. OR `78` bit 0, poll to zero with an iteration bound slightly above 1,000,000 (continues on expiry; an iteration count, not a timeout in microseconds).
3. Read `20`, then `04`, then write `00=8`.

Quality indices 0..7 map to quantization scales `[1000,200,125,100,25,10,1,100]`
percent; out of range selects 100%. The original uses double arithmetic,
truncates and clamps entries to 1..255. It uploads two scaled tables and two
fixed tables, then the DC/AC tables. The object's quality byte is unchanged by
the upload. Tables (osos image, compared byte for byte): quantization bytes, 256 B at
`osos mem:0x086bd888`; DC luma, 28 B at `0x086bd988`; AC luma, 178 B at
`0x086bd9a4`; DC chroma, 28 B at `0x086bda56`; AC chroma, 178 B at `0x086bda72`.

`mem:0x0811b08c`: enable gate 47, optionally reset, `6c=1`, select mode,
clear `0c`, upload changed quality, write `1c=8`, `7c=ffffffff`, `98` capacity,
set state 4.

`mem:0x0810a3cc`: program dimensions/input/output, start `70=1`, wait 250 OS
event units. State 11 is success only if `18` <= output limit. Overflow (state
7) can decrement quality and retry when requested, with disable/gate/sleep
between retries. Cleanup always clears `6c` bit 0 and gates 47 off.

ISR `mem:0x081337cc` reads `20`, `80`, `00` (bit 3 only), `04`, then `9c`
(`mem:0x081337d2..0x081337e0`). Nonzero `9c` latches state 7, acknowledges and
returns **without signaling the event**. Otherwise it handles DMA bit 31,
error and status bits, setting states 8/9/10/11/12, then signals. The state
byte is software, not the hardware status encoding.

## Original-firmware hazards

- The original subtracts `0x100` from output capacity without an underflow check.
- It has timeout/failure quirks that are not hardware requirements.
- Not known: format-bit semantics, color-conversion coefficients, reset completion timing, error meanings, supported image sizes.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Register offsets, tables, IRQ and retry sequence | original VAs | observed; differential tests reported |
| Literal tables | tables at the addresses above | observed (byte comparison with firmware) |
| Remaining fields and electrical behavior | no datasheet | meaning unknown |
