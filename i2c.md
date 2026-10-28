# I2C controllers and transfer protocol

Two controllers: I2C0 at `0x3c600000` and I2C1 at `0x3c900000`.

Address forms: for a 7-bit slave address `a`, the wire byte is `(a<<1)|read`.
The SMBus EFI
protocol takes the 7-bit address and its Execute method shifts it (see the
[accelerometer reference](accelerometer.md)).

Sources: the restore (disk) image and the PreEfi code.

## Registers

`B=0x3c600000 + bus*0x300000`, bus = 0 or 1. All accesses are 32-bit words;
DS carries one data byte per transfer step.

| Offset | Name | Behavior |
|---|---|---|
| `+0x00` | CON | init `0`, then `0x181`; bit 7 controls ACK generation. Firmware clears bit 4 and sets bit 7 to advance; bit 8 differs between IRQ and polling setup. Divider fields not fully decoded. |
| `+0x04` | STAT | init `0x80`, `0`, `0x10`; TX setup `0xd0`, start `0xf0`; RX setup `0x90`, start `0xb0`. Bit 5 is active/busy/START. |
| `+0x08` | ADD | own-address register; firmware writes `0x40` (not the next slave's address). |
| `+0x0c` | DS | slave wire address, then byte TX/RX data. Init writes `0x40`. |
| `+0x10` | unnamed ready/interlock | PreEfi waits for the whole word to equal 0 before register writes. Bit meaning unknown. |
| `+0x14` | unnamed | init writes `1`. |
| `+0x18` | unnamed | init writes `0`. |
| `+0x20` | extra status/interrupt control | IRQ init `0x3f00`; byte acknowledge `0x100`; wait paths test bits 8 and 13. W1C/control field split not decoded. |
| `+0x28` | unnamed | disk init writes `0`; PreEfi does not touch it. |

## Clocks and interrupts

Restore gate IDs 48/47 select I2C0/1. Physical masks are
`PWRCON1 bit4 + PWRCON4 bit11` (I2C0) and `PWRCON1 bit6 + PWRCON4 bit12`
(I2C1); both halves are needed, see the [clock table](clocks-reset.md).
Restore registers VIC21/22 as the I2C0/1 interrupts at `mem:0x08031844`.

SCL frequency is not derived: the source clock and divider encoding behind
`CON=0x181` are unknown. SDA/SCL pad mapping, pull-ups, rise time and
clock-stretch bounds are not established.

## Initialization order

`disk mem:0x08071c84`:

```text
ADD(+08) = 0x40
REG14    = 1
REG18    = 0
STAT     = 0x80
CON      = 0
STAT     = 0
DS       = 0x40
CON      = 0x181
STAT     = 0x10
REG28    = 0
```

`bootloader mem:0x2200b400` performs the same nine writes in the same order
with no REG28 write (no store to `+0x28` in `0x2200b400..0x2200b462`). It
first clears PWRCON1 (`0x3c50004c`) bit 4, the I2C0 clock gate.

The PreEfi variant polls `+0x10==0` before each write. Restore IRQ mode
`0x0802f030` additionally writes `+0x20=0x3f00` and rewrites CON to itself
(possibly a hardware handshake).

## Transaction sequence

- Write (restore `0x0802c7aa`): `STAT=0xd0` (TX), enable ACK generation, put
  `address<<1` in DS, `STAT=0xf0` issues START.
- Read (`0x08071bbc`): `STAT=0x90` (RX), put `(address<<1)|1` in DS, start
  with `STAT=0xb0`.
- Each byte then proceeds by status/pending polling or interrupt. `0x0802c6e0`
  checks STAT bit 5 together with `+0x20` bits 8/13. Waits in the restore path
  are bounded by a 100,000-µs software deadline.
- IRQ-step tail `0x0802f058`: `STAT=0xd0`, `CON=(CON&~0x10)|0x80`,
  `+0x20=0x100`, then CON rewritten. The decompilation has data-flow gaps, so
  the full ISR state machine is not recovered.

Slave register reads send a subaddress, then receive bytes. Some Apple wrappers
use a STOP between the subaddress write and the receive (PMU
`0x0807133c/0x0807148e`, codec wrappers); reproduce that rather than a
repeated START. The last received byte is NACKed, then STOP is issued;
software must wait for STOP before gating clocks.

## Clients

| Client | Controller | 7-bit / write wire byte | Evidence |
|---|---:|---|---|
| PMU | 0 | `0x73 / 0xe6` | restore PMU access wrappers |
| Audio codec | 0 | `0x4a / 0x94` | OSOS `0x082c2f66/0x082c2e48` |
| Camera sensors | 0 | `0x3c / 0x78` | OSOS sensor probe, [camera reference](camera-sensors.md) |
| Speaker amplifier | OSOS: 0 or 1, board-dependent; diagnostics: 1 always | see [amplifier reference](speaker-amplifier.md) | command-byte interface, not register/subaddress framing |
| Headphone interface (Mikey) | OSOS: 0 for revision <4, 1 for revision >=4; diagnostics: 1 always | `0x39 / 0x72` | see [headphone reference](headphone-interface.md); no GPIO-only detector as on other iPods |
| FM tuner | 0 | `0x63 / 0xc6` in the recovered OS path | command protocol |
| Accelerometer | 0 | normal `0x09 / 0x12`; first recovery write `0x19 / 0x32` | [Accelerometer](accelerometer.md); diagnostic test verifies WHO_AM_I at `0x09`, also after recovery |

## Diagnostics SMBus client

The diagnostics reach the controllers through the SMBus EFI protocol
(`bootloader DxeSmbus.pe+0x534`): op 9 = write, op 8 = read, the "command"
argument is ignored, the slave is 7-bit and shifted inside. The two controller
bases sit in a pool at `DxeSmbus.pe+0x43c/0x440` (I2C0 `0x3c600000`, I2C1
`0x3c900000`). The bus metadata byte at instance `+0x20` is `index==1`; every
diagnostics client picks its bus by that byte (0 = I2C0, 1 = I2C1;
`bootloader DxeSmbus.pe+0x25c`).

- Register read = op 9 write of one register byte, then a separate op 8 read; STOP between, no repeated start (`diag TestI2C.pe+0x10c8`, `TestI2CStress.pe+0x137c`).
- Register write = one op 9 transfer of `[reg, value]` (`diag TestI2CStress.pe+0x13e0`).
- `TestI2C` has no device table. It is a generic tool: `i2c -r/-w/-s <bus> ...` and `i2cdevice -r <bus> <addr> <reg> <len>` (`diag TestI2C.pe+0xb68,+0xed0`, strings `+0x139b/+0x13cf`, bus lookup `+0x122c`). The sweep (`-s`) probes 7-bit addresses `0x00..0x7f` with op 8, length 0 (address-ACK probe) and prints `(7-bit) 0x%x (8-bit) 0x%x` = a and a<<1 (loop `+0xcf6..+0xd2c`). The help text says bus "0, 1, or 2"; only buses 0 and 1 exist (metadata match).
- The client table is `TestI2CStress` (`--fct` all but camera, `--fatp` all). Result bits: 1 PMU, 2 accelerometer, 4 Mikey, 8 codec, 0x10 amplifier ("Pizeo"), 0x20 FM, 0x40 camera (flags `diag TestI2CStress.pe+0xf46..0x1218`, strings `+0x15b9/+0x15e2`).

| Device | Bus | 7-bit addr | Diagnostics check | Evidence |
|---|---:|---|---|---|
| PMU | 0 | `0x73` | scratch reg `0x60` written 0/1, read back | `diag TestI2CStress.pe+0xdc2/+0xdd4` |
| Accelerometer | 0 | `0x09` | reg `0x0f` must read `0x12` | `diag TestI2CStress.pe+0xe24..0xe3c` |
| Mikey | 1 | `0x39` | reg `0x07` must read `0x51` or `0x52` | `diag TestI2CStress.pe+0xe70..0xe98` |
| Audio codec | 0 | `0x4a` | reg `0x1a` read, written back, read again, must match | `diag TestI2CStress.pe+0xed6..0xf10` |
| Speaker amplifier | 1 | `0x6f` | LM48557 protocol, command byte sent twice | `diag DxeLM48557.pe+0x494` |
| FM radio | 0 | `0x63` | Si47xx init + GET_REV `10` | `diag DxeSi47xx.pe+0x420` |
| Camera sensor | 0 | `0x3c` | ID must be `0x7692` via the ISP protocol | `diag TestI2CStress.pe+0xc8e` |

The diagnostics use bus 1 unconditionally for Mikey (`diag CD3282.pe+0xcbc`) and the amplifier (`DxeLM48557.pe+0x240`), bus 0 for PMU, accelerometer, codec (`DxeCS42L58.pe+0xd10`) and radio (`DxeSi47xx.pe+0x960`). OSOS picks bus 0 for revision <4 (both diagnostics modules assume the revision >=4 board).
A Mikey read failure prints "Insert headphone to retry!" (`diag TestI2CStress.pe+0xe80/+0xe86`, string `+0x1100`); the device may answer only with a plug present (inferred).

The diagnostics' IRAM sleep code bit-bangs I2C0 `0x3c600000` for PMU writes directly: wire bytes `0xe6/0xe7`, CON bit 7 set, STAT `0xc0`/`0x80` as setup, then `0x30|setup` starts, STOP by STAT = `0x10|setup`; delays come from the timer-4 counter `0x3c7000b4` in us (`diag IramRoutines.pe+0x5c0..0x650,+0x74c..0x76e,+0xd24`).

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Two bases and complete init writes | `disk mem:0x08071c84`; PreEfi `0x2200b400` (nine writes, no `+0x28`) | observed |
| START modes, byte/status waits | restore `0x0802c7aa/0x08071bbc/0x0802c6e0` | observed; ISR has decompiler gaps |
| Paired gates and IRQ21/22 | restore gate table and registration | observed |
| SCL rates and pad wiring | not resolved | unknown |
| Diagnostics client table, bus per device | `diag TestI2CStress.pe+0xd98..0xf40` | observed |
| Register read = write + separate read, write = `[reg,value]` | `diag TestI2C.pe+0x10c8`, `TestI2CStress.pe+0x13e0` | observed |
| Timer-service waits (`diag CD3282.pe` args 1/10/40/50/150/200, `TestI2CStress` 0xfa): units not confirmed (Stall-based service is microseconds) | `diag CD3282.pe+0xc34` | not decoded |
