# AES DMA engine

Register interface of the SoC AES accelerator. Base is physical `0x38c00000`,
32-bit MMIO accesses. The clock is bit 7 of `0x3c500048`, enabled by clearing
it; the restore OS calls it clock ID 9, ROM/direct-gate numbering calls it 7
(same gate).

Evidence is restore `aes_crypt`, `disk mem:0x080304b8` (checked against the
instruction listing), and ROM `mem:0x20001e34` for the key-size layout.
Register names are working names; why the address/size pairs are duplicated is
unknown.

## Register map

| Offset | Working name / access | Observed value or meaning |
|---|---|---|
| `00` | Enable, W | `1` during an operation, `0` after completion |
| `04` | Go, W | Write `1` after configuration |
| `08` | Reset, R/W | Restore writes `1`, waits until readback is zero; selector 3 skips this reset. ROM writes 1 then 0 with no readback poll (`bootrom mem:0x20001e5c/0x20001e60`). |
| `0c` | Status, R/W | Restore writes `7` before go, then waits for bit 0 set. ROM checks low four bits for a nonzero result. Other status bits: meaning unknown. |
| `10` | Unnamed, W | ROM writes zero; restore path leaves it untouched. |
| `14` | Mode, W | Restore writes `0x0e` for decrypt or `0x0f` for encrypt. Key-size code occupies `[5:4]`: 0/1/2 for 128/192/256 bits in ROM. Other mode combinations not seen. |
| `18` | Output size, W | Byte length |
| `20` | Output address, W | DMA bus address |
| `24` | Input size, W | Same byte length |
| `28` | Input address, W | DMA bus address |
| `2c` | Auxiliary size, W | Same byte length |
| `30` | Auxiliary address, W | Restore writes the input address |
| `34` | Fourth size, W | Same byte length; distinct hardware role unknown |
| `4c…68` | User-key words, W | Eight word positions, key right-aligned to end immediately before `6c` |
| `6c` | Key selector, R/W | 0 user key; 1/2 hardware-key selectors; 3 accepted with reset skipped |
| `70` | Unnamed setup, W | Restore writes `1` before key/selector setup |
| `74…80` | IV, W | Four words; caller-supplied or all zero |
| `88` | Selector complement, W | Bitwise complement of the value read back from `6c` |
| `8c` | Unnamed, W | ROM writes zero |

The observed key ranges are 128-bit `5c…68`, 192-bit `54…68`, and 256-bit
`4c…68`. Firmware copies consecutive CPU words from the key/IV byte arrays.
This gives the register write order; the cipher's internal byte order on
silicon is not verified.

## Operation sequence

The restore routine's ABI is `(direction, selector, key, output, input,
length, iv)`. Zero length returns `0x28`; a non-multiple of 16 or selector
above 3 returns `9`. Normal completion returns zero.

1. Enable the AES clock.
2. Unless selector is 3, write reset=1 and poll reset until zero.
3. Write `+70=1`. For selector 0, copy four key words to `+5c…68`.
4. Write selector at `+6c`, read it back, and write its complement at `+88`.
5. Enable the engine; write mode and the four equal byte-length fields.
   Program input, output and the auxiliary-input address.
6. Write four IV words, or zero IV if no pointer was supplied.
7. Write status=7, go=1, then poll status bit 0.
8. Write enable=0 and disable the clock.

The reset and completion polls are unbounded. The selector error path runs
after clock/reset setup.

The engine reads and writes memory through its own address registers; no PL080
channel or request line is allocated, and no interrupt handler is used. Whether
the block can raise an interrupt is unknown.

## Modes, chaining and ownership

The operations run as CBC with the IV registers. No hardware ECB-select field
was found; software ECB uses independent one-block runs with zero IV. Split CBC
requests carry the chaining value between runs: for decrypt the final input
ciphertext block, for encrypt the final output ciphertext block. In-place
decryption saves the last ciphertext block before DMA overwrites it.

Hardware alignment limits are not known.

The GID/UID labels for the hardware-key selectors are inherited names. The
selector numbers and their use are observed; the physical keys and their scope
are not visible in MMIO traces. Selector 3's state-retention behavior is
unknown ("keep key" is an API name). Whether the IV registers update
automatically is not verified; the API keeps chaining in software.

## Decompiler pitfall

The decompiled C of the disk routine turns the reset polling loop into an
infinite loop and drops the selector-0 key-loading branch (volatile reads not
modeled). The instructions show the reset load/compare/back-branch at
`disk mem:0x080304e8…0x080304ec`, key loads/stores at `0x08030506…0x08030514`,
and completion load/test/back-branch at `0x0803056a…0x0803056e`. The instruction
listing is authoritative for those paths: the decompiled control flow drops
the non-3 selector paths, which the instructions do execute.

## Validation

Not validated: hardware-key outputs, physical byte order, real
192/256-bit operation, DMA coherence on silicon, automatic IV update, unknown
status/error bits, selector-3 behavior.

## Finding ledger

| Finding | Evidence | Type / limit |
|---|---|---|
| Register setup, reset polling, key and IV ordering | `disk mem:0x080304b8…0x0803057e` (instructions), literal `mem:0x08030580=0x38c00040` | observed; checked against instructions |
| Key-size ranges | ROM `mem:0x20001e34` | observed; no silicon known-answer test |
| Decompiler removes real polling/key paths | C versus instruction evidence above | observed; check volatile MMIO at instruction level |
