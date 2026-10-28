# TV output timing and analog encoder

**MMIO base `0x39300000`**, 32-bit accesses. Source: the diagnostic module
`diag Tvout.pe`; code addresses are **offsets from the start of the PE**. Normal
OSOS use and cable exposure of each mode are not established.

## Interface and lifecycle

This bank is separate from video input 39100000 and mixer 39200000.

- `diag Tvout.pe+0x0be8` chooses mode 0/1, sets the low nibble of `08` (0 or 2) and stores 640/768 in software state. It then enables all three banks and ORs `3c` with 7. The sizes do not establish NTSC/PAL timing.
- `diag Tvout.pe+0x10f4` and 0x11ec initialize timing/level/configuration registers. The former writes many `1c`..`30` values 800; the latter uses 77d, 774, 76d, 77d, 774, 77d. Purposes and units unknown. Both clear the `100`..`15c` word range, write many fixed analog/filter words and choose further parameter tables.
- `diag Tvout.pe+0x0ca0` derives `3c4`/`3cc` and `3c8`/`3d4` from software flags and a small computed check field. `14` is written with the value 440c (a register value, not a RAM pointer).
- `diag Tvout.pe+0x0808` writes `280=1`; `0814` then ORs `284` bit 0; `082c` writes `284=0`.
- `diag Tvout.pe+0x0f04` uses this handshake before clearing `3c` low 4 bits, clearing `00` bit 0 and polling `00` bit 1. It then stops the mixer and video banks in order, each with a maximum 10-iteration status loop (a software count, not a settling time).
- Start path `diag Tvout.pe+0x102c` requests clock selectors 0x06, 0x44 and 0x45 through the clock protocol (plus two other protocol calls). Stop path `diag Tvout.pe+0x1008` releases only selector 0x47. The two sets differ, and a selector is not a clock-gate bit number without the OSOS table.

## Write inventory

Every direct absolute-MMIO assignment to this bank in the diagnostic
functions, in source order, including repeated writes and branch alternatives
(see the function for conditions). `DAT` names are decompiler labels: the
address of a low constant may just be a literal; a peripheral `DAT` read is a
register read. Indirect pointer writes and unnamed functions are not included.

| Function offset | Register offset | Original write expression |
|---|---|---|
| `00000808` | `280` | `1` |
| `00000814` | `284` | `_DAT_39300284 \| 1` |
| `0000082c` | `284` | `0` |
| `00000870` | `01c` | `param_1` |
| `00000870` | `020` | `param_2` |
| `00000870` | `024` | `param_3` |
| `00000adc` | `008` | `_DAT_39300008 & 0x3f` |
| `00000adc` | `034` | `0x200` |
| `00000adc` | `034` | `0` |
| `00000adc` | `034` | `0x200` |
| `00000adc` | `034` | `0` |
| `00000adc` | `008` | `_DAT_39300008 \| 0x200` |
| `00000adc` | `034` | `0` |
| `00000adc` | `008` | `_DAT_39300008 \| 0x40040` |
| `00000adc` | `008` | `_DAT_39300008 \| uVar1` |
| `00000adc` | `01c` | `0x800` |
| `00000adc` | `020` | `0x800` |
| `00000adc` | `024` | `0x800` |
| `00000b84` | `008` | `_DAT_39300008 & 0xffffffef` |
| `00000b84` | `008` | `_DAT_39300008 \| 0x10` |
| `00000be8` | `008` | `_DAT_39300008 & 0xfffffff0` |
| `00000be8` | `008` | `_DAT_39300008 \| 2` |
| `00000be8` | `000` | `_DAT_39300000 \| 1` |
| `00000be8` | `03c` | `_DAT_3930003c \| 7` |
| `00000ca0` | `3c4` | `iVar2 << 0xe \| uVar1` |
| `00000ca0` | `3cc` | `_DAT_393003c4` |
| `00000ca0` | `3c8` | `iVar2 << 0xc \| uVar3` |
| `00000ca0` | `3d4` | `_DAT_393003c8` |
| `00000ca0` | `014` | `&DAT_0000440c` |
| `00000f04` | `03c` | `_DAT_3930003c & 0xfffffff0` |
| `00000f04` | `000` | `_DAT_39300000 & 0xfffffffe` |
| `00000f04` | `000` | `0` |
| `000010f4` | `00c` | `5` |
| `000010f4` | `00c` | `6` |
| `000010f4` | `010` | `1` |
| `000010f4` | `01c` | `0x800` |
| `000010f4` | `020` | `0x800` |
| `000010f4` | `024` | `0x800` |
| `000010f4` | `028` | `0x800` |
| `000010f4` | `02c` | `0x800` |
| `000010f4` | `030` | `0x800` |
| `000010f4` | `038` | `0` |
| `000010f4` | `03c` | `0x1000700` |
| `000010f4` | `0f0` | `0` |
| `000010f4` | `100` | `0` |
| `000010f4` | `104` | `0` |
| `000010f4` | `108` | `0` |
| `000010f4` | `10c` | `0` |
| `000010f4` | `110` | `0` |
| `000010f4` | `114` | `0` |
| `000010f4` | `118` | `0` |
| `000010f4` | `11c` | `0` |
| `000010f4` | `120` | `0` |
| `000010f4` | `124` | `0` |
| `000010f4` | `128` | `0` |
| `000010f4` | `12c` | `0` |
| `000010f4` | `130` | `0` |
| `000010f4` | `134` | `0` |
| `000010f4` | `138` | `0` |
| `000010f4` | `13c` | `0` |
| `000010f4` | `140` | `0` |
| `000010f4` | `144` | `0` |
| `000010f4` | `148` | `0` |
| `000010f4` | `14c` | `0` |
| `000010f4` | `150` | `0` |
| `000010f4` | `154` | `0` |
| `000010f4` | `158` | `0` |
| `000010f4` | `15c` | `0` |
| `000010f4` | `180` | `_DAT_39300180 & 0xffffffe0 \| 0x10` |
| `000010f4` | `184` | `0x800000` |
| `000010f4` | `188` | `0x800000` |
| `000010f4` | `18c` | `0x80` |
| `000010f4` | `190` | `0` |
| `000010f4` | `194` | `&DAT_0000eb10` |
| `000010f4` | `198` | `0x2000000` |
| `000010f4` | `19c` | `0x3ff0200` |
| `000010f4` | `1a0` | `0x1ff` |
| `000010f4` | `1a4` | `0x3ff0000` |
| `000010f4` | `1a8` | `0x1ff` |
| `000010f4` | `1c0` | `0x11` |
| `000010f4` | `260` | `0` |
| `000010f4` | `264` | `0` |
| `000010f4` | `268` | `0` |
| `000010f4` | `26c` | `&DAT_00011a00` |
| `000010f4` | `280` | `0` |
| `000010f4` | `3c0` | `0` |
| `000010f4` | `3d0` | `1` |
| `000011ec` | `00c` | `6` |
| `000011ec` | `010` | `1` |
| `000011ec` | `01c` | `0x77d` |
| `000011ec` | `020` | `0x774` |
| `000011ec` | `024` | `0x76d` |
| `000011ec` | `028` | `0x77d` |
| `000011ec` | `02c` | `0x774` |
| `000011ec` | `030` | `0x77d` |
| `000011ec` | `034` | `0` |
| `000011ec` | `038` | `0` |
| `000011ec` | `03c` | `0x1000700` |
| `000011ec` | `0f0` | `0` |
| `000011ec` | `100` | `0` |
| `000011ec` | `104` | `0` |
| `000011ec` | `108` | `0` |
| `000011ec` | `10c` | `0` |
| `000011ec` | `110` | `0` |
| `000011ec` | `114` | `0` |
| `000011ec` | `118` | `0` |
| `000011ec` | `11c` | `0` |
| `000011ec` | `120` | `0` |
| `000011ec` | `124` | `0` |
| `000011ec` | `128` | `0` |
| `000011ec` | `12c` | `0` |
| `000011ec` | `130` | `0` |
| `000011ec` | `134` | `0` |
| `000011ec` | `138` | `0` |
| `000011ec` | `13c` | `0` |
| `000011ec` | `140` | `0` |
| `000011ec` | `144` | `0` |
| `000011ec` | `148` | `0` |
| `000011ec` | `14c` | `0` |
| `000011ec` | `150` | `0` |
| `000011ec` | `154` | `0` |
| `000011ec` | `158` | `0` |
| `000011ec` | `15c` | `0` |
| `000011ec` | `180` | `_DAT_39300180 & 0xffffffe0 \| 0x10` |
| `000011ec` | `184` | `0x800000` |
| `000011ec` | `188` | `0x800000` |
| `000011ec` | `18c` | `0x80` |
| `000011ec` | `190` | `0` |
| `000011ec` | `194` | `&DAT_0000eb10` |
| `000011ec` | `198` | `0x2000000` |
| `000011ec` | `19c` | `0x3ff0200` |
| `000011ec` | `1a0` | `0x1ff` |
| `000011ec` | `1a4` | `0x3ff0000` |
| `000011ec` | `1a8` | `0x1ff` |
| `000011ec` | `1c0` | `0x11` |
| `000011ec` | `260` | `0` |
| `000011ec` | `264` | `0` |
| `000011ec` | `268` | `0` |
| `000011ec` | `26c` | `&DAT_00011a00` |
| `000011ec` | `280` | `0` |
| `000011ec` | `3c0` | `0` |
| `000011ec` | `3d0` | `0` |
| `00001498` | `044` | `0` |
| `00001498` | `048` | `0` |
| `00001498` | `04c` | `0` |
| `00001498` | `050` | `0` |
| `00001498` | `054` | `0` |
| `00001498` | `058` | `0` |
| `00001498` | `05c` | `0` |
| `00001498` | `060` | `0` |
| `00001498` | `064` | `0` |
| `00001498` | `068` | `0` |
| `00001498` | `06c` | `0` |
| `00001498` | `080` | `0` |
| `00001498` | `084` | `0` |
| `00001498` | `088` | `0` |
| `00001498` | `08c` | `0` |
| `00001498` | `090` | `0` |
| `00001498` | `0c0` | `0` |
| `00001498` | `0c4` | `0` |
| `00001498` | `0c8` | `0` |
| `00001498` | `0cc` | `0` |
| `00001498` | `0d0` | `0` |
| `00001498` | `044` | `0` |
| `00001498` | `048` | `0` |
| `00001498` | `04c` | `0` |
| `00001498` | `050` | `0` |
| `00001498` | `054` | `0` |
| `00001498` | `058` | `0` |
| `00001498` | `05c` | `0` |
| `00001498` | `060` | `0` |
| `00001498` | `064` | `0` |
| `00001498` | `068` | `0` |
| `00001498` | `06c` | `0` |
| `00001498` | `080` | `0` |
| `00001498` | `084` | `0` |
| `00001498` | `088` | `0` |
| `00001498` | `08c` | `0` |
| `00001498` | `090` | `0` |
| `00001498` | `0c0` | `0` |
| `00001498` | `0c4` | `0` |
| `00001498` | `0c8` | `0` |
| `00001498` | `0cc` | `0` |
| `00001498` | `0d0` | `0` |
| `00001498` | `070` | `0x25d` |
| `00001498` | `09c` | `0x14` |
| `00001498` | `0a0` | `0x28` |
| `00001498` | `0a4` | `0x3f` |
| `00001498` | `0a8` | `0x52` |
| `00001498` | `0ac` | `0x5a` |
| `00001498` | `0d4` | `1` |
| `00001498` | `0d8` | `9` |
| `00001498` | `0dc` | `0x1c` |
| `00001498` | `0e0` | `0x39` |
| `00001498` | `0e4` | `0x5a` |
| `00001498` | `0e8` | `0x74` |
| `00001498` | `0ec` | `0x7e` |
| `00001498` | `044` | `0` |
| `00001498` | `048` | `0` |
| `00001498` | `04c` | `0` |
| `00001498` | `050` | `0` |
| `00001498` | `054` | `0` |
| `00001498` | `058` | `0` |
| `00001498` | `05c` | `0` |
| `00001498` | `060` | `0` |
| `00001498` | `064` | `0` |
| `00001498` | `068` | `0` |
| `00001498` | `06c` | `0` |
| `00001498` | `080` | `0` |
| `00001498` | `084` | `0` |
| `00001498` | `088` | `0` |
| `00001498` | `08c` | `0` |
| `00001498` | `090` | `0` |
| `00001498` | `0c0` | `0` |
| `00001498` | `0c4` | `0` |
| `00001498` | `0c8` | `0` |
| `00001498` | `0cc` | `0` |
| `00001498` | `0d0` | `0` |
| `00001498` | `044` | `0` |
| `00001498` | `048` | `0` |
| `00001498` | `04c` | `0` |
| `00001498` | `050` | `0` |
| `00001498` | `054` | `0` |
| `00001498` | `058` | `0` |
| `00001498` | `05c` | `0` |
| `00001498` | `060` | `0` |
| `00001498` | `064` | `0` |
| `00001498` | `068` | `0` |
| `00001498` | `06c` | `0` |
| `00001498` | `080` | `0` |
| `00001498` | `084` | `0` |
| `00001498` | `088` | `0` |
| `00001498` | `08c` | `0` |
| `00001498` | `090` | `0` |
| `00001498` | `0c0` | `0` |
| `00001498` | `0c4` | `0` |
| `00001498` | `0c8` | `0` |
| `00001498` | `0cc` | `0` |
| `00001498` | `0d0` | `0` |
| `00001498` | `070` | `0x281` |
| `00001498` | `09c` | `0x15` |
| `00001498` | `0a0` | `0x2a` |
| `00001498` | `0a4` | `0x44` |
| `00001498` | `0a8` | `0x57` |
| `00001498` | `0ac` | `0x5f` |
| `00001498` | `0d4` | `2` |
| `00001498` | `0d8` | `10` |
| `00001498` | `0dc` | `0x1d` |
| `00001498` | `0e0` | `0x3c` |
| `00001498` | `0e4` | `0x5f` |
| `00001498` | `0e8` | `0x7b` |
| `00001498` | `0ec` | `0x86` |
| `00001498` | `098` | `7` |
| `00001498` | `094` | `1` |

## Unknown

Signal names/pinout, output voltage levels, format enums, clock tolerances,
reset values and register side effects. A nonzero value is not necessarily an
enable, and a zero write does not show a feature is absent.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Register writes and branch-dependent values | `diag Tvout.pe` functions | observed; not run on hardware |
| Direct-write extraction for analyzed functions | direct-write extraction of the analyzed `diag Tvout.pe` functions | observed; indirect accesses not listed |
| Pipeline relationship | [TV overview](tvout.md) | inferred (ordered enable/disable, data flow) |
