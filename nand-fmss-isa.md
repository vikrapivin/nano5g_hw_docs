# FMSS sequencer: instruction set, events, registers and programs

The NAND controller (`FMC`, MMIO `0x38a00000`; register map in
[NAND bus controller and FMSS](nand-fmss.md)) contains a small programmable
sequencer that Apple's driver calls the "M2 engine" / FMSS. The CPU writes
parameter words, points the sequencer at a program in system memory, starts
it and waits for a completion interrupt; the program itself drives chip
select, NAND command/address cycles, status polls, the two sector buffers and
two parity buffers, the BCH engine and the controller's own DMA into system
memory. One start handles a whole list of pages over up to eight chips.
The sequencer does not use the PL080 DMA controllers.

The microcode is not ARM code. Everything below was recovered from the
program bytes, from the CPU code that starts and services the sequencer, and
from the dataflow inside the programs. There is no data sheet.
Evidence tags: **obs** = seen in the bytes or in named CPU code (address given);
**inf** = inferred from the instruction stream and its callers.

Coordinates: citations use the standard forms. `bootloader NandReadOnly.pe+0xN`
is an offset into the N33 bootloader NandReadOnly module (flat PE); the DFU
loader module `wtf NandReadOnly.pe` has the same bytes at the same offsets;
`disk mem:0xN` is the restore/disk OS image and `osos mem:0xN` the iPod OS image.

## Encoding (obs)

Every instruction is eight bytes, two little-endian words:

```
word0 = opcode << 24 | R << 16 | arg (u16)
word1 = imm (u32)
```

- The program counter is a byte offset from the start of the program; branch
  targets (`imm` of `bz`/`bnz`) and the fault vector are byte offsets. Programs
  are placed at an eight-byte-aligned bus address written to `FMSS_PROG`
  (`FMC+0xc04`) and D-cache-cleaned by the CPU before the start (obs:
  _m2fmssInit `bootloader NandReadOnly.pe+0x591c`, `disk mem:0x08054cc8`).
- Eight 32-bit registers `r0..r7`. They are CPU-visible as `FMC+0xc18..0xc34`
  (stride 4). The CPU writes `r5/r6/r7` (`0xc2c/0xc30/0xc34`) as inputs of the
  boot-page programs and reads `r6` (`0xc30`, `FMSS_SEQ_RESULT`) after END
  (obs: m2fmssReadBLPage `disk mem:0x08055586..0x08055590`,
  m2fmssWriteBLPage `disk mem:0x08055ad6..0x08055ae0`; result reads in
  m2fmssReadPages `disk mem:0x0803a7c0`, m2fmssWritePages `disk mem:0x0803b12a`,
  m2fmssEraseSingleBlock `disk mem:0x080549a2`).
- `arg` of an MMIO operation is a 12-bit offset from the FMC base
  (`0x000..` FMC, `0x800..` ECC, `0xc00..` FMSS, `0xd00..` parameter RAM).
- Re-assembling the six programs from their decoded instruction streams
  reproduces every copy in every image byte for byte (18 copies; obs).

## Opcodes (obs for the encoding and the counts; semantics inf)

Seventeen opcode values occur in the six distinct programs; no other value
occurs in any copy (obs: opcode histogram over the 1,372 instructions of the
six programs, counted from the image bytes).

| op | mnemonic | semantics (inf) | count |
|---|---|---|---:|
| `00` | `end` | Stop; raise `CS_IRQ` bit 0; `r6` is presented as `FMSS_SEQ_RESULT`. Both words zero. | 12 |
| `01` | `wr [REG], #imm` | `MMIO[arg] = imm` | 450 |
| `02` | `st [REG], rR` | `MMIO[arg] = rR` | 137 |
| `03` | `ldi rR, [rA]` | `rR = mem32[rA]` (system memory through a bus-address pointer held in `rA`) | 24 |
| `04` | `ld rR, [REG], mask` | `rR = MMIO[arg] & imm` (`imm = 0xffffffff` reads the whole word) | 198 |
| `05` | `movi rR, #imm` | `rR = imm` | 33 |
| `06` | `mov rR, rA` | `rR = rA` | 9 |
| `07` | `wait EVT[, arg, mask]` | Block until event `R` fires (table below); event 12 also takes `arg` (expected value) and `imm` (mask) | 138 |
| `0a` | `and` | ALU, see operand rule | 42 |
| `0b` | `or` | ALU | 75 |
| `0c` | `add` | ALU | 43 |
| `0d` | `sub` | ALU | 21 |
| `0e` | `bnz rR, label` | `if (rR != 0) pc = imm` | 42 |
| `11` | `sti [rA], rR` | `mem32[rA] = rR` | 41 |
| `13` | `shl` | ALU (shift amount in `imm` or in a register) | 42 |
| `14` | `shr` | ALU | 53 |
| `17` | `bz rR, label` | `if (rR == 0) pc = imm`; a `bz`/`bnz` pair on the same register is an unconditional jump | 12 |

Per-program counts: READID 46, READ 547, BLREAD 120, ERASE 94, WRITE 456,
BLWRITE 109 instructions; every program contains exactly two `end`
instructions (normal end and fault-handler end).

**ALU operand rule (inferred).** For `and/or/add/sub/shl/shr`:
`imm != 0`: `rR = r(arg) OP imm`; `imm == 0`: `rR = rR OP r(arg)` (register
form). The alternative model (`imm != 0`: `rR = rR OP imm`, `arg` ignored)
was rejected by a dataflow check over all programs: under the documented rule
no instruction reads a register that was never defined and no definition is
dead, under the alternative both occur. This is a consistency argument, not a
silicon measurement.

**Branch encoding (obs).** `bz`/`bnz` carry the register in `R` and the target
byte offset in `imm`; `arg` is zero.

## WAIT events (inf)

A `wait` blocks until the controller event numbered by the `R` field fires.
The event numbers were matched to the `FMC_STATUS` (`FMC+0x048`) bits that the
program acknowledges immediately afterwards; `FMC_STATUS` is write-1-to-clear,
so every `wait` is followed by `wr [FMC_STATUS], #bit`.

| event | meaning | acknowledged with |
|---:|---|---|
| 0 | command byte sent (`FMC_STATUS` bit 1) | `wr [FMC_STATUS], #0x2` |
| 1 | address cycles sent (bit 2) | `#0x4` |
| 2 | DMA transfer done (bit 20) | `#0x100000` |
| 3 | data transfer done (bit 3) | `#0x8` |
| 4 | ECC engine done | `ECC_STATUS` write 1 (`FMC+0x810`) |
| 12 | NAND status poll: after `FMCTRL1 = 0xca` the controller latches the NAND status byte into `FMC_NSTAT` (`FMC+0x04c`); the wait completes when `(FMC_NSTAT & mask) == arg`. Used with mask/arg `0x40` (status bit 6 READY after command 70h) and `0x20` (bit 5 ARRAY READY after command 78h) | status-bit acknowledgements `#0x800000`/`#0x1000000` |

Whether a timed-out `wait` really jumps to the vector in `FMSS_C4C` is
inferred: every program's prologue writes its fault handler's byte offset
there, and the handler ends the program with a fault code (see below).
Nothing in the CPU code reads `FMSS_C4C` back.

## Registers the microcode touches (obs: access pattern; meaning inf unless noted)

Offsets are from `0x38a00000`. The CPU-side register map is in
[NAND bus controller and FMSS](nand-fmss.md); this table records what the
programs do with each word.

| offset | listing name | use by the microcode |
|---|---|---|
| `0x000` | `FMC_CTRL0` | Chip-enable bits `[8:1]` (`CE n = 1 << (n+1)`); timing hold `[14:12]`, setup `[18:16]`, EDO `[30:28]` from `P_TIMING`/`P_WTIMING` or the fixed slowest `0x77000`; bits 0 and 11 always set (`0x801`); READ also sets bit 24 (`0x01000801`), WRITE sets and clears bit 24 around its DMA phases. Deselect = `& 0xfffffe01`. |
| `0x004` | `FMCTRL1` | Transfer trigger; codes below. |
| `0x008` | `FMC_CMD` | NAND command byte (CLE cycle). |
| `0x00c/0x010` | `FMC_ADDR0/1` | Address bytes. Page operations: `ADDR0 = row << 16` (column 0), `ADDR1 = row >> 16`; erase and 78h: `ADDR0 = row`. |
| `0x014` | `FMC_DATA_SEL` | Buffer routing, codes below. |
| `0x018` | `FMC_18` | Written 1 before a data read; meaning unknown. |
| `0x01c` | `FMC_1C` | Which buffer (0/1) the ECC engine works on. |
| `0x028` | `FMC_28` | Written 0 in WRITE; unknown. |
| `0x02c` | `FMC_ANUM` | Address cycles minus 1 (4 = full page address, 2 = row only, 0 = one byte). |
| `0x030` | `FMC_DNUM` | Transfer length minus 1 (`0x1ff` = 512-byte sector; `P_SPARELEN`/`P_PARLEN` for parity; 4 = READ ID). |
| `0x034/0x038` | `FMC_DMA_ADDR/CTRL` | DMA bus address, set once per 2 KiB chunk (the engine advances it per sector, inf); `CTRL = 7` before DMA use, unknown. |
| `0x040` | `FMC_40` | FIFO command: `0x52` latch READ ID bytes; `0xc2` latch corrected meta (poll bit 1 clear); `0xc1` load `FMC_60..68` into the encoder (poll bit 0 clear). |
| `0x048` | `FMC_STATUS` | Event flags, write-1-to-clear: bit 1 cmd, 2 addr, 3 data, 20 DMA, 23/24 NAND status; `0x0ff00ffe` clears all. |
| `0x04c` | `FMC_NSTAT` | NAND status byte latched by `FMCTRL1 = 0xca`. |
| `0x058` | `FMC_58` | Destination of NAND data: 1 data buffer, `0x10`/`0x20` parity buffer 0/1. |
| `0x05c` | `FMC_5C` | `0x200` before the last DMA of a page; unknown. |
| `0x060..0x068` | `FMC_60/64/68` | 12-byte FIFO: READ ID bytes (after `FMC_40 = 0x52`), corrected meta (after `0xc2`), meta to encode (before `0xc1`). |
| `0x804` | `ECC_CTRL` | Start: 1/5 = BCH8/BCH16 decode, 2/6 = encode; `| 0xa00` when sector 0 also covers the ten meta bytes. |
| `0x808` | `ECC_808` | `0x80` in the boot-page programs; unknown. |
| `0x80c` | `ECC_RESULT` | Bit 0 uncorrectable; bits `[20:16]` corrected-bit count. |
| `0x810` | `ECC_STATUS` | Write-1-to-clear: `0x1ff` before a run, 1 acknowledges done; bit 6 = sector was all `0xff` (erased). |
| `0xc0c` | `CS_IRQ` | Write-1-to-clear interrupt flags: bit 0 END, bit 2 fault, bit 3 pause. The prologue writes `0xff`. |
| `0xc10` | `FMSS_C10` | Prologue writes 1 (enable, inf). |
| `0xc30` | `SEQ_RESULT` | `r6` after END. |
| `0xc3c` | `FMSS_C3C` | Read by the fault handler (fault information, unused afterwards). |
| `0xc4c` | `FMSS_C4C` | Fault vector: every prologue writes the byte offset of its fault handler. |
| `0xc50` | `FMSS_C50` | Acknowledged (written) by the fault handler. |
| `0xc58` | `FMSS_C58` | Prologue writes 4 (event-timeout scale, inf). |
| `0xd00..0xd7c` | parameter RAM | Per-program parameter and scratch words (per-listing headers): `P_IDBUF d08`, `P_ROWS d0c`, `P_CEMASKS d10`, `P_ECCBUF d14`, `P_COUNT d18`, `P_META d1c`, `P_DMALIST d20`, ECC words `d2c..d38`, timing words. Pointer parameters are advanced in place (load, add 4, store), so the CPU rewrites them for every run. Several words are reused under different names by different programs. |

The CPU-side code never reads `0xc10/0xc38/0xc3c/0xc4c/0xc50/0xc58` in the
restore OS (obs: no cross-references to those words in `disk`); the
bootloader pause handler is the one CPU write to `FMSS_PC` (`0xc38`), below.

## Code tables (inf)

**`FMCTRL1` transfer codes seen.** The low bits agree with the older Apple
FMC convention (bit 0 address, bit 1 read from NAND, bit 2 write to NAND,
bits 6/7 clear FIFOs): `0x1` send address cycles; `0xe2` read `DNUM+1` bytes
into the buffer chosen by `DATA_SEL`; `0x42` same for sector 0 of a page;
`0x12`/`0x32` read parity/header bytes into a parity buffer; `0xe4` write the
data buffer to NAND; `0x34` write a parity buffer to NAND; `0x2e0` DMA memory
to data buffer; `0x2e4` write the data buffer to NAND while DMA fills the
other; `0x100` DMA data buffer to memory; `0x180`/`0x1e0` same for the last
sector; `0xca` latch the NAND status byte; `0x033338e0` program-start
initialization. `0x20`, `0x10020`, `0x20020`, `0x30020`, `0x100e0` follow
status or data reads and look like transfer-end commands; their meaning is
unknown. The CPU's own status read uses `0xea` then `0x20`
(obs: `disk mem:0x0803ed64`).

**`FMC_DATA_SEL`.** Low byte = buffer being filled (1 data buffer 0, 2 data
buffer 1, `0x10`/`0x20` parity buffer 0/1); bits `[15:8]` = buffer being
drained (`0x100` data buffer 0, `0x200` data buffer 1); `0x1000`/`0x2000` =
parity buffer 0/1 to NAND (encoder output); `0x102`/`0x201` = ping-pong.

**NAND commands issued by the programs.** 90h READ ID (one address byte 00h,
five data bytes); 00h + five address cycles + 30h READ PAGE; 70h READ STATUS
(bit 6 READY, bit 0 FAIL); 00h again to return to data output; 78h READ STATUS
ENHANCED + three row cycles (bit 5 ARRAY READY); 80h + five address cycles +
data + 10h PAGE PROGRAM; 60h + three row cycles + D0h BLOCK ERASE. FFh RESET
is issued by the CPU, not by microcode: there is no RESET program on the 5G
(obs: m2fmssResetAndReadIds `bootloader NandReadOnly.pe+0x5f28` writes
`FMC_CTRL0 = 0x77001 | mask << 1` and `FMC_CMD = 0xff` itself).

**On-flash page format produced and consumed.** Sector 0 =
`[parity P][meta 10][data 512]` with one BCH codeword over meta + data
(order obs: m2GetPageLayout `bootloader NandReadOnly.pe+0x53d4` returns `{P, 10}` for
the meta entry and `{P+10+(k-1)*(P+0x200), 0x200}` for data sector k;
the microcode itself does not show the order);
sectors 1.. = `[parity P][data 512]`; `P` = 13 (BCH8) or 26 (BCH16); eight
sectors per 4 KiB page, sixteen per 8 KiB page. The READ/WRITE loops handle
four sectors per 2 KiB chunk and repeat `P_CHUNKS` times. Boot-region pages
(BLREAD/BLWRITE) are `3 x [parity 26][data 512] = 0x600` bytes, BCH16, no
meta, slowest timing.

## Program structure

Every program has the same six-instruction prologue, a body, `end`, and a
fault handler (obs: all six programs):

```
wr [CS_IRQ], #0xff          ; clear all sequencer IRQ flags
wr [FMSS_C10], #0x1         ; enable (inf)
wr [FMSS_C58], #0x4         ; event-timeout scale (inf)
wr [FMSS_C4C], #fault       ; byte offset of this program's fault handler
wr [FMCTRL1], #0x033338e0   ; start-up init value, meaning unknown
wr [FMC_STATUS], #0x0ff00ffe ; acknowledge every pending FMC event
```

The fault handler reads `FMSS_C3C`, acknowledges `FMSS_C50`, records the
selected chip-enable mask and bit 27 (`0x08000000`) in `r6`, and ends.
READ reports in `r6`: bit 30 uncorrectable-sector summary, bit 29
erased-sector summary, low 16 bits a corrected-count mask; per-sector result
bytes in the ECC buffer are `0..16` (corrected bits), `0xfe` erased, `0xff`
uncorrectable (obs: evaluated by _m2EvalReadResult in `bootloader NandReadOnly.pe`). WRITE
and ERASE report failure in bit 28, evaluated by the callers
(m2fmssWritePages `disk mem:0x0803b12a`, m2fmssEraseSingleBlock `disk mem:0x080549a2`).

Debug constants never read by the CPU: READ `r5` seed `0xa4a3a2a1`, ERASE
`P_STATROW = 0xcafebabe`, BLREAD/BLWRITE `r4` markers
`0x11111111`/`0x22222222`/`0xaaaaaaaa` (obs in the listings; inf that they are
debug markers).

## The six programs (obs)

Sizes are stored in a
u32 word before each program and used by `_m2fmssInit` to clean the D-cache
over the arrays (obs: words at `bootloader NandReadOnly.pe+0x6868`: BLREAD `0x3c0`, READ
`0x1118`, READID `0x170`).

| program | role | insns | bytes | fault vector | SHA-1 of the bytes | started by |
|---|---|---:|---:|---:|---|---|
| READID | probe up to eight CE slots with READ ID (90h), store the first four ID bytes per CE | 46 | `0x170` | `0x128` | `f63cce9c461d69595b437129a989178d09b722b9` | m2fmssResetAndReadIds `bootloader NandReadOnly.pe+0x5f28`; `disk mem:0x080558fc` |
| READ | read pages with BCH decode and DMA, three-stage pipeline over chips | 547 | `0x1118` | `0x10d0` | `814fc8d229467f8d2413ddf5cb2d5351b1ac3307` | `bootloader NandReadOnly.pe` page read; m2fmssReadPages `disk mem:0x0803a7c0` region |
| BLREAD | read one `0x600`-byte boot-region page (BCH16) | 120 | `0x3c0` | `0x370` | `517c578388244f88205e5528bd9a6cf809bf3dc1` | m2fmssReadBLPage `disk mem:0x08055586` region |
| ERASE | erase blocks (60h/D0h) with a status sweep | 94 | `0x2f0` | `0x288` | `512bd93992fe54c8ad89edb1f2c9e380e34cc928` | m2fmssEraseSingleBlock `disk mem:0x080548fc` |
| WRITE | program pages with BCH encode and DMA (80h/10h), multi-chip pipelined | 456 | `0xe40` | `0xdd8` | `084b4a4d9592ed49e80bad8990c7c5949e15d44c` | m2fmssWritePages `disk mem:0x0803b12a` |
| BLWRITE | program one `0x600`-byte boot-region page (BCH16) | 109 | `0x368` | `0x318` | `60b6c6ad38f4f62ddd80b1cad41bbd90154617d4` | m2fmssWriteBLPage `disk mem:0x08055ac4` |

Where the bytes are (mapped addresses):

| program | `bootloader NandReadOnly.pe` and `wtf NandReadOnly.pe` (identical) | disk | osos |
|---|---|---|---|
| READID | `+0x8bb0` | `disk mem:0x08080af8` | `osos mem:0x086bb880` |
| READ | `+0x7a98` | `disk mem:0x0807f9d8` | `osos mem:0x086ba760` |
| BLREAD | `+0x76d8` | `disk mem:0x08081ab8` | `osos mem:0x086bc840` |
| ERASE | not present | `disk mem:0x0807f6e0` | `osos mem:0x086ba468` |
| WRITE | not present | `disk mem:0x08080c70` | `osos mem:0x086bb9f8` |
| BLWRITE | not present | `disk mem:0x08081e80` | `osos mem:0x086bcc08` |

The bootloader and the DFU loader carry only the three read programs. Bounded
search: the program prologue occurs in the osos, disk, N33 bootloader and WTF
images and their extracted EFI modules exactly at the copies above, and nowhere
in the diagnostics image (all 217 extracted module files and the raw image were
searched). Other bootloader or restore-OS builds were not searched.

### Diagnostics image

The diagnostics image carries no sequencer program and its NAND tests do not
touch the FMC, FMSS or ECC registers (`diag TestNand.pe`, whole module). They
reach the NAND through an EFI protocol, `NANDChipID`
(`d5406504-2822-4af7-ab27-71eae90e1e4d`; `diag TestNand.pe+0xb8c`, `+0xf50`):

- method 0 returns eight 64-bit chip IDs, one per CE 0..7 (low word 0 = no chip);
- method 1 returns 64 slots of 16 bytes, a unique ID per die (slot `2*CE` lower
  die, `2*CE+1` upper die).

No provider of this protocol exists in the diagnostics, bootloader or WTF
images: a byte search of all images and modules finds its GUID only in the
test itself. Dynamic analysis confirms it for the shipped pairing. In a
simulated boot of the N33 bootloader into the diagnostics (diagnostics branch
taken through the wheel input the bootloader reads), every driver of both
images is loaded and started: 340 protocol installs of 111 distinct protocols,
after which the diagnostics shell waits for input. The bootloader's own
protocols stay resident and are used by the diagnostics (system configuration
`ca7feebd…`, timer `4eeecd0c…`), and the bootloader's NAND driver installs its
NAND protocol `c4fe7984…`, which no diagnostics module uses. `NANDChipID` is
never installed, so with this bootloader the diagnostics NAND commands find no
provider. The per-die 16-byte unique ID is also something none of the six
programs above reads (READID returns 4 ID bytes per CE); the driver that
supplies it, with either a further sequencer program or CPU-driven NAND cycles,
belongs to a firmware build that is not among these images.

Checks behind this conclusion:

- Both bootloader builds shipped for this model (with iPod OS 1.0.1 and 1.0.2)
  were unpacked: 43 drivers each, the same set, no `NANDChipID`.
- The diagnostics image is byte-identical in the 1.0.1 and 1.0.2 restore
  bundles (only the disk OS, resource volume and iPod OS differ).
- Neither the GUID's first word nor the strings "NANDChipID", "ChipID" or
  "Unique" occur in any driver outside the diagnostics tests.
- The test loads no driver before its lookup; on failure it prints
  "ERROR: could not locate NANDChipID protocol".

A GUID assembled at run time from immediates would evade the byte search;
none of the examined drivers stores GUIDs that way. The likely provider is a
factory or engineering bootloader build that was not shipped.

## How the CPU runs a program (obs)

1. Write the parameter words (`+0xd00..`). `_m2SetEccMode(dir, bch16)`
   (`bootloader NandReadOnly.pe+0x4f4c`) fills the ECC words: decode (dir 2)
   `P_ECCMODE (0xd2c) = 1` (BCH8) / `5` (BCH16), `P_ECCCFG (0xd30) = 0xa01/0xa05`;
   encode (dir 1) `2/6` and `0xa02/0xa06`; `P_SPARELEN (0xd34) = parity + 9`
   (22/35); `P_PARLEN (0xd38) = parity - 1` (12/25), parity = 13 or 26.
2. `FMSS_PROG (0xc04)` = bus address of the program.
3. `_startFMSS` (`bootloader NandReadOnly.pe+0x513c`; `disk mem:0x08043268`):
   `CS_IRQ (0xc0c) = 0xff`; then either the buffer-reset handshake
   (`BUF_CFG (0xc6c) = 0; CS_BUF_RST (0xc60) = 1; poll CS_BUF_RST_OK (0xc64) != 0;
   CS_BUF_RST = 0`) when the context's buffer-reset flag is set, or
   `BUF_CFG = 0x10` otherwise; then `FMSS_CTRL (0xc00) = start word`.
4. The start word is `0xffb5` in the bootloader and the DFU loader
   (obs: literal at `bootloader NandReadOnly.pe+0x5174`) and `0xfff5` in the restore
   OS and iPod OS (obs: literal at `disk mem:0x080432a8`). They differ in
   bit 6 only. The bootloader polls `CS_IRQ` and never touches the VIC; the
   restore OS binds VIC source 54 (`0x36`) to `fmss_isr` (`disk mem:0x0803b2ac`,
   pointer at `disk mem:0x08055060`) in _m2fmssInit (`disk mem:0x08054cc8`) and sleeps on
   an event in its wait routine. Bit 6 is therefore read as the "raise the VIC
   interrupt at END/fault" enable (inf: nothing reads it back). The other
   start-word bits are copied verbatim; their meaning is unknown.
5. Wait: _waitForCSInterrupt `bootloader NandReadOnly.pe+0x51b4` polls
   `CS_IRQ & 0xd`: bit 0 END, bit 2 fault (the handler hook runs and the
   sequencer is reset), bit 3 pause, handled by _m2HandlePauseIrq (`bootloader NandReadOnly.pe+0x4b04`):
   `FMSS_PC (0xc38) += 8; CS_IRQ = 8; FMSS_CTRL |= 1`. The timeout argument is
   in milliseconds (multiplied by 1000 against the microsecond timer); callers
   pass 2000. On timeout `_resetFMSS` runs and the call fails.
   `disk mem:0x0803cce8` waits on the RTXC event signalled by `fmss_isr`.
6. The caller evaluates `FMSS_SEQ_RESULT (0xc30)` as described under
   program structure.

## Known unknowns

- Meaning of `FMSS_C10`, `FMSS_C58`, `FMSS_C3C`, `FMSS_C50` and of the
  start-word bits other than bit 6.
- `FMCTRL1` codes `0x20/0x10020/0x20020/0x30020/0x100e0` and `0x033338e0`;
  the values written to `FMC_18`, `FMC_28`, `FMC_5C`, `ECC_808`, `FMC_DMA_CTRL`.
- Whether a `wait` timeout really vectors through `FMSS_C4C`, and what the
  timeout unit set by `FMSS_C58` is.
- Whether the sequencer executes these byte-identical programs unchanged on
  the 8 KiB-page / BCH8 population (sixteen sectors = four chunks fit the loop
  structure; not run on the device for this reference).

## Finding ledger

| Finding | Evidence | Type / limit |
|---|---|---|
| Eight-byte encoding, 17 opcodes, counts | Image bytes at the offsets above | Observed; semantics are a model |
| ALU operand rule | Dataflow check over all programs (no undefined reads / dead definitions under the documented rule) | Inferred |
| WAIT event numbers | `FMC_STATUS` bits acknowledged after each wait in every listing | Inferred from pairing |
| Start words `0xffb5` / `0xfff5`, bit 6 = IRQ enable | `bootloader NandReadOnly.pe+0x5174`, `disk mem:0x080432a8`, IRQ 54 binding `disk mem:0x08054cc8` | Literals observed; bit meaning inferred |
| Register file at `0xc18..0xc34`, result in `0xc30` | `disk mem:0x08055586..0x08055590`, `0x08055ad6..0x08055ae0`, `0x0803a7c0`, `0x0803b12a`, `0x080549a2` | Observed CPU accesses |
| Pause handling `FMSS_PC += 8` | `bootloader NandReadOnly.pe+0x4b04` | Observed in the bootloader only |
| No RESET program on the 5G | CPU reset in `bootloader NandReadOnly.pe+0x5f28`; prologue search over all decrypted images | Bounded negative search |
| Diagnostics image has no sequencer program; its NAND tests use the `NANDChipID` protocol (per-CE IDs, per-die 16-byte unique IDs) whose provider is not in the examined images | prologue and GUID search over all images and modules; `diag TestNand.pe+0xb8c`, `+0xf50`; simulated bootloader-to-diagnostics boot (111 installed protocols, `NANDChipID` not among them) | observed; dynamic analysis of the bootloader and diagnostics images |
| Unknown words retained | Tables above | Not decoded |
