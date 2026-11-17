# Byte order

The application CPU and every bus master listed here move 32-bit words without changing their value. Byte order
matters only where a block, or the processor inside a block, takes bytes or bit fields out of a word. This page
collects those places; each block's page has the details and the evidence.

## Rules per block

| Block | Byte order | Evidence |
|---|---|---|
| [CPU](cpu-memory.md) (ARM1176) | little-endian: none of the SCTLR writes on the CPU page sets the B bit (bit 7); byte `a` of a word is bits `[8(a&3)+7 : 8(a&3)]` | observed (SCTLR writes) |
| [PL080 DMA](dma.md) | has a per-master endianness bit (`DMACCONFIG` bits 1, 2) | observed (register layout) |
| [Video encoder](video-encoder.md#byte-order-between-the-host-and-the-core) core | word accesses and ARM fetch: the word's value; byte and halfword **loads** and Thumb fetch: the big-endian lane (BE-32, byte `a` is bits `[31-8(a&3) : 24-8(a&3)]`); byte and halfword **stores**: the little-endian lane. A byte stored at `a` is loaded back at `a ^ 3` | device |
| Video encoder, host-core paths (host DMA in, mode-5 copy out, `c0c`, matrix words, `a00..a10`) | 32-bit values kept; no swap in either direction | device |
| Video encoder, bitstream ring | H.264 bytes in stream order at ascending addresses, each byte most significant bit first | device |
| Video encoder, statistics table | the iPod OS reads its words as big-endian | observed (`osos` result reader) |
| [AVC decoder](video-decoder.md) input | NAL bytes in stream order at ascending addresses, as read from the file (the iPod OS converts only the big-endian NAL length prefix); the frontend removes emulation-prevention bytes itself and `1400+4*n` returns the next `n` bits most significant first | observed (driver); device (streams supplied that way decode bit-exact) |
| AVC decoder pictures | 8-bit samples, one byte per sample at ascending addresses (luma plane, interleaved Cb/Cr plane) | device |
| [MPEG-4/H.263/JPEG decoder](video-decoder.md#mpeg-4-part-2-h263-and-jpeg-decoder-0x39600000) input | access-unit bytes in stream order | inferred (the driver passes the file's bytes) |
| [SHA](sha.md) | message byte `n` is bits `[8(n&3)+7 : 8(n&3)]` of word `n & ~3`: memory order of the little-endian CPU, as DMA reads it; the length field is big-endian in the padding (standard SHA); the digest words are stored little-endian without a swap | observed |
| [Public-key accelerator](pka.md) | operands are little-endian word arrays, word 0 least significant; the ROM's read-out takes the top word first and byte-swaps each word, giving a big-endian byte string | observed |
| [AES](aes.md) | key and IV are written as consecutive CPU words from the byte arrays; the cipher's internal byte order is not established | observed (write order) |
| [FMSS](nand-fmss-isa.md) | each instruction is two little-endian words; READ packs four sector-status bytes into a word with the first sector in the most significant byte (reverse order in memory) | observed |
| [Compositor](display-compositor.md) | the eleven-byte table at `b8+12*index` is packed into three words in big-endian order | observed |
| iPod OS MP4 reader | container fields are big-endian in the file; the stream's read mode 1 swaps 2-, 4- and 8-byte fields (`osos mem:0x0812c210`), helpers read 32/16-bit big-endian values (`mem:0x08215302`, `mem:0x08205204`) | observed |
| [ADC](adc.md) thresholds | little-endian 16-bit values in the iPod OS table | observed |
| External devices: [camera sensor](camera-sensors.md) OV7739, [FM radio](fm-radio.md) | 16-bit register indices (OV7739) and 16-bit values (FM radio) go over I2C most significant byte first | observed |

## The encoder core in one example

The host fills a word at host address `h` with the bytes `00 00 00 01` (H.264 start code; word value
`0x01000000`) and sends it to the core's RAM at `a`:

| Read by the core | Result |
|---|---|
| `ldr` at `a` | `0x01000000` |
| `ldrb` at `a`, `a+1`, `a+2`, `a+3` | `01 00 00 00`: the bytes in reverse order |
| `ldrb` after the host byte-swapped the word before sending | `00 00 00 01` |

The core then writes the bytes `00 00 00 01` with `strb` at `a ^ 3, (a+1) ^ 3, (a+2) ^ 3, (a+3) ^ 3`, so its own
byte loads read them in order. The word now holds `0x00000001`. A mode-5 copy sends that value; the host reads
`01 00 00 00` unless the core byte-swaps the word before the copy, after which the host reads `00 00 00 01`.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| CPU runs little-endian | SCTLR writes in the boot ROM, bootloader and iPod OS start-up ([CPU](cpu-memory.md)) | observed |
| Encoder core: loads BE-32, stores little-endian lane, words kept between host and core | 843 single-instruction tests, byte/halfword/word read-back of one word, word patterns and byte records each way ([video encoder](video-encoder.md)) | device |
| AVC decoder reads NAL bytes in memory order, most significant bit first | input set-up `osos mem:0x081f0a24`; only the NAL length prefix is converted; bit-exact decodes of byte streams in memory order | observed, device |
| MPEG-4 engine input in stream order | driver passes file bytes | inferred |
| SHA, PKA, AES, FMSS, compositor, ADC, I2C devices | per-block pages | observed |
