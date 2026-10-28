# I2S digital audio interfaces

The codec's PCM data goes through I2S2 at **`0x3d400000`**; its control
registers go over I2C. OSOS builds separate I2S2 TX and RX objects in
`osos mem:0x082a4444`; the radio is I2S1 RX at `0x3cd00000`. I2S0 at
`0x3ca00000` is in the firmware base table but unused by the codec/radio
objects. This is a register-interface reconstruction; pinout and waveforms are
not measured.

## Registers and signals

All registers are 32-bit MMIO at offsets from the selected I2S base. Names
follow use; TXCON and RXCON bitfields are not decoded.

| Offset | Function and values | Evidence |
|---|---|---|
| `+00` | Clock/interface enable: write 1 to enable; disable writes 0 then waits for bit 1 set | `mem:0x22006f5c` |
| `+04` | TX format/control: codec mode 1 writes `0x03100099`; mode 0 uses `(old & 0x780) \| 0x0b000019` | `mem:0x22006ea4` |
| `+08` | TX command: start ORs `6`; stop clears bit 2; full disable writes 0 | `mem:0x22006f36`, `0x22006f10`, `0x22006f5c` |
| `+10` | TX FIFO, PCM playback DMA destination | `mem:0x08551f20` peripheral table |
| `+30` | RX format/control: `0x1000` in the codec setup | `mem:0x22006ea4` |
| `+34` | RX command: same start/stop pattern as TX | `mem:0x22006f36`, `0x22006f10` |
| `+38` | RX FIFO, recording DMA source | Peripheral ID `0x11` |
| `+3c` | Status register in the shared interface map; bits not decoded | shared interface map |
| `+40` | Clock divider; write `D/2`, values below | `mem:0x22006ff4` |

The diagnostics' I2S driver writes `+00 = 1` after the clock-gate enable and a
25 ms wait (`Stall 0x61a8`); a separate method writes `+00 = 0x40`; start is
`(+08 or +34) |= 6`, stop clears bit 2; divider `D/2` goes to `+40` and `0x1000`
to `+30` (`diag I2s.pe+0x5c8,+0x648,+0x4d8,+0x470,+0x300`). Clock IDs
`0x2d/0x2c/0x2b` (I2S0/1/2) are confirmed there.

OSOS sets codec register 04 bit 5, so the codec drives the serial clocks and
the SoC receives them, although Apple still writes the SoC divider. Clock-service
IDs: I2S2 `0x2b` clears `0x3c50004c` bit 16; I2S1 `0x2c` clears bit 10; I2S0
`0x2d` clears bit 7. Set the masks to gate off. MCLK comes from the CLKCON4 path
in [audio-codec.md](audio-codec.md).

## Pins and DMA peripheral IDs (diagnostics)

Pins are selected through the GPIO protocol (pin, cfg): cfg 3 for mode 1 (codec,
TX control `0x03100099`), cfg 2 for mode 0 (TX control `0x0b000099`)
(`diag I2s.pe+0x300..0x44a`, table `+0x8a0..+0x8b4`, literals `+0x45c/+0x460`).

| Block | Pins (port<<16 \| bit) | DMA TX / RX ID |
|---|---|---|
| I2S0 | `0x00000007`, `0x00020004` | `0x0c` / `0x0d` |
| I2S1 | `0x90001`, `0x90002` | `0x0e` / `0x0f` |
| I2S2 | `0xc0001`, `0xc0002` | `0x10` / `0x11` |

DMA IDs from `diag I2s.pe+0x498`, table `+0x8d0`.

## Sample rates and DMA

| Sample rate (Hz) | I2S `+40` | Codec reg 05 low five bits |
|---:|---:|---:|
| 8000 | 750 | `1d` |
| 11025 | 544 | `1b` |
| 12000 | 500 | `19` |
| 16000 | 375 | `15` |
| 22050 | 272 | `13` |
| 24000 | 250 | `11` |
| 32000 | 125 | `0d` |
| 44100 | 136 | `0b` |
| 48000 | 125 | `09` |

The 32000 divider (125) is as observed. The diagnostics use rate codes (stream
descriptor byte +6) 0..8 = 8000, 11025, 12000, 16000, 22050, 24000, 32000, 44100,
48000 in both `AudioTransportEx` (`+0x27c`, table `+0x284`) and `I2s` (`+0x3a2`,
table `21 23 26 29 2b 2e 31 32 31`); the `I2s` switch gives D = 1500, 1088, 1000, 750,
544, 500, 250, 272, 250, written as D/2. `TestI2CStress` passes rate code 6 (32000)
in its radio descriptor (inferred). On platform string "N33" with I2S index 1,
`AudioTransportEx` sets the Si47xx sample-rate property `0104` from the stream's
rate code before starting DMA (0 on stop); DMA uses 4096-halfword (`0x1000`)
LLI chunks (`diag AudioTransportEx.pe+0x7a4`). OSOS rate selection at
`mem:0x082bb304`/`0x082c4718` rounds down to a supported rate. Initial playback
rate is 44100. The sequence is: write the divider, disable MCLK, configure
source 0/div 4, re-enable MCLK, update codec reg 05.

[PL080 DMA](dma.md) moves 16-bit samples in bursts of four. I2S2 TX: peripheral
ID `0x10`, FIFO `0x3d400010`, request DMAC0/0 or DMAC1/4. RX: ID `0x11`, FIFO
`0x3d400038`, request DMAC0/1 or DMAC1/5. IDs are firmware table indices;
request numbers are controller signals. I2S1 TX/RX use DMAC1 requests 2/3. The
transfer-count field counts halfwords, at most 4095 per LLI. Apple allocates a
free channel dynamically.

Start order: gate/CLKCON, retire old DMA, format, rate, queue DMA, then OR 6 into
TXCOM/RXCOM. Stop clears command bit 2 and retires DMA. Full disable zeros both
command registers, writes CLKCON0, waits for bit 1, then gates the block. OSOS RX
uses two 4096-byte pending buffers; DMA completion must not release a buffer the
peripheral still reads.

## Unknown

Electrical pad characteristics, TX/RX bitfields, actual MCLK
frequency, setup/hold times. `0x03100099`, `0x0b000019`, `0x1000` are firmware
literals. 6 MHz MCLK is inferred from source 0/div 4, not measured. Radio mode 0
differs from codec mode 1.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Three bases and codec/radio endpoints | `mem:0x220064cc`, `0x082a4444` | observed |
| Register writes and rate table | `mem:0x22006ea4`–`0x22006ff4`, `0x082bb304` | observed |
| Pin selectors, DMA IDs 0x0c..0x11, 25 ms settle | `diag I2s.pe+0x300..0x498` | observed |
| Rate-code map and N33 radio rate hook | `diag AudioTransportEx.pe+0x27c,+0x7a4`, `I2s.pe+0x3a2` | observed |
