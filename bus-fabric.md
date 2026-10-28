# Bus-fabric and memory-system control

A few 32-bit MMIO words outside the SDRAM controller are written during early
boot. Their values and order are known; what the blocks are (bus matrix,
arbiter, memory interface unit) is not, so they are listed as raw operations.
They are separate from the [SDRAM controller](sdram.md).

## Startup sequence (observed)

`bootloader mem:0x2200b46c` (PreEfi), in this order:

| Step | Address | Operation | Meaning |
|---|---|---|---|
| 1 | `3ff00104` | write `0c150000` | unknown |
| 2 | `3ff0077c` | write `000000ff` | command |
| 3 | `3ff0077c` | poll until the low 8 bits read 0 | completion |
| 4 | `3ff00220` | write `000001ff` | unknown enable mask |
| 5 | `3ff00100` | write 0 | command |
| 6 | `3ff00100` | poll until bit 0 reads 0 | completion |
| 7 | `3810008c` | write 1 | unknown |

The polls have no timeout.

The disk image repeats the same sequence at `disk mem:0x2200043c` before its
normal driver initialization.

Related single writes elsewhere in the boot chain:

| Address | Value | Where | Meaning |
|---|---|---|---|
| `38501000` | 2 | PreEfi clock setup | unknown; not a DDR timing value |
| `3e000008` | `1f` | SDRAM initialization ([sdram.md](sdram.md)) | unknown |
| `3d500400` | 7 | SDRAM initialization ([sdram.md](sdram.md)) | unknown |

## Unknown

Register layout of each block, arbitration priorities, memory protection,
cache interaction, idle acknowledgements and reset prerequisites.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Seven-step startup sequence | `bootloader mem:0x2200b46c` | observed |
| Disk image repeats it | `disk mem:0x2200043c` | observed |
| `38501000` = 2, `3e000008` = `1f`, `3d500400` = 7 | PreEfi clock setup and SDRAM initialization | observed, meaning unknown |
