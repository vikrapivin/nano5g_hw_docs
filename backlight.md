# Backlight: PMU control and serial engine

OSOS uses both PMU registers and a dedicated serial block at **`0x3de00000`**.
The bootloader/restore path sets brightness directly through a PMU register.
The serial block is the PMU's single-wire interface, not an I2C controller:
the diagnostics module `Swif` drives it (protocol signature `swic` = `0x63697773`,
size 4) and its consumers are the backlight test ("backlight test using SWIF",
"Test pmu swi interface", "D1759 SysControl") and the D1759 PMU diagnostic
(GUID `3622282d-c57f-4a7a-adca-37d13caf3322`: `diag TestBackLight.pe+0x18ec`,
`DxeD1759Diagnostic.pe+0x8e8`; strings `TestBackLight.pe+0x1030/0x11ac/0x1611/0x162b`;
`diag Swif.pe+0x280..0x28e`). The consumer link is observed; that the output reaches the PMU backlight is inferred. The output pad is unknown.
IRQ 52 is this interface ([interrupts](interrupts.md)).

## Serial registers

All 32-bit MMIO. The 15 source-access rows behind this table keep the masks and
unknown constants from the original code.

| Offset | Operation |
|---|---|
| `00` | Init `1703`; format/clock fields unknown |
| `04` | Init 1 |
| `0c` | Read then write the same status value to acknowledge a latched event |
| `10` | Init clears low 2 bits |
| `14` | Synchronous length/control: 8-bit 0, 16-bit 2, then OR 1 to start |
| `18` | Synchronous transmit data |
| `1c` | Asynchronous length/control: 8-bit 0, 16-bit 2, then OR 1 |
| `20` | Asynchronous transmit data |
| `24` | Period parameter from the caller; units unknown |

`mem:0x0825cb18` initializes. `mem:0x082bf360` accepts 8 or 16 bits, else returns 9.
For 8 bits it writes `(data0&7f)<<8`; for 16 bits `data1|((data0&7f)<<8)|80`.
It starts 14 then calls serial-wait with `40` (physical duration unknown).
Asynchronous `mem:0x082bf3a8` uses 1c/20 and **does not mask data0 bit 7** in OSOS;
the diagnostics driver masks data0 bit 7 on both channels (`diag Swif.pe+0x2c0..0x30c`).
Acknowledge `mem:0x0801cdac` writes back 0c.

### Diagnostics driver (`diag Swif.pe`)

| Step | Operation | Evidence |
|---|---|---|
| Init | `+04 = 1`; poll `+04` until it reads 0 (self-clearing); `+00 = 0x1703` | `+0x2a4..0x2b6` |
| Channel 0 | control `+14`, data `+18` | `+0x2c8` |
| Channel 1 | control `+1c`, data `+20` | `+0x2d2` |
| 8-bit (format 0) | control 0; data `(b0&7f)<<8` | `+0x2e0..0x2e8` |
| 16-bit (format 1) | control 2; data `((b0&7f)<<8)\|80\|b1` | `+0x2ec..0x302` |
| Start | control `\|= 1`, spin until control bit 0 reads 0 (no timeout) | `+0x304..0x30e` |

Any other channel or format returns `0x80000003`. Protocol slots: +0 identity
(returns `swic`; a NULL out-pointer returns `0x80000002`), +4 init, +8 transmit,
+0xc stub returning `0x80000003` (`+0x31c..0x328`). `+0x240` installs two
protocols: the identity GUID `487e8e93-ef97-4e77-a428-163e70715f62` (common to
about 20 driver modules) and the function GUID above.

## Brightness and power sequence

`mem:0x082544d0` initializes serial once. On a changed brightness byte it
updates its cache before I/O. If serial PMU mode is not enabled, it locks
resources 17 then 5, reads PMU 0a, sets bit 5, writes and unlocks. It sends two
bytes `[8+((value&ff)>>7),value&7f]` (8-bit level). If the PMU operation fails the cache is
still updated, so repeating the same request skips the retry.

`mem:0x08143d60` maps percentage p using the panel's `(low,mid,high)`: for
p<=40, `low+(mid-low)*p/40`; otherwise `high-(high-mid)*(100-p)/60`. The
arithmetic wraps at 32 bits, uses signed division and does not clamp. It
subtracts enhancement compensation before sending. Calibration triples are in
[lcd-panels.md](lcd-panels.md); the values are neither linear luminance nor
safe maximum drive.

Power `mem:0x08121a58` clears PMU 0a bit 5 before updating 31 bit 0 to the
requested on/off state and clearing 31 bit 1. Only a successful first write
clears the cached serial-mode state. PMU addresses are byte indices at the
PMU's I2C address, not offsets from `0x3de00000`.

The bootloader `bootloader DxeD1759.pe+0xd54` temporarily masks 0a with 8f,
writes PMU 30 brightness and 31 enable, then restores 0a. The boot-logo caller
supplies a0 and applies an external-power-dependent clamp. Device runs kept a
low brightness bound.

### Diagnostics brightness commands (`diag TestBackLight.pe`)

The `swif -l <n>` path (`backlight_swif_set`, `+0x1068`) reads PMU `0a`, writes
`0a | 0x30` (bits 5 and 4; OSOS sets bit 5 only) and sends two bytes
`[8 + ((n & 3ff) >> 7), n & 7f]`: a 10-bit level, where OSOS uses 8 bits.
Errors: "Failed to read/write D1759 SysControl", "Failed to send SWIF data".

The `pmuswif` self-test (`+0xe70`): PMU `0a |= 0x30`; SWIF words `0x8008` (bytes `08 80`),
`0xe309` (`09 e3`), `0xc609` (`09 c6`) = levels 0, `e3`, `c6` as `[8+(L>>7), (L&7f)|80]`;
after each, PMU `30` must read back `00` / `e3` / `c6` (the level's low byte); then it restores the saved `30`
as `[8 + (old>>7), old|80]`, restores `0a` and `31`.

Protocol presets of the `bl` command (level 0..255, `+0xcbc..0xe34`, strings `0x14ed..0x15e0`, range `+0x1589`):

| Option | Level |
|---|---|
| `-f` | `00` |
| `-l` | `64` |
| `-m` | `e3` |
| `-n` | `a0` after a flash `ff`, `00`, `ff` (500 ms steps) |
| `-h` | `ff` |
| `-r` | ramp `ff 00 c6 e3 ff`, 500 ms (`0x7a120` µs) steps |
| `-s <n>` | `n`; the startup script runs `bl -s 100` |

`TestHighPower` sets `ff` through the serial path (slot 7 of GUID `e8a9e232`, mode 1: two bytes
`[8 + (v>>7 & 1), v & 7f]` via protocol `3622282d`) and restores `c6`; mode 0 uses the PMU path
(protocol `28b7e144`) (`diag DxeD1759Diagnostic.pe+0x7f4..0x82c`, `TestHighPower.pe+0xc68..0xc9c`, `+0xdf0..0xe00`).
`SystemPowerManagement` sleep entry sets 0 then `3fd4147f(1)`; wake sets `3fd4147f(0)` then `a0`
(`diag SystemPowerManagement.pe+0x3cc..0x420`).

## Unknown

Serial line/pad routing, modulation frequency, register 24 period units, status
bits, LED topology and current rating, physical brightness curve. Registers
00/04/10 and the transfer packing are kept as observed. The exact byte values of
the Swif consumers in `TestBackLight` are not all read.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Serial programming/acknowledge | `mem:0x0825cb18`, `0x082bf360`, `0x082bf3a8`, `0x0801cdac` | observed |
| PMU sequencing/calibration, including cache/error quirks | `mem:0x082544d0`, `0x08143d60`, `0x08121a58` | observed |
| Serial block is the PMU single-wire interface | `diag Swif.pe+0x280..0x310`; `TestBackLight.pe+0x162b` | observed (consumers, strings); PMU backlight reach inferred |
| Diag init `+04 = 1`, `+00 = 0x1703`, channel 0/1 registers, data0 bit 7 masked on both | `diag Swif.pe+0x2a4..0x310` | observed |
| Diag `swif -l`: `0a \| 0x30`, 10-bit level; `bl` presets | `diag TestBackLight.pe+0x1068`, `+0xcbc..0xe34` | observed |
| Physical brightness and electrical limits | none in these routines | not measured |
