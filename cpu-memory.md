# CPU, address spaces and SRAM

Firmware-visible interface of the S5L8730 as configured by the N33 images.
Addresses are physical/MMIO unless called virtual or code addresses. Register
names such as `DREX` are working names.

## Evidence and address domains

- Mask ROM: `bootrom`, `mem:0x20000000` = boot ROM file offset 0.
- N33 PreEfi: `bootloader` PreEfi, XIP addresses such as `mem:0x2200b4b4`.
- Restore OS: `disk` SRAM at `0x22000000` and SDRAM at `0x08000000`.
- `DACR=3` is domain 0 manager (ARM encoding `11`).

## CPU and memory map

| Region | Address / parameter | Established by firmware |
|---|---|---|
| CPU | ARM1176/ARMv6, CP15 | ARM and Thumb code, banked CPSR modes, cache and MMU operations below. ARM1176JZF-S is the working identification. |
| Mask ROM | `0x20000000..0x2000ffff` | 64 KiB image, executes here. Aliasing at reset vector VA/PA zero not measured. |
| SRAM, ROM-used range | `0x22000000..0x2202ffff` | 192 KiB addressed by ROM code, stacks and buffers. |
| SRAM, used by later stages | `0x22000000..0x2206ffff` | Declared 448 KiB (`0x70000` bytes) by the bootloader; evidence and limits in [SRAM size](#sram-size). |
| SDRAM | `0x08000000..0x0bffffff` | PreEfi returns base `0x08000000`, size `0x04000000` (64 MiB) without probing capacity. |
| SDRAM uncached VA alias | `0x88000000..0x8bffffff` | bootloader page tables map it to `0x08000000..0x0bffffff`. Bit 31 is a software alias; a DMA engine may need the translated address. |
| Peripheral window | `0x38000000..0x3fffffff` | individually decoded blocks; not every address exists |
| DREX/DRAM controller | `0x3d700000` | initialized by `bootloader mem:0x2200b4b4` |
| GPIO memory-pad controls | `0x3cf00380/384` | written during GPIO/DRAM init; bit meanings unknown |
| Bus/arbiter windows | `0x38100000`, `0x3d500000`, `0x3e000000`, `0x3ff00000` | known writes in [bus-fabric](bus-fabric.md) and [sdram.md](sdram.md) |

A section descriptor maps a whole 1-MiB region, so a request ending at
`0x2203ffff` does not show that only 256 KiB exists or that the whole mapped
MiB is SRAM.

## CP15, cache and MMU

SecCore `bootloader mem:0x2202257c`:

1. Mask IRQ/FIQ and clear SCTLR with mask `0x04001805` (MMU, D-cache, branch
   prediction, I-cache, bit 26).
2. Invalidate caches and TLB, DSB.
3. Set ACTLR bit 0, then bits 0..2.
4. Reset FCSE to zero.
5. Set SCTLR bits `0x802`, clear bit `0x400000`.

ROM startup instead sets the unaligned-access bit `0x400000`, so the stages
differ in CPU setup. Vendor ACTLR/SCTLR interactions are not decoded.

| Operation | CP15 register | Operand / behavior |
|---|---|---|
| Peripheral port remap | `mcr p15,0,r0,c15,c2,4` | `0x38000012`; written by ROM and SecCore. SoC routing beyond the architectural field is unmeasured. |
| Data cache clean/invalidate | `c7,c14,2` | PreEfi `0x2200a2e4` iterates 128 sets (stride 0x20, limit 0x1000) for 8 values of the way/index field (stride 0x20000000, bits 31:29). The 4-way 16-KiB geometry is the ARM1176 default, not read from the code. |
| Instruction cache invalidate | `c7,c5,0` | PreEfi `0x2200a318`, then barrier |
| TLB invalidate | `c8,c7,0` | PreEfi `0x2200a33c` |
| Write buffer / DSB | `c7,c10,4` | before changing mappings or transferring control |
| TTBR0 | `c2` via `0x2200a32c` | 16-KiB aligned level-one table; no TTBR1/TTBCR in this path |
| DACR | `c3` via `0x2200a334` | `3`: domain 0 manager, other domains no access |
| Enable sequence | SCTLR | clear XP `0x800000`, D `4`, I `0x1000`, M `1`; maintain caches; install table; enable I, D, then M |

### Cache maintenance before sleep (diagnostics image)

The diagnostics image has a module, `IramRoutines`, whose code runs from internal
SRAM ("IRAM") during sleep and idle: it saves the interrupt state, writes the
PMU over I2C0, puts the SDRAM controller into its low-power state and later
re-initializes it, and stops the core with WFI. Code that switches SDRAM off
must not run from SDRAM, and every dirty cache line must reach SDRAM first, so
the routine cleans and disables the caches before touching the SDRAM
controller:

1. Clean and invalidate the data cache by set/way (`c7,c14,2`): 128 sets
   (set index advanced by `0x20`, loop limit `0x1000`) for each of 8 values of
   the way field (advanced by `0x20000000`, bits 31:29).
2. Drain the write buffer (`c7,c10,4`).
3. Disable the data cache (clear SCTLR bit 2) and the instruction cache
   (clear SCTLR bit 12); invalidate the instruction cache (`c7,c5,0`).
4. Wait for interrupt (`c7,c0,4`).

The module also has small helpers that read and set the stack pointer
(`diag IramRoutines.pe+0x220..0x2b8`).

The set/way loop is the same as the one PreEfi runs at boot (`bootloader mem:0x2200a2e4`,
table row above): same set stride and limit, same way stride. Two independently
written firmware stages agree on the cache geometry they assume (32-byte lines,
128 sets, way index in the top bits), which supports the row above; the
geometry itself is still not read from the CPU's cache-type register.

PreEfi `0x2200b06c`, section writer `0x2200cb58`, builds:

| Virtual range | Physical target | Attribute literal (ORed with section address) |
|---|---|---|
| `0x00000000..0x000fffff` | `0x22000000` | `0x4c02` |
| SRAM reservation | identity, section-rounded | `0x4c02` |
| `0x20000000..0x21ffffff` | identity | `0x4c02` |
| `0x08000000..0x0bffffff` | identity | `0x5c06` |
| `0x88000000..0x8bffffff` | SDRAM low physical addresses | `0x4c02` |
| `0x38000000..0x3fffffff` | identity | `0x4c02` |

With the ARMv6 descriptor format, `0x4c02` is normal non-cacheable and
`0x5c06` is write-back/write-allocate. Drivers keep the firmware's ordering
and barriers for MMIO.

## SDRAM and bus programming

The [SDRAM controller reference](sdram.md) lists every recovered setup offset,
the cold/warm command sequences, the DLL poll, timing literals, unknown
pad/bus writes and the final delay loop (1000 dummy reads of `+0x108`).

## Diagnostic memory access

`TestDirectMemoryRW` (`r / w / o / a addr [val]`) does 32-bit `ldr` / `str` on the parsed
address (read, write, OR, AND), prints `0x%X 0x%X` (address, value), takes argc >= 3 for read and 4 otherwise,
and checks neither alignment nor range (`diag TestDirectMemoryRW.pe+0xb54..0xba8`). `TestRam`
is described in [sdram.md](sdram.md).

## Handoff

Bds leaves the CPU in SVC `CPSR=0xd3`, IRQ/FIQ masked, MMU and caches off,
SDRAM stack still selected. Firmware arguments are memory records at
`0x22028cc0`, `0x2206ff10` and `0x2206ff18` (software ABI, not registers). A
replacement runtime sets up its own mappings, vectors, stack, cache coherency
and interrupt ownership.

Unknown: physical SRAM capacity and aliasing beyond used addresses; DRAM part,
bus width and timing-field units; reset defaults of unused registers; power
retention in each sleep state; arbitration/QoS fields; whether each firmware
delay is a hardware minimum.

## SRAM size

The boot ROM only addresses `0x22000000..0x2202ffff` (192 KiB), so the ROM says
nothing about memory above that. The size comes from the later stages and from
one device test. Each source and what it shows:

| # | Source | What the code or test does | What it shows |
|---|---|---|---|
| 1 | `bootloader SystemConfig.pe+0x9e8..0x9f6` | `movs r0,#0x11; lsls r0,#0x19` stores `0x22000000` into the system-configuration record, then `movs r0,#7; lsls r0,#0x10` stores `0x70000` into the adjacent size field | The bootloader declares an SRAM region of base `0x22000000`, size `0x70000` (448 KiB). A declaration, not a probe. |
| 2 | `bootloader mem:0x2200aabe..0x2200aace` (PreEfi, before DXE starts) | builds a resource descriptor: type 5 (reserved memory), attributes `0x407` (literal at `mem:0x2200ab78`), base `0x22001000` (literal at `mem:0x2200ab74`), length `0x6f << 12` = `0x6f000` | Hands the DXE phase a reserved range ending at `0x22070000`, consistent with #1. Also a declaration. |
| 3 | `bootloader Bds.pe+0xc9a..0xcb6` | clears `0x14` bytes at `0x2206ff18` (literal at `Bds.pe+0xd2c`) through a boot-service call, then stores a word at `0x2206ff10` | Firmware writes data near the top of the declared range and relies on it. |
| 4 | iPod OS scatter table (`osos mem:0x22005fa0`, copied by `osos mem:0x22005ec8`) | copies `0x20000` bytes to `0x22030000`; a further entry at `0x22050000` has length 0 | The iPod OS uses SRAM up to `0x2204ffff` (320 KiB). It needs at least that much. |
| 5 | Device test (read-only first, then write) | reads `0x2206ff00..0x2206ff3f`, writes a pattern, reads it back, and checks `0x2200ff00`, `0x2202ff00`, `0x2203ff00` for the pattern | The pattern reads back at `0x2206ff00` and does not appear at the three lower addresses. |

What the device test rules out and what it does not:

- Rules out a mirror with period `0x10000`, `0x20000`, `0x30000` or `0x60000`
  (each would alias `0x2206ff00` onto `0x2200ff00`) and with period `0x40000`
  (alias at `0x2202ff00`). So the top of the range is not a 64, 128, 192 or
  256 KiB SRAM seen through an address mirror.
- Does not rule out a `0x50000` (320 KiB) mirror, which would alias onto
  `0x2201ff00`; that address was not checked.
- Says nothing above `0x2206ff40`: the SRAM could be larger than 448 KiB.
- No full memory test of `0x22030000..0x2206ffff` was run.

Conclusion: at least 320 KiB is in use by the iPod OS; the bootloader declares
448 KiB; one device probe at the top of that range behaves as distinct RAM.
448 KiB is the best-supported figure, not a measured capacity.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| 64-MiB SDRAM contract | `bootloader mem:0x2200b4b4` | observed; no capacity probing |
| Boot ROM addresses only `0x22000000..0x2202ffff` (192 KiB) | ROM memory map, stacks and buffers | observed |
| Bootloader declares SRAM base `0x22000000`, size `0x70000` (448 KiB) | `bootloader SystemConfig.pe+0x9e8..0x9f6` | observed (declaration) |
| Bootloader reserves `0x22001000` + `0x6f000` (to `0x22070000`) for the DXE phase | `bootloader mem:0x2200aabe..0x2200aace` | observed (declaration) |
| Bootloader writes data at `0x2206ff10` and `0x2206ff18` | `bootloader Bds.pe+0xc9a..0xcb6` | observed |
| iPod OS loads code/data up to `0x2204ffff` (320 KiB) | `osos mem:0x22005fa0` scatter table, copied by `osos mem:0x22005ec8` | observed |
| RAM at `0x2206ff00` is distinct from `0x2200ff00`, `0x2202ff00`, `0x2203ff00` | device test (read, write, read back) | device (single probe; `0x50000` mirror not excluded) |
| Ordered DRAM and bus writes | `bootloader mem:0x2200b4b4/0x2200b46c`; restore startup `0x2200043c` | observed; fields partly unknown |
| MMU descriptors and CP15 operations | `bootloader mem:0x2200b06c/0x2200cb58`; SecCore `0x2202257c` | observed |
| In a simulated boot of the bootloader image, PreEfi completes the SDRAM controller initialization and the boot continues with code and data in SDRAM (DXE phase at `0x08000000` upward) | dynamic analysis of the bootloader image | dynamic analysis: confirms the sequence is complete and in order; the simulation answers status polls without DRAM timing, so it does not validate the timing values |
| The diagnostics sleep code (run from SRAM) cleans and invalidates the data cache with the same set/way loop as PreEfi, drains the write buffer and disables both caches before switching SDRAM off | `diag IramRoutines.pe+0x220..0x2b8`; PreEfi `bootloader mem:0x2200a2e4` | observed |
| Diag `r/w/o/a` 32-bit access, no checks | `diag TestDirectMemoryRW.pe+0xb54..0xba8` | observed |
