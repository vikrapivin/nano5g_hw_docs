# LCD command/pixel interface

The command-mode LCD bus controller is at **`0x38300000`**. It sends commands
and pixel data to a panel with internal GRAM; it is not a CPU-addressable
framebuffer. OSOS feeds it either from the [CLCD compositor](display-compositor.md)
or by programmed I/O / PL080 DMA. Panel commands are in [lcd-panels.md](lcd-panels.md).

## Register map

All accesses are 32-bit MMIO, even for a single command/data byte. The
table covers all 21 reconstructed source accesses with their exact expressions.

| Offset | Firmware access / meaning |
|---|---|
| `00` | bus mode: pixel `01100db8`; command `01000c20`, OSOS preserving old bits 31:30 and 2:0 |
| `04` | command output; 8- and 16-bit command paths |
| `08` | 16-bit data/pixel path, used in OSOS and restore firmware |
| `10` | write 0 to request one bus read cycle |
| `14` | read result; delivered byte is register>>1 |
| `18` | diagnostics write `000101bf` (not written by OSOS; meaning unknown) (`diag Clcd.pe+0x434`) |
| `1c` | bit 0 read ready; bit 1 interface idle; bit 4 transmit busy |
| `20` | initialization `11`; phase/timing meaning incomplete |
| `40` | command-data bytes and streamed RGB565 pixel data; DMA destination |
| `68` | diagnostics write `(mode == 1)` from the layer-format routine (`diag Clcd.pe+0xaec`); meaning unknown |
| `70` | CLCD interface setup 1 |
| `74` | CLCD transfer width<<16 \| height |
| `78` | CLCD setup `000a000a`; timing fields unknown |
| `7c` | initialization `201`; meaning incomplete |
| `80` | GRAM-window setup bracket: 1 then 0 |
| `88` | initialization `01000000`; meaning incomplete |
| `8c` | low 2 bits busy during CLCD-to-panel transfer; wait until both are 0 |

Bits not listed are not decoded. Bootloader, restore OS and OSOS have separate
initialization paths with different values.

The diagnostics Clcd module writes, in this order: `18 = 000101bf`, `78 = 000a000a`,
`74 = W<<16 | H` (W, H from DisplayPlatform, default table `0xf0`, `0x140`), `70 = 1`, `7c = 201`
(`diag Clcd.pe+0x434`, `+0xaec`). It does not write `00`, `20` or `88`.

## Clocks and initialization

- Bootloader `DisplayPlatform.pe+0x3f0` writes CLKCON2 `0x3c500008=80001014` and polls readback. Restore/OSOS preserve the high half and write low `1014`. Field meaning incomplete.
- Clock-service ID 15 clears bit 1 at `0x3c500048` and bit 16 at `0x3c50006c` (clear = clock enabled).
- OSOS initialization `mem:0x0801c968`, after clock setup: LCD `00=01100db8`, `88=01000000`, `20=11`, `7c=201`.

Mode and command rules:

- Before changing LCD `00`, wait for `1c` bit 1 set, then delay 1 µs.
- OSOS enter-command `mem:0x081a63cc` uses `((old&c0000007)+01000000)|00000c20`; save the old mode and restore it after the commands.
- Before each command or data write, wait for `1c` bit 4 clear (OSOS `mem:0x081a6d5c`; bootloader).
- The bootloader read trigger (`Lcd.pe+0x2f8`) also waits. The OSOS and restore read-byte routines (`mem:0x080220e8`, `disk mem:0x0803d718`) instead delay 100 µs before writing `+0x10` and do not test bit 4.
- Electrical timing is not recovered.

Read byte `mem:0x080220e8`: delay 100 µs, `10=0`, poll `1c` bit 0, return `14>>1`.
The caller timeout is in milliseconds against the microsecond timer.
Multi-byte read `mem:0x0801f918` includes a dummy read. The panel-ID
transaction is DCS `04` followed by three useful bytes.

## Pixel and composition transport

Rectangular write:

1. Enter command mode.
2. Send `2a` with inclusive column start/end.
3. Send `2b` with inclusive page start/end.
4. Send `2c`.
5. Restore pixel mode and stream data.

RGB565: red bits 15:11, green 10:5, blue 4:0 (bootloader conversion). The
panel's `3a` = `66` or `06` is kept although input words are 16-bit; expansion
in the controller path is inferred, not decoded.

PL080 direct transfer: DMAC0, destination `0x38300040`, request 3, 16-bit
width, transfer count in halfwords, source increment, config `88c1`. The
firmware memory-endpoint ID is `1c` in the bootloader and `1f` in OSOS (software
IDs, not DMA request lines). OSOS `mem:0x080444d4` does allocation, start, wait
and release. Direct blit `mem:0x081a9094` needs an even width and 4-byte-aligned
source. The disk image cleans the D-cache over the line buffer before the blit
(`mem:0x08071860` calls `0x08030eb6`); the OSOS direct path (`mem:0x080444d4`,
`0x081a9094`) and the bootloader (`Lcd.pe+0x494`) do no cache maintenance in
these routines.

CLCD output uses LCD `70`/`74`/`78` and `80`/`8c`
([display-compositor.md](display-compositor.md)). Mixing direct writes with
CLCD ownership without waiting on `8c` can corrupt output; the blocking wait
exists only in the forced GRAM-begin variant (see the compositor page).
Suspend/wake must drain transfers, run the panel-specific sleep/init, and
coordinate gate and backlight state; display-on alone does not reproduce the
original wake path.

## Panel reset

The bootloader LCD modules do not toggle a panel reset line. The diagnostics `lcd` test drives the
panel reset on GPIO port 9 bit 7 (selector `0x00090007`, `diag TestLCDDisplay.pe+0x1d2c..0x1d6c`):
`lcd on` drives it high (`0xf`) and waits 120 ms; `lcd off` waits 120 ms and drives it low (`0xe`). Details in
[lcd-panels.md](lcd-panels.md).

## Unknown

Panel connector data width, pad-to-pin mapping, every LCDCON bit, and the meaning of
`18`/`20`/`68`/`78`/`7c`/`88`. Early GPIO configuration elsewhere is a separate prerequisite.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Controller offsets, mode, read protocol | bootloader, restore and OSOS controller code | observed (bootloader, restore, OSOS) |
| Direct DMA and color packing | `bootloader DmaLight.pe+0x554`, `bootloader Gop.pe+0x2e6`, OSOS `mem:0x080444d4` | observed |
| CLCD handshake registers | OSOS CLCD state and hardware init routines | observed |
| Panel reset = GPIO port 9 bit 7 | `diag TestLCDDisplay.pe+0x1d2c..0x1d6c`, literal `0x90007` | observed (selector semantics inferred from the codec reset) |
| Diagnostics write `18`, `68` and the `78`/`74`/`70`/`7c` order | `diag Clcd.pe+0x434`, `+0xaec` | observed |
