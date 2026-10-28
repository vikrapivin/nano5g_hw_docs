# DMAC2 memory-copy engine

The four-channel engine at physical `0x38800000` is used by the mask ROM for
memory copies. Its register spacing resembles the general DMA blocks, but its
transfer-control word differs from the [DMAC0/1 PL080 control word](dma.md).

Evidence is ROM `mem:0x20009478…0x20009d30`. Addresses beginning `0x200…` are
ROM execution addresses; other offsets are relative to `0x38800000`.

## Resources and register map

ROM clock ID `0x44` selects bit 8 of `0x3c500068`; clearing the bit enables
the block. Restore OS registers IRQ 29 for this block, but the ROM memory-copy
path polls completion and does not use that IRQ.

| Offset | Role | Observed behavior |
|---|---|---|
| `04` | Terminal-count status, one bit per channel | Read by the restore ISR `disk mem:0x0802ef7c` (IRQ 29) over four channels; not accessed by the ROM copy path. |
| `08` | Completion interrupt clear | Restore ISR writes `1<<channel` (`disk mem:0x0802ef7c`); ROM helper `mem:0x20009478` reads/ORs/writes `1<<channel` and has no identified ROM callers. |
| `10` | Error interrupt clear | Same helper when error argument is nonzero. IRQ/error semantics unknown. |
| `1c` | Channel busy mask | ROM waits while bit `channel` is set. |
| `30` | Global control | Bit 0 enables the engine. Helpers also manipulate bits 1…4; meaning unknown. |
| `40…4c` | Four status words | Helpers read `+40 + 4*channel`; complete encoding unknown. |
| `50` | Unnamed status | Read by an unused ROM helper. |
| `100 + ch*20` | Channel source | 32-bit bus address |
| `104 + ch*20` | Channel destination | 32-bit bus address |
| `108 + ch*20` | Channel linked-list pointer | 32-bit descriptor address |
| `10c + ch*20` | Channel control | Low 16 bits count; bit 16 mode discussed below; bit 17 last item; bit 18 linked-list enable. Bits 4/5 are pulsed during allocation/free. |
| `110 + ch*20` | Channel start | Bit 0 set starts, cleared stops. |
| `174` | Channel-3 `+14` / reset-like control | Unused helper ORs bit 0 and clears the software allocation mask; reset semantics unknown. |
| `178` | Channel-3 `+18` status | Unused helper reads it; meaning unknown. |

All accesses are 32-bit words. The reconstruction names CTRL bit 16 `inc`, but
the ROM copy passes zero while advancing source and destination, so bit 16 does
not enable address increment. A hold/fixed-address meaning is plausible
(inferred).

The fields pulsed during channel allocation overlap the low 16 count bits; the
pulses occur before count setup and after use. Their reset/flush role is
inferred from ordering.

## Firmware transfer descriptor

The ROM's software descriptor is 0x24 bytes (channel, source, destination, LLI
pointer, mode/count, list selection). It is a RAM structure, not hardware
registers; the C layout is under "Structure layout" below. An LLI is four words:
`{source, destination, next_pointer, control}`. The hardware channel handles
the first chunk; descriptors represent subsequent chunks.

| Descriptor offset | Width | Meaning |
|---|---:|---|
| `00` | u8 | Channel index, followed by three padding bytes |
| `04`, `08`, `0c` | u32 each | Source, destination, LLI bus addresses |
| `10` | u32 | Unused in the copied path (meaning unknown) |
| `14` | u8 | Control-bit-16 argument, followed by padding |
| `18` | u32 | Transfer byte length |
| `1c` | u8 | Loaded by wait helper, then discarded/reused; meaning unknown |
| `1d` | u8 | Use-LLI flag, followed by two padding bytes |
| `20` | u32 | Unused in the copied path (meaning unknown) |

For a short transfer, `mem:0x20009b1c` clears the last-item bit, sets source,
destination and bit 16, selects whether a list is used, and sets count if it is
below `0x10000`. `mem:0x20009d0c` sets start bit 0. Completion
`mem:0x20009ca8` polls `+1c` up to `0x10000000` busy iterations, but returns
zero even when that bound expires. The counter is an iteration count, not a time.

### Structure layout

From ROM type recovery (descriptor and register file of the second DMA controller at `0x38800000`):

```c
/* DMAC2 transfer descriptor */
typedef struct ROM xfer {
    u8  ch;                            /* 00 channel */
    u8  pad1[3];
    u32 src;                           /* 04 */
    u32 dst;                           /* 08 */
    u32 lli;                           /* 0c linked-list address */
    u32 unk10;                         /* 10 */
    u8  inc;                           /* 14 address-increment flags */
    u8  pad15[3];
    u32 len;                           /* 18 */
    u8  unk1c;                         /* 1c */
    u8  use_lli;                       /* 1d */
    u8  pad1e[2];
    u32 unk20;                         /* 20 */
} ROM xfer;

typedef struct dmac2_ch {              /* 0x100 + ch*0x20, 4 channels */
    u32 SRC;          /* 00 */
    u32 DST;          /* 04 */
    u32 LLI;          /* 08 */
    u32 CTRL;         /* 0c [15:0] count, b17 last, b18 LLI enable, b4/b5 cleared at alloc */
    u32 START;        /* 10 b0 = go */
    u32 R14;          /* 14 channel 3: +0x174 set to 1 by ROM reset */
    u32 R18;          /* 18 channel 3: +0x178 read by ROM get_status178 */
    u32 R1C;
} dmac2_ch;
typedef struct dmac2_regs {
    u32 REG00;        /* 00 */
    u32 REG04;        /* 04 */
    u32 REG08;        /* 08 */
    u32 REG0C;        /* 0c */
    u32 REG10;        /* 10 */
    u32 rsv14[2];
    u32 BUSY;         /* 1c bit ch set while channel ch is active (ROM wait_done) */
    u32 rsv20[4];
    u32 CTL;          /* 30 b0 enable, b1/b2, b3/b4 config bits */
    u32 rsv34[3];
    u32 STAT40[4];    /* 40 + ch*4 */
    u32 REG50;        /* 50 ROM get_status50 */
    u8  pad54[0x100 - 0x54];
    dmac2_ch CH[4];   /* 100 */
} dmac2_regs;
```

## Used copy path and unused library paths

The live ROM helper `mem:0x2000085c`:

1. Enables the clock and global control.
2. Allocates a channel from a four-bit RAM allocation mask.
3. Copies in chunks of at most `0xf000` bytes, with bit 16 zero.
4. Programs each chunk, starts, waits for busy to clear, advances memory pointers.
5. Releases the channel, clears its source/destination/LLI/control/start words,
   disables the controller and gates its clock.

The function calls the allocator twice on the same descriptor
(`mem:0x200008a4`, `0x200008ac`) and releases only the second channel. The
allocation wrapper ignores allocation failure and always returns zero. These
are firmware bugs, not hardware requirements.

For length ≥`0x10000`, the unused descriptor-builder path constructs chunks of
`0xff80` bytes in channel-private buffers at `0x0fff0000 + channel*0x400`.
A final list item sets control bit 17, intermediate items set bit 18. The
`0xf000`-chunk copy helper never reaches this path; an exact-multiple remainder
appears to generate a zero-count last item. This code is unverified.

The unused self-test/reset path writes test patterns to `base + channel*0x20`
(the global area) instead of the channel registers at `base+0x100`, then expects
zero values. It gives no reset values or reset recipe.

## Completion and unknowns

The busy bit is the only completion signal the ROM copy path checks. No
peripheral request selector is programmed; this is a plain memory copy. The restore OS ISR
`disk mem:0x0802ef7c` (IRQ 29) reads `+04`, acknowledges each set bit
through `+08` and calls a per-channel callback table; error status (`+10`) is not handled there. `+1c`, `+30` bit 0 and
`+110` bit 0 correspond to the PL080 ENBLDCHNS, CONFIG.E and channel CONFIG.E
roles; only the channel control word differs from PL080.

Unknown: bit-16 meaning, control bits 1…4, the `+40` status words, `+50`,
`+174`, `+178`, list edge cases, reset values, outstanding bus-request drain,
transfer alignment, maximum list capacity, bandwidth.

## Finding ledger

| Finding | Evidence | Type / limit |
|---|---|---|
| Base, four channels, start and count encoding | ROM `mem:0x200094e4/0x200096b8/0x20009b1c/0x20009d0c` | observed; used path only |
| Gate selection | ROM `mem:0x20009a14` | observed |
| Busy-poll bound and ignored timeout | ROM `mem:0x20009ca8` | observed; not a success guarantee |
| Double-allocation and dead self-test anomalies | ROM `mem:0x2000085c/0x20009894` | observed; firmware defects |
| Basic copy in the boot ROM | simulation of the boot ROM | dynamic analysis; no silicon timing |
