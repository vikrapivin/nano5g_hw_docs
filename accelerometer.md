# Accelerometer: diagnostic driver and register interface

The diagnostics name a `DxeLIS331DLF` driver and expect identity byte `0x12`,
so the sensor is LIS331-family; the retail suffix is unknown. The driver covers
bus selection, axis transformation, range selection, power modes and self-test.
Slave address: this driver initializes its slave parameter to **0x09**; a
separate diagnostic recovery path writes to **0x19**.

Evidence: the `diag DxeLIS331DLF.pe` driver (including its four additional
protocol methods), the `bootloader DxeSmbus.pe` SMBus driver, the
`diag TestAccelerometer.pe` test application.
Addresses such as `+0x0508` below are **module-relative firmware addresses**,
not sensor registers or physical memory.

A byte comparison against the original images reproduces the table and constant
checks. It checks transcription, not hardware behavior.

## Bus, addressing and framing

`diag DxeLIS331DLF.pe+0x444` enumerates SMBus handles and selects the one whose
metadata byte is 0. Its protocol GUIDs match the N33 SMBus driver's bus 0/bus 1
metadata methods (`bootloader DxeSmbus.pe+0x468/+0x47c`):

| Purpose | Diagnostic offset | Bootloader offset | GUID bytes, in file order |
|---|---|---|---|
| SMBus transfer protocol | `0xb48` | `0x674` | `ed339de43d513446b6986f55aa751c1b` |
| Controller metadata | `0xb68` | `0x694` | `938e7e4897ef774ea428163e70715f62` |

Bus 0 therefore resolves to controller `0x3c600000`. The driver copies word
`0xaf0` into runtime slave word `0xae0`. The three consecutive words at
`0xaf0/af4/af8` are `9,0x18,0x19` (read from the `diag DxeLIS331DLF.pe`
module); the captured entry uses the first. No automatic scan is evidenced.

SMBus Execute (`bootloader DxeSmbus.pe+0x534`) shifts `(slave&0x7f)<<1` and adds
1 for read, with no logical-device table: the initial parameter is 7-bit `0x09`
(wire `0x12/0x13`). The test module's recovery begins with one write to 7-bit
`0x19` (wire `0x32/0x33`), then returns to `0x09` (sequence below). Device
responses are not tested.

Register write helper `+0x3f4` prefixes the register byte to up to 8 payload
bytes and performs SMBus operation 9. Read helper `+0x3c4` first writes a
one-byte subaddress with that helper, then operation 8 receives the requested
bytes: two transactions, no auto-increment. The driver reads axis high bytes
individually.

## Register map

Offsets are **8-bit sensor subaddresses** on I2C, not SoC MMIO.

| Register | Access | Meaning |
|---|---|---|
| `0x08` | Recovery writes 0 | Unknown recovery/test configuration. |
| `0x0b` | Recovery writes 0 | Unknown recovery/test configuration. |
| `0x0f` | Diagnostic identity check against `0x12` | WHO_AM_I. |
| `0x20` | Init `0x27`; mode-dependent RMW of upper 3 bits; mode 2 literal `0xc7`; self-test `0x2f` | Power/rate/axis control. Nonzero bits 7:5 means active to the status method. ODR per encoding not recovered. |
| `0x21` | Mode 1 OR `0x80`, then poll that bit clear; mode 2 `0x15`; self-test 0 | Bit 7 is reboot/reinit (inferred from self-clear and the following data-ready wait). Other bits unknown. |
| `0x22` | Mode 2 `0x80`; another control method 2/0; self-test 0 | Interrupt/control selection; pin routing and polarity not fully recovered. |
| `0x23` | Range bits 5:4; self-test 2 then 0; mode 2 = 0 | Range codes 0,1,2 map to firmware limits ±2,±4,±8. Bit 1 enables the diagnostic stimulus. |
| `0x24` | Mode 2 writes 0 | Unknown control. |
| `0x25` | Mode 2 reads and discards | Unknown status/source; read-to-clear not shown. |
| `0x27` | Poll bit 3 before axis reads | All-axis data-ready used by diagnostics. Other bits unknown. |
| `0x29`, `0x2b`, `0x2d` | Separate one-byte reads, signed | X, Y, Z high bytes. Low-byte addresses exist but this driver does not use them. |
| `0x30` | Mode 2 writes `0x2a` | Event/interrupt configuration (inferred); axes/edge mapping unknown. |
| `0x32` | Mode 2 writes `0x10` | Event threshold by family convention; unit not shown by firmware. |
| `0x33` | Mode 2 writes 0 | Event duration by family convention; unit not shown. |

## Power and sampling sequences

Entry calls init `+0x630`, which writes `0x20=0x27`. Power query `+0x612`
returns true if `(reg20>>5)!=0`. Mode setter `+0x958` reads `0x20` and `0x21`,
then modifies `0x20`:

| Mode argument | First write to 0x20 | Remaining operation |
|---:|---|---|
| 0 | `old&0x1f` | Clears upper 3 bits (power-down, per the query method). |
| 1 | `(old&0x1f)\|0x27` | Write `reg21 old\|0x80`; poll bit 7 clear; poll reg27 bit 3 or stop on read error. |
| 2 | `(old&0x1f)\|0xc0` | Then the sequence below. Product-level name unknown. |
| Other | none | Returns EFI-style invalid-parameter status. |

Mode 2's following sequence:

```text
W20=c7, W21=15, W22=80, W23=00, W24=00,
W32=10, W33=00, R25 (discard), W30=2a
```

Method `+0xa80` writes `0x22=2`, drains X/Y/Z high bytes, and (when its
argument is nonzero) waits until reg27 bit 3; zero writes `0x22=0`. Its GPIO test
is below. Several polls have no firmware timeout.

Read-vector method `+0x508` reads signed 8-bit values from `0x29/2b/2d`. Its
matrix at module `0xafc` is:

```text
 1  0  0
 0 -1  0
 0  0  1
```

It asks the range method for scale and multiplies the transformed value by
`0x400` for range 2, `0x800` for range 4, otherwise `0x1000`.
`diag TestAccelerometer.pe+0xb7c` multiplies each output by 1000, shifts
arithmetically right 16, and prints the result in G: **signed Q16.16 g**,
`0x10000` = 1 g. The driver negates Y. Axis orientation relative to the case is
not confirmed on the device.

## Address recovery and interrupt signal test

`diag TestAccelerometer.pe+0xb7c` reads register `0x0f` at slave `0x09` and
requires `0x12`. If the transfer fails or the byte differs, it runs:

| Order | Slave (7-bit) | Operation |
|---:|---|---|
| 1 | `0x19` | Write `0x20=0x27`. |
| 2 | — | EFI Stall `40000` µs. |
| 3 | `0x09` | Write `0x21=0x80`. |
| 4 | — | EFI Stall `40000` µs. |
| 5 | `0x09` | Write `0x08=0`. |
| 6 | — | EFI Stall `40000` µs. |
| 7 | `0x09` | Write `0x0b=0`. |

It then retries the identity read at `0x09`; failure or any byte other than
`0x12` returns an error. Disassembly at `+0xc40/+0xc5c/+0xc78/+0xc92` shows **only
the first recovery write uses `0x19`**. Why the first write goes to `0x19`, and
the roles of `0x08/0x0b`, are unknown.

The interrupt test (`+0xe30..+0xe84`) powers the sensor on, calls control method
`+0x18` with argument 1 (driver method `+0xa80`, `reg22=2`), samples GPIO
protocol selector **`0x000e0005`**, waits 2000 µs, calls the same method with
argument 0 (`reg22=0`), and samples again. It reports PASS if the two levels
differ, then powers down; no fixed polarity is tested. The GPIO getter,
`bootloader Gpio.pe+0x240`, uses the selector's upper 16 bits as port and lower
16 bits as the shift argument, then reads `0x3cf00000 + port*0x20 + 4`: this
selector is **port 14 pin 5**, an input in the PreEfi table. The physical ball
and board trace are not verified. `TestGpio` names this selector `ACCEL_INT1_C`
(`diag TestGpio.pe+0x137c..0x15f0`, cfg 0 input, then read). The PMU also latches an
accelerometer interrupt: reg `04` bit 2 = `ACCEL_INT_P`, read after `0c &= f5`
(`diag TestGpio.pe+0x1c04..0x1cce`). `TestI2CStress` checks the sensor on bus 0,
7-bit `0x09`, reg `0x0f` must read `0x12` (`diag TestI2CStress.pe+0xe24..0xe3c`).

## Range selection and self-test

Range query `+0x5a4` extracts `(reg23>>4)&3`: codes 0/1/2 report limits
±2/±4/±8 on all three axes; code 3 is not decoded. Setter `+0x8bc` finds the
largest requested magnitude, rejects bounds outside ±8, preserves reg23 with
mask `0xcf`, and selects 0 for magnitudes ≤2, `0x10` for ≤4, `0x20` for ≤8.

Self-test `+0x642` powers down via the protocol; writes 21=0,22=0,23=0,20=0x2f;
waits for and discards 4 ready samples; averages 5 signed samples per axis;
writes 23=2; discards 4 more; averages 5; writes 23=0 and powers down. Accepted
raw differences: X:+5..+55, Y:−55..−5, Z:+5..+55 (firmware pass criteria in raw
signed-byte units).

## Unknown

Retail part suffix, mounted slave address, reason for the initial `0x19`
recovery write, sample-rate and low-power encodings, interrupt polarity and
wiring, low-byte resolution, calibration, temperature dependence, power
requirements.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Controller 0 selection | Entry `+0x444`, matching metadata GUIDs and SMBus `+0x468/+0x47c` | observed (bytes read); external wiring not measured |
| Initial 7-bit slave 0x09 | Entry loads `0xaf0=9`; SMBus `+0x534` shifts address directly | observed; differs from recovery 0x19 and the provisional port default |
| Register/literal/mode map | Full module and four additional methods | observed; unknown fields listed above |
| Axis transformation and Q16.16 g scale | Driver `+0x508`, matrix `0xafc`, range method `+0x5a4`; test application `+0xb7c` displays G | observed; case orientation unknown |
| Recovery address sequence | Test application `+0xc40..+0xcae` | observed; first write to `0x19`, rest to `0x09`; reason unknown |
| Interrupt signal check | Test application `+0xe30..+0xe84` | observed; GPIO selector `0x000e0005`; compares change, not polarity |
| Interrupt pad name `ACCEL_INT1_C`; PMU `ACCEL_INT_P` = reg 04 bit 2 | `diag TestGpio.pe+0x137c..0x15f0,+0x1c04..0x1cce` | observed |
| Stress-test identity check, bus 0, `0x09`, reg `0x0f` = `0x12` | `diag TestI2CStress.pe+0xe24..0xe3c` | observed |
| Self-test sequence and bounds | `+0x642` | observed algorithm; no device results |
