# TV mixer and graphics planes

**MMIO base `0x39200000`**, 32-bit accesses. Source: the diagnostic module
`diag Tvout.pe`; code addresses are **offsets from the start of the PE**. Normal
OSOS use and cable exposure of each mode are not established.

## Interface and lifecycle

- Two graphics-plane banks begin at `20` and `40`. Per plane: masked buffer pointer at `24`/`44`, stride at `28`/`48`, dimensions at `30`/`50`. Position is written by the helper at `diag Tvout.pe+0x0a9c`, which packs the caller coordinates into the plane's `34`/`54` word.
- Plane mode bits 8..11 map caller enums 0,1,4,5,6,7 to codes 2,3,4,5,6,7. Pixel-format names not known.
- `diag Tvout.pe+0x07bc` uses global `04` bits 4/5 for the two graphics planes and bit 3 for the video plane.
- Helpers at `08e8` and `092c` change low 8-bit controls and higher blend/control fields; alpha/blend meaning is provisional.
- `diag Tvout.pe+0x083c` writes `60`..`6c` from its six arguments, combining low 11 bits and flags `10000`/`e0000` in `60`. Color space and key-comparison meaning not known.
- Global `00` bit 0 enables; shutdown polls bit 1.

## Write inventory

Every direct absolute-MMIO assignment to this bank in the diagnostic
functions, in source order, including repeated writes and branch alternatives
(see the function for conditions). `DAT` names are decompiler labels: the
address of a low constant may just be a literal; a peripheral `DAT` read is a
register read. Indirect pointer writes and unnamed functions are not included.

| Function offset | Register offset | Original write expression |
|---|---|---|
| `000007bc` | `004` | `_DAT_39200004 \| 8` |
| `000007bc` | `004` | `_DAT_39200004 & 0xfffffff7` |
| `000007bc` | `004` | `_DAT_39200004 & ~uVar1` |
| `000007bc` | `004` | `_DAT_39200004 \| uVar1` |
| `0000083c` | `060` | `param_3 & 0x7ff \| uVar1` |
| `0000083c` | `064` | `param_4` |
| `0000083c` | `068` | `param_5` |
| `0000083c` | `06c` | `param_6` |
| `000008e8` | `020` | `_DAT_39200020 & 0xffffff00 \| param_2` |
| `000008e8` | `040` | `_DAT_39200040 & 0xffffff00 \| param_2` |
| `000008e8` | `014` | `_DAT_39200014 & 0xffffff00 \| param_2` |
| `0000092c` | `014` | `0` |
| `0000092c` | `014` | `_DAT_39200014 \| 0x10000` |
| `000009a8` | `024` | `*param_2 & 0x7fffffff` |
| `000009a8` | `044` | `*param_2 & 0x7fffffff` |
| `00000a54` | `028` | `param_4` |
| `00000a54` | `030` | `uVar1` |
| `00000a54` | `048` | `param_4` |
| `00000a54` | `050` | `uVar1` |
| `00000a9c` | `054` | `uVar2` |
| `00000a9c` | `034` | `uVar1` |
| `00000b84` | `004` | `_DAT_39200004 & 0xfffffffb` |
| `00000b84` | `004` | `_DAT_39200004 \| 4` |
| `00000be8` | `004` | `_DAT_39200004 \| 2` |
| `00000be8` | `000` | `_DAT_39200000 \| 1` |
| `00000c74` | `010` | `param_1` |
| `00000f04` | `000` | `_DAT_39200000 & 0xfffffffe` |
| `00000f04` | `000` | `_DAT_39200000 & 0xfffffffe` |
| `00001098` | `004` | `0` |
| `00001098` | `014` | `0` |
| `00001098` | `020` | `0` |
| `00001098` | `040` | `0` |
| `00001098` | `024` | `0` |
| `00001098` | `034` | `0` |
| `00001098` | `030` | `0` |
| `00001098` | `044` | `0` |
| `00001098` | `054` | `0` |
| `00001098` | `050` | `0` |
| `00001098` | `038` | `0x80808080` |
| `00001098` | `058` | `0x80808080` |
| `00001098` | `064` | `&DAT_00108080` |
| `00001098` | `068` | `&DAT_00108080` |
| `00001098` | `06c` | `&DAT_00108080` |
| `00001098` | `008` | `0` |
| `00001098` | `080` | `0x8440832` |
| `00001098` | `084` | `0x3b4dace1` |
| `00001098` | `088` | `0xe1d13dc` |
| `00001098` | `010` | `0x123` |

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
