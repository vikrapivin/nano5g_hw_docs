# NAND bus controller and FMSS sequencer

Programming interface of the S5L8730 flash block, reconstructed from observed
behavior; the register names are working names. Addresses are physical MMIO
unless prefixed with a firmware image. Evidence comes from the nano 5G bootROM,
the flat PE `bootloader NandReadOnly.pe`, restore OS `disk`, and OSOS. The flat
PE's `mem:` values are offsets within that module.

The CPU supplies a program, page descriptors and buffers. FMSS issues the NAND
command/address/data phases, schedules the local sector buffers and the ECC
engine, and uses the FMC's own DMA to move payloads. Normal FIL page I/O does
not allocate a PL080 channel. A separate general-DMA peripheral table also
exposes the FMC FIFO; the normal FIL path does not use it.

## Blocks and signals

| Block | Base | Role |
|---|---|---|
| FMC | `0x38a00000` | NAND bus phases, chip selects, buffering, dedicated DMA |
| BCH ECC | `0x38a00800` | Encode/decode local 512-byte sector buffers; [ECC reference](ecc.md) |
| FMSS | `0x38a00c00` | Sequencer control, registers and interrupts |
| Shared parameters | `0x38a00d00` | CPU/sequencer parameter and scratch words |

Registers are 32-bit words. FMC clock gate is bit 5 and ECC clock gate bit 9 of
`0x3c500048`; clearing a bit enables its clock. ROM `mem:0x20003004` enables
both before NAND access. ROM `mem:0x20001b3c` programs the pin mux:

| GPIO register | NAND mux assignment |
|---|---|
| `0x3cf000c0` | `(old & 0x0000000f) | 0x22222220` |
| `0x3cf000e0` | `0x22222222` |
| `0x3cf00100` | `(old & 0xff0000f0) | 0x00333302` |

These set the pin groups and alternate functions. Which package pad is D0, CLE,
ALE, WE#, RE# or R/B# is not decoded. `CTRL0[8:1]` selects CE0..CE7, one bit
per CE; the polarity of the external chip-select pins is not visible in the
software mask. Writes to `CMD` start command cycles; address cycles follow an
explicit transfer command. Ready/busy is seen through status polling and
controller events. Pin correspondence and signal timing waveforms are not
measured.

## FMC register map

Offsets here and in the FMSS table are from `0x38a00000`. "W1C" means firmware
acknowledges events by writing the set-bit mask.

| Offset | Name / access | Fields and observed use |
|---|---|---|
| `000` | `CTRL0`, R/W | CE mask `[8:1]`; hold `[14:12]`; setup `[18:16]`; read EDO `[30:28]`; DMA path bit 24; CPU FIFO-window bit 25. READ ORs timing with `0x01000801`; WRITE and ERASE OR `0x801` (WRITE sets and clears bit 24 around its DMA phases); BLREAD uses `0x77801`; CPU reset uses `0x77001 | CE_mask<<1`. Remaining bits unknown. |
| `004` | `CTRL1`, R/W | Transfer-operation word; codes below. Reset request `0x800`, wait for bit 30, release with `0x80000000`. |
| `008` | `CMD`, W | NAND command byte; completion is `STATUS` bit 1. |
| `00c` | `ADDR0`, W | Page operation: `column[15:0] | row[15:0]<<16`; row-only erase/status operations use row directly. |
| `010` | `ADDR1`, W | Page row bits `[23:16]`. |
| `014` | `DATA_SEL`, W | Buffer source/destination selection; table below. |
| `018` | unnamed, W | `1` before data reads; meaning unknown. |
| `01c` | ECC slot, W | Select local sector buffer 0 or 1. |
| `024` | unnamed, W | ROM writes zero before reads. |
| `028` | unnamed, W | WRITE program writes zero. |
| `02c` | address count, W | Address cycles minus one: page `4`, row-only `2`, ID `0`. |
| `030` | byte count, W | Transfer bytes minus one: sector `511`; parity `12` or `25`. |
| `034` | DMA address, W | Bus address for dedicated FMC DMA. The page-list ABI supplies one pointer per 2 KiB chunk. |
| `038` | DMA control, W | Observed value `7`; bits not decoded. |
| `040` | FIFO command, R/W | `0x52` latches ID; `0xc2` latches corrected metadata; `0xc1` loads metadata into ECC. Firmware polls bit 1 for latch completion, bit 0 for load completion. |
| `048` | event status, R/W1C | Bits 1 command, 2 address, 3 data, 20 DMA, 23 ready poll, 27 ECC data present. `0x0ff00ffe` clears the events the driver uses. Bit 24 is also cleared in status-poll setup; meaning unknown. |
| `04c` | NAND status, R | Status byte latched by transfer code `0xca`. |
| `058` | NAND input route, W | `1` data buffer; `0x10`/`0x20` parity buffers. |
| `05c` | unnamed, W | `0x200` before the last page DMA; meaning unknown. |
| `060`, `064`, `068` | FIFO words, R/W | READ ID data or 12-byte metadata staging. Only ten metadata bytes are protected; the last two returned bytes are forced to `ff ff`. |
| `080…` | CPU FIFO window, R/W | Word-wise raw-buffer access while `CTRL0` bit 25 is set. Window capacity not determined. |

Evidence: bootROM `mem:0x200091e8`; `bootloader NandReadOnly.pe+0x4730`,
`0x4bac`, `0x4fac`, `0x5fc8`; `disk mem:0x0803ed64`, `0x08040148`,
`0x080546c0`.

### Register definitions

From the NAND reference library's controller header (offsets relative to
`FMC_BASE`; "obs" = access seen in code or microcode, "inf" = meaning inferred):

```c
#define S5L8730_FMC_H
#define FMC_BASE            0x38a00000u   /* physical base of the FMC/ECC/FMSS block (obs) */
/* ===================== FMC registers as driven by the bootROM CPU code ===================== */
#define FMC_CTRL0           0x000   /* chip select / mode / DMA start. bits[8:1] = CE select (1 << (ce+1)), bit0 + bit11 =
                                     * enable/mode (always set together, inf), [14:12]/[18:16]/[30:28] = bus timing
                                     * , bit24 = DMA path */
#define FMC_CTRL1           0x004   /* transfer command: what the next bus phase does (send address, read N bytes into
                                     * FIFO / ECC buffer, DMA a buffer out, read status). Values below */
#define FMC_CMD             0x008   /* NAND command byte; writing it issues one command cycle (STATUS b1 when done) */
#define FMC_ADDR0           0x00c   /* address cycles 0..3: column (16 bit) | row[15:0] << 16 */
#define FMC_ADDR1           0x010   /* address cycle 4: row[23:16] (row = page index within the CE) */
#define FMC_DATA_SEL        0x014   /* data path select bitmask (inf): 0 none, 1 data FIFO, 0x10 spare/ECC bytes,
                                     * 1 << (8+n) DMA from ECC buffer n; microcode uses 0x102/0x201 ping-pong */
#define FMC_ECC_SLOT        0x01c   /* (inf) written 0 before ECC decode: which sector buffer (0/1) the ECC engine reads */
#define FMC_UNK24           0x024   /* (inf) written 0 before each FIFO read; meaning unknown */
#define FMC_ANUM            0x02c   /* address cycle count - 1 (4 = 5 cycles for a page address, 0 = 1 cycle READ ID) */
#define FMC_DNUM            0x030   /* byte count - 1 for the next data phase (0x19 = 26 B parity, 0x1ff = 512 B data) */
#define FMC_DMA_ADDR        0x034   /* DMA destination (bus address) for the next "DMA out" phase */
#define FMC_DMA_CTRL        0x038   /* DMA control, 7 (written before every DMA by the ROM; bits not decoded) */
#define FMC_STATUS          0x048   /* write-1-to-clear status; bits FMC_ST_* below */
#define FMC_ECC_CTRL        0x804   /* ECC engine command: writing it starts an encode/decode */
#define FMC_ECC_RESULT      0x80c   /* ECC result of the last decode: b0 uncorrectable, b[20:16] corrected bit count */
#define FMC_ECC_STATUS      0x810   /* ECC status, w1c: b2 done, b6 sector was all 0xFF (erased) */
/* FMC_CTRL0 bits (obs: ROM uses (1<<(bank+1)) | 0x43801) */
#define FMC_CTRL0_CE(bank)      (1u << ((bank) + 1))   /* chip-enable select: bit (ce+1); ROM uses bank == CE */
#define FMC_CTRL0_ROM_SETUP     0x43801u    /* ROM mode word: bit0|bit11 enable (inf) + timing fields [14:12]=3,
                                             * [18:16]=4 (same layout as m2CalcTiming output, inf), CE bits clear */
#define FMC_CTRL0_ROM_IDLE      0x43802u    /* written at the end of a ROM page read (bootrom FUN_200092b0 step 7):
                                             * bit0 cleared, bit1 set; immediately followed by &= ~0x1ff (deselect) */
#define FMC_CTRL0_DMA_START     0x01000001u /* ORed in before a DMA out: bit24 = DMA path, bit0 */
#define FMC_CTRL0_BIT10         0x400u      /* cleared before DMA (obs, ROM); meaning unknown */
/* FMC_CTRL1 command values used by the ROM (names inf from the phase each one precedes) */
#define FMC_CTRL1_SETUP         0x1f0e0u    /* first write of a ROM page read, before CTRL0/CMD (inf: init/latch mode) */
#define FMC_CTRL1_SEND_ADDR     0xe1u       /* clock out ANUM+1 address cycles from ADDR0/ADDR1 (STATUS b2 when done) */
#define FMC_CTRL1_STATUS_PRE    0x1000e0u   /* before CMD 0x70 in fmc_wait_ready (inf: status-read mode) */
#define FMC_CTRL1_READ_STATUS   0x2au       /* poll the NAND status until ready; sets STATUS b23 (R/B) */
#define FMC_CTRL1_RESTORE       0xe0u       /* back to the idle mode after the status poll */
#define FMC_CTRL1_READ_SPARE    0x32u       /* read DNUM+1 bytes into the ECC parity buffer (DATA_SEL 0x10) */
#define FMC_CTRL1_READ_DATA     0x22u       /* read DNUM+1 bytes into the sector data buffer (DATA_SEL 1) */
#define FMC_CTRL1_DMA_OUT       0x1a0u      /* DMA the (corrected) sector buffer to DMA_ADDR (STATUS b20 when done) */
/* FMC_STATUS bits */
#define FMC_ST_CMD_DONE         (1u << 1)   /* command cycle sent (microcode WAIT event 0) */
#define FMC_ST_ADDR_DONE        (1u << 2)   /* address cycles sent (WAIT event 1) */
#define FMC_ST_DATA_DONE        (1u << 3)   /* data phase finished (WAIT event 3) */
#define FMC_ST_DMA_DONE         (1u << 20)  /* DMA out finished (WAIT event 2) */
#define FMC_ST_READY            (1u << 23)  /* NAND R/B ready seen during FMC_CTRL1_READ_STATUS */
#define FMC_ST_ECC_DATA         (1u << 27)  /* ECC data present: ROM only runs the decoder when this is set */
/* FMC_DATA_SEL values */
#define FMC_DSEL_NONE           0u          /* after the status read */
#define FMC_DSEL_DATA           1u          /* data FIFO / sector buffer */
#define FMC_DSEL_SPARE          0x10u       /* spare / ECC parity bytes (also READ ID FIFO in the microcode) */
#define FMC_DSEL_DMA(n)         (1u << (8 + (n)))   /* DMA source = ECC buffer n */
/* ECC */
#define FMC_ECC_CTRL_DECODE     5u          /* ROM: BCH16 (4) | decode (1) = start a BCH16 decode */
#define FMC_ECC_ST_DONE         (1u << 2)   /* decode finished (ROM spins on it without a bound) */
#define FMC_ECC_ST_ERASED       (1u << 6)   /* sector is all 0xFF */
#define FMC_ECC_ST_CLEAR        0x7fu       /* w1c value clearing bits 0..6 (ROM) */
#define FMC_ECC_RES_UNCORR      1u          /* uncorrectable; bits[20:16] = corrected count */
/* ===================== FMSS / M2 engine (bootloader NandReadOnly) ============
 * obs = access seen in code/microcode, inf = meaning inferred.
 * "µ" in the comments = written only by the microcode; the CPU never touches it on the read path. */
#define FMSS_FMCTRL1            0x004   /* = FMC_CTRL1 ("FMSS__FMCTRL1" in disk-image strings) */
#define FMSS_FMCTRL1_RESET      0x800u          /* _resetFMSS: request FMC reset (obs) */
#define FMSS_FMCTRL1_IDLE       (1u << 30)      /* polled until set (obs) */
#define FMSS_FMCTRL1_RELEASE    0x80000000u     /* written after idle (obs) */
#define FMC_18                  0x018   /* microcode: 1 before each data read (inf) */
#define FMC_ECC_SLOT_SEL        0x01c   /* microcode: ECC buffer 0/1 (inf; same as FMC_ECC_SLOT) */
#define FMC_FIFO_CMD            0x040   /* microcode: 0x52 latch ID FIFO, 0xc2 latch meta; bit1 busy (inf) */
#define FMC_XFER_TARGET         0x058   /* microcode: 0x20/0x10 parity buffers, 1 data (inf) */
#define FMC_5C                  0x05c   /* microcode: 0x200 before the last DMA (inf) */
#define FMC_FIFO0               0x060   /* ID bytes / corrected meta word 0 */
#define FMC_FIFO1               0x064   /* corrected meta word 1 */
#define FMC_FIFO2               0x068   /* corrected meta word 2 (| 0xffff0000) */
#define FMC_ECC_RESET           0x81c   /* _resetFMSS writes 1 (obs) */
/* ===================== nano 5G additions: CPU-side chip reset / status poll ======================
 * bootloader NandReadOnly.pe fmc_select_ces 0x4b98, fmc_send_cmd 0x4c98, fmc_wait_status 0x50f8, fmc_wait_ready_ces 0x4730,
 * m2fmssResetChips 0x5fc8; disk mem:0x0803dd78 / 0x08043254 / 0x08041c38 / 0x0803ed64 / 0x0805599c. (obs) */
#define FMC_CTRL0_CPU_BASE      0x77001u        /* FMC_CTRL0 for CPU-driven commands: bit0, timing [14:12] = [18:16] = 7 (slowest), no CE, no DMA */
#define FMC_CTRL0_CPU(mask)     (FMC_CTRL0_CPU_BASE | ((uint32_t)(mask) << 1))   /* CE mask bit n selects CE n (bit n+1 of CTRL0) */
#define FMC_CTRL1_STATUS_5G     0xeau           /* FMCTRL1: read the NAND status byte after CMD 0x70 */
#define FMC_CTRL1_STATUS_END    0x20u           /* FMCTRL1 written after the status read */
#define FMC_ST_CE_POLL_CLR      0x1800000u      /* FMC_STATUS w1c before a status read (b23 ready, b24) */
#define FMC_ST_ALL_CLR          0x0ff00ffeu     /* FMC_STATUS w1c after a status read (fmc_wait_ready_ces literal 0x4794) */
#define FMC_FIFO_WINDOW         0x080           /* disk image only: word-wise read/write of the current buffer (CTRL0 b25 set while used) */
#define FMC_CTRL0_FIFO_WINDOW   (1u << 25)
#define FMC_CPU_WAIT_FW_US      20000u          /* disk fmc_wait_status bound (mem:0x08041c38); the bootloader's bound is UNVERIFIED */
#define M2_CHIP_RESET_STALL_US  10000u          /* Stall after the all-CE reset (NandReadOnly.pe+0x5fc8 -> efi_Stall(10000)) */
/* ECC engine as used by the FMSS microcode (inf from microcode + NandReadOnly.pe+0x4ae0) */
#define FMC_ECC_CTRL_BCH8_DEC   1u              /* decode, BCH8 (13 B parity per sector) */
#define FMC_ECC_CTRL_ENC        2u              /* encode (write path) */
#define FMC_ECC_CTRL_BCH16      4u              /* strength select: BCH16 (26 B parity per sector) */
#define FMC_ECC_CTRL_META10     0xa00u          /* codeword carries 10 meta bytes (sector 0) */
#define FMC_ECC_RES_NBITS(v)    (((v) >> 16) & 0x1fu)   /* corrected bit count of the last decode (0..16) */
#define FMSS_CTRL               0xc00   /* sequencer control (obs) */
#define FMSS_CTRL_STOP          8u              /* _resetFMSS first write: stop the running program */
#define FMSS_CTRL_ABORT         2u              /* written when CS_STATUS reads 4 (inf: force abort) */
#define FMSS_CTRL_PRE_FMC_RESET 0x80000u        /* written just before FMCTRL1 = 0x800 in _resetFMSS */
#define FMSS_CTRL_START_DISK    0xfff5u         /* _startFMSS of the disk image (disk mem:0x080432a8; IRQ-driven wait; bit meaning unknown) */
#define FMSS_CTRL_START_BL      0xffb5u         /* _startFMSS of the bootloader (NandReadOnly.pe+0x5174; polled wait; bit meaning unknown) */
#define FMSS_CTRL_RESUME        1u              /* ORed in after a pause IRQ */
#define FMSS_PROG               0xc04   /* bus address of the microcode program */
#define FMSS_CS_STATUS          0xc08   /* 0 idle, 3, 4 (obs) */
#define FMSS_CS_STATUS_IDLE     0u
#define FMSS_CS_STATUS_3        3u              /* accepted as "stopped" after an abort (inf) */
#define FMSS_CS_STATUS_4        4u              /* needs FMSS_CTRL_ABORT to stop (inf: waiting on an event) */
#define FMSS_CS_IRQ             0xc0c   /* write-1-to-clear */
#define FMSS_IRQ_DONE           (1u << 0)       /* program executed END */
#define FMSS_IRQ_FAULT          (1u << 2)       /* WAIT timeout -> fault handler ran (inf) */
#define FMSS_IRQ_PAUSE          (1u << 3)       /* program paused; CPU skips the instruction and resumes */
#define FMSS_IRQ_MASK           0xdu            /* DONE | FAULT | PAUSE: the bits the CPU looks at */
#define FMSS_IRQ_ALL            0xffu           /* clear everything (also the microcode prologue value) */
#define FMSS_C10                0xc10   /* microcode: 1 at start (inf enable) */
#define FMSS_R(n)               (0xc18 + 4 * (n))   /* microcode register file r0..r7 (inf) */
#define FMSS_SEQ_RESULT         0xc30   /* = r6 at END */
#define FMSS_SEQ_UECC           (1u << 30)      /* READ: some sector uncorrectable */
#define FMSS_SEQ_ERASED         (1u << 29)      /* READ: some sector erased */
#define FMSS_SEQ_FAULT          (1u << 27)      /* set by the microcode fault handler */
#define FMSS_SEQ_NCORR_MASK     0xffffu         /* OR of (1 << corrected bits) */
#define FMSS_PC                 0xc38   /* byte offset of the current instruction */
#define FMSS_PC_STEP            8u              /* one microcode instruction = 8 bytes */
#define FMSS_FAULT_INFO         0xc3c   /* µ: read into r2 by the fault handler (meaning unknown) */
#define FMSS_FAULT_VECTOR       0xc4c   /* µ: byte offset of the program's fault handler (0x90 / 0x128 / 0x10d0) */
#define FMSS_C50                0xc50   /* µ: fault handler reads b0 and writes 1 (meaning unknown) */
#define FMSS_C58                0xc58   /* µ: 4 at program start (inf: event timeout scale) */
#define FMSS_CS_BUF_RST         0xc60   /* sequencer buffer reset: 1 = request, 0 = release */
#define FMSS_CS_BUF_RST_OK      0xc64   /* reads 1 when the buffer reset is done */
#define FMSS_BUF_CFG            0xc6c   /* buffer mode; _resetFMSS skips the buffer reset handshake when it reads 0x10 */
#define FMSS_BUF_CFG_NORMAL     0u              /* _startFMSS with use_buf_reset = 1 (the default, mem:0x6408) */
#define FMSS_BUF_CFG_BYPASS     0x10u           /* _startFMSS with use_buf_reset = 0 */
/* parameter / scratch words read by the microcode */
/* Written by the CPU before each run (NandReadOnly.pe m2fmssReadPages 0x4d9c, _m2SetEccMode 0x4fcc, m2fmssResetAndReadIds 0x5ab0).
 * Pointers are bus addresses; the microcode advances P_ROWS/P_CEMASKS/P_DMALIST/P_META/P_ECCBUF in place, so all
 * of them are rewritten before every run. */
#define FMSS_P_IDBUF            0xd08   /* READID: destination (4 bytes per CE) */
#define FMSS_P_ROWS             0xd0c   /* READ: u32 *rows; READID: CE count */
#define FMSS_P_CEMASKS          0xd10   /* u32 *, 1 << ce */
#define FMSS_P_ECCBUF           0xd14   /* u8 * per-sector status, 0 = none */
#define FMSS_P_COUNT            0xd18   /* pages */
#define FMSS_P_META             0xd1c   /* 12 bytes per page */
#define FMSS_P_DMALIST          0xd20   /* u32 * per 2 KiB chunk */
#define FMSS_P_STATS            0xd24   /* 0x48-byte ECC histogram */
#define FMSS_P_CHUNKS           0xd28   /* sectors per page / 4 */
#define FMSS_P_ECCMODE          0xd2c   /* ECC_CTRL value for sectors 1..n */
#define FMSS_P_ECCCFG           0xd30   /* ECC_CTRL value for sector 0 */
#define FMSS_P_HDRLEN           0xd34   /* DNUM for meta+parity of sector 0 (parity+9) */
#define FMSS_P_PARLEN           0xd38   /* DNUM for parity (parity-1) */
#define FMSS_V_SUMMARY          0xd3c   /* microcode scratch */
#define FMSS_V_CHUNKCNT         0xd40   /* microcode scratch */
#define FMSS_P_TIMING           0xd4c   /* FMC_CTRL0 timing bits for READ (rSetup<<16 | rHold<<12 | rEdo<<28) */
#define FMSS_V_ECCRES           0xd7c   /* microcode scratch */
/* FMC_CTRL0 timing fields (FIL_CalcTiming bootloader NandReadOnly.pe+0x542c / disk mem:0x08054418): clock counts minus one, 3 bits each */
#define FMC_CTRL0_HOLD(x)       (((x) & 7u) << 12)  /* [14:12] hold clocks (read or write) */
#define FMC_CTRL0_SETUP(x)      (((x) & 7u) << 16)  /* [18:16] setup clocks */
#define FMC_CTRL0_EDO(x)        (((x) & 7u) << 28)  /* [30:28] EDO (read data valid) clocks, read word only */
#define FMSS__FMCTRL0__SETUP_HOLD__MAX 8u           /* asserted limits of FIL_CalcTiming: every setup/hold/EDO clock count < 8 */
#define NAND_CMD_READ_ID        0x90        /* READ ID (microcode READID program, address cycle 0x00) */
/* ===================== restore-OS write path registers / values (disk mem:0x0803b088 .. 0x08055ef4; microcode, byte-identical on the 5G) ============ */
#define FMC_DMA_CTRL_ON         7u              /* FMC_DMA_CTRL value written before every transfer (obs) */
#define FMC_28                  0x028   /* WRITE microcode: 0 before the first page (inf) */
#define FMC_NSTAT               0x04c   /* NAND status byte latched by the microcode status read; b0 = FAIL, b6 = ready (inf) */
#define FMC_FIFO_DATA           0x080   /* CPU PIO data window = FMC_FIFO_WINDOW (read_no_ecc / write_no_ecc: disk mem:0x080546c0 read, 0x08040148 write, obs) */
#define FMC_CTRL0_PIO           (1u << 25)      /* set around CPU FIFO access (obs disk mem:0x080546c0 / 0x08040148) */
#define FMC_ECC_808             0x808   /* boot-page microcode: 0x80 before each ECC operation (inf) */
#define FMSS_R5                 0xc2c   /* microcode r5 (boot-page programs: DMA address) */
#define FMSS_R6                 0xc30   /* microcode r6 (boot-page programs: CE select 1 << (ce+1)); = SEQ_RESULT */
#define FMSS_R7                 0xc34   /* microcode r7 (boot-page programs: row) */
#define FMSS_SEQ_PROGFAIL       (1u << 28)      /* WRITE/ERASE: NAND status FAIL seen (or fault handler) */
#define FMSS_V_D00              0xd00   /* BLREAD: ECC_RESULT of sector 0 (d04, d08 = sectors 1, 2) */
#define FMSS_V_FAILCE           0xd04   /* WRITE: CE mask of the chip whose status read FAILed; ERASE: CPU writes count */
#define FMSS_P_WTIMING          0xd48   /* FMC_CTRL0 timing for program/erase (WRITE program) */
#define FMSS_P_STATROW          0xd78   /* WRITE: 0 = status 0x70 per CE; else row of 0x78 per-plane status reads */
#define NAND_CMD_ERASE1         0x60        /* BLOCK ERASE setup (3 row address cycles) */
#define NAND_CMD_ERASE2         0xd0        /* BLOCK ERASE confirm */
#define NAND_CMD_PROG1          0x80        /* PAGE PROGRAM setup (5 address cycles + data) */
#define NAND_CMD_PROG2          0x10        /* PAGE PROGRAM confirm */
```

### Transfer words and buffer routing

The `CTRL1` values are compound operations, not decomposed into bitfields.

| `CTRL1` value | Observed phase |
|---|---|
| `0x1` | Address cycles |
| `0xe2` | NAND data to selected local buffer |
| `0x42` | First-sector data-read variant |
| `0x12`, `0x32` | Parity/header input |
| `0xe4` | Local data buffer to NAND |
| `0x34` | Parity buffer to NAND |
| `0x2e0` | Memory DMA into data buffer |
| `0x2e4` | Drain one buffer to NAND while DMA fills the other |
| `0x100` | Corrected local data buffer to memory DMA |
| `0x180`, `0x1e0` | Final-sector/window transfer variants |
| `0xca` | NAND status latch used by sequencer WAIT 12 |
| `0xea`, then `0x20` | CPU ready-status poll, then terminate poll |
| `0x033338e0` | Common sequencer initialization write; bits not decoded |
| `0x10020`, `0x20020`, `0x30020`, `0x100e0` | Phase endings; semantics not decoded |
| `0x100f2` | Raw spare-tail input |

`DATA_SEL` low-byte values `1/2` select filled data buffers 0/1; `0x10/0x20`
select parity buffers. `0x100/0x200` select drained data buffers; `0x1000/0x2000`
drain parity. `0x102/0x201` are opposite-slot ping-pong combinations. These
meanings come from producer/consumer access order; concurrent behavior on silicon is not verified. ROM uses a related
set of compound words (`0x1f0e0`, `0xe1`, `0x1000e0`, `0x2a`, `0xe0`, `0x32`,
`0x22`, `0x1a0`) in its CPU-driven reader.

## FMSS control map

| Offset | Role | Observed operation |
|---|---|---|
| `c00` | Control | `8` stop; `2` abort transition; `0x80000` FMC-reset mode; start `0xffb5` in polled bootloader and `0xfff5` in IRQ-driven restore OS; bit 0 resumes a paused program. |
| `c04` | Program bus address | Points to cache-cleaned sequencer instructions, eight bytes each. |
| `c08` | State | Reset code handles states `0`, `3`, `4`; zero is idle. Names for 3/4 are inferred from transitions. |
| `c0c` | IRQ W1C | Bit 0 END, bit 2 fault, bit 3 pause. Clear values `0xff`, `0x0d`, `8` occur. |
| `c10` | unnamed | Common prologue writes `1`. |
| `c18…c34` | R0…R7 | Eight 32-bit sequencer registers, stride four. R6 at `c30` is the completion result. |
| `c38` | PC | Byte offset within program; CPU adds eight when handling pause. |
| `c3c` | Fault information | Read by program fault handler; encoding unknown. |
| `c4c` | Fault vector | Program-relative byte offset of fault handler. |
| `c50` | Fault acknowledgement | Fault handler writes `1`. |
| `c58` | unnamed | Common prologue writes `4`; timeout scale (inferred). |
| `c60`, `c64` | Buffer reset request / acknowledgement | Request `1`, wait acknowledgement `1`, release request `0`. |
| `c6c` | Buffer reset mode | `0` selects handshake path; `0x10` bypasses it. |

The two start words differ only at bit 6. From IRQ use, that bit probably
enables the external completion interrupt (inferred); the other start-word
bits are not decoded. The restore driver binds VIC IRQ 54 in
`disk mem:0x08054cc8`, ISR `0x0803b2ac`. The polled bootloader needs no IRQ
setup.

### Program parameter ABI

This layout belongs to the original programs, not to fixed hardware registers.
CPU and sequencer share the writable words; pointer parameters are advanced in
place and must be rebuilt for each run. The observed parameter window is `d00…d7c`;
whether the silicon holds words beyond `d7c` is not established.

| Offset | Original READ/WRITE use |
|---|---|
| `d00` | Scratch / boot-sector-0 ECC result |
| `d04` | Failed-CE mask (WRITE/ERASE result); boot-sector-1 ECC result (BLREAD). The CPU also stores `nCE*banks` here before ERASE (`disk mem:0x080548fc`), but the ERASE program never reads it |
| `d08` | READID destination; WRITE final status-sweep CE count (`8`); boot-sector-2 ECC result |
| `d0c` | Row-list pointer; READID uses CE scan count |
| `d10` | READ: CE-mask-list pointer, one `1<<physical_ce` word/page. WRITE: descriptor/row-pair list pointer. ERASE: pointer to one CE index (0..7) per block; the program forms `1<<index` itself. |
| `d14` | Per-sector status destination |
| `d18` | READ page count; ERASE block count |
| `d1c` | Metadata pointer, 12 host bytes/page |
| `d20` | DMA-address-list pointer, one word/2 KiB chunk |
| `d24` | ECC statistics/histogram pointer |
| `d28` | Chunks/page: sectors/page divided by four |
| `d2c`, `d30` | Ordinary-sector ECC mode / first-sector ECC+metadata mode |
| `d34`, `d38` | First parity+metadata transfer count / later parity transfer count, both minus one |
| `d3c`, `d40` | Program scratch/counters |
| `d48`, `d4c` | Write / read timing word |
| `d78` | Enhanced-status row selection for WRITE |
| `d7c` | Program scratch/ECC result |

BLREAD/BLWRITE start with R5 = buffer address, R6 = `1<<(ce+1)`, R7 = row.
Their 1,536-byte boot-page format differs from normal 4/8 KiB pages. WRITE's
eight-byte pair is `{descriptor, row}`, terminated by descriptor zero;
`descriptor = status_command<<24 | setup_command<<16 | confirm_command<<8 |
(1<<physical_ce)`, so it carries the command bytes as well as the CE selection.
READ packs four sector-status bytes into a word with the first sector in the
most-significant byte; in little-endian memory the bytes appear in reverse
sector order.

## Initialization and transfer sequence

1. Enable the FMC/ECC clocks and board NAND supply; select the pin mux above.
   The supply is board-dependent; a generic CE GPIO does not replace it.
2. Stop FMSS (`c00=8`). If state is 4, write `2` and wait for state 0 or 3.
   Use the `c60/c64` handshake unless buffer mode is `0x10`.
3. Set `c00=0x80000`, write `004=0x800`, wait for bit 30, then write
   `004=0x80000000`; reset ECC with `81c=1`.
4. CPU-reset the selected chips: `CTRL0=0x77001 | mask<<1`, `CMD=ff`, wait
   command-done, acknowledge it, deselect. Initial all-CE probing waits 10 ms;
   later resets poll per-CE ready status.
5. Run READID across eight CE slots; reject zero/all-ones IDs, form the CE map,
   match [population/geometry tables](nand-chips.md), calculate timing and ECC.
6. Clean program/descriptors/source buffers; clean/invalidate DMA destinations.
   Write the program address and the full parameter ABI. The CPU must not
   reuse a source/destination while FMSS or its local DMA still owns it.
7. Clear sequencer IRQ, perform buffer reset if selected, write the start word.
8. Wait for END/fault. On pause: advance PC by eight, clear pause bit, resume.
   On fault/timeout: stop/reset and treat the output as failed.
9. Read R6 and per-sector ECC results, invalidate destination cache lines,
   then release ownership. END does not make a NAND write atomic; power-loss
   consistency is handled by the VFL/FTL software.

Original CPU waits: 100 µs around sequencer reset/IRQ handshakes, 20,000 µs for
FMC status, 2,000 ms for a program. The start-time buffer-reset wait has no
bound. The bootloader FMC bound is the literal `0x4e20` at
`bootloader NandReadOnly.pe+0x5130`.

## Timing, instruction interface and errors

The sequencer's instruction set, events, registers and program inventory are in
[the FMSS instruction-set page](nand-fmss-isa.md).

`CTRL0` timing holds setup/hold/EDO as three-bit fields. Firmware derives them
from the active bus clock and eight population-table nanosecond parameters;
[chip timing details](nand-chips.md) give the formula. The timing word depends
on bus clock and chip population.

Each instruction is two little-endian words:
`word0 = opcode<<24 | register<<16 | argument16`, `word1 = immediate32`.
Program counters and branch targets are byte offsets. Opcodes: END `00`,
immediate MMIO write `01`, register MMIO store `02`, indirect memory load `03`,
masked MMIO load `04`, immediate `05`, register move `06`, WAIT `07`,
AND/OR/ADD/SUB `0a/0b/0c/0d`, nonzero branch `0e`, indirect memory store `11`,
left/right shift `13/14`, zero branch `17`. Operand details and program
listings are in the sequencer reference.
WAIT event 0 is command completion, 1 address completion, 2 DMA, 3 data, 4 ECC;
12 waits for `(NAND_status & mask) == expected`. Event meanings are inferred
from the surrounding register accesses.

READ result R6: bit 30 uncorrectable-sector summary, bit 29 erased-sector
summary, low 16 bits corrected-count mask. Per-sector bytes are `0…16`
correction counts, `fe` erased, `ff` uncorrectable. WRITE/ERASE bit 28 means
program/status failure; fault handlers also record the selected CE. A
low-level success needs the chip status as well as sequencer END.

## Diagnostics

The diagnostics `TestNand` module touches no FMC/FMSS register; it uses the NANDChipID and SystemConfig protocols only ([nand-chips.md](nand-chips.md)). Bounded to that module.

## Not established

Electrical pinout, setup/hold waveforms, maximum bus frequency by population,
DMA arbitration, reset values, reserved-bit readback and the full fault
encoding. The compound transfer words and unnamed fields above are literal
observed operations. Physical BCH correctness, NAND endurance, multi-plane timing and
recovery after power loss are not shown.

## Finding ledger

| Finding | Source and evidence | Type / limit |
|---|---|---|
| Register offsets and reset/start sequence | `bootloader NandReadOnly.pe+0x4fac/0x513c`, `disk mem:0x08043268` | observed; some field names inferred |
| CPU ready poll and FIFO window | `disk mem:0x0803ed64/0x080546c0/0x08040148` | observed; transfer-word bitfields not decoded |
| IRQ 54 and interrupt start-word difference | `disk mem:0x08054cc8/0x0803b2ac` | observed; bit-6 meaning inferred |
| 20 ms bootloader FMC bound | `bootloader NandReadOnly.pe+0x5130` | observed literal and call argument; not a silicon timeout register |
| Parameter RAM above `d7c` unproven | hardware contract and recorded native-driver failure | observed access boundary; RAM extent not proven |
