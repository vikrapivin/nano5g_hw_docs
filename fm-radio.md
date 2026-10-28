# FM radio command interface

OSOS talks to the radio on **I2C0, 7-bit address 0x63** (wire bytes `0xc6/0xc7`).
The command set is Si47xx-like; the chip revision is unconfirmed. Digital audio
arrives through **I2S1 RX at 0x3cd00000**, separate from codec I2S2. Package
pins, antenna network and RF specifications are not recovered.

Evidence: the OSOS transport, property and rate routines, the ready and lock
helpers, and the transport callers.
Addresses below are OSOS runtime `mem:` addresses, not offsets inside the radio.

Command transport in the diagnostics: wait CTS (bit 7 of a status read), write
op 9, wait CTS again, read op 8, all on address `0x63` (99), bus 0
(`diag DxeSi47xx.pe+0x2c4,+0x734,+0x950`).

## Command-ready and bus ownership

`0x081b8314` repeatedly calls `0x082b4680`, which reads one status byte from
I2C0/0x63 and returns when bit 7 is set or the I2C operation fails. The loop has
no timeout. `0x0818f5e4` waits ready, takes the I2C serialization service, writes
a command, releases it, and, if a response is wanted, waits ready again before a
separately serialized read. Command-buffer bytes must stay valid across those
service calls. `0x0818f578/0x0818f638` acquire/release the higher-level resource
numbered 1a (software state, not an IRQ or I2C address).

## Commands

Bytes are sent in the order shown; hi/lo are big-endian 16-bit values. Read
lengths include status. Command bytes, properties, addresses and masks are hex;
response lengths, sample rates and clock frequencies are decimal.

| Command / transmitted bytes | Response bytes | OSOS function | Meaning |
|---|---|---|---|
| `10` | 16 | `082ae0b4` | Revision/query (GET_REV). Diagnostics print bytes 1..8: part number, firmware major/minor, patch hi/lo, component major/minor, chip revision (`diag TestFm.pe` strings `+0x128c..+0x16b4`); the part-number check accepts <=6 or `0x14..0x2d` (`diag DxeSi47xx.pe+0x420`) |
| `11` | 0 | `082a1e14` | Issued in radio power-down transition; the diagnostics send it only if the powered flag is set (`diag DxeSi47xx.pe+0x5e4`) |
| `12 00 property_hi property_lo value_hi value_lo` | 0 | `0822fb84` | Set property |
| `13 00 property_hi property_lo` | 4 | `08198008` | Get property; bytes 2/3 are the returned 16-bit value |
| `14` | 1 | `08001ae4` | Status/query; bits not decoded |
| `20 00 value_hi value_lo 00` | 0 | `081d87cc` | Tune-like; frequency unit not established. Diagnostics: `20 00 freq_hi freq_lo cap`, the fifth byte is the antenna capacitor argument (`TestFm` "capacitor"), then Stall 1,000,000 us (`diag DxeSi47xx.pe+0x900`) |
| `21 (flags & 0c)` | 0 | `081bf924` | Seek-like; flag bits 2/3 |
| `22 flags` | 8 | `0819ab74` | Tune-status-like response; diagnostics send `22 00` after the tune wait (`diag DxeSi47xx.pe+0x6f6`) |
| `23 flags` | 8 | `081ccf4c` | Signal-quality-like response; the interrupt-acknowledge bit comes through flags (`diag DxeSi47xx.pe+0x6dc`) |
| `24 flags` | 13 | `0819abac` | RDS-like response (`diag DxeSi47xx.pe+0x5e4`) |
| `80 value` | 0 | `081b41c4` | Antenna-switch control (diagnostics) |
| `81 value` | 0 | `082b46a0` | Antenna-switch control (diagnostics) |
| `01 40 b0` | 0 | `082a7e9c` | Short initialization variant, then delay 6e |
| `01 0f 00` | 8 | `082ae0e8` | Initialization/query variant; purpose unknown |
| `01 40 b0 00 e7 e5` | 1 | `082b460c` | Initialization without a command stream. The diagnostics send `01 40 b0 00 00 00` (6 bytes, no response) then Stall 110000 us (`diag DxeSi47xx.pe+0x60c`) |
| `01 60 b0 00 e7 e5` | 1, then 1 per block | `082b460c` | Initialization with a stream, length divisible by 8; the diagnostics use the same form with 8-byte chunks, status bit 6 = error (`diag DxeSi47xx.pe+0x650`) |

The stream path waits with argument `0x6e` after the initial command, then writes
8-byte chunks. Status bit 6 on a response produces software error 0x51. The
`0x6e` wait goes through the millisecond delay routine `0x220023f8` (argument
times 1000 against the microsecond timer): 110 ms. The stream format and
device-side meaning are unknown; it is not a PMU-style register/value list.

Most wrappers require software state at 0866dc38 to be >=2, otherwise they return
0x23 (an OSOS state byte, not the radio status register). The caller at 082a7eb4
is `osos mem:0x082a7e9c`: it sends exactly `01 40 b0` (3 bytes), then calls the
millisecond delay (`0x220023f8`) with `0x6e` (110 ms) on success. It is distinct
from the 6-byte variant.

## Power, reset and initialization

- `mem:0x0819f714` writes **PMU 0c=05**, then **PMU 0e=d0** under the shared
  power/I2C locks (early PreEfi uses PMU 0c=09). Rail meanings unknown.
- `mem:0x081f6668` drives GPIO `0x4b` low, clears bit 0 of `0x3cf0012c`, waits
  `0xfa` = 250 µs through the raw-timer busy wait `0x2200241e` (microsecond
  base, no scaling), drives GPIO `0x4b` high, sets that bit, then waits 1 µs.
  GPIO `0x4b` is port 9 pin 3; its reset/enable role is inferred from use.
  The diagnostics use the same sequence on selector `0x90003` (port 9.3): cfg `0xe`, Stall 250 us, cfg `0xf`, Stall 1 us (`diag DxeSi47xx.pe+0x764`), and then start I2S1 RX. The radio interrupt pad is `0x90000` (`FM_INT_C`, port 9.0, input; `diag TestGpio.pe+0x137c..0x15f0`).
- `mem:0x082a7d94` performs that reset, queries with the command 01 variant,
  chooses an operation table from response byte 1 (special cases 06/2b), calls
  its initialization, sets software state 2 and queries command 10. The
  operation table and stream payload are not fully decoded.

Initialization then writes these properties after its revision checks:

| Condition | Property/value writes, in order |
|---|---|
| Software flag 08550949 is 0 | 1303=4, 1302=1f, 1301=4 |
| Flag nonzero | 1302=0 |
| Both branches | 1403=3, 1404=10 decimal, 1501=8, 1502=ff01 |
| After helper 082b35ec(0), variant callback and event registration | 0001=45 |

The diagnostics set these properties after init (number=value): `0001=1`, `0202=0x1092`,
`0201=0x806c`, `1100=2`, `1302=0`, `1403=3`, `1404=0x14` (20 decimal, versus OSOS 10 decimal),
`1501=8`, `1502=0xff01`, `4001=0`, then `4000=0x3f`, `1107=0` (antenna 0), `1800=0x31`,
`1801=0x1e`, `1804=0x1b`, `1805=0x0e`, `1808=0x14`, `1809=0x3c`
(`diag DxeSi47xx.pe+0x464,+0x4fa..0x5ae,+0x53c`). Of these, `0202 = 0x1000|146` and
`0201 = 32876` (4.8 MHz / 146); `0102` = OSIZE code (8 -> 3, 16 -> 0, 20 -> 1,
24 -> 2); `0104` = sample rate 32000..48000, or 0 (`+0x7e4,+0x8c0`). The OSOS
value is read at `osos mem:0x082a7d94`.

Antenna selection in the diagnostics (`diag DxeSi47xx.pe+0x314,+0x434,+0x44c,+0x7a4`;
`TestFm +0xd68..0xd9c`): antenna 0 ("fmi") = `80 02`, `81 02` + property `1107=0`;
antenna 1 ("lpi") = `80 02`, `81 00` + `1107=1`; `80 00` alone = both off.

Event registration uses GPIO-like selector 48 and callback 082ae33d; IRQ polarity
and status handling are not traced. Property names are not decoded.

## Digital-audio properties and clock sequence

`0x08198044` reads and saves property 0104, then sets it to 0 before changing
clocks. Rate 0 disables the clock service. Accepted nonzero rates are
**32000, 44100, 48000 samples/s**; other values return 0x13.

| Rate | Clock parameters |
|---|---|
| 32000 | 4,800,000 when the caller requests that value, otherwise 8,000,000; 24,000,000 is divided by it for clock-out configuration |
| 44100 | Source 22,579,200; derived clock 2,822,400, source selector 2, divisor code 5; extra service argument `0x71` |
| 48000 | 2,400,000, source selector 0, divisor code 10 |

The routine configures clock-out ID 11 (`0x0b`), I2S frame ratio and bit-clock
divisor. It computes `divider = clock / 32768`, checks it fits 12 bits and that
the reference rate is 31130..34406 inclusive, then writes property 0202 as
`1000 | divider`, property 0201 as the 16-bit reference value, then 0104 as the
sample rate. Failure paths can restore the saved 0104, but the initial
property-read status is ignored, so a failed read can leave the saved value
invalid. Clock-out units and pins are in [clocks](clocks-reset.md).

`0x082a1e14` starts clock-out 11 and I2S1 RX for the active path. The idle path
sends 11, stops I2S1 RX, gates clock-out 11 and releases resources. Tuning,
RDS and regional-band policies are not covered. `TestFm` `sweep/rec/pbk/dqe/getrds`
internals are not decoded beyond the protocol calls.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| I2C bus/address, ready bit 7, serialized write/read | Transport and ready captures | observed |
| Command bytes and response lengths | Caller capture | observed; chip ID unconfirmed |
| Rate choices and property sequence | `mem:0x08198044` with resolved literal pools | observed (arithmetic); clock accuracy not measured |
| Radio uses I2S1 RX | `mem:0x082193b0`, active/idle functions, [I2S](audio-i2s.md) | observed |
| `80`/`81` are antenna-switch controls; antenna 0 `80 02`,`81 02`, antenna 1 `80 02`,`81 00` | `diag DxeSi47xx.pe+0x434,+0x44c,+0x7a4` | observed |
| Diagnostics init: `01 40 b0 00 00 00` or stream form, 1404 = 0x14, property set | `diag DxeSi47xx.pe+0x464,+0x60c` | observed |
| Initialization variants in OSOS | commands above | meaning unknown |
