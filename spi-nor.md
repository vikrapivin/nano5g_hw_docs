# SPI controllers and the ROM's serial-NOR interface

The mask ROM has a polled serial-NOR boot interface. Whether a NOR chip is
populated on a given nano 5G is not known. The interface covers read, status and
identification commands, not programming or erase.

Evidence: the boot ROM. Function addresses below are
`bootrom mem:`; subtract `0x20000000` for the offset in the ROM image.

## Instances and external signals

| Controller | Base | Recovered use |
|---|---|---|
| SPI0 | `0x3c300000` | NOR boot variant 0. |
| SPI1 | `0x3ce00000` | NOR boot variant 1. |
| SPI2 | `0x3d200000` | ROM helpers accept variant 2; normal boot strap selection does not select it. |
| SPI3 | `0x3da00000` | Fourth ROM offset-table entry; generic bootloader SPI inventory. |
| SPI4 | `0x3e100000` | Bootloader SPI inventory; not in the four-entry ROM table. |

The ROM table at `0x2000bdc4` contains offsets
`0,0x00b00000,0x00f00000,0x01700000` relative to SPI0. ROM gate indices
`0x17/0x20/0x24` enable SPI0/1/2 with paired peripheral gates; see
[clock numbering](clocks-reset.md).

`rom_spi_gpio_init:0x20001ba0` selects SPI0 pads port 0.1..3 function 2 and
port 0.0 as GPIO CS, pulsed high/low/high. Variant 1 selects port 4.7 and
port 5.0..1 function 2, with CS on port 4.6, also high/low/high. Which of the three alternate-function pads is clock, MOSI or MISO is not
decoded. CS is active low (from its placement around commands). CPOL/CPHA are
not decoded from the config word.

## Register inventory

Accesses are 32-bit MMIO; data transfers use the low byte. "Count" and "FIFO
reset" are inferred from use.

| Offset | Read/write behavior | Meaning |
|---|---|---|
| `+0x00` | Write 1 enable, 0 disable; OR `0xc` during initialization | Control. Bits 2/3 called FIFO reset by sequence; disable returns whether prior bit 1 was clear. Bit 1 state meaning unknown. |
| `+0x04` | Init `0x00040618`; set bit 0 while reading payload, clear afterward | Configuration and automatic receive/clocking control inferred. Other fields unknown. |
| `+0x08` | Init write `0xf`; RX waits `(value&0xf800)!=0`; TX waits while `(value&0x7c0)==0x400` | Combined status/flags. Fields 15:11 and 10:6 behave as FIFO occupancy indicators; depths and encoding unknown. |
| `+0x0c` | Init write 6 | Unknown. |
| `+0x10` | Write outgoing byte | TX data. |
| `+0x20` | Read incoming byte | RX data. |
| `+0x30` | `pclk_hz/6000000` | Clock divider chosen for nominal 6 MHz. Integer divider; frequency not measured. |
| `+0x34` | Write 100 for ready check, then `payload_len+4` | Transfer-count word; difference from +4c unknown. |
| `+0x3c` | Init `0xffffffff` | Unknown. |
| `+0x40` | Init `0xffffffff` | Unknown. |
| `+0x4c` | Same count as +34, masked with `0x1fffff` | 21-bit count field in the helper; hardware transfer limit unknown. |

## ROM initialization and transactions

`rom_spi_init:0x20008bc0` writes `+08=0xf`; resets FIFO via `+00|=0xc`;
sets `+30=rom_spi_div_clk(6000000)`; writes `+0c=6`, `+40=-1`, `+3c=-1`,
`+04=0x40618`; resets FIFO again; then writes `+00=1`.
`rom_spi_div_clk:0x20005304` divides the saved PCLK variable (54 MHz in ROM
startup) by requested 6 MHz, so the ROM's programmed divider is 9.

Byte TX (`0x20004f30`) checks status bits 10:6 against `0x400` for up to 500
iterations and writes the byte to +0x10. Byte RX (`0x20004ed8`) waits up to 500
iterations for a nonzero status field 15:11, then reads +0x20; it returns a timeout
indicator. These are poll counts, not 500 µs. TX still writes after its wait expires.

| Device command | Firmware sequence | Meaning / constraints |
|---|---|---|
| `0x05` | CS low; TX command and consume RX; TX `0xff` and consume RX; CS high | Read status; bit 0 is the ready/busy criterion. |
| `0x9f` | CS low; command, then three dummy `0xff` bytes with RX; CS high | Read three-byte JEDEC ID, assembled most significant byte first. |
| `0x03` | CS low; command then address bits 23:16,15:8,7:0, consuming RX after each; set config bit 0; read payload; clear bit 0; CS high | Normal 24-bit addressed read. No fast-read dummy cycle in this path. |

`rom_nor_wait_ready:0x20004fec` repeats status transactions until bit 0 clears
or 500 tries are reached. `rom_nor_read_image:0x20008ca8` writes both count
registers 100, calls the ready helper, then writes both counts `len+4` and calls
`rom_nor_read:0x20005124`. Its wrapper returns 4 for success, 5 for failure.
These result codes are firmware return values, not chip status bytes.

The boot caller `0x20003114` reads a 0x600-byte container header at flash offset
`0x8000` into SRAM `0x22025200`, and a body at `0x8600` into SRAM `0x22000000`.
These are boot-image layout choices.
At successful handoff the ROM disables SPI (`+00=0`) and records the selected
source. Early-failure cleanup is not uniformly explicit; not every error path
visibly deasserts CS.

## Not established

NOR part, capacity, page/sector geometry, erase/program opcodes, write-enable
sequence, voltage, maximum SCK rate, FIFO depth, count semantics, DMA/IRQ
interface, CPOL/CPHA, SPI3/4 board connection. The values `0x40618`, +0x0c = 6
and the all-ones writes to +0x3c/+0x40 have no field names.
SPI0/1/2 TX/RX data registers (+0x10/+0x20) do appear as DMA endpoints in the
bootloader DmaLight table (see [DMA](dma.md)); the ROM boot path itself uses no DMA.

## Finding ledger

| Finding | Evidence | Type / limit |
|---|---|---|
| Base offsets, mux, CS | ROM `0x2000bdc4`, `0x20001ba0/0x20005318` | observed; pad roles not decoded |
| Complete known init writes | ROM `0x20008bc0` | observed; some values unexplained |
| FIFO-status tests and counts | ROM `0x20004ed8/0x20004f30/0x20004e94/0x20004eb0` | observed; names partly inferred |
| Read/status/ID command framing | ROM `0x20004fec/0x20005078/0x20005124` | observed; no attached-flash trace |
| NOR population on N33 | not seen in firmware | unknown |
