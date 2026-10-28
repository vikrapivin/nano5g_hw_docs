# LCD panel variants and command scripts

The visible display is **240×376**, with command-mode GRAM accessed through
[the LCD bus](lcd-controller.md). Firmware recognizes five ID prefixes from DCS
command `04`: `38b3`, `38c4`, `38e6`, `38f7`, `48c4`. A prefix does not
identify the manufacturer or part. Some boot stages keep the third ID byte, but
the main table-selection branches ignore it.

## Per-panel table inventory

Every byte of the OSOS scripts was read from the firmware and compared to the
reconstructed tables. The init and sleep bytes of each variant are contiguous at the VA below
(212, 205, 216, 207 and 236 bytes in total). Vendor commands with unknown fields
are included.

| Variant | Original OSOS table VA | Init bytes | Following sleep bytes | Total bytes at VA |
|---|---:|---:|---:|---|
| 38c4 | `084322ac` | 203 | 9 | 212 |
| 48c4 | `08432380` | 196 | 9 | 205 |
| 38b3 | `0843244d` | 207 | 9 | 216 |
| 38e6 | `08432525` | 207 | uses 38b3 sleep | 207 |
| 38f7 | `084325f4` | 219 | 17 | 236 |

The OSOS sequence interpreter at `mem:0x081a6e84` is length-bounded, with no
end opcode:

| Opcode | Meaning |
|---|---|
| 0 | `command,length,data...` |
| 1 | draw/fill |
| 2 | millisecond delay |
| 3 | send 16-bit command/data |

Bootloader scripts instead use opcode 3 for fill and 4 for end, so decode each
stage's bytes with its own interpreter.

## Command meanings

| Command | Observed role / payload |
|---|---|
| `04` | read ID with dummy read |
| `11` | sleep out, followed by 120 ms at initialization |
| `13` | normal mode near end of initialization |
| `29` | display on |
| `28` | display off before sleep |
| `10` | sleep in |
| `35` | payload `00`, tearing-effect setting; TE routing not identified |
| `36` | address/orientation; payload `00` in the 38xx init scripts and from the window routine `mem:0x080221bc` (`00`, or `60` for orientation index 5); the 48c4 init script has no `36` |
| `3a` | pixel format `66` for 38xx, `06` for 48c4 |
| `2a` | column window `00..239` |
| `2b` | initial page window `00..399` on 38c4/38b3/38e6; `00..375` on 48c4/38f7 |
| `38`/`39` | exit/enter idle-mode control |

The `00..399` page window does not mean a 400-row display; rendering geometry
is 240×376. Vendor sequences differ:

- 38xx: `b1`-`b4`, `b6`, `c0`-`c5`, `ea` (`eb` on 38b3/38e6 only) and `e0`-`e5` arrays.
- 48c4: unlock `f0`/`f1` = `5a`; write `b0`/`b1`/`b2`/`b8` and `c0`-`c5` arrays; lock `f0`/`f1` = `a5`.
- 38f7: additionally `f0` = `aa 55 25` and `f6` = `7f 8a 34 00 04 26 07`.

Exact bytes and order are in the inventories. The `c*`/`e*` bytes are not
decoded to voltages or gamma formulas.

Detection differs by stage:

- OSOS/restore: 38e6, unknown IDs and failed reads all give default index 0 (38c4), although a 38e6 table exists. Six tries with 10 ms delays.
- Bootloader: separate 38e6 branch; retries unsupported IDs up to five times.

## Orientation, sleep and brightness calibration

- OSOS `mem:0x080221bc` emits MADCTL and inclusive windows. Wrapper `mem:0x0801f984` maps orientation 5 to MADCTL `60` and every other orientation to `0`.
- Panel sleep scripts: blank, display off, sleep in. 38f7 inserts `f6` ending `00` and has no trailing delay in its OSOS script; other variants include 120 ms or 50 ms as recorded.
- Bootloader 38b3/38e6 sleep differs from OSOS/restore; keep stage-specific timing.
- Wake replays initialization.

The five descriptors at `mem:0x0855212c`, stride 36, hold dimensions, format
byte `3d`, function pointers, three brightness calibration values and the ID.
Calibration triples: `93,b4,e3` for 38c4/48c4 and `93,c0,ed` for the other
three. [Backlight](backlight.md) interpolates at 40%. Format `3d` is a
descriptor value, separate from the direct-blitter format 34 (`disk
mem:0x0803e98e` passes it to the idle-mode command: `0x0b` selects DCS `39`,
anything else DCS `38`).

## Diagnostic LCD commands

The diagnostics `lcd` test (`diag TestLCDDisplay.pe`) reaches the panel through the bootloader LCD protocol
(GUID 144d4aca, provider `Lcd.pe`) and the DisplayPlatform protocol (GUID 3fd4147f, same module as the
bootloader `DisplayPlatform.pe`):

| Command | Action | Evidence |
|---|---|---|
| `on` | GPIO port 9 bit 7 (`0x00090007`) high (`0xf`), Stall 120 ms (out of reset) | `TestLCDDisplay.pe+0x1d2c` |
| `off` | DisplayPlatform method 0 with argument 1, Stall 120 ms, port 9 bit 7 low (`0xe`, in reset) | `+0x1d50` |
| `enable` / `en` | DisplayPlatform method 0 with argument 0 (default power-on script); methods 3/4 (`+0xc`/`+0x10`) bracket the panel read as lock/unlock | `+0x1d70..0x1dc6` |
| `id` | read register `04`, 3 bytes, print `ID: 0x.. 0x.. 0x..` | `+0x1d92` |
| `read <reg> <n>` | LCD protocol method 3 (`+0xc`) read(reg, len, buf), n <= 0x80 | `+0x1de6` |
| `write <reg> [bytes]` | LCD method 6 (`+0x18`) write script `00, reg, n, data..., 04` (opcode 0 command+data, opcode 4 end), n <= 0x7c | `+0x1e7c` |

Method 2 (`+8`) sends a command with no data. The three ID bytes are named ControllerID (byte 0), VendorID (byte 1)
and BuildID (byte 2) through the DisplayPlatform property "DisplayId", packed little-endian
(`TestLCDDisplay.pe+0x1da0..0x1dd2`). Grey-step test levels: `10 28 40 58 70 88 a0 b8 d0 ec` (each repeated in R, G, B),
plus `ffffff`, `00ffff`, `ff00ff` and the primaries; patterns are BMPs from the diagnostics `LCMPatterns` directory
drawn through GOP Blt; options `--iqc`, `--fatp`, `--dqe` (`+0x1266..0x1436`).
`DisplayPlatform` method 0 with argument 1 is used only by `lcd off`; its sleep-script role is inferred.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| OSOS init/sleep literals | the VA table above | observed (firmware byte match) |
| Detection/default discrepancy | `mem:0x0801ea28`, `0x0801d5b8`; `bootloader` offset `0x220` | observed, stage-specific |
| Panel pinout, ratings, vendor fields | not in recovered scripts | meaning unknown |
| Panel reset GPIO port 9 bit 7, 120 ms waits | `diag TestLCDDisplay.pe+0x1d2c..0x1d6c` | observed |
| LCD protocol methods and write-script grammar, ID register `04` byte names | `diag TestLCDDisplay.pe+0x1e20..0x1f28`, `+0x1da0..0x1dd2` | observed |
