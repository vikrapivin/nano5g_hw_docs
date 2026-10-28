# NAND BCH encoder/decoder

The BCH block works on the FMC's local sector/parity buffers; it has no
memory-to-memory interface. The firmware feeds parity and data through the
[NAND controller](nand-fmss.md), selects a local slot, starts ECC, then drains
corrected data or generated parity. No ECC interrupt is used: ROM polls the
status, and FMSS waits on the ECC event.

Base is `0x38a00800`; FMC is `0x38a00000`. Clock gate is bit 9 of
`0x3c500048`, enabled by clearing it. All observed accesses are 32-bit words.

## Registers and associated FMC routing

| ECC offset | Absolute address | Observed interface |
|---|---|---|
| `+04` | `0x38a00804` | Write starts encode/decode. Values: `1` BCH8 decode, `2` BCH8 encode, `5` BCH16 decode, `6` BCH16 encode. OR `0xa00` on normal-page sector zero to include ten metadata bytes. |
| `+08` | `0x38a00808` | Boot-format programs write `0x80`; field meaning unknown. |
| `+0c` | `0x38a0080c` | Decode result: bit 0 uncorrectable; bits `[20:16]` corrected-bit count. Other bits unknown. |
| `+10` | `0x38a00810` | W1C status. ROM polls bit 2 for decode completion and clears `0x7f`; sequencer initializes with `0x1ff` and acknowledges its ECC event with `1`. Bit 6 is treated as erased/all-FF. ROM (bit 2 poll) and the sequencer (event acknowledge) use different completion mechanisms. |
| `+1c` | `0x38a0081c` | FMSS reset path writes `1`. Reset completion semantics unknown. |

FMC `+0x1c` selects ECC slot 0/1. FMC `+0x14` and `+0x58` route the data and
parity buffers. FMC `+0x40=0xc1` loads CPU-written metadata from FIFO words
`+0x60/64/68`; `0xc2` latches corrected metadata for CPU reads. The third host
metadata word has its high half forced to `ffff` when returned. A destination
DMA must finish reading a slot before a new ECC operation overwrites it.

Evidence: bootROM `mem:0x200091e8`; `bootloader NandReadOnly.pe+0x4f4c` mode setter,
`0x48dc` result decoder, `0x53d4` physical-layout descriptor; the original READ, WRITE,
BLREAD and BLWRITE FMSS programs (see the FMSS ISA page).

Register definitions, from the NAND reference library's controller header
(offsets relative to `FMC_BASE` = `0x38a00000`):

```c
#define FMC_ECC_CTRL        0x804   /* ECC engine command: writing it starts an encode/decode */
#define FMC_ECC_808         0x808   /* boot-page microcode: 0x80 before each ECC operation (inf) */
#define FMC_ECC_RESULT      0x80c   /* ECC result of the last decode: b0 uncorrectable, b[20:16] corrected bit count */
#define FMC_ECC_STATUS      0x810   /* ECC status, w1c: b2 done, b6 sector was all 0xFF (erased) */
#define FMC_ECC_RESET       0x81c   /* _resetFMSS writes 1 (obs) */

#define FMC_ECC_CTRL_DECODE     5u          /* ROM: BCH16 (4) | decode (1) = start a BCH16 decode */
#define FMC_ECC_ST_DONE         (1u << 2)   /* decode finished (ROM spins on it without a bound) */
#define FMC_ECC_ST_ERASED       (1u << 6)   /* sector is all 0xFF */
#define FMC_ECC_ST_CLEAR        0x7fu       /* w1c value clearing bits 0..6 (ROM) */
#define FMC_ECC_RES_UNCORR      1u          /* uncorrectable; bits[20:16] = corrected count */

/* ECC engine as used by the FMSS microcode */
#define FMC_ECC_CTRL_BCH8_DEC   1u              /* decode, BCH8 (13 B parity per sector) */
#define FMC_ECC_CTRL_ENC        2u              /* encode (write path) */
#define FMC_ECC_CTRL_BCH16      4u              /* strength select: BCH16 (26 B parity per sector) */
#define FMC_ECC_CTRL_META10     0xa00u          /* codeword carries 10 meta bytes (sector 0) */
#define FMC_ECC_RES_NBITS(v)    (((v) >> 16) & 0x1fu)   /* corrected bit count of the last decode (0..16) */
```

## Supported codewords and parameter values

The normal payload unit is 512 bytes. The firmware supports strengths 8 and
16 with parity lengths 13 and 26 bytes respectively. The normal first sector
also protects ten metadata bytes. That is 104/208 parity bits, consistent
with a 13-bit-symbol BCH construction. The polynomial, parity bit order,
shortening convention and erased-codeword transform are not recovered.

| Mode | Strength | Parity bytes | Ordinary-sector control | First-sector control | First transfer count minus one | Later parity count minus one |
|---|---:|---:|---:|---:|---:|---:|
| Decode | 8 | 13 | `1` | `0xa01` | `22` | `12` |
| Encode | 8 | 13 | `2` | `0xa02` | `22` | `12` |
| Decode | 16 | 26 | `5` | `0xa05` | `35` | `25` |
| Encode | 16 | 26 | `6` | `0xa06` | `35` | `25` |

Those four parameters are stored at FMC offsets `+0xd2c`, `+0xd30`, `+0xd34`,
`+0xd38` (the shared parameter words at `0x38a00d2c..`, not the FMSS block
at `0x38a00c00`) by the original mode setter. Only the first sector's transfer
count includes the ten metadata bytes; counts are not data-sector lengths.
The data-sector byte count stays 511 in FMC `+0x30`.

Strength selection follows geometry: compute
`available = floor((physical_spare_bytes - 10) / sectors_per_page)`; choose
BCH16 if `available >= 26`, otherwise BCH8 if `available >= 13`, otherwise
assert unsupported. The lookup table is at flat-PE
`bootloader NandReadOnly.pe+0x6728`: pairs `{26,16}`, `{13,8}`, then zero terminator.
Refresh thresholds are 12 for BCH16 and 6 for BCH8 (software policy).

## Physical stream layout

The layout-descriptor routine, `bootloader NandReadOnly.pe+0x53d4`, gives the ordering:

```text
sector 0:     [parity P][metadata 10][data 512]
sector 1…N:   [parity P][data 512]
P = 13 or 26
```

The metadata segment starts at offset `P`; data sector `k` starts at
`P + 10 + k*(P+512)`. Normal 4-KiB pages have eight sectors; 8-KiB pages
have sixteen. Thus a 4096+218 device can consume all 218 spare bytes as
`10 + 8*26`, while an 8192+376 BCH8 device consumes `10 + 16*13 = 218`
bytes of its physical spare budget. Unused physical spare is not logical
payload.

The independent-driver requirements packet groups this as `[metadata+parity]`
or puts metadata first; the descriptor routine places parity first, and this
page follows the descriptor.

Boot-region pages are separate: three sectors, each `[parity 26][data 512]`,
1,536 payload bytes total, BCH16 and no normal metadata. BootROM and the
BLREAD/BLWRITE programs implement that format even on parts whose ordinary
pages use BCH8.

## Decode, encode and result interpretation

For a decode, NAND input fills the selected parity/header and data buffers.
The program selects ECC slot, clears prior status, writes the mode and waits
for completion. It classifies the result before reusing the slot, gets
corrected metadata if this is sector zero, and transfers corrected payload
to memory with FMC DMA. The multi-sector programs overlap stages on
opposite slots.

For encode, memory DMA fills a data buffer, the first sector loads metadata,
and ECC produces parity for that slot. The program writes parity/header and
data to NAND in the format above, then issues the chip program-confirm/status
sequence. ECC completion does not mean the NAND page was programmed.

Per-sector software result bytes are `0…16` corrected-bit counts, `fe` erased,
`ff` uncorrectable. READ builds an aggregate in FMSS R6: bit 30 if any
uncorrectable sector, bit 29 if erased sectors occurred, and a low-16-bit
correction-count mask. The sequencer generates these aggregate bits; they are not hardware ECC status bits.

`_m2EvalReadResult` behavior: a single page containing only
`fe/ff` sectors, with fewer than three `ff` sectors, is treated as noisy erased
media; an erased summary with no correction mask is also clean. The returned
`maxcorr` is the bit-length of the summary mask rather than a direct copy of
`ECC_RESULT[20:16]`. Corrected count 16 falls outside the low-16-bit mask.

## Recovery and open fields

The FMSS reset procedure resets ECC with `+0x1c=1`; the driver has no ECC-only
recovery protocol. BootROM's ECC completion poll has no timeout.

The reset-state map and
all-zero/all-one fault matrix are not measured on the device.

## Finding ledger

| Finding | Evidence | Type / limit |
|---|---|---|
| Decode/encode values and parity sizes | `bootloader NandReadOnly.pe+0x4f4c/0x6728`; mode setter; ECC register definitions above | observed |
| Parity precedes metadata | `bootloader NandReadOnly.pe+0x53d4` | observed; independent-driver packet disagrees |
| ROM poll and sequencer acknowledgement differ | BootROM reader and original programs linked above | observed; status-bit meanings incomplete |
| Software erased/error aggregation | `bootloader NandReadOnly.pe+0x48dc`; reference decoder | observed; driver policy |
| Physical BCH polynomial/order | not decoded | needs parity/data vectors from silicon |
