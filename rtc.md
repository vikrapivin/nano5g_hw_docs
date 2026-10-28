# PMU real-time clock and alarm

The RTC is a set of [PMU](pmu.md) byte registers on I2C0/address 0x73.
Functions: `disk mem:0x080712ee` (read) and `mem:0x080713f0` (write).

## Register and structure mapping

Read mode 0 reads 6 bytes from 50, clears bit 6 in the first byte, and copies
the bytes to structure offsets 0,1,2,4,5,6 (offset 3 is not filled). Nonzero
mode reads 5 bytes from 56 into offsets 1,2,4,5,6 and sets structure byte 0 to zero.

| PMU time reg | PMU alarm reg | Caller structure byte | Treatment |
|---|---|---|---|
| `50` | — | `+0` | Bit 6 masked out on read; OR 40 on write |
| `51` | `56` | `+1` | Byte copied |
| `52` | `57` | `+2` | Byte copied |
| — | — | `+3` | Not transferred |
| `53` | `58` | `+4` | Byte copied |
| `54` | `59` | `+5` | Byte copied |
| `55` | `5a` | `+6` | Alarm write ORs 40 into this byte |

The time write is **six separate one-byte writes**, 50 through 55 in order; the
alarm write is **five**, 56 through 5a. The alarm transfer ends at 5a. The
record looks like a calendar, but BCD/binary, day-of-week, epoch and
month/year ranges are not traced.

## Bootloader RTC protocol

GUID `e9a6aa07`, provider `bootloader DxeD1759.pe` (`+0x1488..0x14c8`, `+0x15cc..0x1614`, `+0x1418`).
It uses a different layout from the disk functions above.

| Operation | Behavior |
|---|---|
| GetTime | One 12-byte read of regs `50..5b`: second = `50 & bf`, minute `51`, hour `52`, day `53`, month `54`, year = `55` + 2000 |
| SetTime | Reads the 12 bytes, replaces `50..55` (year - 2000), ORs `40` into reg **`56`**, writes all 12 back in 8-byte chunks (`+0x15f6..0x160c`) |

So the bootloader masks bit 6 of `50` on read only and ORs `40` into `56`; the disk
write above ORs `40` into the first byte (reg `50`, `mem:0x080713f0`) for the time and into reg `5a` for the alarm.

## Unknown

Retention across rail loss, oscillator source and frequency, read atomicity
across rollover, the alarm IRQ bit, and the meaning of the two bit 6 uses.
Suspend/wakeup integration is not traced.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Byte offsets, read spans, write order | Disk functions above | observed (decompilation); no device RTC test |
| Two bit 6 placements | `mem:0x080713f0` | observed; meaning unknown |
| Structure offset 3 unused | `mem:0x080712ee` | observed (this helper only) |
| Bootloader GetTime/SetTime: 12-byte block `50..5b`, `40` OR-ed into `56` | `bootloader DxeD1759.pe+0x1488..0x1614` | observed |
