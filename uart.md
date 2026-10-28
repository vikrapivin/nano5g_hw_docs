# UART serial controllers

`disk mem:0x080723e4` selects one of four bases: index 0 = `0x3cc00000`,
1 = `0x3db00000`, 2 = `0x3dc00000`, other = `0x3dd00000`. The map is not a
contiguous `0x4000`-stride map.

The [DmaLight routing table](dma.md) also names endpoints at
`0x3e500020/24` (UART4). The disk selector does not cover that fifth block,
and its initialization is not decoded.

## Setup writes

All accesses are 32-bit MMIO.

| Offset | Startup value | Later use |
|---|---|---|
| `00` | `3` | line-format word; framing fields not decoded |
| `04` | `405` | control; OR `0c` for one DMA direction, OR `03` for the other |
| `08` | `7` (last write) | FIFO control/reset word; field meanings unknown |
| `0c` | `1` | replaced by 11 when caller config word1 is nonzero |
| `10` | `170` | status/control startup write; clear semantics unknown |
| `18 bit9` | read during transmit | while set, the transmitter waits before writing the next byte |
| `20` | per-byte data | low byte of the caller buffer, written in a 32-bit transaction |
| `28` | `0003000f` | later `(divide(001c2b89, requested_rate)-1) \| 30000` |

Init order: `04`, `00`, `0c`, `10`, `28`, `08`. Baud configuration
`mem:0x080724cc` calls this init, optionally writes `0c` = 11, programs `28`,
stores DMA flags, and requests DMA channels when DMA is selected. How the
constant `001c2b89` relates to the input clock or oversampling is unknown.

## Transfer

`mem:0x080725a0` validates the buffer, count and device descriptor.

- Without DMA: spin while status `18` bit 9 is 1, write one byte to `20`, repeat.
- With DMA: submit source pointer, count and the descriptor's channel/request
  parameters to the common DMA functions, then wait.

Neither loop has a timeout. The descriptor byte at +9 selects the controller;
flags at +18 and channel/request bytes around +1c..+21 are software RAM
fields, not UART offsets.

## Diagnostics serial services

- `TestUart baud <rate>`: finds the EFI SerialIo handle whose device path has an ACPI node `_HID = 0x050141d0` (PNP0501) and `_UID = 0`, then calls SetAttributes with the baud in the rate slot and constants `0x10` (FIFO depth), `1000000` (timeout µs), `1`, `8`, `1` (`diag TestUart.pe+0x2e0..0x37e`, constants `+0x3c0/+0x3c4`). The mapping of UID 0 to a physical controller is not shown, and the first SetAttributes argument was clobbered in the compiled code (argument order not confirmable).
- `SerialTextInOut` reports an 80 x 25 text mode (`diag SerialTextInOut.pe+0x5dc..0x62a`) and decodes `ESC` sequences to EFI scan codes (`+0x604..0x700`): `ESC [ A/B/C/D` = Up / Down / Right / Left (1..4); `h` Home 5; `k` End 6; `+` Insert 7; `-` Delete 8; `?` PageUp 9; `/` PageDown 0xa; `1`..`9` F1..F9 (0xb..0x13); `!` 0x15; a bare `ESC` as read by the loop. UID 1 in this module is likewise unmapped.
- `DiagInit` sets GPIO port 0 pins 4 and 5 to function 2 (dock serial lines, [GPIO](gpio-pinmux.md)); `TestTwif` sends its debug bytes on "the dock UART" (role inferred).

Unknown: connector pin assignment, voltage levels, RX path, interrupt masks,
flow control.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Bases, raw init writes | `disk mem:0x080723e4` | observed |
| Rate/configuration/DMA choice | `disk mem:0x080724cc` | observed; service semantics partly decoded |
| TX wait bit and data register | `disk mem:0x080725a0` | observed; no serial measurement |
| `TestUart` SerialIo arguments (PNP0501, UID 0) and `SerialTextInOut` ESC decoding | `diag TestUart.pe+0x2e0..0x37e`; `diag SerialTextInOut.pe+0x5dc..0x700` | observed; UID to controller mapping not shown |
