# General-purpose DMA controllers (DMAC0 and DMAC1)

The nano 5G has two eight-channel PL080-style DMA controllers at physical
`0x38200000` and `0x38700000`. They service peripheral request signals and
memory copies. The separate ROM memory-copy engine at `0x38800000` has a
different channel-control format; see [DMAC2](dma2.md). NAND's normal FIL path
uses [dedicated FMC DMA](nand-fmss.md), not these channels.

DmaLight numbers the controllers 1 for DMAC0 and 2 for DMAC1. The bases differ
by `0x500000`.

## Resources and addressing

| Property | DMAC0 | DMAC1 |
|---|---|---|
| Physical base | `0x38200000` | `0x38700000` |
| VIC interrupt | 16 | 17 |
| Clock gate | `0x3c500048` bit 11 | `0x3c500048` bit 12 |
| Channels | 0…7 | 0…7 |
| Channel base | `base + 0x100 + channel*0x20` | Same formula |

Gate bits clear to enable. IRQ assignments are observed in restore registration
`disk mem:0x08031844` and handlers `0x0802efb0/0x0802efb4`.
`disk mem:0x08071504` (stop path) shows the bases, channel stride and
terminal-count acknowledgement; function `0x08071540` shows global enable and
control-word construction.

## Register map

All registers are 32-bit words. The map comes from the PL080 definitions
(reproduced below); "observed" rows also appear in nano 5G firmware paths.

| Global offset | Role | Evidence |
|---|---|---|
| `00` | Combined interrupt status | Core definition |
| `04` | Terminal-count interrupt status | Observed completion polling |
| `08` | Terminal-count clear, one bit/channel | Observed W1C write |
| `0c`, `10` | Error status / error clear | Core definition and shared driver |
| `14`, `18` | Raw terminal-count / error status | Core definition |
| `1c` | Enabled-channel mask | Core definition |
| `20`, `24` | Software burst / single requests | Core definition; no nano 5G use seen |
| `28`, `2c` | Software last burst / last single requests | Core definition; no nano 5G use seen |
| `30` | Global configuration, bit 0 enable | Observed write `1` |
| `34` | Request synchronization control | Core definition; field use not seen |

| Channel offset | Role | Use |
|---|---|---|
| `00` | Source bus address | Memory pointer or peripheral data register |
| `04` | Destination bus address | Memory pointer or peripheral data register |
| `08` | Linked-list pointer | Four-word descriptor chain; DmaLight's simple path writes zero |
| `0c` | Transfer control | Count, width, burst, increment and terminal-count interrupt |
| `10` | Channel configuration | Enable, source/destination request selectors, transfer flow, interrupt enables |

### Control and configuration fields

`CONTROL[11:0]` is a transfer-unit count (not always bytes). Width codes
0/1/2 are 8/16/32 bits; source width is `[20:18]`, destination width `[23:21]`.
Source burst is `[14:12]`, destination burst `[17:15]`; standard codes 0…7
represent 1,4,8,16,32,64,128,256 transfers. Source/destination increment are
bits 26/27. Bit 31 requests terminal-count interrupt. DmaLight's masks/tables at module offsets `0x750…0x7d0` and its
programmer at `0x554`.

The shared core also defines source/destination AHB master bits 24/25
and protection bits `[30:28]`; nano 5G wiring is not decoded. `CONFIG` bit 0 enables the channel; source request
is `[4:1]`, destination request `[9:6]`, flow `[13:11]`. DmaLight uses flow 0 for
memory→memory, 1 for memory→peripheral, 2 for peripheral→memory, 3 for
peripheral→peripheral, plus bit 15 for terminal-count interrupt. The common core
also defines error interrupt bit 14, lock 16, active 17, halt 18 and additional
peripheral-controlled flows; the original firmware examples do not exercise all of them.

A linked-list item contains `{source, destination, next, control}`, four
32-bit words. Standard next-pointer bits `[31:2]` carry the aligned address;
bit 0 selects an AHB master. DmaLight ignores its passed LLI argument and
writes zero; audio playback uses descriptor chaining.

### Register definitions

PL080 core register definitions, taken from the Rockbox PL080 header (Rockbox contributors, GPL-2.0-or-later; not every field is used on the nano 5G):

```c
/* general defines */
#define DMAC_CH_COUNT               8
#define DMAC_LLI_MAX_COUNT          0xfff
#define DMAC_CH_PRIO(x)             (x)
#define DMAC_CH_BASE(dmac_ba,ch_n)  ((dmac_ba) + 0x100 + ((ch_n) << 5))

/* PL080 controller registers */
#define DMACINTSTS(base)            (*((uint32_t volatile*)((base) + 0x00)))
#define DMACINTTCSTS(base)          (*((uint32_t volatile*)((base) + 0x04)))
#define DMACINTTCCLR(base)          (*((uint32_t volatile*)((base) + 0x08)))
#define DMACINTERRSTS(base)         (*((uint32_t volatile*)((base) + 0x0c)))
#define DMACINTERRCLR(base)         (*((uint32_t volatile*)((base) + 0x10)))
#define DMACRAWINTTCSTS(base)       (*((uint32_t volatile*)((base) + 0x14)))
#define DMACRAWINTERRSTS(base)      (*((uint32_t volatile*)((base) + 0x18)))
#define DMACENABLEDCHANS(base)      (*((uint32_t volatile*)((base) + 0x1c)))
#define DMACSOFTBREQ(base)          (*((uint32_t volatile*)((base) + 0x20)))
#define DMACSOFTSREQ(base)          (*((uint32_t volatile*)((base) + 0x24)))
#define DMACSOFTLBREQ(base)         (*((uint32_t volatile*)((base) + 0x28)))
#define DMACSOFTLSREQ(base)         (*((uint32_t volatile*)((base) + 0x2c)))
#define DMACCONFIG(base)            (*((uint32_t volatile*)((base) + 0x30)))
#define DMACSYNC(base)              (*((uint32_t volatile*)((base) + 0x34)))

/* PL080 controller channel registers */
#define DMACCxSRCADDR(base)         (*((void* volatile*)((base) + 0x00)))
#define DMACCxDESTADDR(base)        (*((void* volatile*)((base) + 0x04)))
#define DMACCxLINK(base)            (*((uint32_t volatile*)((base) + 0x08)))
#define DMACCxCONTROL(base)         (*((uint32_t volatile*)((base) + 0x0c)))
#define DMACCxCONFIG(base)          (*((uint32_t volatile*)((base) + 0x10)))

/* PL080 controller channel LLI */
#define DMACCxLLI(base)             ((struct dmac_lli volatile*)(base))

/* PL080 DMA controller configuration register */
#define DMACCONFIG_E_POS            0       /* DMAC enable */
#define DMACCONFIG_E_MSK            0x1
#define DMACCONFIG_M1_POS           1       /* AHB Master 1 endianness */
#define DMACCONFIG_M1_MSK           0x1
#define DMACCONFIG_M2_POS           2       /* AHB Master 2 endianness */
#define DMACCONFIG_M2_MSK           0x1

#define DMACCONFIG_E_BIT            (1 << DMACCONFIG_E_POS)
#define DMACCONFIG_M1_BIT           (1 << DMACCONFIG_M1_POS)
#define DMACCONFIG_M2_BIT           (1 << DMACCONFIG_M2_POS)

#define DMACCONFIG_M_LITTLE_ENDIAN  0
#define DMACCONFIG_M_BIG_ENDIAN     1

/* PL080 DMA controller channel LLI register */
#define DMACCxLINK_LM_POS           0
#define DMACCxLINK_LM_MSK           0x1
#define DMACCxLINK_NEXTLLI_POS      2
#define DMACCxLINK_NEXTLLI_MSK      0x3fffffff

/* PL080 channel control register */
#define DMACCxCONTROL_I_POS         31      /* terminal count interrupt */
#define DMACCxCONTROL_I_MSK         0x1
#define DMACCxCONTROL_PROT_POS      28      /* protection bits */
#define DMACCxCONTROL_PROT_MSK      0x7
#define DMACCxCONTROL_DI_POS        27      /* destination addr increment */
#define DMACCxCONTROL_DI_MSK        0x1
#define DMACCxCONTROL_SI_POS        26      /* source addr increment */
#define DMACCxCONTROL_SI_MSK        0x1
#define DMACCxCONTROL_D_POS         25      /* destination AHB master */
#define DMACCxCONTROL_D_MSK         0x1
#define DMACCxCONTROL_S_POS         24      /* source AHB master */
#define DMACCxCONTROL_S_MSK         0x1
#define DMACCxCONTROL_DWIDTH_POS    21      /* destination transfer width */
#define DMACCxCONTROL_DWIDTH_MSK    0x7
#define DMACCxCONTROL_SWIDTH_POS    18      /* source transfer width */
#define DMACCxCONTROL_SWIDTH_MSK    0x7
#define DMACCxCONTROL_DBSIZE_POS    15      /* destination burst size */
#define DMACCxCONTROL_DBSIZE_MSK    0x7
#define DMACCxCONTROL_SBSIZE_POS    12      /* source burst size */
#define DMACCxCONTROL_SBSIZE_MSK    0x7
#define DMACCxCONTROL_COUNT_POS     0       /* n SWIDTH size transfers */
#define DMACCxCONTROL_COUNT_MSK     0xfff

#define DMACCxCONTROL_WIDTH_8       0
#define DMACCxCONTROL_WIDTH_16      1
#define DMACCxCONTROL_WIDTH_32      2

#define DMACCxCONTROL_BSIZE_1       0
#define DMACCxCONTROL_BSIZE_4       1
#define DMACCxCONTROL_BSIZE_8       2
#define DMACCxCONTROL_BSIZE_16      3
#define DMACCxCONTROL_BSIZE_32      4
#define DMACCxCONTROL_BSIZE_64      5
#define DMACCxCONTROL_BSIZE_128     6
#define DMACCxCONTROL_BSIZE_256     7

#define DMACCxCONTROL_INC_DISABLE   0
#define DMACCxCONTROL_INC_ENABLE    1

#define DMACCxCONTROL_I_BIT         (1 << DMACCxCONTROL_I_POS)

/* protection bits */
#define DMAC_PROT_PRIV              (1 << 0)
#define DMAC_PROT_BUFF              (1 << 1)
#define DMAC_PROT_CACH              (1 << 2)

/* bus */
#define DMAC_MASTER_AHB1            0
#define DMAC_MASTER_AHB2            1

/* PL080 channel configuration register */
#define DMACCxCONFIG_E_POS          0       /* enable */
#define DMACCxCONFIG_E_MSK          0x1
#define DMACCxCONFIG_SRCPERI_POS    1       /* source peripheral */
#define DMACCxCONFIG_SRCPERI_MSK    0xf
#define DMACCxCONFIG_DESTPERI_POS   6       /* destination peripheral */
#define DMACCxCONFIG_DESTPERI_MSK   0xf
#define DMACCxCONFIG_FLOWCNTRL_POS  11      /* DMA transfer type */
#define DMACCxCONFIG_FLOWCNTRL_MSK  0x7
#define DMACCxCONFIG_IE_POS         14      /* interrupt error mask */
#define DMACCxCONFIG_IE_MSK         0x1
#define DMACCxCONFIG_ITC_POS        15      /* interrupt terminal count mask */
#define DMACCxCONFIG_ITC_MSK        0x1
#define DMACCxCONFIG_L_POS          16      /* lock */
#define DMACCxCONFIG_L_MSK          0x1
#define DMACCxCONFIG_A_POS          17      /* active */
#define DMACCxCONFIG_A_MSK          0x1
#define DMACCxCONFIG_H_POS          18      /* halt */
#define DMACCxCONFIG_H_MSK          0x1

#define DMACCxCONFIG_E_BIT          (1 << DMACCxCONFIG_E_POS)
#define DMACCxCONFIG_IE_BIT         (1 << DMACCxCONFIG_IE_POS)
#define DMACCxCONFIG_ITC_BIT        (1 << DMACCxCONFIG_ITC_POS)
#define DMACCxCONFIG_L_BIT          (1 << DMACCxCONFIG_L_POS)
#define DMACCxCONFIG_A_BIT          (1 << DMACCxCONFIG_A_POS)
#define DMACCxCONFIG_H_BIT          (1 << DMACCxCONFIG_H_POS)

#define DMACCxCONFIG_FLOWCNTRL_MEMMEM_DMA         0
#define DMACCxCONFIG_FLOWCNTRL_MEMPERI_DMA        1
#define DMACCxCONFIG_FLOWCNTRL_PERIMEM_DMA        2
#define DMACCxCONFIG_FLOWCNTRL_PERIPERI_DMA       3
#define DMACCxCONFIG_FLOWCNTRL_PERIPERI_DSTPERI   4
#define DMACCxCONFIG_FLOWCNTRL_MEMPERI_PERI       5
#define DMACCxCONFIG_FLOWCNTRL_PERIMEM_PERI       6
#define DMACCxCONFIG_FLOWCNTRL_PERIPERI_SRCPERI   7

/*
 * types
 */
struct dmac_lli {
    void* srcaddr;
    void* dstaddr;
    uint32_t link;
    uint32_t control;
} __attribute__((aligned(16)));

struct dmac_tsk {
    struct dmac_lli volatile *start_lli;
    struct dmac_lli volatile *end_lli;
    uint32_t size;
    void *cb_data;
};

/* used when src/dst peri is memory */
#define DMAC_PERI_NONE    0x80

struct dmac_ch_cfg {
    uint8_t srcperi;
    uint8_t dstperi;
    uint8_t sbsize;
    uint8_t dbsize;
    uint8_t swidth;

struct dmac_lli {
    void* srcaddr;
    void* dstaddr;
    uint32_t link;
    uint32_t control;
} __attribute__((aligned(16)));
```

## Peripheral routing table

The 28-entry DmaLight table, checked against `bootloader DmaLight.pe`. `—` means
unavailable (`request=0x10`, no controller mask). Names come from the addressed
peripheral and shared register map; the request numbers are from firmware.

| Apple endpoint index | Data register | Role / working name | DMAC0 request | DMAC1 request |
|---:|---|---|---:|---:|
| 0 | `0x38300040` | LCD write FIFO | 3 | — |
| 1 | `0` | Unused | — | — |
| 2 | `0x3cc00020` | UART0 TX | 6 | — |
| 3 | `0x3cc00024` | UART0 RX | 7 | — |
| 4 | `0x3db00020` | UART1 TX | 8 | — |
| 5 | `0x3db00024` | UART1 RX | 9 | — |
| 6 | `0x3dc00020` | UART2 TX | — | 8 |
| 7 | `0x3dc00024` | UART2 RX | — | 9 |
| 8 | `0x3dd00020` | UART3 TX | — | 12 |
| 9 | `0x3dd00024` | UART3 RX | — | 13 |
| 10 | `0x3e500020` | UART4 TX | 4 | — |
| 11 | `0x3e500024` | UART4 RX | 5 | — |
| 12 | `0x3ca00010` | I2S0 TX | 10 | — |
| 13 | `0x3ca00038` | I2S0 RX | 11 | — |
| 14 | `0x3cd00010` | I2S1 TX | — | 2 |
| 15 | `0x3cd00038` | I2S1 RX | — | 3 |
| 16 | `0x3d400010` | I2S2 TX | 0 | 4 |
| 17 | `0x3d400038` | I2S2 RX | 1 | 5 |
| 18 | `0` | Unused | — | — |
| 19 | `0x38a00080` | FMC FIFO window | — | 0 |
| 20 | `0` | Unused | — | — |
| 21 | `0x3c300010` | SPI0 TX | — | 10 |
| 22 | `0x3c300020` | SPI0 RX | — | 11 |
| 23 | `0x3ce00010` | SPI1 TX | 14 | — |
| 24 | `0x3ce00020` | SPI1 RX | 15 | — |
| 25 | `0x3d200010` | SPI2 TX | 12 | — |
| 26 | `0x3d200020` | SPI2 RX | 13 | — |
| 27 | `0` | Unused | — | — |

Index 28 is the allocator's memory pseudo-endpoint; both controllers are
allowed and request selector zero is used. Raw addresses are passed to the
transfer programmer for memory. DMAC0 requests 4/5 point to `0x3e500020/24`, and DMAC1 request 0 to the
NAND controller FIFO `0x38a00080`.

### DmaHeavy table (diagnostics)

`diag DmaHeavy.pe` is a second PL080 driver (tree `anbu_n33_138`) for DMAC0 `0x38200000` and DMAC1 `0x38700000`
with a 28-endpoint table (`+0xa8c` addresses, `+0x9a4` request pairs, `+0x984` controller masks). It equals the DmaLight table above except:

- Endpoints 10 / 11 (UART4 TX / RX `0x3e500020/24`): address 0, no request.
- Endpoints 21 / 22 (SPI0 TX / RX): requests DMAC0 4 / 5 and DMAC1 10 / 11, mask 3 (DmaLight: DMAC1 only). This matches the shared S5L8702 header, which labels request 4 / 5 on DMAC0 as SPI0.

Controller masks (bit 0 DMAC0, bit 1 DMAC1) by endpoint index 0..28: `1,0,1,1,1,1,2,2,2,2,0,0,1,1,2,2,3,3,0,2,0,3,3,1,1,1,1,0,3`
(`diag DmaHeavy.pe+0x984`; DmaLight `bootloader DmaLight.pe+0x7f4/0x814/0x8fc`).

DmaHeavy chunking and control (`diag DmaHeavy.pe+0x2d8..0x4f0`, `+0x54a`, `+0x6e0..0x860`, `+0x868..0x8c4`):

- At most `0xfff` units per descriptor: word width if `(src | dst | len) & 3 == 0` (chunk `0x3ffc` bytes), halfword if `& 1 == 0` (`0x1ffe` bytes), else byte (`0xfff`).
- Per channel and controller, a 16-byte LLI `{src, dst, next, control}` in a `0x100`-page pool; control = old `& 0xf3fff000` `|` count `|` bit 26 `|` bit 27.
- Config = `(dst req & 0xf) << 6 | (src req & 0xf) << 1 | flow << 11 | 0x8001`; flow 0 / 1 / 2 / 3 is chosen from whether the source and destination index is below `0x1c`.
- Completion: poll `+0x04` bit ch, write the bit to `+0x08`, timeout result `0x80000012`.

A request selector chooses an internal peripheral handshake signal, not a GPIO
pin. Request timing, FIFO thresholds and synchronization latency are not
known.

## Start, completion and shutdown protocol

DmaLight allocation intersects source/destination controller masks, claims the
first available channel, enables the controller (`+30=1`), and writes width,
burst and terminal-count control. When no channel is free, the bootloader code spins; the restore OS uses a
mutex/event-based allocator.

The programmer writes source/destination addresses, zero LLI for its simple
path, count and increment bits, then channel CONFIG with request selectors,
flow, terminal-count enable and channel-enable bit. DmaLight derives count
from byte length shifted by destination width; the core definition uses
source-width units. They agree for same-width transfers, which are the only
ones seen.

DmaLight waits for terminal-count status, clears the channel bit in `+08`, then
releases. A nonzero timeout returns without clearing the status; zero timeout
waits indefinitely. Its stop routine clears CONFIG bit 0 and writes the
terminal-count clear bit. It does not check that in-flight bus transactions
have drained (no idle/active check). It does not poll DMA errors while waiting,
so a failed transfer looks like a timeout.

For linked audio buffers, the CPU publishes descriptors/data before enabling
DMA and does not recycle them while the active chain can still reach them.
Physical request timing and cache-coherence behavior are not validated on the
device.

## Finding ledger

| Finding | Evidence | Type / limit |
|---|---|---|
| Two controller bases, stride, enable and stop | `disk mem:0x08071504/0x08071540` | observed |
| Endpoint/request tables and width/burst fields | Flat-PE offsets `0x750/0x75c/0x768/0x79c/0x7f4/0x814/0x8fc`, decoded tables; all arrays read from bytes | observed; specific to this model, not all S5L87xx |
| Exact table image | `bootloader DmaLight.pe` SHA-256 `22fcdda3c2e9ea53ce3a5bcaf4e8b9fd99de8222080fdea16d9db171c2d5501a` | observed; one firmware version |
| Start/count/flow and completion behavior | `bootloader DmaLight.pe` module offsets `0x2c0/0x438/0x554/0x6dc`; reference, comparison tests | observed; mixed-width and handshake timing not tested |
| Additional PL080 fields | PL080 definitions (Register definitions below) | core definitions; not each seen on the nano 5G |
| DmaHeavy tables equal DmaLight except endpoints 10 / 11 and 21 / 22 | `diag DmaHeavy.pe+0x984/0x9a4/0xa8c` | observed |
| DmaHeavy chunking, LLI and config encoding | `diag DmaHeavy.pe+0x2d8..0x8c4` | observed |
