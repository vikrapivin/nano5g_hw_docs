# TV video input and scaler

**MMIO base `0x39100000`**, 32-bit accesses. Source: the diagnostic module
`diag Tvout.pe`; code addresses are **offsets from the start of the PE**. Normal
OSOS use and cable exposure of each mode are not established.

## Interface and lifecycle

This bank holds the video input's buffer pointers, source/destination
dimensions, crop/position fields and fixed-point scale factors. Field names are
inferred from arithmetic and call arguments.

- `diag Tvout.pe+0x09a8` strips bit 31 from pointers. Format branch 8 uses caller pointer 0 and pointer 2; branch 9 uses pointer 0 and pointer 1. It writes base `28`/`30` and derived addresses `2c`/`34` using the caller's stride. Planar-format names and the CPU-to-physical mapping are not known.
- `diag Tvout.pe+0x0d1c` writes output width `4c`, height through 0x1b38, positions `54`/`58`, source width `5c`, source height through 0x1b38, and ratios `64`/`68` computed as `(output_dimension << 16) / input_dimension`. Check the direction of the ratio before writing a resampler. Write `08=1` commits the video-path update.
- The height helper `diag Tvout.pe+0x1b38` writes `+0x50` or `+0x60`. When the software mode variable `DAT_00004eb4` equals 1 it halves the requested height first; otherwise it writes it unchanged. The mode's meaning is unknown.
- `diag Tvout.pe+0x12dc` initializes all pointer slots to 08000000, dimensions to 64x16, ratios to 10000 (unity in 16.16), then writes coefficient words and other controls (table below).
- Enable is `00` bit 0; shutdown clears it and polls `00` bit 1.

## Write inventory

Every direct absolute-MMIO assignment to this bank in the diagnostic
functions, in source order, including repeated writes and branch alternatives
(see the function for conditions). `DAT` names are decompiler labels: the
address of a low constant may just be a literal; a peripheral `DAT` read is a
register read. Indirect pointer writes and unnamed functions are not included.

| Function offset | Register offset | Original write expression |
|---|---|---|
| `000009a8` | `028` | `*param_2` |
| `000009a8` | `030` | `param_2[2]` |
| `000009a8` | `028` | `*param_2` |
| `000009a8` | `030` | `param_2[1]` |
| `000009a8` | `030` | `_DAT_39100030 & 0x7fffffff` |
| `000009a8` | `02c` | `uVar2 + DAT_0019a2d8 & 0x7fffffff` |
| `000009a8` | `028` | `_DAT_39100028 & 0x7fffffff` |
| `000009a8` | `034` | `uVar1 + DAT_0019a2d8 & 0x7fffffff` |
| `000009a8` | `014` | `_DAT_39100014 & 0xffff \| DAT_0019a2d8 << 0x10` |
| `000009a8` | `018` | `DAT_0019a2d8 << 0x10 \| _DAT_39100018 & 0xffff` |
| `00000a9c` | `058` | `param_3 & 0x3ff` |
| `00000a9c` | `054` | `param_2 & 0x3ff` |
| `00000be8` | `000` | `_DAT_39100000 \| 1` |
| `00000d1c` | `04c` | `param_11` |
| `00000d1c` | `044` | `param_9 << 4` |
| `00000d1c` | `054` | `param_7` |
| `00000d1c` | `058` | `param_8` |
| `00000d1c` | `048` | `param_10` |
| `00000d1c` | `05c` | `param_5` |
| `00000d1c` | `064` | `(undefined4)uVar2` |
| `00000d1c` | `068` | `(undefined4)uVar2` |
| `00000d1c` | `008` | `1` |
| `00000f04` | `000` | `_DAT_39100000 & 0xfffffffe` |
| `00000f04` | `000` | `_DAT_39100000 & 0xfffffffe` |
| `000012dc` | `010` | `0` |
| `000012dc` | `010` | `4` |
| `000012dc` | `00c` | `0` |
| `000012dc` | `028` | `0x8000000` |
| `000012dc` | `030` | `0x8000000` |
| `000012dc` | `02c` | `0x8000000` |
| `000012dc` | `034` | `0x8000000` |
| `000012dc` | `044` | `0` |
| `000012dc` | `048` | `0` |
| `000012dc` | `04c` | `0x40` |
| `000012dc` | `050` | `0x10` |
| `000012dc` | `054` | `0` |
| `000012dc` | `058` | `0` |
| `000012dc` | `05c` | `0x40` |
| `000012dc` | `060` | `0x10` |
| `000012dc` | `064` | `0x10000` |
| `000012dc` | `068` | `0x10000` |
| `000012dc` | `3cc` | `1` |
| `000012dc` | `06c` | `&DAT_00070707` |
| `000012dc` | `070` | `0x7070707` |
| `000012dc` | `074` | `0x7070707` |
| `000012dc` | `078` | `0x7000000` |
| `000012dc` | `07c` | `&DAT_00020405` |
| `000012dc` | `080` | `0x6060606` |
| `000012dc` | `084` | `0x6050504` |
| `000012dc` | `088` | `0x3020101` |
| `000012dc` | `08c` | `0x7a7470` |
| `000012dc` | `090` | `0x6e6c6b6c` |
| `000012dc` | `094` | `0x6c6e7073` |
| `000012dc` | `098` | `0x76787b7e` |
| `000012dc` | `09c` | `0x7f7e7d79` |
| `000012dc` | `0a0` | `0x726b6359` |
| `000012dc` | `0a4` | `0x4f44392e` |
| `000012dc` | `0a8` | `0x23191008` |
| `000012dc` | `0ec` | `0x3d3a38` |
| `000012dc` | `0f0` | `0x38383839` |
| `000012dc` | `0f4` | `0x3a3b3c3d` |
| `000012dc` | `0f8` | `0x3e3f3f00` |
| `000012dc` | `0fc` | `0x7f7e7c76` |
| `000012dc` | `100` | `0x6f665c51` |
| `000012dc` | `104` | `0x463b3025` |
| `000012dc` | `108` | `0x1b130b05` |
| `000012dc` | `10c` | `&DAT_00050b13` |
| `000012dc` | `110` | `0x1b25303b` |
| `000012dc` | `114` | `0x46515c66` |
| `000012dc` | `118` | `0x6f767c7e` |
| `000012dc` | `11c` | `&DAT_00003f3f` |
| `000012dc` | `120` | `0x3e3d3c3b` |
| `000012dc` | `124` | `0x3a393838` |
| `000012dc` | `128` | `0x38383a3d` |
| `000012dc` | `12c` | `0x6b6b6d6f` |
| `000012dc` | `130` | `0x3336393d` |
| `000012dc` | `134` | `0x3f010203` |
| `000012dc` | `138` | `0x3030202` |
| `000012dc` | `13c` | `0xaaa69f95` |
| `000012dc` | `140` | `0x88786754` |
| `000012dc` | `144` | `0x412e1e0f` |
| `000012dc` | `148` | `0x279726d` |
| `000012dc` | `200` | `1` |
| `000012dc` | `20c` | `0` |
| `000012dc` | `210` | `0` |
| `000012dc` | `218` | `0x80` |
| `000012dc` | `21c` | `0x80000080` |
| `000012dc` | `220` | `0x80` |
| `000012dc` | `224` | `0x80` |
| `000012dc` | `228` | `0x80` |
| `000012dc` | `22c` | `0x80` |
| `000012dc` | `230` | `0x80` |
| `000012dc` | `234` | `0x80` |
| `000012dc` | `238` | `0` |
| `00001b38` | `050` | `param_1` |
| `00001b38` | `060` | `param_1` |

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
