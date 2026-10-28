# Random-number block

**MMIO base `0x3c100000`**. The entropy source (true random, deterministic
or mixed) is unknown, and output quality is not measured.

## Operations

| Register | Access | Behavior |
|---|---|---|
| `+00` | 32-bit write/read | restore init writes 0 (`disk mem:0x0807206c`); bootloader seeding writes `0x60` (`Prng.pe+0x328`); acquisition waits until `(value & 7) != 0` |
| `+04` | 32-bit read | next word once that status condition holds |
| `+08` | 32-bit write | seed word written by the bootloader (`Prng.pe+0x32c`); the restore OS never touches it |

- `disk mem:0x0807206c` reads the previous state of gate ID51 (`0x33`),
  enables the gate, writes 0 to base+00 and returns the prior gate state. It
  does not read random data, despite its old `prng_read` symbol name.
- `mem:0x080720a0` is the status poll and data read.
- `mem:0x08072090` restores a supplied gate state.

Lifecycle: prepare, acquire, restore. Status bits 0..2 are tested as a group;
whether they mean ready, count, overflow or fault is unknown. The acquisition
routine has no timeout. Output is 32 bits wide; throughput, startup latency
and entropy are not measured.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Initialization and prior gate state | `disk mem:0x0807206c` | observed |
| Ready poll and result | `disk mem:0x080720a0` | observed |
| Gate restoration | `disk mem:0x08072090` | observed |
