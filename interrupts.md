# Interrupt controllers: VIC and GPIO interrupt groups

Two VIC instances route up to 64 numbered sources to the CPU. A separate GPIO
interrupt block groups pin events before they enter the VIC. Four layers must
each be cleared: peripheral pending state, GPIO-group state, the VIC side-clear
words, and VIC end-of-interrupt.

Evidence: the boot ROM, the restore
dispatch routine (`disk mem:0x08044054`).
Register names follow the PL192-style interface used by the firmware.

## VIC registers

`V(n)=0x38e00000 + n*0x1000`, n = 0, 1. Sources 0..31 use VIC0; 32..63 use
VIC1 with local bit `source-32`. 32-bit accesses.

| Offset | Register | Evidence |
|---|---|---|
| `+0x000` | IRQSTATUS | pending-IRQ test; restore `0x08044054` |
| `+0x004` | FIQSTATUS | ROM FIQ dispatch `0x2000159c` |
| `+0x008` | raw pending | interface name; source polarity unknown |
| `+0x00c` | IRQ/FIQ selection | ROM `0x200013d8` writes zero |
| `+0x010` | interrupt enable | set bit enables the source. Restore `vic_enable_line` `0x08044038` / `irq_enable` `0x0804d62c`, bootloader `bootloader InterruptController.pe+0x2a0` |
| `+0x014` | interrupt enable-clear | write bit disables the source (restore `vic_disable_irq` `0x08043fc0`, bootloader `+0x254`); reset writes `0xffffffff` |
| `+0x018/+0x01c` | software interrupt set/clear | defined by the VIC interface; firmware use not established |
| `+0x020/+0x024/+0x028` | protection, software priority mask, daisy priority | interface only; semantics not validated |
| `+0x100 + 4*i` | vector slot, i=0..31 | restore init stores numeric source IDs; ROM stores callback addresses |
| `+0x200 + 4*i` | vector priority | layout from headers; priority policy not established |
| `+0xf00` | selected vector / EOI | read selects the vector; write completes service. ROM init writes 0 (`0x200013d8`) after the reset entry wrote `0xffffffff` (`0x200000bc/0x200000c8`); the ROM IRQ handler writes no EOI. Restore dispatch writes `0x01010101`, bootloader dispatch writes `0xffffffff` |
| `0x38e02008` | VIC0 source-side clear | write `1<<source` during dispatch |
| `0x38e0200c` | VIC1 source-side clear | write `1<<(source-32)` during dispatch |

The side-clear words are outside both VIC windows and are not the standard
`INTENCLEAR` (which masks a source). "Edge clear" describes their firmware use;
integration semantics unknown.

## Startup and dispatch

Restore init `mem:0x080440b0` masks both VICs and writes the vector slots. The
dispatcher `0x08044054`:

1. Check VIC0 IRQSTATUS, otherwise VIC1.
2. Read `VICADDRESS`.
3. Call the registered callback.
4. Clear the side word.
5. Write EOI.

ROM init `0x200013d8` also clears IRQ/FIQ selection and initializes the
GPIO-group handlers. A driver acknowledges its own pending condition before
the final VIC completion.

ROM IRQ handler `0x20001570` calls the function returned in VICADDRESS. Its
FIQ path `0x2000159c` appears to re-test VIC0 in the branch meant for VIC1
(firmware behavior). At handoff the CPU IRQ/FIQ mask still matters even with
sources enabled. Bds's CPU teardown does not mask all VIC sources.

## Source assignments

| VIC source | Block | Evidence |
|---:|---|---|
| 0,1,2,3,31,32,33 | GPIO groups 6,5,4,3,2,1,0 respectively | ROM init/group ISR map; restore uses groups 0..4 |
| 7 | bootloader timer 5 tick | `bootloader Timer.pe+0x2fc/+0x408` |
| 8 | general timer dispatcher | `disk mem:0x0802f51c`, registration `0x08031844` |
| 16,17 | DMAC0, DMAC1 | restore ISR `0x0802efb0/0x0802efb4` |
| 19 | USB OTG | restore `0x08048adc` |
| 21,22 | I2C0, I2C1 | restore registration `0x08031844` |
| 23 | click wheel | OSOS wheel thread `0x080005f0`, ISR `0x08000534` |
| 29 | DMA block `0x38800000` | restore ISR `0x0802ef7c` |
| 50 | USB PHY | restore `0x080726c0`, ISR `0x08038c51` |
| 52 | PMU single-wire interface ([backlight](backlight.md)) | restore `0x080720b0` |
| 57 (`0x39`) | camera interface | diag `IspDriver.pe+0x60c` installs the handler; it sets Scaler `0x39000004 \|= 1` when CAMIF `+44` bit 0 is set, then clears `+44` and `+40` (`+0x5e8`) |

Video, NAND and other driver IRQs are in their subsystem references.
Header constants labelled `TBC` in related chips are not confirmed routing.

## GPIO interrupt controller

Base `G=0x39700000`, seven groups `g=0..6`, 32 bits per group.

| Offset | Operation | Meaning |
|---|---|---|
| `+0x00` | bits 25:24 set by the diag IRAM sleep code before WFI; bits 10:9 (`0x600`) cleared by the `TestHighPower` low-battery stop before WFI | meaning unknown (`diag IramRoutines.pe+0xbb8..0xd0c`, `TestHighPower.pe+0xeb0..0xec0`) |
| `+0x80+4*g` | read/modify; ISR may toggle a bit | polarity / active level. Toggling emulates both-edge handling; truth table not decoded |
| `+0xa0+4*g` | read status, write bit one | pending status and acknowledge (W1C) |
| `+0xc0+4*g` | set/clear enable bits | per-pin interrupt enable |
| `+0xe0+4*g` | event type | header names suggest edge=0 / level=1; per-pin configuration not known |

GPIO interrupt numbering (7*32 slots) differs from GPIO pad numbering
(18*8 pads). The pad-to-group wiring map is not recovered. The GPIO interrupt
registration table is separate from pinmux commands.

PreEfi zeros `+80`/`+c0`/`+e0` and writes all ones to `+a0` for all groups.
Restore `0x08040bc8` uses these nonzero defaults:

| Address | Value |
|---|---|
| `0x397000c0` | `0x00000040` |
| `0x397000c8` | `0x80000000` |
| `0x397000cc` | `0x00010000` |
| `0x397000e8` | `0x80000000` |
| `0x3970008c` | `0x00010000` |

Restore `0x080409f8` clears status at `0x397000a0/a8/ac` with all ones. The
physical signals behind these masks are not identified. ROM group handlers
dispatch a set bit through the RAM table at state offset
`+0x19c + group*0x80 + bit*4` (software table, not MMIO).

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Two VICs, side-clear and EOI sequence | ROM `0x200013d8/0x20001570`, restore `0x08044054` | observed; timing not measured |
| Vector register holds a number (restore) or a callback (ROM) | restore init vs ROM dispatch | observed |
| Seven GPIO groups and per-group offsets | ROM `0x200011f0..0x20001614`; restore `0x08071ae8/0x08071b50` | observed |
| Wheel IRQ 23 | OSOS `0x080005f0` | observed |
| Camera IRQ 57 handler in the diag | `diag IspDriver.pe+0x5e8/+0x60c` | observed |
| `0x39700000` bits 25:24 / 10:9 touched before WFI | `diag IramRoutines.pe+0xbb8..0xd0c`; `TestHighPower.pe+0xeb0` | observed; meaning unknown |
| Pad/source wiring | no complete map | unknown |
