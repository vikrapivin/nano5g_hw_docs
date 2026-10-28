# Click wheel, buttons and hold interaction

The wheel is an external controller on a SoC serial interface at `0x3c200000`.
Firmware receives packets carrying buttons and touch positions; it does not
measure the capacitive electrodes. Packet semantics are mostly recovered. The
four wire roles are known from the bootloader bit-bang driver; the controller's
register read/write commands are known from the diagnostics driver
([protocol](#register-protocol-and-controllers)); the electrical framing is not.

Addresses `0x080.../0x081...` below are **OSOS runtime**.

## Connections, clock and registers

Pins `0x57..0x5a` are flattened GPIO IDs: port 10.7, port 11.0, port 11.1,
port 11.2. Init selects function 2. Wire roles are in the
[bit-bang table](#bit-bang-wire-protocol-bootloader). Apple restore gate ID 50 is `PWRCON1 bit1`.
The OS wheel interrupt is VIC 23. Hold is a PMU input, separate from the packet
stream: PMU register 5 bit 0 set means switch released, clear means hold engaged.

All accesses are 32-bit words at `W=0x3c200000`.

| Offset | Name | Fields / access |
|---|---|---|
| `+0x00` | Control | OSOS init clears bits 0..19 and 22, sets 20 and 21 (`osos mem:0x081bf9f0`). The diagnostics init writes `0x800000` (reset bit 23, self-clearing, polled to 0), then `0x500000` (bits 22 and 20), then ORs `0x200000`: final value `0x700000` with bit 22 set (`diag Twif.pe+0x4f8..0x540`). TX pulses bit 21 low/high. Meaning of the low field and bit 22 unknown. |
| `+0x04` | Send request | OR bit 0 after writing TX. Other fields unknown. |
| `+0x08` | Timing/config | Init literal `0x0003a980` (240000 decimal). Unit and fields unknown. |
| `+0x0c` | Status | Bit 0 gates a valid receive-data read; bit 3 polled until clear after transmit. Other bits unknown. |
| `+0x10` | Interrupt enable | Bit 0 set at OS init; whole value saved/zeroed/restored around a synchronous exchange. Diagnostics: init leaves `+10=1`; exchange sets `+10=3` (bits 0,1); send-only path sets `+10=2`; saved value restored (`diag Twif.pe+0x29a,+0x378`). |
| `+0x14` | Interrupt flags/ack | Write 1 in exchange, write 7 in ISR/init. Bit 0 signals a receive event. Bits 1/2 unknown. Diagnostics write `+14=2` after TX and `+14=1` after RX, and loop `+14=7` until it reads 0 at init. |
| `+0x18` | RX word | One packet read when status bit 0 indicates data. Whether the read clears status is not established. |
| `+0x1c` | TX word | Firmware writes `(logical_packet>>1)|0x80000000`. |

## Register protocol and controllers

Registers inside the wheel controller are read and written with the same
`0x3c200000` interface (Click Wheel Diagnostic protocol installed by `diag Twif.pe`).

| Operation | Logical word | Notes |
|---|---|---|
| Read register r | `0x8000003a \| r<<8` | reply value `(rx>>16)&0x7fff` (`diag Twif.pe+0x40c`); OSOS `0x8000023a` (TX word `0xc000011d`) is read register 2 |
| Write register r | `0x8000002a \| r<<8 \| (v&0x7fff)<<16` | no reply wait (`diag Twif.pe+0x608`; send-only path `+0x368`) |

TX register word = `(logical>>1)+0x80000000`. Query words `0x8000013a` and
`0x8000003a` read back to back with a 10 ms stall (`Stall 0x2710`) form the
32-bit ID `reg1<<16 | reg0` (`diag Twif.pe+0x448..0x486`).

| Register | Content | Evidence |
|---|---|---|
| 0 | vendor ID, 15 bit (`v & 0x7fff`) | `diag Twif.pe+0x448..0x480`, `TestClickwheel.pe+0x2168..0x21a4` |
| 1 | firmware version, 15 bit (`(v>>16)&0x7fff`) | same |
| 2 | button state (OSOS query) | `osos mem:0x0801cdb8` |

Vendor `0x104` = Cypress, `0x1288` = Synaptics; any other value prints "Invalid
Vendor!" (`diag TestClickWheelUpdate.pe+0xe02..0xe4e`). Both updater modules ship
in the diagnostics image and the vendor is read at run time, so both parts exist
on N33 boards; the part on a given unit is read from register 0.

Diagnostics driver behaviour (`diag Twif.pe`):

1. Run the full init (control `0x800000`, wait 0, `0x500000`, `+08=0x3a980`, `+14=7` until clear, `+10=1`, control `|= 0x200000`) before every register read or write (`+0x4f8`).
2. Exchange (`+0x284`): `+10=3`, `+1c=(cmd>>1)+0x80000000`, `+04=1`, dummy read `+0c`, `+14=2`, spin without timeout until `+14` bit 0 and `+0c` bit 0 are both set (flag set with status clear writes `+14=1` and keeps waiting), result `(RX>>shift)&mask` from `+18`, then `+14=1`.
3. Send-only (`+0x368`): save `+10`, init, `+10=2`, `+1c`, `+04=1`, read `+0c`, `+14=2`, restore `+10`.

Hold test: `diag TestClickwheel.pe+0x2f08..0x2f22` reads PMU register 5 and
requires bit 0 = 1 ("please open the hold switch" when 0); `diag TestPowerMgmt.pe+0x2b4..0x2c8` (`-w`) and `TestSleep.pe+0xc1a..0xc2c` loop until bit 0 is clear (hold engaged); `diag TestGpio` passes the hold test when bit 0 reads 0 for "on" and 1 for "off" (10 ms polls).

### Bit-bang wire protocol (bootloader)

`bootloader TouchWheel.pe+0x294..0x4a0`, interface GUID
`cabe8131-a192-41ba-9034-d9f8e49a323d` at `+0x744` = {+0 read buttons `0x294`,
+4 command `0x5e8`, +8 byte read `0x4c4`}. Pads are driven with cfg `0xf` high,
`0xe` low, 0 input, 2 hardware function.

| Pad | Selector | Role |
|---|---|---|
| 11.0 | `0xb0000` | clock, sampled as input |
| 11.1 | `0xb0001` | host to wheel data |
| 11.2 | `0xb0002` | wheel to host data |
| 10.7 | `0xa0007` | host-driven frame/attention: low during transmit, high during receive |

- Transmit: for each of 32 bits, LSB first, wait the falling edge of 11.0, drive the bit on 11.1, wait the rising edge.
- Receive: 32 bits LSB first, sample 11.2 on each rising edge of 11.0. Reply valid if `(w&0x8000ffff)==0x8000023a`, buttons = `(w>>16)&0x1f`.
- Edge-wait timeout 5000 us (`0x1388`). Pads return to function 2 afterwards. Same TX word `0xc000011d` as the MMIO path.
- Set command word: `(((b2<<16)|(b1<<8))>>1)|0xc0000015` = logical `0x8000002a|(b1<<8)|(b2<<16)` (`+0x5e8`).
- Byte read (`+0x4c4`): MSB first from 11.2, rising-edge sampled, 100,000 us timeout.

### Debug dump and updaters (diagnostics)

- `TestTwif -r/-s/-b`: set command 0x2a, sub 6, value `0x800|mode` (raw `0x100`, signal `0x200`, baseline `0x300`) through the bit-bang slot +4; then 10.7 becomes input with pad-control mode 1, 1000 us stall, wait (100,000 us timeout) for 10.7 low then high, then 99 (`0x63`) bytes are read MSB first from 11.2 and printed on the dock UART. 10.7 returns to cfg 2, pad mode 0 (`diag TestTwif.pe+0xa80..0xc78`, calls `+0xba0`, `+0xc30`).
- Update flow (`diag TestClickWheelUpdate.pe+0xd60..0xf98`): options `-u` update, `-f` force; files `nandfs:\AppleInternal\Diags\ClickWheel\Cypress.bin` / `Synaptics.bin`; "Aha, it's the expected version" compares expected and current firmware version; prompts "Put Hold Switch on/off".
- Cypress updater: ISSP-style bit-bang. SDATA = GPIO `0xa0007` (port 10.7, pin 0x57), SCLK = GPIO `0xb0002` (port 11.2, pin 0x5a); SDATA read via cfg 0 (input) after SCLK pulses; vectors are 22 bit (3+16+3), sent by `+0xf50`/`+0x1018`; 6-byte vector records in .data (e.g. `06 00 00 f7 07 00`); write-register vector `(4, addr<<8|data, 7)`; "ChipID = 0x%x" read as two 8-bit shifts (`diag FirmwareUpdaterCypress.pe+0xc74,+0xdb8,+0xf10,+0xf50,+0x1018,+0x1084,+0x10d0`, data `+0x1944..+0x1b10`). Pins and framing observed; ISSP identification inferred.
- Synaptics updater: 16-bit words. `0xa0007` is low for the word and high after (frame); `0xb0000` toggles as clock (low/high with Stall 15/5/10 us); `0xb0001` driven per bit; `0xb0002` sampled per bit and also used outside words (`diag FirmwareUpdaterSynaptics.pe+0x10b4,+0x1170,+0x1228,+0x12cc`). Same four pins 0x57..0x5a; roles inferred.
- Sleep entry drives pad 11.1 (`0xb0001`) cfg `0xe`, 1 ms, then 10.7 (`0xa0007`) cfg `0xe`; wake calls protocol `8d59d32b` slot 0 (arg 1) (`diag SystemPowerManagement.pe+0x254..0x2c0,+0x360..0x3ba,+0xaa0..0xaf8`).

## Initialization and synchronous command path

OSOS init `mem:0x081bf9f0`:

1. Set GPIO `0x57` function 2, wait 1 ms, then `0x58`, `0x59`, `0x5a` function 2.
2. Clear control low 16, bits 16..18, bit 19 and bit 22 (combined mask `0x4fffff`).
3. Acknowledge flags with `+14=7`; enable receive interrupt with `+10|=1`.
4. Write timing `+08=0x3a980`; set control bits 20 and 21.

Transmit `0x0801ea98` clears control bit 21, waits 1 ms, sets it, writes the
encoded packet to `+1c`, sets send-request bit 0, and waits for status bit 3 to
clear, bounded by 1500 µs (`0x5dc`, software timeout).

Synchronous exchange `0x0801db18` saves interrupt enable, disables it through
`0x081a4a98`, writes flag bit 0 twice, sends the packet, then polls for up to
2000 µs. The loop exits early only when flag bit 0 and status bit 0 are both
set. After the loop (or the timeout) RX is read whenever status bit 0 is set,
without re-testing the flag. If only the flag is set, it acknowledges 1 and
continues. It acknowledges 1 after the read and restores the saved enable word.

Button query `0x0801cdb8` repeats exchanges of logical command `0x8000023a`
(TX register word `0xc000011d`). A valid reply satisfies
`(reply&0x8000ffff)==0x8000023a`. The retry counter of 5 is decremented only
when an exchange returns 0; a non-matching non-zero reply is retried without
bound. Buttons are bits 16..20.

## Receive packets and signals

OSOS ISR `0x08000534`, installed by thread `0x080005f0` on VIC 23, tests status
bit 0, reads RX once, classifies it, acknowledges 7 and posts deferred event 5.
Software state lives at RAM `0x0855232c`; `0x08552370/74` are software fields,
not MMIO registers.

| Packet classification | Meaning | Fields |
|---|---|---|
| `(word&0x800000ff)==0x8000001a` | Button/touch state | Buttons bits 8..12; touch bit 30; position `(word&0x03ffffff)>>16` (10 bits kept by OS). The diagnostics keep `(w&0x7fffff)>>16` (7 bits) and force the touch flag to -1 (`diag Twif.pe+0x49c..0x4f6`). |
| `(word&0x800000ff)==0x8000005a` | Two-position state | Buttons bits 24..28; positions bits 8..15 and 16..23. `0x7f` means no finger; the OS two-position path requires valid positions. Diagnostics: buttons `(w>>24)&0x1f`, position A `(w&0xffff)>>8`, position B `(w&0xffffff)>>16`. |
| `(word&0x8000ffff)==0x8000023a` | Synchronous query reply | Buttons bits 16..20. |
| Other | Unclassified | Raw word kept; not a scroll or button event. |

Button field: bit 0 Select, bit 1 Next/Right, bit 2 Previous/Left, bit 3 Play,
bit 4 Menu. The diagnostics map bits 0..4 to UEFI scan codes 7, 0xa, 9, 6, 5; combos
bits 0+3 = 0xb, 0+2 = 0xc, 0+1 = 0xd (`diag Twif.pe+0x394..0x400`). OSOS debug text prints `M P < > S` for bits 4..0 and its
button-change routine `0x08000b4e` maps them to internal key IDs 3,1,2,4,0.

A test driver used 96 positions per revolution and masked 7 bits. The OS keeps 10 bits,
so positions above 95 are not ruled out. Scroll acceleration, long-hold timing
and treating two positions as one cursor are software policy.

## Hold, resume and acknowledgement

OSOS hold handling `0x0819c3f0/0x081988fc` reads PMU 5 bit 0. When held it
writes pins `0x59`, then `0x57` to configuration 1; on release it restores
`0x57`, then `0x59` to function 2. The controller and gate stay active.
Configuration 1 is GPIO output in the ROM's pin interface; held-line levels
follow the wrapper's data argument.

## Device observations

Two device runs with different interrupt-handler variants fix the receive
behaviour that the OS code alone does not show.

1. **One packet per interrupt, acknowledge, no wait** (first driver). A single
   SELECT press registered only on the next press: the press packet was taken
   at release time and the release packet at the following press. The OS's own
   polled exchange (`osos mem:0x0801db18`) acknowledges the flag once when it
   finds the flag set without the data-ready bit and polls again, so the flag
   can precede its data, and a second packet can wait behind the one just read.
2. **Drain loop** (next variant: after each read keep reading `WHEELRX` while
   `WHEEL0C` bit 0 stays set, acknowledging after each read). The device hung
   at boot in the first query (button changes and the query reply are read
   before interrupts are on). Cause: `WHEEL0C` bit 0 does not drop when
   `WHEELRX` is read; it stays asserted until the flags are acknowledged, and
   further reads return words that are not packets, so the loop never ends.
3. **Final test driver** (device boots and the wheel works): one `WHEELRX` read per
   interrupt, kept only if the word is a state or two-finger packet; when only
   the flag is up the handler waits up to 300 µs for the data-ready bit; one
   acknowledge `WHEELINT = 7` after the read; the tick handler reads once more
   only while no flag is pending and does not acknowledge. The stock OS order
   (one read, then acknowledge) is what the hardware requires; the 300 µs
   bound and the tick fallback are test-driver safeguards, not hardware timing. The
   packet stream of one click (expected: two state packets, buttons `0x01`
   then `0x00`) has not been captured on the device.

## Unknown

Bus clock, edge polarity, framing/parity, receive FIFO depth, flag
bits, touch resolution, noise filtering. Controller registers other than 0..2 and the write sub-commands other than sub 6 are not decoded.
Update files: layout of the `*.bin` files and the source of the "expected firmware version" (`diag TestClickWheelUpdate.pe+0x22b4,+0x215c`) not decoded; erase/program/verify vectors of both updaters and the Synaptics word semantics not decoded. Protocol `99722de2-2389-b64f-81f0-9ef7f81d01c5` (used by `TestClicker -s` as slot0(7,1,1) and `TestSleep`): provider not found.
The `0x3a980` timing word and the unexplained control bits are kept as observed.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Init, TX encoding and exchange waits | OSOS `0x081bf9f0/0x0801ea98/0x0801db18` | observed |
| IRQ 23 and packet formats | OSOS `0x080005f0/0x08000534` | observed; not the full controller protocol |
| Hold polarity and pin restore | OSOS `0x0819c3f0/0x081988fc` | observed |
| Read then acknowledge; reading RX does not clear `WHEEL0C` bit 0 | Stock ISR; device runs above (drain loop hangs, one-read driver works) | device |
| Register read `0x8000003a\|r<<8`, write `0x8000002a\|r<<8\|v<<16`; reg 0 vendor, reg 1 firmware | `diag Twif.pe+0x40c,+0x608,+0x448` | observed |
| Vendor IDs `0x104` Cypress, `0x1288` Synaptics | `diag TestClickWheelUpdate.pe+0xe02..0xe4e` | observed |
| Wire roles 11.0 clock, 11.1 host out, 11.2 host in, 10.7 frame | `bootloader TouchWheel.pe+0x294..0x4a0` | observed |
| Diagnostics init ends with control `0x700000` | `diag Twif.pe+0x4f8` | observed |
| Updater bit-bang pins and framing | `diag FirmwareUpdaterCypress.pe`, `FirmwareUpdaterSynaptics.pe` | observed; roles inferred |
