# USB PHY

**MMIO `0x3c400000`**, with wake/status interrupt **50**. It has its own reset,
separate from the [USB controller](usb-controller.md). Pin voltages and analog
tuning are unknown; the `tuning` names below only label firmware writes.

Evidence: `disk mem:0x080726c0` and `mem:0x08039c5c`. Write order, preserved
bits and delays matter.

### Addresses, IRQs, clock gates

| Item | nano 5G value | Evidence |
|---|---|---|
| OTG core base | 0x38400000 (GOTGCTL +0, GINTSTS +0x14, HPRT +0x440, DCFG +0x800, PCGCCTL +0xe00) | literals in `disk mem:0x08048ba4`, `0x08048cd0`, `0x0803d164` |
| OTG IRQ | **19** (VIC0 0x38e00000 bit 19), handler `disk mem:0x08038568` | `0x08048adc`: `irq_handle_init(19)`, `irq_register_enable(.., 0x08038569)`; VIC bases from `0x080440b0` |
| PHY IRQ | **50** (VIC1 0x38e01000 bit 18), handler `0x08038c50` acks PHY+0x34[1:0] | `disk mem:0x080726c0` registers it, `0x0807267e` removes it |
| PHY base | 0x3c400000 (built as `movs #0xf1; lsls #22`) | `disk mem:0x08039c64` |
| OTG clock gate | PWRCON0 0x3c500048 bit 2 = Apple gate id 0x0e | gate table `disk mem:0x0807d634` entry 0x0e; `bootloader Usb.pe+0x234`; `wtf UsbHAL.pe+0x14ac` |
| PHY clock gate | PWRCON1 0x3c50004c bit 3 = Apple gate id 0x31 | table entry 0x31; `bootloader Usb.pe+0x23c`; `wtf UsbHAL.pe+0x14ac` |
| gate order | on: PHY (0x31) then OTG (0x0e), then **5 ms**; off: OTG then PHY; refcounted | `disk mem:0x0803e318` |
| secondary gate bits | none for 0x0e / 0x31 (table rows have one mask each) | `0x0807d634` |
| ChipID (for PHY+4) | 0x3d100000, needs gate 0x23 = 0x4c bit 14 | `disk mem:0x08038c58` |

### PHY

#### Register map (names inferred; offsets observed)

| Off | Apple use (disk) | Name (inf) |
|---|---|---|
| +0x00 | 0 = power up, 0x1f = power down | OPHYPWR |
| +0x04 | 0x10 \| (chipinfo_get(2) != 0) (disk), 1 (WTF) | OPHYCLK (ref clock select) |
| +0x08 | 9 = reset asserted, 0 = released; 1 at power-off (disk), \|=7 (WTF) | ORSTCON |
| +0x1c | 4 (powered/idle), 6 (around core soft reset), 0 (running) | "UNKCON" (u-boot) |
| +0x28 | bit 0 read = B-session (with GOTGCTL b19) | PHY status |
| +0x30 | bit 2 cleared (run mode 1) / set (run mode 2) | inf: VBUS/ID override |
| +0x34 | [1:0] = PHY interrupt enable/ack (IRQ 50; set on suspend) | inf: wake IRQ |
| +0x38 | 0x22e0 at power-up (WTF: 0 / 0xffff toggles) | observed |
| +0x3c | 2 at power-up (WTF: 0 / 0xffff) | observed |
| +0x40 | RMW [10:8]=3, b7=0, [6:4]=4, [3:1]=3, b0=1 -> low 11 bits 0x347 | tuning 0 |
| +0x44 | RMW [11:10]=3, [9:6]=8, b5=1, b4=1, [3:0]=0xf -> low 12 bits 0xe3f (WTF [9:6]=9 -> 0xe7f) | tuning 1 |
| +0x48 | 2 / 4 / 0 written by the brick test: analog test select for the PMU ADC DP/DM channel (2 = DP, 4 = DM, 0 idle) | `diag TestPowerBrick.pe+0xb0c`, `+0xb34`, `+0xb5a` |
| +0x100 | bit 0 = 1 | tuning 2 |

#### Disk power-on sequence — `usb_phy_power_on` 0x080726c0 + `0x08039c5c`

```
PCGCCTL(0x38400e00) = 0;                udelay(800)      0x0803d834
PHY+0x00 = 0                                             0x080726e2
PHY+0x04 = 0x10 | (chipinfo[2] != 0)                     0x080726e6..f2  (chipinfo[2] = CHIPID 0x3d100008 bit0, 0x080409b4)
PHY+0x3c = 2                                             0x080726f6
PHY+0x38 = 0x000022e0                                    0x080726fa (literal 0x0807271c)
udelay(10)
PHY+0x1c = 4                                             0x0803d178(0)
DCTL(0x38400804) = 2  (soft disconnect)                  0x08039c62
PHY+0x08 = 9;                           udelay(10)       0x08039c6a
PHY+0x40: [10:8]=3 [7]=0 [6:4]=4 [3:1]=3 [0]=1           0x08039c80..cb4 (reg_set_field 0x0804eb08)
PHY+0x44: [11:10]=3 [9:6]=8 [5]=1 [4]=1 [3:0]=0xf        0x08039cc4..cf8
PHY+0x100 |= 1;                         udelay(10)       0x08039d08
PHY+0x08 = 0;                           udelay(815)      0x08039d14 (literal 0x32f)
PCGCCTL = 1;                            udelay(10)       0x0803d164
```
Then the core soft reset `0x08048cd0` writes **PHY+0x1c = 6** first (0x08072720), and after the device is
connected `usb_phy_set_run_mode(1)` (0x0807263c) does PCGCCTL=0/800 µs, PHY+0x30 &= ~4, **PHY+0x1c = 0**,
PCGCCTL=1/10 µs.

Power off (`0x0807267e`): PHY+0x1c=4, PHY+0x34&=~3, IRQ 50 off, PCGCCTL=0/800 µs, PHY+0=0x1f, 10 µs,
PHY+8=1, 10 µs, PCGCCTL=1.

**PCGCCTL (observed, meaning unknown):** Apple writes PCGCCTL=0 before and **1 after** every PHY register
sequence, also after the run-mode switch, so the running core has PCGCCTL=1. In stock DWC2 bit 0 is StopPclk;
the S5L87xx may use it differently or ignore it here.

#### WTF variant (`wtf UsbHAL.pe:0xf1c`, `0xf74`)

gates on; PCGCCTL &= ~3; 100 µs; PHY+0 = 0; 10 µs; **PHY+4 = 1**; PHY+0x1c = 4; PHY+8 = 9; **20 µs**;
same +0x40 fields; +0x44 with **[9:6] = 9** (0xe7f); +0x100 |= 1; 10 µs; PHY+8 = 0; **1000 µs**. No PHY+0x38/+0x3c
write at power-up. Core init `0xc04`: DCTL |= 2; 4 ms; PHY init; GRSTCTL = 1, wait; 1 ms; DCTL &= ~2; 4 ms.
Off (`0xed0`): PCGCCTL |= 3, PHY+0 = 0x1f, PHY+8 |= 7, 1 ms, gates off.

#### Diagnostics brick test (`diag TestPowerBrick.pe+0xb00..0xb5a`)

1. Enable the PHY clock: PWRCON1 `0x3c50004c` bit 3 cleared.
2. `+0x48 = 2`; wait 100 ms (`0x186a0` µs); read the PMU ADC DP/DM channel (driver channel 4, hardware code 6, [ADC](adc.md)).
3. `+0x48 = 4`; wait 100 ms; read.
4. `+0x48 = 0`.

`bootloader DxeD1759.pe+0x49c` does the same measurement for charger classification.

#### Diagnostics `UsbHAL`

`diag UsbHAL.pe` differs from `wtf UsbHAL.pe` in 70 bytes (the header size field
at `0x1db0` and the build-path text `0x1de0..0x1e28`); function bodies, constants and
offsets (FIFO sizing `+0xb00`, gates `+0x14ac`, PHY `+0xf1c/0xf74`, off `+0xed0`) are
identical, so the WTF facts above apply unchanged to the diagnostics image
(byte diff of `203_UsbHAL.pe` and `084_UsbHAL.pe`).

## Signals and charger detection

PHY `+0x28 bit0` and core GOTGCTL bit 19 give session/power state. OSOS reads
core HPRT bits 11:10 after a settling delay to classify a supply (this does not
measure current). Line state 3 is used by the wall-supply path; budgets are in
[PMU policy](pmu.md). The bootloader classifies a charger brick by the ratio
DM_raw * 100 / DP_raw measured through `+0x48`: 130..149 % is a 1000 mA brick,
91..111 % with DM_raw < 367 a 500 mA brick (`bootloader DxeD1759.pe+0x49c`). Connector pin numbering and analog detection thresholds
are not recovered.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Power/reset/tuning writes and delays | Disk functions above | observed; electrical meaning unknown |
| PCGCCTL ends at 1 | Core PCGCCTL and PHY0/8 accesses | observed; meaning unknown |
| Clock choice depends on CHIPID-info bit 0 (disk); WTF writes fixed 1 | disk and WTF paths above | observed |
| Charger line state 3 | OSOS `mem:0x081a9170` | observed (fallback path only) |
| PHY `+0x48` = 2 / 4 / 0 selects DP / DM / idle for the ADC | `diag TestPowerBrick.pe+0xb00..0xb5a` | observed; analog meaning inferred |
| Diag `UsbHAL` code equals WTF `UsbHAL` | 70-byte diff, header and build path only | observed |
