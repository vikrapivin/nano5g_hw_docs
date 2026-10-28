# Public-key accelerator (PKA)

Register interface of the SoC big-number accelerator. Base is physical `0x3d000000`, 32-bit MMIO accesses.
The block has a command register that starts one two-source operation on operand slots held in a private
operand RAM at `+0x800`, a per-slot flag word, a geometry register that sets the operand width, and one
interrupt source (enable and status registers; never used by the ROM or the bootloader). The clock gate is
called PKE: disk ID 36, PWRCON1 bit 13 (ROM direct-gate ID `0x22`, `+0x4c` bit 13); see
[clocks-reset.md](clocks-reset.md).

Evidence is the boot ROM driver (`rom_pka_*`, `bootrom mem:0x20008494..0x20008f30`,
`rom_pka_run_program` `bootrom mem:0x2000b174`, `rom_pka_modexp` `bootrom mem:0x20008d7c`), the bootloader
module `PKE.pe` (10 functions; `wtf PKE.pe` has the same 10 functions at the same offsets) and the restore
OS copy (`disk mem:0x0803290c..0x08032c56`, `disk mem:0x08073ebc`). The three drivers write the same register
values in the same order. Register names are working names.

## Register map

| Offset | Working name / access | What the driver does |
|---|---|---|
| `0x00` | Geometry, R/W | Full-word write `((units-1) << 3) \| mode` (`bootrom mem:0x200086c4..0x200086cc`, `bootloader PKE.pe+0x8aa`). Bits `[6:3]` = unit count minus 1, bits `[1:0]` = mode (words per unit minus 1). All other bits written 0. Read back as operand word count `(([6:3]+1) * ([1:0]+1))`. See geometry below. |
| `0x04` | Unused | No access found. |
| `0x08` | Start / busy, R/W | Write `1` (or `9`) to start one operation, then poll until bit 0 reads 0. Only bit 0 is ever tested. |
| `0x0c` | Operation word, W | `dst \| (src1 << 16) \| (src2 << 24)`: three slot numbers, bits `[15:8]` always 0 as written. Written before every start. |
| `0x10` | Slot flags, R/W | Bit n belongs to slot n. Driver writes 0 to clear all, clears single bits, copies a bit with a slot copy; never sets one directly. Meaning below. |
| `0x14` | Control, R/W | Bits `[7:6]` slot-size code, bit 0 set together with it, bit 1 operation modifier (set and cleared around single operations). Meaning of bits 0 and 1 unknown. |
| `0x18` | Unnamed, W | `rom_pka_set_reg18` writes a value and returns 1; no caller. Meaning unknown. |
| `0x1c` | Interrupt status, R/W | Bit 0 tested by the ISR; the ISR writes back the value it read. |
| `0x20` | Interrupt enable, R/W | Bit 0 = enable. `rom_pka_irq_enable` writes 1, `rom_pka_irq_disable` writes 0. |
| `0x800..0xfff` | Operand RAM | 0x800 bytes, 32-bit word accesses by the CPU. |

Software start value: `1` for every operation except the first operation of a program and the operand-load
operation, which write `9`. Meaning of bit 3 unknown.

The ROM start helper writes `+0x08` and polls it (`rom_pka_start` `bootrom mem:0x20008790`; bootloader
`PKE.pe+0x804` and `+0x902`; disk `mem:0x08032c56`). A separate wait helper (`bootrom mem:0x2000877c`, no
caller) polls without writing.

## Operand RAM

The RAM is 0x800 bytes at `+0x800`, split into equal slots. The slot size is chosen by `+0x14` bits `[7:6]`
and is the maximum operand width in bytes.

| `+0x14` `[7:6]` | Slot size | Slots | Maximum width | Size code (`rom_pka_size_code`) | Top scratch slot `T` |
|---|---|---|---|---|---|
| `0` | `0x100` bytes | 8 | 2048 bits | `7` | `6` |
| `1` | `0x80` bytes | 16 | 1024 bits | `0xf` | `0xe` |
| `2` | `0x40` bytes | 32 | 512 bits | `0x1e` | `0x1d` |

`rom_pka_select_size(bytes)` takes the slot size in bytes. It writes the whole `+0x14` word: `0x80 \| 1` for
`0x40`, `0x40 \| 1` for `0x80`, `0x01` for `0x100`, and returns 1. Any other argument leaves `+0x14`
unchanged and returns the current `[7:6]` field (`bootrom mem:0x20008538..0x20008578`). Slot `n` starts at
`+0x800 + (n << 8)`, `(n << 7)` or `(n << 6)` for the three codes (`bootrom mem:0x200087c4..0x200087e8`,
`mem:0x20008990..0x200089a8`). Value `3` in `[7:6]` is never written. The size codes in the table are
returned by `rom_pka_size_code(0x40/0x80/0x100)` (`bootrom mem:0x20008a84`); the "top scratch slot" column is
that code minus 1 (`T`), the value placed in the source fields of the constant-conditioning operations
(`bootrom mem:0x20008de4..0x20008df4`).

Two helpers disagree with this spacing and have no callers: `rom_pka_slot_addr` (`bootrom mem:0x2000873c`)
and `rom_pka_clear_slot` (`mem:0x200085c4`) compute `{0x40, 0x20, 0x10} * n` bytes, a factor 4 smaller.

**Word order (observed).** A slot is an array of 32-bit words with word 0 at the lowest address. The
CPU-side carry chains (add, shift, subtract) run from word 0 upward, so word 0 is the least significant
word (inferred). `rom_pka_store_operand` (`bootrom mem:0x20008ab4`) copies the source words straight into
the slot, zero-fills the rest of the slot, and clears the slot's flag bit. `rom_pka_read_operand`
(`mem:0x20008b4c`) reads the top word first and byte-swaps each word, so the output buffer is a big-endian
byte string. `rom_pka_load_operand` (`mem:0x2000883c`) zeroes the slot, copies `min(buffer bytes, geometry
bytes)` and then starts an operation (below).

**Reset of the RAM.** `rom_pka_clear_ram` (`mem:0x2000857c`) writes `0x200` zero words over the whole RAM and
then writes 0 to `+0x10`. It runs right after the clock is enabled. Nothing clears the RAM before the clock
is gated off again.

### Geometry (`rom_pka_calc_geometry`)

Input is the operand width in bits (`bootrom mem:0x20008630`; same code at `bootloader PKE.pe+0x812`). The
unit width grows with the width so that the unit count fits the 4-bit field (at most 16 units).

| Bits | Mode (`[1:0]`) | Unit | Units |
|---|---|---|---|
| up to `0x200` (values below `0x80` become `0x80`) | 0 | 32 bits | ceil(bits / 32) |
| `0x201..0x400`, except `0x2a0`, `0x360` | 1 | 64 bits | ceil(bits / 64) |
| `0x2a0`, `0x360`, and `0x401..0x600` except `0x500`, `0x580` | 2 | 96 bits | ceil(bits / 96) |
| `0x500`, `0x580`, and above `0x600` | 3 | 128 bits | ceil(bits / 128) |

The five special sizes are exact multiples of the unit that the table gives them (inferred: that is why
they are special-cased). Mode 2 divides by 3 with a multiply by `0xaaaaaaab` in the ROM
(`bootrom mem:0x200086e4`) and with a software divide helper in the bootloader (`PKE.pe+0x220`). Above `0x800`
bits the unit count no longer fits `[6:3]`; the driver has no check for that. The maximum operand width the
geometry can describe is 16 units of 128 bits = 2048 bits, equal to the largest slot.

`rom_pka_max_bits` (`bootrom mem:0x20008f30`, no caller) returns `32 * ([6:3] + 1)` read back from `+0x00`
and succeeds when that is in `0x80..0x200`; it ignores the mode field.

## Operation sequence

Per operation (ROM `rom_pka_run_op_a` `bootrom mem:0x20008908`, bootloader `PKE.pe+0x5ac`, disk
`mem:0x0803290c`):

1. Write `+0x0c` = `dst | src1 << 16 | src2 << 24` (slot numbers).
2. Write `+0x08` = `1` (or `9`).
3. Poll `+0x08` until bit 0 is 0. There is no timeout.

An optional modifier is set in `+0x14` bit 1 before step 1 and cleared after step 3 (read-modify-write;
`rom_pka_set_flag14` `bootrom mem:0x2000871c`, bootloader `PKE.pe+0x8ae`).

After the final operation of a chain the driver tests the flag bit of the result slot in `+0x10`; if it is
set, the driver clears the bit and adds slot 0 into the result slot over the geometry word count with carry
(`bootrom mem:0x20008954`, `bootloader PKE.pe+0x794`).

## Operations

| Operation | Command | Start | Notes |
|---|---|---|---|
| Two-source multiply | `dst \| src1<<16 \| src2<<24` | `1` | Used as square (`src1 == src2`) and as general multiply (`rom_pka_run_op_a`). Always followed by its own poll. The arithmetic is modular with the modulus in slot 0 (inferred: slot 0 is the add-back operand and the source of the CPU-side reductions). A Montgomery-style multiplier is consistent with the constant-conditioning steps below (inferred). |
| Multiply with modifier | same, `+0x14` bit 1 set around it | `1` | `rom_pka_run_op_c` (`bootrom mem:0x20008a04`); followed by the conditional add of slot 0. Effect of the modifier unknown. |
| Raw command | full `+0x0c` word | `1` | `rom_pka_run_op_b` (`mem:0x20008934`), no caller. |
| Operand load | `dst=a, src1=top scratch slot, src2=slot` | `9` | `rom_pka_load_operand`: the loaded slot is `src2`, the product goes to `dst`. |
| Copy | none | none | CPU `memcpy` of `geometry bytes` between two slots, then the flag bit is copied (`rom_pka_copy_slot` `bootrom mem:0x200087a8`). Not a block operation. |
| Clear | none | none | CPU zero fill (`rom_pka_clear_ram`, store-operand tail). Not a block operation. |

No single-command modular exponentiation was found. `rom_pka_modexp` builds exponentiation from the
multiply operation, two slot copies, and CPU bit scanning (below).

### `rom_pka_modexp` (`bootrom mem:0x20008d7c`)

Called by one function outside the block driver (`bootrom mem:0x20005ab8`). Arguments: output buffer, bit
width, three `{pointer, length}` buffers. The three buffers play these roles (inferred from the slots
they reach): one is stored into slot 0, one is loaded into slot 1 and converted into slot 3, one is only
scanned bit by bit from CPU memory and never enters the RAM.

| Step | Action |
|---|---|
| 1 | Clock on: `rom_pka_clock_on` equivalent, ID `0x22` (`bootrom mem:0x20008d98`). |
| 2 | `rom_pka_clear_ram`. |
| 3 | Slot size by width: up to `0x200` bits `0x40`, up to `0x400` `0x80`, else `0x100`; size code from `rom_pka_size_code`. |
| 4 | `rom_pka_calc_geometry(bits)`, then `rom_pka_select_size(slot size)`. |
| 5 | Store the first buffer into slot 0. |
| 6 | `rom_pka_run_program(bits)` (below): leaves a constant in the top scratch slot. |
| 7 | Clear flag bits of slots 1, 2, 3. Load second buffer into slot 1, product into slot 3 (start `9`). |
| 8 | Set `+0x14` bit 1; multiply with `dst=2`, both sources the top scratch slot; clear bit 1. |
| 9 | For each bit of the scanned buffer, most significant first, 32 per word: multiply `dst=1, src=2,2`; then if the bit is 1 multiply `dst=2, src1=3, src2=1`, else copy slot 1 to slot 2. |
| 10 | Final multiply with modifier, `dst=1, src=2,2`, then the conditional add of slot 0. |
| 11 | Read slot 1 out (word-reversed, byte-swapped), clock off (set the gate bit). |

## `rom_pka_run_program` (`bootrom mem:0x2000b174`)

Not a table interpreter. It is a CPU routine plus a short fixed chain of hardware steps. With `T` the top
scratch slot of the slot size in use (`6`, `0xe`, `0x1d`):

1. The CPU zeroes slots `T-2..T` (`bootrom mem:0x2000b178..0x2000b1ec`).
2. The CPU does multiword shift-left, compare and subtract on the operand RAM, working against slot 0 and an
   accumulator slot (`+0x500`, `+0x600`, `+0x700` from the RAM base for the `0x100`, `0x80`, `0x40` slot
   sizes: slot `5`, `0xc`, `0x1c`). The arithmetic is not decoded further.
3. The hardware chain below runs; every step squares one slot into another (`src1 == src2`).
4. If flag bit `T` of `+0x10` is set (bit `6`, `14`, `29`), the CPU adds slot 0 into slot `T` with carry.
5. The CPU zeroes slots `T-2` and `T-1` and clears flag bit `T`. The result stays in slot `T`.

Table `bootrom mem:0x2000b8e0`: 12 words in the `+0x0c` format, `dst` in `[7:0]`, `src` in both `[23:16]`
and `[31:24]`. The same 12 words are at `bootloader PKE.pe+0x107c`, `wtf PKE.pe+0x107c` and
`disk mem:0x08074628`; the literal `0x3d000800` precedes the table (`mem:0x2000b8dc`). The code picks a word
by slot size, then by step. The chain length depends on the geometry mode (`+0x00` bits `[1:0]`), observed at
`bootrom mem:0x2000b560..0x2000b73c`. Steps below are `dst<-src,src`, all numbers hex.

| Mode | Steps | `0x40` slots | `0x100` slots | `0x80` slots |
|---|---|---|---|---|
| 0, 2 | 4 | `1d<-1c`, `1c<-1d`, `1b<-1c`, `1d<-1b` | `6<-5`, `5<-6`, `4<-5`, `6<-4` | `e<-c`, `d<-e`, `c<-d`, `e<-c` |
| 1 | 5 | `1d<-1c`, `1c<-1d`, `1d<-1c`, `1c<-1d`, `1d<-1c` | `6<-5`, `5<-6`, `6<-5`, `5<-6`, `6<-5` | `e<-c`, `d<-e`, `e<-d`, `d<-e`, `e<-d` |
| 3 | 6 | `1d<-1c`, `1c<-1d`, `1d<-1c`, `1c<-1d`, `1b<-1c`, `1d<-1b` | `6<-5`, `5<-6`, `6<-5`, `5<-6`, `4<-5`, `6<-4` | `e<-c`, `d<-e`, `e<-d`, `d<-e`, `c<-d`, `e<-c` |

The first step is written with start `9`, later steps with start `1`. Each step is followed by the busy poll:
`bootrom mem:0x2000b754` writes `+0x0c` and `+0x08`, then falls into the wait at `mem:0x2000b75c`. The restore
copy (`disk mem:0x08073ebc`) has the same three start-`9` writes, the same table and the same chain; the
bootloader copy has them at `PKE.pe+0xd24`, `+0xdac`, `+0xe54` in code not defined as a function; the start writes were confirmed from the instruction stream.

## Interrupt

| Item | Value |
|---|---|
| Enable | `+0x20` bit 0 |
| Status | `+0x1c` bit 0 |
| ISR | `rom_pka_isr` (`bootrom mem:0x200084fc`): reads `+0x1c` and `+0x20`, ANDs them, and if bit 0 is set calls the function pointer stored at `[state + 0x528]` (if nonzero, no arguments); then writes the read `+0x1c` value back to `+0x1c` (acknowledge, inferred write-1-to-clear). The acknowledge is written whether or not the test passed. |
| Install | `rom_pka_irq_enable(cb)` writes `+0x20 = 1` and `cb` into `[state + 0x528]` (`state` is the pointer loaded through `bootrom mem:0x20008bb4`); `rom_pka_irq_disable` writes both back to 0. Only tail-call veneers reach them (`mem:0x20008f28`, `mem:0x20008f2c`), and nothing calls the veneers. |

No reference to `rom_pka_isr`, no VIC registration for it, and no write of `+0x20` in the modexp path were
found in the boot ROM. The ten functions of the bootloader module touch only `+0x00/08/0c/10/14` and the RAM.
The restore copy has no absolute-address reference to `+0x18/1c/20`.

## Clock and reset

- Gate: ROM ID `0x22` (`+0x4c` bit 13, enabled by clearing). The ROM turns it on at the start of
  `rom_pka_modexp` (`bootrom mem:0x20008d9c`) and off at the end (`mem:0x20008f18`). Helpers
  `rom_pka_clock_on/off` are `mov r0,#0x22; b rom_clkgen_enable/disable_gate` (`mem:0x20008d6c`, `0x20008d74`).
- No block reset register or reset write exists in any of the three drivers. After the gate is enabled the
  driver clears the RAM and flags itself, then writes `+0x00` and `+0x14` as full words. Reset values of the
  registers are unknown.

## Timing

All completion waits are unbounded poll loops on `+0x08` bit 0 (no counter, no delay, no timeout) in the
ROM, the bootloader and the restore copy. Operation latency, block clock rate and the clock-enable settling
time are not measured. Loop bounds in the driver: RAM clear `0x200` words, exponent scan 32 iterations per
word, run-program chain 4, 5 or 6 steps.

## Unknowns

- `+0x08` bit 3 (start value `9`), `+0x14` bits 0 and 1, `+0x18`, `+0x04`.
- What the slot flags in `+0x10` mean in hardware (inferred only as a result-sign/overflow marker, because
  the driver adds slot 0 back when one is set).
- IRQ line number, and what sets `+0x1c` bit 0.
- Reset values, whether `+0x00` bits are read-only capability or plain storage, behaviour above 2048 bits.
- Whether the RAM keeps its contents with the gate off, and whether the CPU may read the RAM while an
  operation runs.
- Exact semantics of the multiply (modulus slot, Montgomery factor); only the slot pattern is known.

## Finding ledger

| Finding | Evidence | Type | Limit |
|---|---|---|---|
| Block base, gate ID `0x22` / `+0x4c` bit 13 | `bootrom mem:0x20008d98`, `mem:0x2000169c` | observed | disk ID 36 from [clocks-reset.md](clocks-reset.md) |
| `+0x00` geometry layout and mode table | `bootrom mem:0x20008630..0x20008700`, `bootloader PKE.pe+0x812` | observed | |
| Units are exact fits for the five special sizes | same code | inferred | |
| Slot size, slot count, spacing from `+0x14 [7:6]` | `bootrom mem:0x20008538`, `mem:0x200087a8`, `mem:0x2000883c` | observed | codes 3 never written |
| Size code to scratch-slot index | `bootrom mem:0x20008a84`, `mem:0x20008de4` | observed | |
| `rom_pka_slot_addr` / `clear_slot` use 4x smaller spacing | `bootrom mem:0x2000873c`, `mem:0x200085c4` | observed | no callers |
| Word order, little-endian words, word 0 lowest | `bootrom mem:0x20008ab4`, `mem:0x20008b4c`, CPU carry loops | observed; ordering of significance inferred | |
| Command word format `dst/src1<<16/src2<<24` | `bootrom mem:0x20008704`, `0x20008908`, table `mem:0x2000b8e0` | observed | role names (dst, src) inferred |
| Start values `1` and `9`, poll on bit 0 | `bootrom mem:0x20008790`, `mem:0x2000b588`, `mem:0x2000b75c` | observed | bit 3 meaning unknown |
| Operation is a two-source modular multiply | `bootrom mem:0x20008e6c..0x20008ee8`, `0x2000b8e0` | inferred | no spec; modulus and domain unknown |
| No single-command exponentiation | `rom_pka_modexp` loop, `bootrom mem:0x20008e9c..0x20008ef0` | observed | bounded: ROM, bootloader PKE, restore copy |
| `run_program` is CPU code plus 4-6 table steps | `bootrom mem:0x2000b174..0x2000b73c`, `disk mem:0x08073ebc` | observed | CPU arithmetic not decoded |
| Table identical in ROM, bootloader, WTF, restore | `bootrom mem:0x2000b8e0`, `bootloader PKE.pe+0x107c`, `disk mem:0x08074628` | observed | bootloader program routine not defined as a function (writes at `PKE.pe+0xd24`, `0xdac`, `0xe54`) |
| ISR, enable, acknowledge | `bootrom mem:0x200084a0..0x20008534` | observed | write-1-to-clear inferred |
| ISR never installed | x-refs to `0x200084fc`, `0x200084a0`, `0x200084bc` | observed | searched bootrom only; bootloader PKE.pe has no `+0x1c/0x20` access in its 10 functions; restore image checked by absolute-address xrefs only |
| Gate toggled per modexp; no reset write | `bootrom mem:0x20008d9c`, `mem:0x20008f18` | observed | |
| All waits unbounded | `bootrom mem:0x20008798`, `0x200088f8`, `0x2000b75c`; `bootloader PKE.pe+0x902` | observed | |
