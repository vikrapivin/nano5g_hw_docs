# PMU analog-to-digital converter

The converter is inside the [PMU](pmu.md), reached over I2C0/slave 0x73. Its
registers are PMU bytes `40..42`, plus an early configuration write `44=72`.
These are PMU register numbers, not SoC MMIO addresses.

## Registers and handshake

| Register/field | Operation | Meaning |
|---|---|---|
| `40` | Conversion control write, then repeated reads | Low nibble = hardware channel code |
| `40 bit4` | Poll until clear (up to 4 reads) | Start / busy |
| `40 bit5` | Set with code 3 only | Extra source enable (ACC_ID path) |
| `40 bit6` | Set during init and idle, and in every diag conversion byte (all start from `40`) | Power-mode meaning unknown |
| `41 bits1:0` | Read after conversion | Low 2 bits of 10-bit sample |
| `42 bits7:0` | Read after conversion | High 8 bits of sample; 8-bit mode (flag 4) reads only `42` |
| `44` | PreEfi writes 72 | Unknown configuration bits |
| `b3` (write), `b9`, `ba` (read) | Second capture path for driver channel 0 with flag bit 8 (`0x100`) | No diag caller; calibration variables not identified |
| `bb` | Trim nibbles | Low nibble and high nibble used by the calibrated read below; flag-bit-8 channels 4/5 read `bb`, channel 7 reads `b9 & f` |

Register protocol: slot 2 raw read (`bootloader DxeD1759.pe+0xac8`, table `+0xb9a`),
slot 3 calibrated read (`+0xc98`); see [PMU](pmu.md). Diag reader: `diag TestADC.pe+0x15d0`.

### Driver channels

Driver channel index to control byte (start bit 4 OR-ed in at run time;
`bootloader DxeD1759.pe+0xb9a`, `+0xbb0..0xc0a`). Indexes 1, 7..13 and 19 and above are unsupported.
Hardware code = low nibble of the control byte.

| Index | Control | Code | Name (diag) | Conversion | Evidence |
|---:|---|---:|---|---|---|
| 0 | `40` | 0 | | | |
| 14 | `41` | 1 | Charge current | 8-bit reg `42` only; mA = raw8 * 390 / 1000 (prints "%d.%03d mA") | `diag TestADC.pe+0xe2c` |
| 6 | `42` | 2 | NTC | R = (raw10 * 625 >> 8) * 20 ohm ("The NTC Resistance is %6d R"); read with control `12`; 2.5 V full scale, 50 µA bias inferred from the x20 | `diag TestADC.pe+0x18da..0x18f6` |
| 3 | `63` | 3 + bit 5 | ACC_ID | 80 ms settle (`0x13880` µs; the diag waits 50 ms), bit 5 cleared afterwards; mV = raw10 * 2400 >> 10 | `bootloader DxeD1759.pe+0xbb0`; `diag TestADC.pe+0x15fa..0x1700`, `+0x17c0..0x1810` |
| 2 | `44` | 4 | VBAT | mV = (raw10 * 2000 >> 10) + 2500; battery protocol `ecca55d7` slot 0 uses this index | `bootloader DxeD1759.pe+0x470` |
| 17 | `45` | 5 | IBAT_OUT part | pair below | |
| 4 | `46` | 6 | DP / DM monitor | calibrated read; [USB PHY](usb-phy.md) `+0x48` selects the line: 2 = DP, 4 = DM | `diag TestPowerBrick.pe+0xb00..0xb5a` |
| 5 | `47` | 7 | DP / DM monitor (second channel) | calibrated read; the diag names codes 6 and 7 the DP/DM monitor channels | `diag TestADC.pe+0x1728..0x1a00` |
| 15 | `4d` | 0xd | IBAT_IN part | pair below | |
| 18 | `4e` | 0xe | IBAT_OUT part | pair below | |
| 16 | `4f` | 0xf | IBAT_IN part | pair below | |

Current pairs: mA = `(rawA - rawB) * 19531 / 10000` (about 1.953 mA per LSB, 2000/1024); negative differences print 0.
"Charging IBAT_IN" = code 0xd minus code 0xf; "Discharging IBAT_OUT" = code 5 minus code 0xe
(`diag TestADC.pe+0xc0c..0xd02`, `+0x1928..0x19f6`).

Calibrated read (slot 3, channels 4 and 5 only): `value = lo + (hi - lo) * raw / full`,
`lo = 115 + 5 * (reg bb & f)`, `hi = 4965 + 5 * (reg bb >> 4)`, `full = 0x3ff` (`0xff` in 8-bit mode)
(`bootloader DxeD1759.pe+0xc98..0xd4e`, `+0xac8..0xb92`, `+0x3bc`).

OSOS's battery service selects physical channel 4 (`mem:0x08198390`, battery
case at `mem:0x081983b8`); the control byte for that transaction is 54
(code 4, start bit, bit 6; the VBAT row above).
After completion, `raw = (reg42 << 2) | (reg41 & 3)`, then control 40 is restored.
The bus lock must cover the whole conversion, not each I2C transfer. Other
firmware ADC modes exist, including an 8-bit read path; their channel routing
and packed service arguments are not mapped.

Disk `mem:0x080711ba` shows a four-iteration control poll, result-byte reads
and idle restore. Its decompiled argument expression is unreliable (division
helper return convention), and its label `ch|41` does not explain OSOS's
channel 4 selection.

## Battery conversion and percentage

OSOS `mem:0x081219ec` converts a 10-bit sample to millivolts:

`millivolts = 2500 + floor(raw * 2000 / 1023)`.

The bootloader and the diag use `(raw * 2000 >> 10) + 2500` (see VBAT above).

PreEfi `mem:0x2200c77c` uses a 1024 denominator in its early battery check;
that is a separate routine. The 1023 expression matches all 1,024 OSOS inputs
in the original-instruction comparison.

OSOS's table at `mem:0x085521e2` is 15 little-endian 16-bit thresholds:

`542, 557, 581, 590, 600, 607, 620, 628, 643, 660, 685, 707, 733, 756, 790`.

The number of thresholds satisfied gives 0..15; the ordinary service scales
this to 0..256. The same table is used with and
without external power. It is a voltage lookup, not a coulomb count. OSOS's UI
smooths decreases of its 22-pixel bar; increases can jump on a new high sample.

## Unknown

ADC reference accuracy, impedance/loading, temperature calibration, conversion
time, reg 44 bits, error-state behavior, the channel-0 flag-bit-8 capture
(`b3/b9/ba`), channel code 0 and the OSOS packed service arguments. The software
poll count is not the converter's conversion time.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Control/poll/result/idle | `disk mem:0x080711ba`; OSOS conversion routine | observed; packed disk argument not decoded |
| Physical battery channel and 1023 scaling | original-code software test | dynamic analysis (original routine executed for all 1,024 codes) |
| Different early denominator | PreEfi code | observed (PreEfi only) |
| Control byte layout, 11 driver channels, result registers | `bootloader DxeD1759.pe+0xac8/+0xb9a/+0xbb0..0xc0a`; `diag TestADC.pe+0x15d0` | observed |
| Channel names, current-pair and NTC/ACC_ID formulas | `diag TestADC.pe+0xc0c..0xf14`, `+0x1728..0x1a00` | observed (constants); NTC bias inferred |
| Calibrated read with trim reg `bb` | `bootloader DxeD1759.pe+0xc98..0xd4e` | observed |
| Voltage jump allowed by original display code | original display code | dynamic analysis (original display code executed with controlled inputs); not an analog test |
