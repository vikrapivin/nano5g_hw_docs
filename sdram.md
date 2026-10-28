# SDRAM controller and memory-pad interface

The N33 PreEfi initializes a 64-MiB SDRAM window at `0x08000000` through the
controller at `0x3d700000`. Ordered programming reference reconstructed from
`bootloader mem:0x2200b4b4`. See [CPU/address spaces](cpu-memory.md) for
mapping and cache requirements and [clocks](clocks-reset.md) for the clock
state it runs in.

The final delay is a loop of 1000 iterations (counter seeded with 999, `bcs`
loops while no borrow). Each iteration does a dummy read of `+0x108`
(`ldr r1,[r2,#8]` at `mem:0x2200b5a2`, r2 = `0x3d700100`); the decompiler
hides the unused volatile load.

## Registers

`B = 0x3d700000`, 32-bit accesses. Names are behavioral labels.

| Offset | Writes / tests | Meaning |
|---|---|---|
| `+0x000` | `1` early | controller setup; bits unknown |
| `+0x008` | `1` last | final controller activation (inferred from order) |
| `+0x100` | `0x1030`, later OR `0x09100000` | memory configuration / final operating mode; bits not decoded |
| `+0x104` | cold sequence below; warm `0x11`; final `0x33` | command issue; wait while `&0x00110000 != 0` |
| `+0x108` | `0x0002000b`, then `0x0002100b` | refresh/control; bit `0x1000` set after setup; period encoding unknown |
| `+0x10c` | `0x40` | unknown |
| `+0x110` | `0x33`, `0x8040` | data for the mode/extended-mode command (inferred) |
| `+0x114` | `0x008aae21` | timing word A; fields unknown |
| `+0x118` | `8` | unknown |
| `+0x11c` | `0xff`, then `0x19` | unknown |
| `+0x120` | `0xff`, then `1` | unknown |
| `+0x124` | `0x050e6b46` | timing word B; fields unknown |
| `+0x140` | `0x0003b3b2`, `0x0003b3b3`, then the formula below | DLL/control sequence; untested bits unknown |
| `+0x144` | poll `+140` bit 1 first, then this word `&3 == 3`; read bits 25:18 | DLL status / lock tap (inferred); not a measured margin |
| `+0x148` | `4` | unknown |
| `+0x14c` | `0` | unknown |
| `+0x150` | `0x10` | unknown |

Final `+0x140` value (32-bit arithmetic):

```text
(((read(B + 0x144) >> 18) & 0xff) << 20) + 0xff13b3b0
```

This adjusts the locked value by 15 in the encoded field. The lock sequence
supports calling it a DLL adjustment; the duration of one tap is unknown.

Cold path, writes to `+104` in order (`WAIT` = poll `(+104 & 0x110000) == 0`):

```text
0x33, 0x233, WAIT, 0x33, 0x33, 0x33,
0x333, WAIT, 0x33, 0x33, 0x33,
0x333, WAIT, 0x33, 0x33, 0x33,
(+110 = 0x33), 0x133, WAIT, 0x33, 0x33, 0x33,
(+110 = 0x8040), 0x133, WAIT, 0x33, 0x33
```

From the sequence, `0x233`/`0x333`/`0x133` are precharge-all, refresh and
mode-register operations (command pins not captured). The warm path (PreEfi
mode `0x11`, selected by saved PMU state) writes `+104=0x11` instead and
preserves memory for hibernate resume. Both paths then:

1. write `+104=0x33`
2. `+10c=0x40`
3. `+108=0x2100b`
4. OR `+100` with `0x09100000`
5. `+11c=0x19`
6. `+120=1`
7. run the 1000-iteration loop reading `+0x108` once per pass (counter 999 down to 0)
8. `+008=1`

## IRAM re-initialization (diagnostics)

The diagnostics `IramRoutines` module (`diag IramRoutines.pe+0x2e0..0x42e`, literals `+0x430..0x460`)
re-initializes the controller from IRAM with different literals from PreEfi:

| Offset | PreEfi | IRAM routine |
|---|---|---|
| `+0x000` | `1` | `2` |
| `+0x100` | `0x1030` | `0x830` |
| `+0x108` | `0x0002000b`, then `0x0002100b` | `0x2002e`, then `0x2102e` |
| `+0x10c` | `0x40` | `0x10` |
| `+0x114` | `0x008aae21` | `0x008aac25` |
| `+0x124` | `0x050e6b46` | `0x050d6828` |
| `+0x14c` | `0` | `0x2780` |
| final `+0x140` constant | `0xff13b3b0` | `0xff53b3b0` |

Final `+0x140` there is `(((+0x144 >> 18) & 0xff) << 20) + 0xff53b3b0`. These writes match PreEfi:
`0x3cf00384 = 0x555`; `+0x11c` / `+0x120 = 0xff` then `0x19` / `1`; `+0x118 = 8`; `+0x148 = 4`; `+0x150 = 0x10`; `+0x008 = 1`; `0x3e000008 = 0x1f`.
Full order for the cold mode: `0x3cf00384 = 0x555`; `+00 = 2`; `+100 = 0x830`; `+11c = ff`, `+120 = ff`; `+114`; `+124`; `+118 = 8`;
`+108 = 0x2002e`; `+148 = 4`; `+14c = 0x2780`; `+140 = 0x3b3b2`, poll bit 1, `0x3b3b3`, poll `+144 & 3 == 3`, then the final value;
`+150 = 0x10`; the command list; `+10c = 0x10`; `+100 |= 0x09100000`; `+11c = 0x19`; `+120 = 1`; `+108 = 0x2102e`; `+008 = 1`; `0x3e000008 = 0x1f`.

Modes of the routine:

| Mode | Operation |
|---|---|
| 2 | Cold init: the command sequence below |
| 4 | Warm: `+0x104 = 0x11`, then `0x33` |
| 1 | Low-power entry: `+0x100` bit 20 cleared, field `[27:24]` = 2, `\| 0x08100000`, `+0x11c = 1`, `+0x120 = 1`, wait `+0x104 & 0x440000 == 0x440000` |
| 3 | Exit: bit 20 cleared, field `[27:24]` = 1, `\| 0x08100000`, `+0x11c = 0x19`, `+0x120 = 1` |

Cold command order: `0x33, 0x233, WAIT, 0x33 x3, 0x333, WAIT, 0x33 x3, 0x333, WAIT, 0x33 x3, (+110 = 0x33) 0x133, WAIT, 0x33 x3,
(+110 = 0x8040) 0x133, WAIT, 0x33 x3` (WAIT = `+0x104 & 0x110000 == 0`, as in PreEfi; one more `0x33` per group).

`TestRam` (`diag TestRam.pe+0xc34..0xd90`, `+0xfb8..0x1024`, `+0x10b4..0x1294`) scans a 16-MiB (`0x01000000`) EFI pool buffer, not fixed
SDRAM addresses: `-f` full (pool type 4, 16-bit walking patterns, two passes x 16 patterns x 2-KiB blocks, plus `0x5678` / `0xffff` passes);
`-q` quick (pool type 6, address-line test writing `1<<i` at word `(1<<i) ^ 0xffff` and the complement at word `0x10000 + (1<<i)`, i = 0..15).
It has no alias or aperture test.

## Other bus writes

| Address | Operation | Evidence |
|---|---|---|
| `0x3cf00380` | `0x00155195` | PreEfi GPIO init `0x2200b320`; likely pad setup, bits unknown |
| `0x3cf00384` | `0x00000555` | GPIO init and SDRAM init; likely memory-pad setup |
| `0x3e000008` | `0x1f` after DRAM starts | `0x2200b4b4`; function unknown |
| `0x3ff00104` | `0x0c150000` | PreEfi bus helper `0x2200b46c`, repeated by restore OS startup |
| `0x3ff0077c` | `0xff`, poll low byte zero | same helper; command completion (inferred) |
| `0x3ff00220` | `0x1ff` | same helper; unknown |
| `0x3ff00100` | `0`, poll bit 0 zero | bus helper and SDRAM helper; unknown |
| `0x3810008c` | `1` | same; unknown bus-routing/control |
| `0x3d500400` | `7` | end of SDRAM helper; unknown |

## Unknown

The firmware returns 64 MiB unconditionally. DRAM vendor/part, data-bus width,
timing-field units, maximum rates, reset values, DLL tap duration and
retention/stability margins are not known. Warm mode `0x11` is the firmware
path meant to retain SDRAM; self-refresh current and retention time are not
measured. Setup literals with unknown meaning are kept as written.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Cold/warm setup offsets and values | `bootloader mem:0x2200b4b4` | observed (code and literal pools) |
| DLL polls and lock adjustment | same function, `+140/+144` | observed; tap meaning inferred |
| End-of-init delay: 1000 passes, one dummy read of `+0x108` each | instructions `mem:0x2200b5a2..0x2200b5a6`, counter `0x333+0xb4=999` | observed; the decompiler drops the volatile read |
| Shared bus initialization | `bootloader mem:0x2200b46c`, restore startup `0x2200043c` | observed; bus/QoS fields unknown |
| 64-MiB configured range | return values of `0x2200b4b4` | observed; no capacity discovery |
| IRAM re-init literals and modes 1..4 | `diag IramRoutines.pe+0x2e0..0x42e` | observed |
| `TestRam` buffer scan, no alias test | `diag TestRam.pe+0xc34..0x1294` | observed (bounded to this module) |
