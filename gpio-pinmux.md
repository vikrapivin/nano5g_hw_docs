# GPIO pads and pin multiplexing

18 GPIO ports with eight pins each. No ball map is available: port/pin
identifiers below are logical pad IDs, not package balls. GPIO interrupt group
numbering is a separate space, see [interrupts](interrupts.md).

Evidence: the boot ROM, the PreEfi code. A byte
comparison against the original images reproduces the tables and constants;
it checks transcription only.

## Addressing and commands

`G=0x3cf00000`, port `p=0..17`, pin `b=0..7`, port stride `0x20`. The flattened
pad number is `pin_id=p*8+b`; the command register uses separate port and bit
fields.

The EFI GPIO protocol selector is `port<<16 | pin`. Its getter
(`bootloader Gpio.pe+0x240`) takes the upper 16 bits as the port and the lower
16 bits as the shift argument, then reads `G + port*0x20 + 4`. The code uses
`UXTH`; the decompiler's `&0xff` in the shift expression is the register-shift
operation, not an 8-bit selector. Valid pins are 0-7. The diagnostic
accelerometer test's selector `0x000e0005` reads logical pad 14.5.

| Offset | Access | Meaning |
|---|---|---|
| `+0x20*p+0x00` | 32-bit read/write, nibble at `4*b` | PCON, pin function |
| `+0x20*p+0x04` | 32-bit read/write, bit `b` | PDAT, data level; firmware uses RMW for single pins |
| `+0x20*p+0x0c` | per-bit RMW | unnamed pad-control word; "pull/drive" inferred, direction and units unknown. Mutually exclusive with `+0x10` per pin (protocol mode 1 below) |
| `+0x20*p+0x10` | per-bit RMW | second unnamed pad-control word; function unknown. Protocol mode 2 below |
| `+0x240` | write packed command | `(p&31)<<16 \| (b&7)<<8 \| (cfg&15)`; changes one pin's configuration |
| `+0x380` | `0x00155195` at boot | memory-pad/control literal, fields unknown; the diag `mclk` command ORs 3 into it before programming the MCLK clock (`diag TestMonitor.pe+0xec8..0xed0`) |
| `+0x384` | `0x00000555` at boot | memory-pad/control literal, fields unknown |

EFI GPIO protocol (GUID `17a0a3d7-c0a5-4635-bbd5-072187dfe2ee`, `bootloader Gpio.pe+0x240..0x314`, table `+0x31c`):

| Slot | Operation |
|---|---|
| +0 (`+0x240`) | read PDAT bit (`G + port*0x20 + 4`) |
| +4 (`+0x2bc`) | set config: writes `0x3cf00240 = port<<16 \| pin<<8 \| cfg` |
| +8 (`+0x274`) | read PCON nibble |
| +0xc (`+0x2d4`) | pad-control mode: 0 clears the pin's bit in both `+0x0c` and `+0x10`; 1 sets it in `+0x0c` and clears `+0x10`; 2 sets it in `+0x10` and clears `+0x0c` |

The two pad-control words are therefore mutually exclusive per pin. In the
EFI protocol `cfg` `0xe` is low and `0xf` is high (codec reset on port 14
bit 4, selector `0xe0004`: low, 10 µs, high, 10 µs: `diag DxeCS42L58.pe+0x166c`).

ROM `0x20001c3c` writes the command; `0x20001c64` reads a configuration
nibble; `0x20001b80/0x20001c80` write/read data. `cfg=0` is input, `cfg=1` is
GPIO output (the NOR CS path uses it), `2/3` select alternate peripheral
functions. Firmware wrappers also issue `0xe/0xf` for output-low/high. The
truth table of all 16 configurations and their readback encoding is unknown.
A wheel hold write of `cfg=1` is an output, not high impedance; its effect
depends on the data level and the wrapper.

## PreEfi startup pad literals

PreEfi `mem:0x2200b320` initializes every port from three 18-word tables at
`mem:0x2200cc44`, `0x2200cc8c`, `0x2200ccd4`, read from the original N33
image. They are boot configuration values, not reset defaults.

| Port | PCON (+0) | Unnamed +0x0c | Unnamed +0x10 |
|---:|---|---|---|
| 0 | `0xee02eeee` | `0x00000020` | `0x00000000` |
| 1 | `0xe0ee22ee` | `0x00000000` | `0x00000040` |
| 2 | `0x2eeeeeee` | `0x00000000` | `0x00000000` |
| 3 | `0x33333332` | `0x00000000` | `0x00000000` |
| 4 | `0xee333e33` | `0x00000000` | `0x00000000` |
| 5 | `0xeeeee2ee` | `0x00000000` | `0x00000000` |
| 6 | `0x2222222e` | `0x00000000` | `0x00000000` |
| 7 | `0x22222222` | `0x00000000` | `0x00000000` |
| 8 | `0x0fee33e2` | `0x00000080` | `0x00000000` |
| 9 | `0xfee2e220` | `0x00000000` | `0x00000000` |
| 10 | `0x2ee0e0e0` | `0x00000000` | `0x00000020` |
| 11 | `0xeeee0222` | `0x00000000` | `0x00000000` |
| 12 | `0xeeeee332` | `0x00000000` | `0x00000000` |
| 13 | `0xeeeeeeee` | `0x00000000` | `0x00000000` |
| 14 | `0x220eeeee` | `0x00000000` | `0x00000000` |
| 15 | `0xeeeeeeee` | `0x00000000` | `0x00000000` |
| 16 | `0xee2eefee` | `0x00000000` | `0x00000000` |
| 17 | `0xeeeeeeee` | `0x00000000` | `0x00000000` |

Then the firmware resets all GPIO interrupt groups (polarity/type/enable zero,
pending all ones). To read the board straps it clears PCON3 bits 31:8, waits
1,000 µs, takes `(PDAT3 & 0xff) >> 2` (six bits) and restores
`PCON3=0x33333332`. The value is stored in RAM `0x2200cc40` and PMU register
`0x7f`. This board-ID sample is separate from the ROM's three boot-source pins.

## Recovered connections

| Logical pad(s) | Interface / signal | Meaning and open points |
|---|---|---|
| Port 3 bits 5,6,7 | ROM boot-source straps | `sel=(3.5<<2)\|(3.6<<1)\|3.7`. Values 4..7 NAND; 0/2 NOR variant0; 1/3 NOR variant1 |
| Port 3 bits 2..7 | PreEfi board-ID straps | six-bit board identity; resistor network unknown |
| Pin `0x4b` = port 9.3; port 9 unnamed +0x0c bit 0 | OSOS FM power/reset path `0x081f6668` | drives the pin low, clears `0x3cf0012c` bit 0, delays `0xfa` via a firmware helper, drives high, sets bit 0, delays 1. The second bit is a pad-control bit, not a second output signal. See [FM](fm-radio.md) |
| Selector `0x000e0005`, likely port 14.5 | diagnostic accelerometer interrupt test (`ACCEL_INT_C` / `ACCEL_INT1_C`) | samples the level with sensor control enabled and disabled; PASS requires a change. EFI encoding and physical mapping inferred; polarity unknown. See [accelerometer](accelerometer.md) |
| Port 11 bit 3 (`0x5b`) | `M2_FORCE_DFU` (selector `0xb0003`) | ROM takes DFU if high with the PHY VBUS-present bit; the diag sets cfg 0 and reads it (`diag TestGpio.pe+0x1084..0x1090`) |
| Port 10 bits 4,2 | ROM force-DFU key path | two-stage held/released input sequence with PHY VBUS check; mapping to external switches not proven |
| Port 12 bit 3 | ROM sets output high before DFU | purpose unknown |
| Port 0 bits 0..3 | SPI0 | pin 0 CS GPIO; pins 1..3 alternate function 2; SCK/MOSI/MISO mapping unknown |
| Port 4 bits 6,7; port 5 bits 0,1 | SPI1 | port 4.6 CS GPIO; other three alternate function 2; data/clock mapping unknown |
| Ports 6,7,8 | NAND | ROM mux words below; per-pad signal assignment in [FMSS](nand-fmss.md) |
| Pins `0x31..0x45` | LCD parallel/control group | contiguous group configured by firmware; transaction interface in [LCD controller](lcd-controller.md) |
| Pins `0x57..0x5a` = port 10.7, port 11.0..2 | click wheel | function 2 at init; hold changes `0x57/0x59` to function 1 then back to 2. Bootloader bit-bang driver roles (cfg `f`/`e` = drive high/low, 0 = input, 2 = hardware function; `bootloader TouchWheel.pe+0x294..0x4a0`): 11.0 clock (input), 11.1 host data out, 11.2 host data in, 10.7 host-driven frame/attention (low while transmitting, high while receiving); pads return to function 2 afterwards. The Cypress updater bit-bangs SDATA = 10.7 and SCLK = 11.2; the Synaptics updater uses `0xa0007` as frame, `0xb0000` as clock, `0xb0001` as driven data, `0xb0002` sampled (`diag FirmwareUpdaterCypress.pe+0xc74..0x10d0`, `FirmwareUpdaterSynaptics.pe+0x10b4..0x12cc`; Synaptics roles inferred). The wheel debug mode sets 10.7 to input with pad-control mode 1, then restores cfg 2, mode 0 (`diag TestTwif.pe+0xa80..0xc78`) |
| Port 8 bit 7 (`0x47`, selector `0x80007`) | `PM_IRQ_L` in diagnostics | active-low name from diagnostic text; GPIO-group routing not traced; the diag sets cfg 0 and reads it |
| Port 14 bit 4 (`0x74`, selector `0xe0004`) | audio codec reset | OSOS drives low, waits 1 ms, high, waits 1 ms; the diag waits 10 µs each, then writes codec reg 04 = 0 (`diag DxeCS42L58.pe+0x166c`) |
| Port 14 bit 3 (`0xe0003`) | speaker-amp enable | `DevicePowerManagement` device 0x40: state 6 drives cfg `f`, state 7 cfg `e` (`diag DevicePowerManagement.pe+0x2ec..0x32c`); the amp driver calls state 6 at load (`diag DxeLM48557.pe+0x2f0`) |
| Port 16 bits 2,3 (`0x100002`, `0x100003`) | camera power/control pair | camera driver uses opposite levels for on/off: off = `0x100002` high, `0x100003` low; on = `0x100002` low, `0x100003` high (`diag IspDriver.pe+0x24b8`); external reset/powerdown names are subsystem-specific |
| Port 9 bit 7 (`0x90007`) | LCD panel reset | `lcd on` drives cfg `f` then waits 120 ms (out of reset); `lcd off` waits 120 ms then drives `e` (`diag TestLCDDisplay.pe+0x1d2c..0x1d6c`, literal `+0x1fb0`) |
| Port 1 bit 6 | LCD controller bring-up | the diag Clcd sets function 3 (PCON1 nibble 6, bits 27:24); the PreEfi nibble is 0 (`diag Clcd.pe+0x5de`, `+0x5b4..0x640`) |
| Port 1 bits 2,3; port 15 bits 0..6; port 16 bits 0,1,5 | camera interface pads | function 2, set by the diag sensor-on path (`diag IspDriver.pe+0x894`); port 1 bits 2,3 and port 16 bit 5 are already function 2 in the PreEfi table, ports 15 and 16 bits 0,1 read `e` there |
| Port 9 bit 0 (`0x90000`), port 10 bit 0 (`0xa0000`) | `FM_INT_C`, `MIKEY_INT_C` | read as inputs (cfg 0) by the diag (`diag TestGpio.pe+0x137c..0x15f0`); `0xa0000` is also the AP-side `MIKEY_INT_L` read of the jack-controller test (`diag CD3282.pe+0x8dc..0x9a8`) |
| Port 1 bits 0,1 (`0x10000`, `0x10001`) | clock-test outputs; speaker drive | clock-test: cfg 6 for monitor 0 / 1 (`diag TestMonitor.pe+0xe3a..0xe44`). `TestClicker -s` drives them as a complementary 1 kHz pair (cfg `e` low / `f` high, 1 µs between edges, 500 µs half period, `time*1000` iterations; `-h` = `0x10000` high + `0x10001` low, `-l` opposite) (`diag TestClicker.pe+0xc84..0xeee`); the sleep test toggles `0x10001` 5000 cycles at 1 kHz (`diag TestSleep.pe+0xc9a`) |
| Port 0 bits 4,5 (selectors 4, 5) | dock serial lines | `DiagInit` sets both to function 2 (startup PCON0 has pin 4 = 2, pin 5 = 0); `TestTwif` sends its debug bytes "on the dock UART" (dock role inferred) (`diag DiagInit.pe+0x240..0x26a`) |
| Ports 6.2, 6.3, 7.1..7.7, 8.0 | NAND supply test pads | "nand ldo1: on" sets cfg 2, "off" cfg `e` (`diag TestADC.pe+0x14b8..0x1584`, table `+0x260c`, selector `port<<16\|pin`) |
| Port 16.2, 4.6, 11.1, 10.7 | sleep / wake actions | enter: 16.2 cfg `f`, 4.6 cfg `e` then 1 ms delay and protocol `12b34146` slot 0 with arg 2, 11.1 cfg `e`, 1 ms, 10.7 cfg `e`; wake: 16.2 `e`, 4.6 `f`, wheel protocol `8d59d32b` slot 0 (arg 1) (`diag SystemPowerManagement.pe+0x254..0x2c0`, `+0x360..0x3ba`, `+0xaa0..0xaf8`) |
| Port 8.6 (`0x80006`) | LDO1 enable alternative | driven cfg `f` on a PMU revision whose reg 0 low nibble is nonzero ([PMU](pmu.md)) |
| Ports 0.7, 2.4; 9.1, 9.2; 12.1, 12.2 | I2S0; I2S1; I2S2 pins | selectors I2S0 `0x7`, `0x20004`; I2S1 `0x90001`, `0x90002`; I2S2 `0xc0001`, `0xc0002`; cfg 3 for codec mode 1, cfg 2 for mode 0 (`diag I2s.pe+0x300..0x44a`, table `+0x8a0..+0x8b4`) |

ROM NAND mux `0x20001b3c`:

```text
PCON6 = (old & 0x0000000f) | 0x22222220
PCON7 = 0x22222222
PCON8 = (old & 0xff0000f0) | 0x00333302
```

ROM SPI0 mux: `PCON0=(old&0xffff0000)|0x2221`, then CS data high/low/high.
SPI1: `PCON4=(old&0x00ffffff)|0x21000000`, `PCON5=(old&0xffffff00)|0x22`, then
port 4.6 high/low/high. The pulse width is the instruction sequence; no
minimum is known.

## Diagnostic sleep pad tables

The IRAM sleep code carries three 18-word tables: PCON at `diag IramRoutines.pe+0xe20`,
`+0x0c` words at `+0xe68`, `+0x10` words at `+0xeb0`. PCON differs from the PreEfi
table: port 3 `0x333333e2`, ports 6 and 7 `0xeeeeeeee`, port 8 `0x0eeeeeee`, port 9
`0xfeeeeeee`, port 10 `0xeee0e0e0`, port 11 `0xeeee02e2`, port 12 `0xeeeeeeee`,
port 14 `0xee0eeeee`; ports 0, 1, 2, 4, 5, 13, 15, 16, 17 equal PreEfi. `+0x0c`: port 0 `0x20`,
port 8 `0x80`; `+0x10`: port 1 `0x40` only (PreEfi has port 10 = `0x20`).
The restore loop runs ports 0..18 (19 passes, `cmp r0,#0x12; bls`), so pass 19
writes table word 18 (data after the table) to the port-18 slot, the command
register `0x3cf00240` and `0x3cf0024c/50`; the save loop covers ports 0..16 (17 passes).
Intent of the extra pass is unknown (`diag IramRoutines.pe+0xa98..0xaf6`, restore `0xa9a..0xaf8`, save `0xd64..0xd8e`).

## State save/restore

Restore routines `0x08070d28/0x08070d50` loop over 18 ports to save and
restore pin configuration. A peripheral's sleep path may change only its own
pins and leave controller clocks on (the wheel does this). Restoring a whole
PCON word can clobber another owner's live state, so follow the reference
routine's ownership and RMW behavior.

Unknown: package ball mapping, external nets for most pads, drive current and
pull values, voltage domains, input tolerance, all mux alternatives,
GPIO-to-interrupt-group mapping, reset defaults.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| 18 ports, stride, single-pin command | restore `0x08070d28/0x08071ab4`; ROM `0x20001c3c` | observed |
| Startup pad tables | PreEfi table addresses and original image bytes | observed (read from bytes, all 54 words); +0xc/+0x10 meaning unknown |
| Board and boot strap sampling | PreEfi `0x2200b320`; ROM `0x20003298/0x200035e4` | observed; resistor wiring unknown |
| GPIO function 1 is output in the ROM CS path | ROM `0x20001ba0` and data toggles | observed |
| Peripheral pin groups | cited subsystem routines | observed logical pads; not a netlist |
| GPIO protocol slots; `+0x0c` / `+0x10` mutually exclusive per pin | `bootloader Gpio.pe+0x240..0x314` | observed |
| Diag-seen pads (LCD reset 9.7, camera pads, amp enable 14.3, NAND supply pads, sleep pads, wheel bit-bang roles, signal names) | `diag` citations in Recovered connections | observed; names from diag strings |
| IRAM sleep pad tables differ from PreEfi; 19-pass restore loop | `diag IramRoutines.pe+0xa98..0xaf6`, data `+0xe20` | observed; intent of the 19th pass unknown |
