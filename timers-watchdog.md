# Timers and watchdog

Eight general-purpose timer instances plus a separate counter at the `+0x80`
slot. Use the numeric indices and addresses below, not older S5L header names.
Only a small watchdog command interface is recovered.

## Sources

The boot ROM, the restore timer ISR (`disk mem:0x0802f51c`) and rate setter
(`disk mem:0x080722cc`). Addresses `0x080...` are `disk` code.

## Timer addresses

| Timer | Base | Role |
|---|---|---|
| 0 | `0x3c700000` | general; scanned by restore IRQ 8 |
| 1 | `0x3c700020` | general |
| 2 | `0x3c700040` | general |
| 3 | `0x3c700060` | general |
| separate counter | `0x3c700080` | ROM counter setup, different register layout (below) |
| 4 | `0x3c7000a0` | free-running microsecond time source |
| 5 | `0x3c7000c0` | N33 bootloader tick, VIC line 7 |
| 6 | `0x3c7000e0` | general |
| 7 | `0x3c700100` | general |
| shared status | `0x3c700118` | timer 4..7 interrupt status |
| additional status read | `0x3c710000` | bootloader tick ISR reads it and writes the value to `0x3c700118`; alias/aggregate semantics unknown |

Registers of an ordinary timer:

| Offset | Use | Meaning |
|---|---|---|
| `+0x00` | control: `0x40`, `0x140`, `0x440`; interrupt enables `0x1000..0x4000` | clock/control; bits 10:8 select the input (see [Clock source and rates](#clock-source-and-rates)); bit 6 always set in these values (meaning inferred) |
| `+0x04` | commands `0x10`, `3`, OR `2` | stop/reset and start/update; individual bits not decoded. PWM use: `0x10` = reset (wait bit 4 clear), bit 0 = run |
| `+0x08` | period/reload | DATA0; width differs between timer classes, not every instance is known to take a 32-bit reload. In PWM use: high ticks |
| `+0x0c` | second data register | DATA1; use as timer unknown. In PWM use: period ticks |
| `+0x10` | prescaler (`0xb`, `0x17`, rate-calculated) | consistent with divide by `(PRE+1)`; clock source partly inferred |
| `+0x14` | counter read | timer 4 at `0x3c7000b4` feeds firmware microsecond deadlines, including the IRAM sleep code (`diag IramRoutines.pe+0xd24`) |

## Initialization and timing

PreEfi `mem:0x2200b8bc`:

1. Save the clock words and ungate the timers: `PWRCON1 &= 0xe07fffdf`,
   `PWRCON2 &= ~0x60`, `PWRCON4 &= 0xff3fff80`.
2. `CMD=0x10` to timers 0..3.
3. Timer 4: `CON=0x40`, `PRE=0xb`, `DATA0=0xffffffff`, `CMD=3`.
4. `CMD=0x10` to timers 5..7.
5. Clear shared status `+0x118` with `0xffffffff` (stores at
   `mem:0x2200b8c6..0x2200b8f4`).
6. Restore the saved gate words.

`bootloader mem:0x2200cb7c` reads timer 4 for delays.

Restore OS `0x080710d4` uses `CON=0x440`, `PRE=0x17` with the same
microsecond-counter API; this fits a 24-MHz /24 source (inferred).
`disk mem:0x080722cc` uses the literal `3000000` with `CON=0x140` and fits a
requested rate through prescaler/reload values; keep that constant when
reconstructing the path.

`bootloader Timer.pe` `+0x2fc/+0x364` programs timer 5 with `CON=0x1040`,
`PRE=0xb`, the requested period in DATA0 and `CMD=3`, then uses VIC line 7.
Its ISR at `+0x408` writes the value read from `0x3c710000` into
`0x3c700118`. Restore OS registers its dispatcher on line 8. How every timer
routes to both lines is not decoded.

## PWM use of timer 0

The diagnostics `Pwm` module uses timer 0 (`0x3c700000`) as a PWM generator
(`diag Pwm.pe+0x420..0x558`); the buzzer test `TestClicker -b` goes through it
(clock id `0x1e`, `diag TestClicker.pe+0xc84..0xeee`).

1. `+0x04 = 0x10`; wait until bit 4 clears (reset).
2. Enable gate `0x1e` (restore gate table row 30: gate word `+4c` bit 5 and `+6c` bit 0, [clocks](clocks-reset.md)).
3. `+0x00 = 0x50`, with bits 10:8 as the clock divider select: `0` = /2, `0x100` = /4, `0x200` = /16, `0x300` = /64 (requested divisors 2 / 4 / 0x10 / 0x40).
4. Tick rate = source clock (ClockAndReset get-hz, clock id 5) / divisor.
5. `+0x0c` = period ticks = rate / frequency (frequency must be below the tick rate).
6. `+0x08` = high ticks = period * duty% / 100 (duty <= 100).
7. `+0x04 |= 1` starts; the module waits the requested time using the Timer protocol (`4eeecd0c`); `+0x04 &= ~1` stops.

## Clock source and rates

The timers count from the 24 MHz oscillator: the diagnostics clock-output
command labels oscillator / 4 as "6 MHz" (`diag TestMonitor.pe+0xbfc`).
Control bits 10:8 select the timer input; the diagnostics PWM driver maps
codes 0..3 to /2, /4, /16, /64 (`diag Pwm.pe+0x420..0x4b0`), and code 4 is the
undivided oscillator (inferred). The prescaler divides by `PRE + 1`.

| Stage | Timer 4 setting | Rate |
|---|---|---|
| Bootloader PreEfi (`bootloader mem:0x2200b8bc`) | `CON = 0x40` (code 0: 24 MHz / 2), `PRE = 0xb` (/12) | 1 MHz |
| Restore/disk OS (`disk mem:0x080710d4`) | `CON = 0x440` (code 4: 24 MHz), `PRE = 0x17` (/24) | 1 MHz |

Both land on 1 MHz, and the device run measured the microsecond counter at
`0x3c7000b4` at 1 MHz within +0.05 % of host time. Open point: the disk image
computes a timer with `CON = 0x140` as 3 MHz (`disk mem:0x080722cc`), which does
not match code 1 = /4 of 24 MHz (6 MHz), so the select encoding is not fully
settled.

## Interrupt status and separate counter

Timers 0..3: enables in CON bits 12, 13, 14; pending bits 16, 17, 18.
Restore ISR `0x0802f51c` tests enable and pending, calls the callback, then
writes the pending bit as one.

Timers 4..7: pending status in `0x3c700118`: timer 4 bits 24..26, timer 5
bits 16..18, timer 6 bits 8..10, timer 7 bits 0..2. Enables stay in each CON.
Pending bits are cleared by writing ones. The peripheral acknowledge precedes
VIC completion.

ROM `0x200033b8` configures the separate counter in this order:
`0x3c70008c/90/94/98 = 0xffffffff`, then `0x3c700088 = 0x00010050`, then
`|= 0x10000000` (final `0x10010050`). The count is read at `0x3c700084`.
The 64-bit counter/latch/compare interpretation is not proven. ROM gate
number `0x31` is used, see [clock numbering](clocks-reset.md).

## Watchdog and reset

| Address | Write | Evidence / meaning |
|---|---|---|
| `0x3c800000` | `0x000000a5` | ROM reset entry `0x200000e0`, SecCore reset stub `0x220226b0`; interpreted as disabling/controlling the watchdog, key semantics unknown |
| `0x3c800000` | `0x00100000` repeatedly | ROM `0x2000386c` failure/reboot loop; firmware expects a reset. Timeout units unknown |
| `0x3c800000` | `0x00100000`, then `0x3c500050 = 0xa5`, then spin | Production software reset: ClockAndReset slot 3 (`bootloader ClockAndReset.pe+0x5a8`, ARM), used by the diag `SystemReset` (`diag SystemReset.pe+0x220..0x22e`). Only for reset-type argument 0; other types return `0x8000001c` |
| `0x3c800000` | `0` | restore system reset `0x08070c04`, right before software reset |
| `0x3c500050` | `0xa5` | restore system reset and SecCore stub; software reset command |

Power-off is a different path: PMU register `0x31=2`, then `0x0a=1`
(`disk mem:0x08056592`). Other watchdog offsets, timeout formula, reset-cause
register and refresh protocol are not known.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Nonuniform timer placement | PreEfi and `disk mem:0x08072350/0x080723c4` | observed |
| Timer 4 microsecond API at 1 MHz from the 24 MHz oscillator | PreEfi `0x2200b8bc/0x2200cb7c`; restore `0x080710d4/0x0802c920`; `diag TestMonitor.pe+0xbfc`, `diag Pwm.pe+0x420`; device run | observed; device (1 MHz within +0.05 %); select encoding partly inferred |
| IRQ masks and W1C acknowledgement | restore `0x0802f51c` | observed |
| Tick timer 5, IRQ 7 | `bootloader Timer.pe+0x2fc/+0x364/+0x408` | observed; exercised by dynamic analysis of the bootloader image; routing combinations unresolved |
| Watchdog and software reset literals | ROM `0x200000e0/0x2000386c`; restore `0x08070c04` | observed; field meanings incomplete |
| Reset writes `0x3c800000 = 0x00100000` then `0x3c500050 = 0xa5` | `bootloader ClockAndReset.pe+0x5a8`; `diag SystemReset.pe+0x220..0x22e` | observed |
| Timer 0 as PWM: `+00 = 0x50`, divider bits 10:8, `+0c` period, `+08` high time, `+04` bit 0 run | `diag Pwm.pe+0x420..0x558` | observed |
