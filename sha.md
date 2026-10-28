# SHA-1 accelerator

Hardware SHA-1 engine at MMIO `0x38000000`, 32-bit accesses. Clock gate SHA1: ID 16 in the restore/bootloader
numbering (disk ID 16, PWRCON0 bit 0, cleared to enable; see [clocks-reset.md](clocks-reset.md)), gate 0 in the ROM
call `rom_clkgen_enable_gate(0)` (`bootrom mem:0x20002f40`). Same gate.

The core runs the compression function on one 64-byte block at a time. It does no padding and no length append: the
driver supplies complete, already padded blocks. A block arrives either through the register window `+0x40..+0x7c`
(boot ROM, programmed path) or by DMA from memory (`+0x80..+0x8c`, disk image path). The engine reads memory itself;
no PL080 channel is involved.

Sources: boot ROM `rom_sha1*` (`bootrom mem:0x200022c8..0x20002f40`), restore OS helpers
`disk mem:0x08037fc4`, `0x080396e0`, `0x0803f898`, `0x08038cd8`, `0x08039c2c`, and the bootloader/WTF module `Sha1`.

## Register map

| Offset | Access | Meaning | Evidence |
|---|---|---|---|
| `00` | R/W | Control/status. Bit 0 busy (read; poll until clear). Bit 1 start. Bit 2 set with start when the caller waits on a completion flag instead of polling (ROM only). Bit 3 continue: set when the block continues a previous one, clear for the first block. Disk writes the whole word: `2` first block, `0x0a` (decimal 10) every later block. ROM does read-modify-write of bits 1..3. Other bits: not decoded. | `disk mem:0x080396e0`; `bootrom mem:0x20002414` |
| `04` | W | Reset/start preparation: write `1`. Disk writes it once per hash at init; ROM writes it only when the caller passes the reset flag. | `disk mem:0x08037fc4`; `bootrom mem:0x20002f40` |
| `0c` | R/W | ROM ORs bit 0 only in the completion-flag mode, together with the start (`00 \|= 6`). Disk never touches it. Field not decoded. | `bootrom mem:0x20002414`, `0x20002898` |
| `10` | W | Unknown control; written `0` after `04=1` at init (disk and ROM). | `disk mem:0x08037fc4`; `bootrom mem:0x200022c8` |
| `20..30` | R | Five-word digest, 20 bytes. | `bootrom mem:0x2000231c`; `disk mem:0x08039c2c` |
| `40..7c` | R/W | 16-word (64-byte) message block window, ROM path. | `bootrom mem:0x20002328` |
| `78`, `7c` | W | Last two words of the block: the 64-bit message length in bits (see padding). | `bootrom mem:0x20002898` |
| `80` | W | DMA mode: `1` = block comes from memory, `0` = block comes from the window. Disk writes `1` once per call; ROM writes `1` before a DMA run and `0` before the buffered-tail and padding paths. | `disk mem:0x080396e0`; `bootrom mem:0x200026dc` |
| `84` | W | DMA source address of the caller's buffer, passed through unchanged. | `disk mem:0x080396e0` |
| `8c` | W | DMA byte count: a multiple of 64 (`0x40`). | `disk mem:0x080396e0`; `bootrom mem:0x20002ea8` |

Offsets `00..58` (23 words) are zeroed by `rom_sha1_clear_regs` (`bootrom mem:0x20002f10`, no caller found), so the
register file covers more than the offsets above. Offsets `08`, `14..1c`, `34..3c`, `88`: no access seen, meaning unknown.

## Sequences

Disk image, hash of a buffer (`disk mem:0x0803313e` calls the three steps in order; the context is a RAM structure
placed through its `0x80000000` alias):

1. Init (`disk mem:0x08037fc4`): enable gate 16 (`clkgate_set(0x10,0,1)`), write `04=1`, `10=0`; context `+0x40`
   and `+0x44` = 0 (64-bit byte count, low and high word), context `+0x48` = 2 (command to use next).
2. Update (`disk mem:0x080396e0`): add the length to the 64-bit count. Write `80=1` and `8c=0xffc0`. While the
   remaining length is greater than `0xffc0`: `84=src`, `00=context+0x48`, `context+0x48=0x0a`, remaining -= `0xffc0`,
   src += `0xffc0`, poll `00` bit 0 until clear. Then, if at least 64 bytes remain: `84=src`, `8c=length & ~63`,
   `00=context+0x48`. The residue (`length & 63`) is copied to the start of the context buffer by a byte-copy helper
   (`disk mem:0x08070ec8`) while the engine runs; then poll `00` bit 0 clear and set `context+0x48=0x0a`.
3. Finish (`disk mem:0x08038cd8`): build the last block or two in the context buffer (see padding), run each with
   `0x0803f898`: `80=1`, `84=buffer`, `8c=0x40`, `00=context+0x48`, poll bit 0 clear, `context+0x48=0x0a`.
4. Read digest (`disk mem:0x08039c2c`): five words from `20..30`, then disable gate 16 (`clkgate_set(0x10,0,0)`).

Boot ROM (`rom_sha1`, `bootrom mem:0x20002f40`, service table slot `+0x50`; arguments data, length, out20, flags;
flag bit 0 = reset, bit 1 = finish):

1. Enable gate 0.
2. With reset flag: write `04=1`. Then (both cases) poll `00` bit 0 until clear, write `00=0`, `10=0`, set the
   software done flag (state `+0x8c`) to 1. Without the reset flag the call continues an earlier hash.
3. `rom_sha1_update` (`bootrom mem:0x200026dc`): for each block, set `00` bit 3 (clear it for the first block), then start
   with `00 |= 2` and poll `00` bit 0 until clear. Blocks arrive as below.
4. With finish flag: `rom_sha1_finish` (`bootrom mem:0x20002898`) pads and runs the last block(s) with `80=0`.
5. Copy 20 bytes from `20` to the output; disable gate 0.

Per-block start, all drivers: command word has start (bit 1) set; bit 3 clear on the first block of a hash and set
on every later one, so the engine chains from the previous digest state. `00` stays busy until the block is done.

## Padding and length

The driver builds the final block; the hardware appends nothing.

- Disk (`disk mem:0x08038cd8`): byte `0x80` at index `count & 63`; zero fill to index `0x37`. If the `0x80` byte lands at
  index `0x38` or later, zero fill to `0x3f`, run that block, then zero index `0..0x37` and use it as the last block.
  Bytes `0x38..0x3f` hold the 64-bit bit count, big-endian: count in bytes shifted left by 3, high word (context
  `+0x44`) first. Then one more DMA block run.
- ROM (`bootrom mem:0x20002898`): `80=0`. Eight length bytes are computed from the byte count (state `+0x94`) times 8,
  most significant byte first. `0x80` is written into the window at the fill position (state `+0x90`), zeros fill to
  `0x38`. If fill is already `0x38` or more, zeros fill to `0x40`, that block runs, then `0x00..0x37` are zeroed. Words `78`
  and `7c` are cleared and the eight length bytes are written byte-wise into them, then the final block runs. A full
  buffer (fill `0x40`) is reset to 0 before padding.

## Byte and word order

- Message block: byte `n` of the message goes to window byte `0x40 + n`. A byte-wise insert puts byte `n` into
  bits `[8*(n&3)+7 : 8*(n&3)]` of the word at `0x40 + (n & ~3)`; that is memory order of a little-endian CPU, the same
  layout DMA reads from RAM. Length bytes use the same placement at `78`/`7c`. No byte swap by software on the message.
- Digest: `20..30` are read as five 32-bit words and stored little-endian into the output byte array, in both the ROM
  (`rom_memcpy` of 20 bytes, `bootrom mem:0x20002f40`) and the disk helper (byte stores of each word, `disk mem:0x08039c2c`). No
  software byte swap on the digest in any of the paths read. Whether the words hold standard H0..H4 values
  (output then in little-endian word order) is not verified against a known-answer vector.
- The bootloader module `Sha1` only defines three functions; none of them reads the digest or the message (see below).

## DMA versus programmed path

| | Disk image (DMA) | Boot ROM, DMA branch | Boot ROM, window branch |
|---|---|---|---|
| Source | caller buffer, any alignment passed to `84` (no check) | caller buffer, only when `state+0x90 + src` is 4-byte aligned | byte-wise inserts into `40..7c` |
| Mode `80` | 1 | 1 | 0 |
| Count `8c` | multiples of 64, max `0xffc0` per run | multiples of 64: `0xff00` per run, last run `n & 0xffc0` | 64 per run |
| Residue < 64 | copied to the RAM context, hashed in step 3 | buffered in the window, state `+0x90` fill count | same |
| Done test | poll `00` bit 0 clear, unbounded | poll `00` bit 0 clear, up to `0xf4240` (1,000,000) reads, then returns silently | same |

Chunking: the disk loop continues while the remaining count is above `0xffc0` and then issues the remainder rounded down
to a multiple of 64. The ROM loop cuts at `0xff00`. `rom_sha1_dma_block` (`bootrom mem:0x20002ea8`, no caller) spins forever
if the length is not a multiple of 64.

## Interrupt, reset, clock, timing

- Interrupt: no handler found. The ROM has a completion-flag mode (`wait = 1`: clear state `+0x8c`, `0c |= 1`, `00 |= 6`,
  spin on the flag) that expects something external to set the flag; its only caller (`rom_sha1`) passes `wait = 0`, so
  this mode and the SHA interrupt number are unobserved.
- Reset: `04 = 1`, then wait `00` bit 0 clear, `00 = 0`, `10 = 0`. The disk init writes `04 = 1` and `10 = 0` without the
  wait and without `00 = 0`.
- Clock: gate before the first register write, gate off after the digest read, both in the disk and ROM paths.
- Timing: only the ROM poll bound is known (1,000,000 reads of `00`, not a time). Per-block latency not measured.

## Bootloader and WTF module

`bootloader Sha1.pe+0x410` (identical code in `wtf Sha1.pe`) does the init steps through a service-table call:
slot `+4` with arguments `0x10, 0, 1` (gate 16), then context `+0x40` = 0, `+0x44` = 0, `+0x48` = 2. It does not
write `04` or `10` itself. `Sha1.pe+0x2cc` stores two fields of its argument block; `Sha1.pe+0x2ec` registers the module
through two table calls (offsets `+0xac`, `+0xb0` of the boot services table). The `.text` area `0x220..0x4ef` also holds
undefined Thumb code (for example near `Sha1.pe+0x3c0`) that was not analysed.

## Unknowns

Unused command bits, fault bits, field meaning of `0c` and `10`, offsets listed as untouched, SHA interrupt, clock limits,
completion latency, DMA source alignment requirement, whether `00=0` is needed after power-up, digest word order against
a known-answer vector.

## Finding ledger

| Finding | Evidence | Type / limit |
|---|---|---|
| Disk init: gate 16, `04=1`, `10=0`, context `+0x40/+0x44=0`, `+0x48=2` | `disk mem:0x08037fc4` | observed |
| DMA command `2` first, `0x0a` after; `80=1`, `8c=0xffc0`, poll `00` bit 0, no timeout | `disk mem:0x080396e0`, `mem:0x0803f898` | observed |
| Residue < 64 handled in RAM, padding built by the driver | `disk mem:0x08038cd8`, `mem:0x08070ec8` | observed |
| Padding: `0x80`, zeros, big-endian bit count in last 8 bytes | `disk mem:0x08038cd8`, `bootrom mem:0x20002898` | observed; hardware appends nothing is inferred |
| Bit 1 start, bit 3 continue, bit 2 flag mode | `bootrom mem:0x20002414`, `mem:0x200026dc`, `disk` command values 2/0x0a | bit 1 observed; bits 2, 3 inferred from use |
| Message byte placement, LE lanes of window words | `bootrom mem:0x20002328` | observed |
| Digest read raw from `20..30`, no swap | `bootrom mem:0x2000231c`, `mem:0x20002f40`; `disk mem:0x08039c2c` | observed; word order vs known answer not verified |
| ROM poll bound `0xf4240`, chunk `0xff00`, mask `0xffc0` | `bootrom mem:0x20002414`, literals `0x20002ea0/0x20002ea4` | observed |
| ROM gate 0 vs disk gate 16 same gate | `bootrom mem:0x20002f40`, `disk mem:0x08037fc4`, [clocks-reset.md](clocks-reset.md) | inferred (equal bit 0 of `0x3c500048`) |
| Bootloader `Sha1.pe+0x410` is the init wrapper only | `bootloader Sha1.pe+0x410`, `+0x2cc`, `+0x2ec` | observed; module's other code not analysed |
| Interrupt not found | ROM `rom_sha1*`, disk helpers; searched those only | not found; other images not searched |
| Error/timing behavior | no status decoded | not measured |

See [AES](aes.md) for the separate cipher engine.
