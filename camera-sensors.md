# Camera sensor variants: 7692 and 7739

OSOS probes two sensor families over **I2C0, seven-bit address `0x3c`** (wire
address `0x78`). The IDs select the firmware paths conventionally named OV7692
and OV7739; which module package is fitted in each nano5g is not established.
The two paths use different register-index widths.

## Identification, transport and selection

| Parameter | 7692 path | 7739 path |
|---|---|---|
| Read framing | Write one-byte index, then read one byte | Write big-endian two-byte index, then read one byte |
| Write framing | `[reg8,value8]` | `[reg_hi,reg_lo,value8]` |
| ID registers/values | `0a=76`, `0b=92` | `300a=77`, `300b=39` |
| Probe function | `mem:0x08127e7c` | `mem:0x081272fc` |
| Sensor clock divisor | 1 | 4 |
| Reset write | `12=80` | `3008=82` |
| Reset delay | 2 ms busy-wait (OSOS); diagnostics 300 ms | 30 OS sleep units (OSOS); diagnostics 300 ms |
| Init table source | `mem:0x08464dc0`, 116 entries | `mem:0x08464ea8`, 124 entries |
| State allocation | 4 bytes | 12 bytes; field04=0, field08=`28000` |

Probe `mem:0x0810a940`:

1. Power on with divisor 1, wait 100 ms, try 7692.
2. On failure: free that object, power off, sleep 20 OS units, power on with divisor 4, try 7739.
3. If both fail, assert.

Transport functions hold resource 5 around each operation. Most sensor writes
ignore I2C errors in OSOS.

## Initialization writes

All 240 writes are listed in order with firmware VA, register and value, taken
from the osos image bytes (compared byte for byte). 7692 uses packed two-byte entries; 7739 uses four-byte records
with a 16-bit register, a value byte and padding.

- 7692: all 116 register/value writes, table at `osos mem:0x08464dc0` (232 bytes).
- 7739: all 124 register/value writes, table at `osos mem:0x08464ea8` (496 bytes).

Duplicate registers occur. 7692 writes register `ff` in a pattern consistent
with bank selection (bank/field semantics unresolved), so a register number
alone does not identify a unique field. Both initialize methods reset twice
before replaying the table. 7692 then waits 10 ms. 7739 additionally writes
`3818=e0` and waits 10 ms. The image-control fields in the tables (gain,
exposure, color) are not decoded.

## Diagnostic driver (`diag IspDriver.pe`)

The module `TestI2CStress` also requires the camera ID `0x7692` on bus 0, address `0x3c` (`diag TestI2CStress.pe+0xc8e`).

Same probe order as OSOS (7692 on registers `0a/0b`, then 7739 on `300a/300b`), I2C address `0x3c`, one-byte index for 7692 and big-endian two-byte index for 7739. The transport is the Smbus protocol (`e49d33ed`): method 0 `(this, 0x3c, 0, 8, 0, ...)` read, method 1 `(this, 0x3c, 2, 9, 0, ...)` write (`+0x1d54`, `+0x2b2c`, `+0x1f14`). The reset is `12=80` (7692) or `3008=82` (7739), then Stall 300 ms (`+0x1ecc`).

- 7692 table (116 pairs, `+0x329c`) equals the OSOS table except entry 10, `0c=10` (OSOS `0c=d0`). The diagnostics then write `0c=d0` after the table and `0e=00` as stream start (OSOS clears bit 3 of `0e`) (`+0x20f8`).
- 7739 table (123 pairs, `+0x2ec4`) is a different register set from the OSOS 124-entry table: 51 pairs only in the diagnostics, 52 only in OSOS. Examples: `3008=42`; colour matrix `5380..5385 = 42 33 0f 0b 42 4d` (OSOS `3b 2e 0d 0a 3b 45`); gamma `5481..5490` differs; `3a0f/3a10/3a11/3a1b/3a1e/3a1f = 35 2c 90 36 2d 10`; `5000`/`5001` are written several times (`0e 01`, `4f 47`, `cf`, `c7`). Tail `3818=e0`; stream start `3008=04` (OSOS `3008=02` plus the `3212/4805` writes) (`+0x20f8..0x21ae`).
- Colour-bar test mode (TestCamisp `--mode 3`; `0` = normal), `+0x1f60`: 7692 sets sensor register `82` bit 3 (and bit 2 through `|0x0c`) after the manual-exposure writes `13=00 00=00 01=40 02=40 03=40 10=00 0f=00`; 7739 writes `5000=01`, `5001=02`, `503d |= 80`, `5380=41 5381=3c 5382=06 5383=17 5384=3a 5385=52 5392=1e`, `3503=03 3508=00 3509=00 350a=00 350b=00`. Normal mode clears `82` bit 3 (7692) or `503d` bit 7 (7739).
- AE toggle (`--ae`, method 7 selector 0): 7692 register `13` bit 1; 7739 register `3503` bits 1:0 (on = set both). AWB toggle (`--awb`, selector 1): 7692 register `13` bits 7 and 0 (on `|0x81`, off `&0x7e`); 7739 register `5001` bit 0. Selectors 2, 3 do nothing; 4 and above are invalid (`+0x19b4`, `+0x1b94`, `+0x1c00`; `TestCamisp.pe+0x27fc..0x29bc`).
- DLI test: the model ID must be `0x7692` or `0x7739`. Clock-register check: A and B read from `3014` and `3011` (7739) or `30` and `31` (7692); either sensor must give (B == 0x83 and A == 4) or (B == 8 and A == 0). The diagnostics 7739 table writes `3011=08`, which fits the second pair; which pair belongs to which sensor is not resolved. Then colour-bar mode, 500 ms, stream stop (method `0x28`), and a Hamming-distance compare of the capture buffer with `colorBar-OV7739-422.bin` / `colorBar-OV7692-422.bin` (`TestCamisp.pe+0x2fd4..0x3140`, `IspDriver.pe+0x1e34`, `+0x28c8`).
- The exposure/metric method reads 7739 `3014`/`3011` and 7692 `30`/`31` (OSOS 7739 reads `568a`) (`+0x1e34`).

Diagnostic 7739 table, `reg=val`, in execution order:

```
3008=42 3011=08 3015=28 3104=03 3017=7f 3018=fc 3602=14 3611=44
3631=22 3622=00 3633=25 370d=04 3620=32 3714=2c 401c=00 401e=11
4702=01 5000=0e 5001=01 3a00=7a 3a18=00 3a19=3f 300f=88 3011=08
4303=ff 4307=ff 430b=ff 4305=00 4309=00 430d=00 5000=4f 5001=47
4300=32 4301=80 501f=01 3017=00 3018=00 300e=04 4801=0f 4601=02
300f=8a 3010=00 3103=40 3618=04 4837=28 300f=8a 4605=00 4608=02
4609=ba 3820=00 3821=2a 381c=00 381d=0e 381e=01 381f=e8 380d=20
380f=f4 3811=04 3834=02 3a0f=35 3a10=2c 3a1b=36 3a1e=2d 3a11=90
3a1f=10 5000=cf 5481=0a 5482=13 5483=23 5484=40 5485=4d 5486=58
5487=64 5488=6e 5489=78 548a=81 548b=92 548c=a1 548d=bb 548e=cf
548f=e3 5490=26 5380=42 5381=33 5382=0f 5383=0b 5384=42 5385=4d
5392=1e 5801=00 5802=12 5803=0a 5804=48 5805=2a 5806=25 5001=c7
5580=02 5583=40 5584=26 5589=10 558a=00 558b=3e 5300=0f 5301=30
5302=0d 5303=02 3504=0e 3505=30 3506=06 3507=40 3c00=80 3c01=32
3c04=12 3c05=60 3c06=00 3c07=20 3c08=00 3c09=c2 300d=21 3c0a=4e
3c0b=1f 3a0b=7d 3a09=96
```

## Size, start and sampled metric

7692 size method `mem:0x08127ea4` selects 320×240 for mode 1, otherwise
640×480. It reads `cc,cd,ce,cf` (two of the reads are unused), then writes:

| Register | Value |
|---|---|
| `cc` | `(old_cc & fc) \| (width>>8)` |
| `cd` | `width & ff` |
| `ce` | `(old_ce & fe) \| (height>>8)` |
| `cf` | `height & ff` |

The 7739 size method in this vtable is a no-op.

Stream start:

- 7692 `mem:0x0812800c`: reads `0e` and clears bit 3, leaving the state set by initialization (`0e=08`).
- 7739 `mem:0x08127418`: writes `3212=00`, `4805=10`, `3212=10`, `3212=a0`, `3008=02`, in that order. The `3212` writes look like group/standby handling; individual bits are not decoded.

Capture stop uses the GPIO/clock sequence in [CAMIF](camera-interface.md).

The 7739 metric method reads `568a` (raw value; not shown to be exposure time).
The 7692 method leaves the caller's `ffffffff` sentinel untouched. Default
capture geometry is 640×480; the worker requests a rate of 30. Source clock and
actual frame rate are not determined.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Probe, reset, transport, size and streaming | sensor and capture drivers, addresses above | observed |
| All 240 init writes | tables at the addresses above, byte comparison | observed |
| Register-field meanings in the init tables | ordered literal inventory | not decoded |
| Diagnostic 7692 table (one entry differs), 7739 table (different register set), 300 ms reset, colour-bar/AE/AWB registers | `diag IspDriver.pe+0x329c`, `+0x2ec4`, `+0x1ecc`, `+0x1f60`, `+0x19b4` | observed |
| Which DLI clock-register pair belongs to which sensor | `diag TestCamisp.pe+0x2fd4..0x3140` | not resolved |
| Pin identities, source clock, analog controls | no firmware proof | not determined |
