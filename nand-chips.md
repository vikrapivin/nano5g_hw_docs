# External NAND devices, population and bus timing

The firmware supports several NAND populations; a table entry shows a
supported configuration, not that a given nano 5G contains that part. The
reported device ID `0x2994d5ec` is bytes `ec d5 94 29`, a 4-KiB-page entry.

The interface is an eight-bit NAND command/address/data bus selected by up to
eight controller CE signals. Five-byte ID probing and table matching give the
geometry; the code does not derive geometry from generic extended-ID bitfields.
[FMSS](nand-fmss.md) drives the phases and [ECC](ecc.md) describes the
protected media stream.

## Identification and configuration records

`bootloader NandReadOnly.pe+0x5f28` resets/probes the chips; `0x522c` matches a
population, `0x53a0` finds geometry, `0x536c` finds a platform and `0x5328`
finds its configuration. Platform selector `mem:0x24e8` returns `0x16` for
this nano 5G. Table offsets below are in the flat `046_NandReadOnly.pe`, where
module `mem:` equals file offset (not OSOS runtime addresses).

| Table | Module offset | Rows / stride | Record fields |
|---|---|---|---|
| Chip geometry | `0x6874` | 18 / `0x18` | Layout below |
| CE population | `0x6a28` | 41 / `0x15` | Group count byte; two `{id[5], CE_count}` groups; eight timing bytes |
| Platform configuration | `0x6d8c` | 58 / `0x1c` | Platform word; 13-byte population key; padding; VSVFL configuration word at `+0x14`; trailing word `4` |
| Platform definitions | `0x73e8` | 3 / `0x10` | Platform word (`8`, `0x0a`, `0x16`), then six u16 values `{1,2,3,4,0,0}` |

Chip-record offsets are `+0…4` ID bytes (fifth `ff` wildcard in these rows),
`+5` padding, `+6` u16 blocks/CE, `+8` u16 pages/block, `+0a` u16 data bytes/page,
`+0c` u16 physical spare bytes, `+0e` u16 attribute (full meaning unknown),
`+10` u32 vendor selector, `+14` u32 dies/CE. All multi-byte fields are little
endian. The vendor selector and the VSVFL configuration word are separate values.

Match groups occupy CE slots `4*g … 4*g+nce-1`; unused slots in the group are
empty, so the slot map can contain holes. A logical bank index is translated
through the map to get the CE number.

## Complete bootloader geometry table

All rows were read from firmware bytes. All rows have 128 pages/block and fifth
ID byte `ff`. ECC strength follows the firmware's spare-budget rule.

| ID bytes | Blocks/CE | Data bytes/page | Spare | `+0e` | Vendor selector | Dies/CE | BCH strength |
|---|---:|---:|---:|---:|---:|---:|---:|
| `ec d7 94 72` | 4152 | 8192 | 436 | 12 | 8 | 1 | 16 |
| `ec d7 d5 29` | 8192 | 4096 | 218 | 8 | 2 | 2 | 16 |
| `ec d5 94 29` | 4096 | 4096 | 218 | 8 | 2 | 1 | 16 |
| `ec d5 14 b6` | 4096 | 4096 | 128 | 4 | 2 | 1 | 8 |
| `ec d7 55 b6` | 8192 | 4096 | 128 | 4 | 2 | 2 | 8 |
| `ad d5 14 b6` | 4096 | 4096 | 128 | 4 | 3 | 1 | 8 |
| `ad d7 55 b6` | 8192 | 4096 | 128 | 4 | 3 | 2 | 8 |
| `ad d7 94 25` | 8192 | 4096 | 224 | 12 | 3 | 1 | 16 |
| `98 d5 94 ba` | 4096 | 4096 | 218 | 8 | 1 | 1 | 16 |
| `98 d7 95 ba` | 8192 | 4096 | 218 | 8 | 1 | 2 | 16 |
| `98 d7 94 32` | 4148 | 8192 | 376 | 8 | 1 | 1 | 8 |
| `98 de 95 32` | 8296 | 8192 | 376 | 8 | 1 | 2 | 8 |
| `98 e7 94 32` | 4100 | 8192 | 448 | 12 | 1 | 1 | 16 |
| `89 d5 94 3e` | 4096 | 4096 | 218 | 8 | 6 | 1 | 16 |
| `89 d7 d5 3e` | 8192 | 4096 | 218 | 8 | 6 | 2 | 16 |
| `2c d5 94 3e` | 4096 | 4096 | 218 | 8 | 5 | 1 | 16 |
| `2c d7 d5 3e` | 8192 | 4096 | 218 | 8 | 5 | 2 | 16 |
| `2c d7 94 3e` | 8192 | 4096 | 218 | 12 | 7 | 1 | 16 |

The restore OS and OSOS contain reduced tables: 10 chip rows, 14 population
rows, 14 configuration rows and one platform row. Restore runtime bases are
`mem:0x080781e4`, `0x080782d8`, `0x08078410`, `0x0807859c`. Their population
rows are `0x16` bytes with an extra leading byte and group count at `+1`;
their configuration keys are 14 bytes, so the bootloader-table parser does not
apply unchanged. The population and timing entries are in the annotated
firmware table dump.

## Addressing and command phases

For page I/O, column is 16 bits and row is 24 bits in the observed five-cycle
transfer. `row = physical_block * pages_per_block + page_in_block`. CPU word
packing is `ADDR0=column | row<<16`, `ADDR1=row>>16`. Erase uses only the three
row cycles. The row is a physical page, not a logical FTL sector.

For multiple dies, the firmware computes blocks/die, rounds the die block
stride up to a power of two, and inserts holes between dense die ranges.
Total addressable blocks are `stride*(dies-1)+blocks_per_die`; holes are not
usable blocks. The table block count is the count of real blocks, which can be
less than the largest address plus one.

| Operation | NAND bus sequence | Completion/failure signal |
|---|---|---|
| Reset | `ff` | Initial probe waits 10 ms; subsequent path polls status |
| Read ID | `90`, address `00`, five input bytes | Controller command/address/data events |
| Read page | `00`, five address cycles, `30` | Status `70`, ready bit 6; `00` returns from status to data output |
| Program page | `80`, five address cycles, data, `10` | Ready bit 6, failure bit 0 |
| Erase block | `60`, three row cycles, `d0` | Ready bit 6, failure bit 0 |
| Multi-plane/cache family | Intermediate `11`, alternate setup `81`, cache confirm `15`, status `71` or `78` as selected by chip policy | Enhanced status `78` includes a row address and array-ready bit 5 |

Command selection depends on vendor/configuration, die and plane pairing,
last-operation position and write pattern. Controller ready, cache ready and
array ready are separate completion conditions; a cached program is complete
only after the chip/die status has been drained.

## Timing parameters and encoding

The eight population timing bytes are nanosecond inputs `t[0…7]`; their datasheet names are not assigned. For the two-/four-CE `98 d7 94 32` population, bytes are
`19 0c 0a 19 0c 0a 14 19` (25, 12, 10, 25, 12, 10, 20, 25 ns).
The two-group population uses `32 18 14 32 18 14 00 00`.

`FIL_CalcTiming`, `disk mem:0x08054418`, obtains two clock bounds; the
existing platform helper supplies equal MHz values. With equal values, let
`T=floor(1000/MHz)` and `cd(x)=max(ceil(x/T)-1,0)`:

```text
write_setup = cd(t1 + 4)
write_hold  = cd(max(abs(T*write_setup - t0), t2 + 4))
read_setup  = cd(t4 + 4)
read_start  = T * (read_setup + 1)
read_hold   = cd(max(abs(max(t6 + 6, t3) - read_start), t5 + 4))
read_edo    = ceil(abs(read_start - (t6 + 6)) / T)
read_word   = read_setup<<16 | read_hold<<12 | read_edo<<28
write_word  = write_setup<<16 | write_hold<<12
```

The code narrows several intermediate arguments to bytes, so the formula holds
for small timing inputs. It asserts every field below eight, `read_edo*T <= (read_hold+1)*T`, and
`read_edo*T < t7+6`. With differing clock bounds the assertion period comes
from the other clock output.
READ consumes the read word; WRITE consumes the write word. ERASE loads both
parameters but the original sequencer uses the read timing word.

The software adds 4 ns and 6 ns margins; assert strings associate the 6 ns with
SoC/NAND rise-time terms. Maximum chip frequency, timing distributions,
voltage/temperature dependence and vendor timing modes are not measured.

## Diagnostic queries

The diagnostics `TestNand` module reaches the NAND only through protocols; it has no FMC/FMSS/ECC register access and no chip or geometry table (bounded to this module).

- NANDChipID protocol (GUID `d5406504-2822-4af7-ab27-71eae90e1e4d`; no provider located in any diagnostics, bootloader or DFU module). Method 0 returns eight records `{id_lo, id_hi}`, one per CE0..7 (`id_lo == 0` = absent). `nandcsid` prints `FCEn = 0x...`, a CE-present mask (MSB first) and `NANDID = 0x...` / `mismatch` / `none` (all present CEs must carry the same 64-bit ID) (`diag TestNand.pe+0xb8c`).
- Method 1 fills 64 slots of 16 bytes: slot `2*CE` = "Lower die", `2*CE+1` = "Upper die" (CE index up to 31 in the print loop); nonzero entries are printed. This fits the dual-die rows above (dies/CE = 2) (`diag TestNand.pe+0xf50`).
- `nandsize`: SystemConfig protocol (`ca7feebd`) GetInfo(1) into a `0x134`-byte record, magic `0x53797349`; size in KB = `info[0x40] * (info[0x3f] >> 10)` (`diag TestNand.pe+0xd30`).
- The SoC chip ID (`diag TestM2ChipID`, `m2chipid`) is a separate block: it prints CID_VALID, CHIPIDL_VER (`CHIPIDL & 0xff00003f`), CHIPIDL_CAL (the rest), CHIPIDH, DIEIDL, DIEIDH (die ID words at `+0x0c/+0x10`) and ECID_VERSION; protocol GUID `ddee1ef2-dc49-4799-90e6-4b0c8bd36810` (`diag TestM2ChipID.pe+0xb10..0xb8e`). See [chipid.md](chipid.md). It is not the NAND ID.

## Not established

Package markings from ID bytes, CE-to-die wiring, R/B# fanout, physical pinout,
unsupported chip revisions, ONFI capabilities, endurance, retention, legal
partial-program counts and factory bad-block marker rules. The factory reader
has several vendor marker algorithms, described in the NAND library. The
device-observed Samsung R/W result covers only that population.

## Finding ledger

| Finding | Evidence | Type / limit |
|---|---|---|
| All 18 geometry records and count words | Firmware bytes at flat-PE `0x6874`, `0x6a24`, `0x6d88`, `0x73e4`; file SHA-256 `6e324c575697e3a90a47d6897eac763a2fdc231e83e0e754f0e8a2236c1bdfc5` | observed; supported configurations, not an installed-part census |
| Five timing outputs and three-bit encodings | `disk mem:0x08054418` | observed; electrical margins not measured |
| ReadID and CE population rules | `bootloader NandReadOnly.pe+0x5f28/0x522c/0x601c`; table dump | observed; eight slots probed, no package pinout |
| Non-power-of-two die strides | reference | observed; software address translation, no extra usable blocks |
| NANDChipID records, per-die unique IDs, `nandsize` formula | `diag TestNand.pe+0xb8c`, `+0xf50`, `+0xd30` | observed; NANDChipID provider not located |
| Reported 4-KiB Samsung device | Device-read ID `0x2994d5ec`, four CEs, 4096-byte pages, 218 spare bytes | device; one unit |
