# Headphone jack and accessory controller

Jack status is read from a separate controller at **7-bit I2C address `0x39`**
(wire write address `0x72`)
and classic. The OSOS worker calls this accessory path "Mikey"; that is a name,
not a part number. The diagnostics module for it is `CD3282`; the chip ID is
register `07` = `51` or `52` (`diag CD3282.pe+0x4d8`, `TestI2CStress.pe+0xe90`).

## Bus operations and registers

`osos mem:0x08190350` locks the shared bus, sends a one-byte register index, then
reads one byte; it writes the output only when both transactions succeed.
`mem:0x08205186` chooses bus 0 for model `001c` revision <4, bus 1 for revision
>=4, from system descriptor word `+0x84`; any other model enters WaitForReset. The
diagnostics hard-code bus 1 (`diag CD3282.pe+0xcbc`); both diagnostics modules
assume the revision >=4 board.

| Register | Access | Meaning / values |
|---|---|---|
| `00` | W | OSOS idle setup writes `01`; the diagnostics idle setup writes `25` (`diag CD3282.pe+0x6a4`, called from `+0x3a4`). Command/mode register: diagnostics modes write `2d`,`af`,`2e` (mode 1), `2d`,`af` (mode 2), `65`,`66` (mode 4), `af` (mode 5); bit 4 is pulsed to sample `0c` bit 1; bit 7 cleared first (`diag CD3282.pe+0x6bc,+0x5bc`) |
| `01` | W | Diagnostics: bits 5:0 threshold, `38` at init, settable (`diag CD3282.pe+0x800,+0x838`) |
| `02` | RMW | Diagnostics: bits 6:3 = 3 (`18`) at init; bit 0 set in mode 1; bit 1 pulsed (set, wait, read `0c` bit 2, clear) (`diag CD3282.pe+0x7e4,+0x614`) |
| `03` | W | Idle setup writes `16` (OSOS and diagnostics); `00` = off in the diagnostics; bits 7:5 settable (`diag CD3282.pe+0x558,+0x80e`); mode-byte fields not decoded |
| `05` | W | Diagnostics: bits 4:0 = 31 (`diag CD3282.pe+0x820`) |
| `07` | R | Chip ID `51` or `52` |
| `08` | RMW | Idle setup ORs `df`, masking accessory events (event mask A; diagnostics idle value `df`, `diag CD3282.pe+0x4f4,+0x512,+0x558`) |
| `09` | RMW | Idle setup writes `(old \| dc) & ~03` (event mask B; diagnostics `or dc`, `and fc`) |
| `0a`, `0b` | RMW | Detect-level fields: `0a` bits 6:4, `0b` bits 4:2 and 7:5, combined into an 11-bit "Detect Level" (`diag CD3282.pe+0x770`); bit names not decoded |
| `0c` | R | Status. Bit 1 tone, bit 2 S0/short, bit 3 jack-inserted flag (polarity unresolved, see below) (`diag CD3282.pe+0x5bc,+0x614,+0x668`) |
| `0d`, `0e` | R | Event status, read twice between steps (read-to-clear inferred). `0d` bit 6 = SHORT; `0e` bits 3/4 = TxAck/open-mic (`diag CD3282.pe+0x5a0,+0x440,+0xbd8`); other bits not decoded |

These are byte register indices, not SoC MMIO offsets. `mem:0x0819f2ec` reads
`0c` and returns `value&8`. The worker calls the status wrapper at `mem:0x080181d8`
(stores nonzero as inserted) and branches on that flag at `mem:0x08018238`; the
zero path restores idle detection on removal. The wrapper ignores the read
function's error result, so a failed read must not be treated as a removal.

Polarity of `0c` bit 3 is unresolved. OSOS returns `value&8` as "inserted"
(`osos mem:0x0819f2ec`). The diagnostics GPIO test labels bit 3 = 0 "Headphone
Status: 0,detect" and bit 3 = 1 "1,no detect", the opposite; the plain `-2`/`hpdet`
print shows only the raw bit (`diag TestGpio.pe+0x1d62..0x1d70`, strings
`+0x1de8`, `+0x1e08`). That detect test (jack controller protocol
`4993c92a-7ade-41e8-a243-e6d403904813`: slot +0 write(reg,val), +4 read(reg,&val),
+0x38 status byte) does reg 3 `&= ~0x1c`, `|= 2`, waits 7000 us, then reads `0c`
bit 3 (`diag TestGpio.pe+0x1098..0x1176,+0x1930..0x1970,+0x1d2a..0x1d70`).

Idle initialization at `mem:0x0819f338` does the 08, 09, 03, 00 operations in
that order; the diagnostics idle sequence is the same for 08, 09, 03.

Interrupt line `MIKEY_INT_L` is read twice by `TestMikey`: AP side = SoC GPIO
`0xa0000` (port 10.0, `MIKEY_INT_C`, input cfg 0, level read), PMU side = PMU
diagnostics protocol pin 8 (configured with `0x210`, bit 0 read)
(`diag CD3282.pe+0x8dc..0x9a8`, `TestMikey.pe` strings `+0x10e4/+0x10fc`; `diag TestGpio.pe+0x137c..0x15f0`).
PMU reg `06` bit 2 = `MIKEY_INT_P`, read after `0e &= 5f` (`diag TestGpio.pe+0x1d78..0x1db8`).

Mikey init waits 20 ms (`Stall 0x4e20`) after enabling "device 0x42" through
DevicePowerManagement; device `0x42` is not in the board's 46-entry table, so the
call returns UNSUPPORTED and the wait is skipped (`diag CD3282.pe+0xa70`,
`DevicePowerManagement.pe+0x3cc..0x35e`).
 Accessory/remote detection, interrupt routing, microphone
negotiation and the remaining mask bits are not decoded.

## Interaction with audio

Headphone state controls output policy; PCM still goes through I2S2 and the
codec. The external amplifier shares this revision-dependent I2C bus.

Source 1 in the OSOS recorder calls accessory setup `mem:0x0819f354`; contacts
and bias voltages (TRRS pin assignment) are not identified.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| I2C0/1 selection and address | `mem:0x08190350`, `0x08205186` | observed |
| Reg 0c bit 3 is the status flag; OSOS reads set as inserted | `mem:0x0819f2ec`, worker branches | observed (caller interpretation) |
| Diagnostics label bit 3 = 0 "detect", bit 3 = 1 "no detect" | `diag TestGpio.pe+0x1d62..0x1d70` | observed; polarity unresolved |
| Chip ID reg 07 = `51`/`52`; diagnostics idle reg 00 = `25` | `diag CD3282.pe+0x4d8,+0x6a4` | observed |
| Diagnostics register use for `01`..`0e` | `diag CD3282.pe+0x4f4..0xc9c` | observed (access order); bit names from `TestMikey` strings |
| Idle literals | `mem:0x0819f338` | observed; some mask meanings unknown |
| Full remote/mic/IRQ protocol | accessory worker not reconstructed | not decoded |
| Bit names of `0a`/`0b` fields and `0d`/`0e` bits other than 6/3/4; whether `0d`/`0e` clear on read; slot +0x38 provider (`fwdetect` status byte); DevicePowerManagement ids `0x3d`, `0x3f` consumers | `diag CD3282.pe`, `TestAudioIpod.pe` | not decoded |
