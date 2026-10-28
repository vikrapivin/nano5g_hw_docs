# Power-management IC and charger

The firmware addresses a D1759-family PMU on **I2C bus 0, 7-bit address
`0x73`** (wire address bytes `0xe6/0xe7`). The family name comes
from the firmware driver name. Registers are **8-bit PMU register numbers**, not SoC
MMIO offsets. Register numbers, masks and byte values are hex; bit positions,
loop counts, times and current budgets are decimal.

The PMU supplies power/status, interrupt events, charging, ADC, RTC and
backlight controls. The latter three have separate references:
[ADC](adc.md), [RTC](rtc.md), [backlight](backlight.md).
The battery percentage is a software estimate; the charger does not use it.

## Bus and interrupt contract

`disk mem:0x0807133c` writes the register index on bus 0/slave 0x73,
then performs a separate read (STOP between, not repeated start). `mem:0x0807148e`
sends register plus bytes. PM_IRQ_L is GPIO port 8 pin 7 (diagnostics); pull-up and
polarity not recovered. Reading the status registers does not acknowledge event
latches; writing `ff ff ff` to event registers 1..3 does.

PMU register protocol (GUID `3d4aa229`, installed by `bootloader DxeD1759.pe`,
table `+0x1658`): slot 0 read(reg, len, buf), slot 1 write(reg, len, buf) (up
to 8 bytes per call), slot 2 raw ADC read (`+0xac8`), slot 3 calibrated ADC
read (`+0xc98`) (`bootloader DxeD1759.pe+0xa98`, `+0x13c8`; client
`diag DxeD1759Diagnostic.pe+0x46c`). `ClockAndReset` is a different protocol
(`3a6e3065`). The IRAM sleep code bit-bangs I2C0 `0x3c600000` directly: wire
bytes `0xe6/0xe7`, CON bit 7 set, STAT `0xc0`/`0x80` setup then `0x30|setup`
start, STOP by STAT = `0x10|setup`; delays from the timer-4 counter `0x3c7000b4`
in µs (`diag IramRoutines.pe+0x5c0..0x650`, `+0xd24`).

## Register map

| Register(s) | Observed use | Meaning/limit |
|---|---|---|
| `00` | Chip ID, read by the register protocol | `pmurw --id` prints "Chip ID: 0x%x" (`diag TestPmuGpioRW.pe+0xdbc..0xe16`); value not in the code. The LDO1 enable reads it as a revision gate (rail section) |
| `01..03` | Save on warm boot; write `ff` to clear | Event latches (CLRPMIRQ in the diag, `diag TestGpio.pe+0x1d78..0x1db8`); some bits not decoded |
| `02 bit7` | Select warm/cold startup path | Warm-event flag in PreEfi |
| `07..09` | Interrupt mask bytes | Initialization variants below |
| `04 bit3 / bit6` | Select based on reg 0a bit 3 | Usable VBUS / raw input-present indication; bit 6 is `USB DETECT` in the diag (`diag TestGpio.pe+0x1162..0x1250`). Reg 4 bit 3 set with line class 3 selects the brick classification below |
| `04 bit5` | Accessory detection path | `ACC DETECT` in the diag; `TestAccsw --auto` prints "Acc detect: 1/0" from it (`diag TestAccsw.pe+0xbac..+0xcb0`). Electrical threshold unknown |
| `04 bit2` | `ACCEL_INT_P` after `0c &= f5` | Accelerometer interrupt, PMU side (`diag TestGpio.pe+0x1162..0x1250`) |
| `05 bit0` | Wheel hold input | 1 = hold released, 0 = held in OSOS. The diag loops reading reg 5 until bit 0 clears (hold engaged): `diag TestPowerMgmt.pe+0x2b4..0x2c8`, `TestSleep.pe+0xc1a..0xc2c`; its hold test passes on 0 for "on" and 1 for "off" (10 ms polls), and `TestClickwheel` requires 1 (`diag TestClickwheel.pe+0x2f08..+0x2f22`) |
| `05 bits2:1` | Charge-status poll | Nonzero busy; zero complete once input and charger are enabled and settling is done |
| `06 bit3` | FireWire-detect input on older boards (diagnostics GPIO test names it `FIREWIRE DETECT` on board id 1); board 0x17 revision > 10 uses bit 0 instead | OSOS `mem:0x0824f080` selects the bit. The bootloader ORs it into the power-source byte as bit 6 ("Firewire"): `bootloader DxeD1759.pe+0x544`, `diag TestCharge.pe+0xcbc..0xcdc`; not a USB-presence bit. Diag board id 5 first does `0d &= 5f` and uses reg 6 bit 0; any other board id prints "Wrong Board ID!" (`diag TestGpio.pe+0x2010..0x2140`) |
| `06 bit2` | `MIKEY_INT_P` after `0e &= 5f` | Jack-controller interrupt, PMU side (`diag TestGpio.pe+0x1c04..0x1cce`) |
| `0a bit3` | Set for USB suspend, clear for enabled input | Input off. Any selector write to reg 33 clears it (`bootloader DxeD1759.pe+0x85c`) |
| `0a bits5:4` | Serial/backlight paths | The diag `swif -l` path writes `0a \| 0x30` (bits 5 and 4); OSOS sets bit 5 only. See [backlight](backlight.md) |
| `0a = 01` | Shutdown after backlight-off | Written as exact value, not OR. Deep sleep ORs 1, hibernate ORs 2 (`diag TestSleep.pe+0xb1c..0xb7e`); the IRAM sleep path writes 1 / 2 / 0 for arg 0 / 1 / other, mode 1 writes 2 and spins (`diag IramRoutines.pe+0x74c..0x76e`) |
| `0b..0f` | 4-bit field per rail id 1..10 | Reg `0b + (id-1)/2`, odd id = low nibble, even id = high nibble; the diag reads bits 0..3 to build state words `0x100020 / 0x20020 / 0x40020 / 0x90 / 0x200 / 0x80000` (`diag DxeD1759Diagnostic.pe+0x5c8`, `+0x6d8`). Bit meanings not decoded. `Test32k --on` writes `0b = (0b & f0) + 7`, `--off` `+ 6` (`diag Test32k.pe+0xbd2..0xc2e`): bit 0 of that nibble switches the 32 kHz output, shared with rail id 1's field. `0c` bit 0 and `0e` bit 4 are cleared by the standby sequence; `0c &= f5`, `0e &= 5f`, `0d &= 5f` precede status reads (above) |
| `10`, `11` | Rail enable bits | See [Rails](#rails) |
| `14`, `18..21` | Rail voltage codes | See [Rails](#rails); `15` is written at startup but is not in the provider's rail tables (role unknown) |
| `1e` | Accessory 3.3 V | `TestAccsw --on` writes `1e = 10` then sets reg `10` bit 7; `--off` clears reg `10` bit 7 (`diag TestAccsw.pe+0xbac..+0xcb0`). `1e` is the LDO6 voltage register |
| `22` | PreEfi writes 0, disk IRQ init writes 5 | Role unknown |
| `30` | Brightness byte | See backlight |
| `31 bit0 / bit1` | On/off RMW; value 2 on shutdown | Backlight control |
| `32` | PMU-GPIO command | Value `(pin<<1)` (pin 0..7) `\| 10` if mode arg is 0 `\| 01` if value arg is 0; the sleep test calls (pin 7, mode 1, value 1) = `0e`; no read-back (slot 1 returns `0x8000001c`). `bootloader DxeD1759.pe+0x1540` (GUID `e22d7299`), `diag TestSleep.pe+0xc88..0xc92`. What pin 7 drives is unknown |
| `33 bits1:0` | Input budget selector | `0` = 100 mA, `1` = 200 mA, `2` = 500 mA, `3` = 1000 mA; see [Input selector](#input-selector-and-power-source-byte) |
| `33 bits7:2` | Enable with `50`, disable clears upper bits | Number `k` in mA units: mA = `((15*4k)+1 >> 3) - 5` (read), `k = (2*mA + 18) / 15` (write); startup `50` gives k = 20 = 145 mA. `TestHighPower` prints it as "Current = %dmA" (`diag DxeD1759Diagnostic.pe+0x48c`, `+0x4f4`, `TestHighPower.pe+0xd34..0xd58`); charge-current limit inferred from the label |
| `34` | `7c` enabled,0 disabled; startup `(old&80)\|54` | Charge setting; not a mA or mV value. No diag module touches reg 34 |
| `35` | Current-like value | Value x 120; slots test it ==0 / !=0 (`diag DxeD1759Diagnostic.pe+0x46c`, `+0x454`, `+0x4d4`); unit unknown |
| `36`, `37` | Bits 7:6 cleared by standby | See [Sleep](#sleep-wake-and-standby) |
| `5a` | Bit 6 cleared by standby | Also the last alarm byte ([RTC](rtc.md)) |
| `40..42`, `44` | ADC interface/configuration | [ADC](adc.md) |
| `50..5a` | RTC and alarm | [RTC](rtc.md) |
| `5b,60,61` | Persistent/scratch use | Per-field lifetime not established. The diag I2C stress test writes 0 and 1 to `60` and reads them back (`diag TestI2CStress.pe+0xdc2`) |
| `7c..7e` | Saved startup event bytes | RAM-like scratch role in PreEfi |
| `7f` | Write board-byte from RAM 2200cc40 | Software-provided board identifier |
| `80, a0 bit0` | Startup latch sequence below | Test/OTP latch (inferred) |

## Early initialization, including unresolved writes

`bootloader mem:0x2200b650`, successful path, after event setup; an error
return stops the sequence. The double write to reg 1a is in the original code.

Before the table it initializes I2C, writes reg 7f from board-byte RAM 2200cc40,
and calls the latch helper: up to five rounds of `80 |= 1; a0 &= ~1; 80 &= ~1`,
testing `(80 | a0) & 1`. It then reads event 02: bit 7 selects mode 11, 01..03 are
saved to 7c..7e, and **`old_event02 & 80`** is written back to event 02
(`bootloader mem:0x2200b68c..0x2200b698`: shift right 7, then left 7). The cold
path calls the saved-event-clear helper instead.

| Order | Register | Written value / operation | Established meaning |
|---|---|---|---|
| 1 | `14` | `13` | Rail 1 (core) = 1200 mV (formula in Rails) |
| 2 | `15` | `0d` | Unknown rail/control parameter |
| 3 | `0b` | `23` | Rail field byte, bits not decoded |
| 4 | `0c` | `09` | Rail field byte, bits not decoded |
| 5 | `1f` | `14` | LDO7 = 3000 mV |
| 6–7 | `1a` | `b2`, twice | LDO2: `b2` = code `12` with `a0` OR-ed, = 3000 mV; written twice |
| 8 | `19` | `14` | LDO1 = 3000 mV (NAND supply) |
| 9 | `21` | `06` | LDO9 = 1800 mV |
| 10 | `1d` | `12` | LDO5 = 3000 mV |
| 11 | `10` | `(old & 7f) \| 60` | Bit 7 (LDO6) cleared, bits 6:5 (LDO5, LDO4) set; bit 5 is also the camera supply (below) |
| 12 | `44` | `72` | Converter/control setting, bit meanings unknown |
| 13 | `40` | `old \| 40` | ADC block setup |
| 14 | `33` | `(old & 03) \| 50` | Preserve budget, enable |
| 15 | `34` | `(old & 80) \| 54` | Startup charge setting |
| 16 | `22` | `00` | Unknown |
| 17 | `07..09` | `50 fe 2b` | Interrupt masks |
| 18 | `01..03` | `ff ff ff` | Clear events |
| 19 | `30` | `64` on cold path only | Brightness literal 100 (device tests used `0x40`) |
| 20 | `31` | `old & ~01` | Backlight off |
| 21 | `0a` | `70` | Startup system-control byte; fields not fully decoded |

Disk `mem:0x080561dc` instead writes reg 22=5, masks `10 fe 2b`, then
`ff ff ff` to events. The two stages differ. Device runs used backlight `0x40`;
the original cold-boot value is `0x64`.

The radio path later writes PMU 0c=05 and 0e=d0, (OSOS
`mem:0x0819f714`; see [FM radio](fm-radio.md)). Do not reapply the startup bytes over active services.

The voltages in the table follow from the Rails formulas applied to the
startup bytes (inferred). Regs `18`, `1b`, `1c`, `1e`, `20` are not written at startup.

## Rails

Rail id = buck number as is, LDO number + 10 (`diag TestADC.pe+0xfec`: `ldo`
adds 10, `buck` as is). All rail data below is from `bootloader DxeD1759.pe`.

Enable bits (`+0x1174`, table `+0x1186`; other rail ids return unsupported):

| Reg | Bit 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| `10` | rail 1 | rail 2 | LDO1 | LDO2 | LDO3 | LDO4 | LDO5 | LDO6 |
| `11` | LDO7 | LDO8 | LDO9 | | | | | |

Voltage codes (`+0xed8`, dispatch table `+0xef0`); mV = base + step * code:

| Rail id | Reg | Base mV | Step mV | Startup byte -> mV |
|---|---|---|---|---|
| 1 (core) | `14` | 725 | 25 | `13` -> 1200; sleep `0d` -> 1050 |
| 2 | `18` | 1200 | 50 | not written |
| 11 (LDO1, NAND) | `19` | 2000 | 50 | `14` -> 3000 |
| 12 (LDO2) | `1a` | 1200 | 100 | `b2` -> 3000 |
| 13, 14, 15 (LDO3..5) | `1b`, `1c`, `1d` | 1200 | 100 | `1d`: `12` -> 3000 |
| 16 (LDO6) | `1e` | 2500 | 50 | not written; accessory `10` -> 3300 |
| 17, 18 (LDO7, 8) | `1f`, `20` | 2000 | 50 | `1f`: `14` -> 3000 |
| 19 (LDO9) | `21` | 1200 | 100 | `06` -> 1800 |

- Reg `1a` is written twice: `code | a0`, then plain `code`.
- Units (mV) are inferred from the test print "set ldo/buck %d voltage:%d mv"; the constants are observed.
- Bypass exists for LDO5 and LDO6 only: reg `1d` / `1e` bit 5 set after enable; normal enable clears it (`& df`) (`+0xe10`, `+0x1268`, `+0x12a4`; `diag TestADC.pe+0x1158..0x1198`, `--bypass` / `--normal` accept 5 or 6).
- LDO1 enable: when reg `00` low nibble is nonzero and bit 7 is clear, it drives GPIO 8.6 (`0x80006`, cfg `f`) instead of setting reg `10` bit 2 (`+0x11d4..0x1222`); the reg 0 meaning (revision gate, polarity) is inferred.
- NAND supply test (`diag TestADC.pe+0x14b8..0x1584`): "nand ldo1: on" enables rail 11, then sets ten pads to cfg 2; "off" sets them to cfg `e` and disables rail 11. Pads are listed in [GPIO](gpio-pinmux.md).
- Camera supply: `diag IspDriver.pe+0x241c` sets / clears reg `10` bit 5 (LDO4 by the enable table, role inferred); "on" also writes `1c = 10` (2800 mV by the formula), re-reads `10` and `1c`, waits 100 ms. The early init sets reg `10` bit 5 already.
- `DevicePowerManagement` device 0x3d sets rail 0x13 (LDO9) to 1800 mV then enables it (`diag DevicePowerManagement.pe+0x364..0x3a2`).

## Input selector and power-source byte

Reg `33` bits 1:0 select the USB input budget: `0` = 100 mA, `1` = 200 mA,
`2` = 500 mA, `3` = 1000 mA (`bootloader DxeD1759.pe+0x85c`,
`Pmu_ConfigureCharger`; `diag TestCharge.pe+0xd32..0xe64`, `--set 0/1/2/5/A`
pass caller masks 2/4/8/0x10/0x20 to slot 0 of GUID `05cf6e3e`). The caller mask
(bit 2 = 100, bit 3 = 200, bit 4 = 500, bit 5 = 1000) is mapped to the
selector; bit 1 (suspend) instead sets reg `0a` bit 3. Reg 33 bits 7:2 are
preserved; any selector write clears `0a` bit 3.

Power-source byte (`bootloader DxeD1759.pe+0xa6c`, `+0x135c`, `+0x544`; names
from `diag TestCharge.pe+0xc52..0xcea`; bit 3 unused):

| Bit | Meaning |
|---|---|
| 0 | Battery (set initially by `Pmu_GetBootPowerState`) |
| 1 | USB host, suspended |
| 2 | USB host |
| 4 | USB 500 mA (reg 4 bit 3 set, line class 3, brick id not 1) |
| 5 | USB 1000 mA (reg 4 bit 3 set, line class 3, brick id 1) |
| 6 | FireWire (reg 6 bit 3) |

Battery mV is shown only when bit 0 is set; below 3000 mV (`0xbb8`) it prints "None".

Brick identification (`bootloader DxeD1759.pe+0x49c`, `+0x135c`; `diag TestPowerBrick.pe+0xb60..0xb74`):
ratio = DM_raw * 100 / DP_raw, both raw 10-bit from the DP/DM ADC channel
(see [USB PHY](usb-phy.md) `+0x48`). 130..149 % -> type 1 (1000 mA, power-source bit 5);
91..111 % with DM_raw < 367 -> type 0 (500 mA); otherwise unknown (`ff`).
Applied only when reg 4 bit 3 is set and the line class is 3.

## Sleep, wake and standby

Diag power states (protocol `736e4495`, slot 1): 0 active, 1 sleep, 2
hibernate, 3 standby. Sleep: low-power entry, wait via slot 0 of `b6d3f663`,
then wake sequence; hibernate: low-power entry, slot 1 of `b6d3f663`, no wake
sequence (`diag SystemPowerManagement.pe+0xa00..0xa80`,
`TestPowerMgmt.pe+0x26c..0x2d0`, `-a/-s/-h/-y`).

1. Enter (`diag SystemPowerManagement.pe+0x84c..0x90a`): GPIO 16.2 cfg `f`, clock gates 0x47 and 0x49 via ClockAndReset, SPI1 CS 4.6 low, wheel pads 11.1 and 10.7 low (pads in [GPIO](gpio-pinmux.md)).
2. `CLKCON0 = 0x40009003` (wait low nibble 3); save reg `14`; write `14 = 0d` (1050 mV by the formula); read back. Matches the PreEfi low-battery `14 = 0d` ([clocks](clocks-reset.md)).
3. Wake: restore the saved reg `14` first, then `CLKCON0 = 0x40009001` (wait low nibble 1).

Standby / power-off (`diag SystemPowerManagement.pe+0x922..0x9d4`, `+0xa32..0xa4e`), in order:
clear bits 7:6 of `36`, clear bits 7:6 of `37`, clear bit 0 of `0c`, clear bit 4 of `0e`,
clear bit 6 of `5a`, write `0a = 01`, spin.

`TestSleep` (`diag TestSleep.pe`): `-d` deep sleep = backlight off, 0.5 s delay (`0x7a120` µs), `0a |= 1`;
`-h` hibernate = same with `0a |= 2`; `-s` returns without PMU writes (`+0xb1c..0xb7e`).
`stbpmu` (`+0xc2e..0xcd0`): after hold is engaged, regs `01, 02, 03 = ff`, then `07 = 90`, `08 = 80`, `09 = 00`,
then 5000 cycles of a 1 kHz toggle of GPIO 1.1 (selector `0x10001`, cfg `f`/`e`, 500 µs per phase), then power state 3.

Low-battery stop in `TestHighPower` (`diag TestHighPower.pe+0xdd0..0xec0`): battery below 3200 mV (`0xc80`)
sets backlight `c6`, `0x39700000 &= ~0x600`, then CP15 wait-for-interrupt (`c7,c0,4`); screen text "CPU Enter Stop Mode / Please Reset iPod".

## Charging state machine

Original OSOS functions (OSOS):
`mem:0x08103436` selects input mode; `mem:0x0801cd24` enables/disables the
charger; `mem:0x0812eff4` reports status.

- Input mode 1 sets reg 0a bit 3 and writes reg 33=0.
- Input mode 4 uses selector 0, mode 6 selector 2; the generic mode 7 uses 3 (selector values: see Input selector).
  Enabled controls are `50`, `52`, `53`. Mode 2 is remapped to 4 in this helper.
- Charge enable preserves reg 33 bits 1:0 and sets upper control to 50, writes
  reg 34=7c, and records an enable timestamp. Disable preserves selector bits,
  clears the upper enable field, and writes reg 34=0.
- The status reader waits until 200ms after enable before trusting completion.
  Upper control zero is disabled; input-off is separate; reg 05 bits 2:1 zero
  yields completed status in the ordinary enabled path.
- Software can disable input and charging by policy. These functions have no percentage cutoff.

OSOS `mem:0x081a9170` waits 100ms and tests usable VBUS plus core HPRT
bits 11:10. The fallback for line state 3 is 500mA; the board's optional
GPIO/ADC signature function returns unsupported. The bootloader charger
driver does use selector 3 (1000 mA) after brick identification.

The reported current is a configured **input budget**, not measured cell current.
Termination voltage, taper threshold, recharge hysteresis, thermal thresholds,
timer behavior and the bits of 7c are not decoded.

Backlight (diag brightness path through the single-wire block) is described in [backlight](backlight.md).

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Early raw writes and stage differences | `bootloader mem:0x2200b650`, disk image startup | observed; rail meanings unknown |
| Mode/budget and completion checks | OSOS charger routines above | observed; transitions by dynamic analysis of the osos image |
| Hold polarity | OSOS `mem:0x0819c3f0`, [wheel reference](clickwheel.md) | observed |
| Battery percentage and charge-full indication are separate | [ADC](adc.md) | observed; physical battery condition unknown |
| Input selector 0..3 = 100 / 200 / 500 / 1000 mA; power-source byte bits | `bootloader DxeD1759.pe+0x85c/+0xa6c/+0x135c/+0x544`; `diag TestCharge.pe+0xc52..0xe64` | observed |
| Rail enable bits, voltage formulas, bypass, LDO1 revision branch | `bootloader DxeD1759.pe+0xed8/+0x1174/+0xe10/+0x11d4` | observed (constants); mV units inferred |
| Startup voltages (1200 / 3000 / 1800 mV) | formulas applied to `bootloader mem:0x2200b650` bytes | inferred |
| Brick identification by DM/DP ratio | `bootloader DxeD1759.pe+0x49c`; `diag TestPowerBrick.pe+0xb60` | observed |
| Standby sequence, sleep core-rail drop | `diag SystemPowerManagement.pe+0x84c..0x9d4` | observed |
| Reg 33 bits 7:2 mA formula | `diag DxeD1759Diagnostic.pe+0x48c/+0x4f4` | observed (formula); charge-current role inferred |
| Reg 32 PMU-GPIO command encoding | `bootloader DxeD1759.pe+0x1540` | observed; pin 7 role unknown |
| Accessory 3.3 V (`1e = 10`, reg 10 bit 7) | `diag TestAccsw.pe+0xbac..+0xcb0` | observed |
| Rail nibbles `0b..0f` | `diag DxeD1759Diagnostic.pe+0x5c8` | observed (layout); meaning unknown |

## Not resolved

- Rail-field bit meanings (`0b..0f`) and whether `Test32k`'s nibble is rail 1's field or a separate control.
- Reg `00` value (not in the code); polarity of the LDO1 revision gate.
- Event-mask polarity of regs `07..09` (`90 / 80 / 00` in the sleep test versus `fe`-style masks at boot).
- Reg `32` pin 7 target; reg `35` x 120 unit; reg `33` bits 7:2 being charge current rests on the label.
- GUIDs `12b34146`, `4993c92a`, `a31f3676`, `67f04d1a`, `3fd4147f`, `b6d3f663`, `8d59d32b` not identified; the providers `ClockAndReset`, `Gpio`, `DxeD1759` proper were read from the bootloader copies.
- `DevicePowerManagement` per-device handlers: the 46-id table at `+0x3cc` has handler words zero in the file (relocated at load); only ids 0x3d and 0x40 have visible code.
- Clock-gate ids 0x47 / 0x49 are in the diag namespace and differ from the restore table above id 56 ([clocks](clocks-reset.md)); their physical bits are not asserted.
