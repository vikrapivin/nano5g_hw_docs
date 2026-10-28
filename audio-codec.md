# Audio codec and analog input routing

The codec has a Cirrus CS42L55/58-compatible register interface on
**I2C0, 7-bit address `0x4a`** (wire write address `0x94`); the exact part number
is not confirmed.
The speaker amplifier and jack controller have their own pages.

## Interface, supplies and clocks

OSOS `mem:0x082c2f66` writes `[register,value]`. `0x081eef52` writes a stereo
pair as `[0x80|register,value,value]`. `0x082c2e48` reads by writing the register
index, then a separate one-byte read. Reads and writes serialize I2C0. The diagnostics driver (`DxeCS42L58`, I2C0
`0x4a`) uses the same three forms: `[reg,val]`, stereo `[0x80|reg,val,val]`, and
read = write index then read (`diag DxeCS42L58.pe+0x199c,+0x1964,+0x161c`). It
sets reg 07 bit 0 before and clears it after every level/route write
(`+0x668,+0x6ae,+0x820`), where OSOS asserts reg 07 bit 4 during the ramp.

GPIO pin `0x74` is port 14 bit 4, active-low reset: low, wait 1 ms, high, wait
1 ms (`mem:0x082c1d64`). The diagnostics driver pulses `0xe0004` (port 14.4)
with cfg `0xe` (low), Stall 10 us, cfg `0xf` (high), Stall 10 us, then writes reg 04 = 0
(`diag DxeCS42L58.pe+0x166c,+0x16b0`). PMU reg 21 receives `(mV-1200)/100` (6 for 1800 mV);
reg 11 bit 2 enables the rail (`mem:0x082cc4b8`). The codec routine assumes its
cached rail value 1800 is already applied.

MCLK is CLKCON4 `0x3c500010`: low 15 bits are replaced with 3 for source 0/div 4;
bit 15 is the off gate. Configuration waits 100 µs. Source frequency unknown.
Codec reg 04 bit 5 sets serial-clock master mode.

## Register map exercised by OSOS

Names are those of the compatible register map.

| Byte register | Use / writes |
|---|---|
| `01` | Version register; OSOS does not check it. The diagnostics `id` command prints it as "Chip Fab_ID" with no expected value (`diag TestAudioIpod.pe+0x17cc..0x17de`, string `+0x1804`) |
| `02` | Power: bit 0 set during shutdown, cleared during power-up; route mask `ee` |
| `03` | Output power/shadow; initialization `af`; route changes write the full byte |
| `04` | Clock: set `20`, clear `0e` and `c0`; bit 0 MCLK disable |
| `05` | Rate low 5 bits from [I2S table](audio-i2s.md); set upper bits `c0` |
| `07` | Initialization mask `1c`, value `0c`; bit 4 asserted during ramp |
| `08` | Route low 4 bits 0, or `f` for route bit 7 |
| `09` | Input/HPF route masks `af`, `a3`, `ac`, or `a0` |
| `0a` | ADC route mask `30`, value 0 or `30` |
| `0b`, `0c` | PGA input values `3d/3d`, `5f/20`, `20/5f`, `20/20`; alternate masks below |
| `0b`, `0c` (diagnostics) | Masks `3f`, and bit 6 (`0x40`) as input select (`diag DxeCS42L58.pe+0xa24`) |
| `0f` | Playback/mix bit 7, ordinary setup `80` |
| `10`, `11` | Analog mixer bit 7, ordinary mute `80` |
| `1a`, `1b` | Headphone level: low 7 bits signed level, bit 7 mute |
| `1c`, `1d` | Line-output level: same representation |
| `0d`, `0e` | Diagnostics only: 8-bit levels, clamp -96..0 (`diag DxeCS42L58.pe+0xb08,+0xbd8`) |
| `12`, `13` | Diagnostics only: bit 7 mute (`+0xde8`) |
| `18`, `19` | Diagnostics only: 8-bit levels, no clamp (`+0xc1a`) |
| `20` | Diagnostics only: four 2-bit fields, masks `03/0c/30/c0`, selected per input bit (`+0xf04,+0xfbc`) |
| `05` (diagnostics) | Mask `c0` = `c0` when argument 2, else 0 (`+0x1442`) |

Mute bit 7 in the diagnostics covers `0f`, `10`, `11` (mixer), `1a`..`1d`, `12`, `13`
(`diag DxeCS42L58.pe+0x6ae`). Other registers are not covered.

## Initialization and shutdown

`mem:0x082bb198`, mode 1: configure MCLK; hold reset low; enable MCLK; reset
pulse; set shadow `af`; update `(reg,mask,value)` in this order:
`(03,ff,af) (04,20,20) (04,0e,00) (04,c0,00) (07,1c,0c) (0f,80,80)`
(`mem:0x082bb210..0x082bb26a`; the `(05,c0,c0)` write at `mem:0x082bb204`
belongs to mode 4 only).

Mode 4: clear 04 bit 0; paired writes 1a and 1c = `c4` (muted −60); set 07 bit 4;
clear 02 bit 0; paired writes 1a and 1c = `44`; delay 240 ms; clear 07 bit 4; set
05 bits `c0`. Mode 2/3 mutes 1a–1d, sets 02 bit 0 and 04 bit 0, waits 1 ms and holds
GPIO 74 low. This path does not switch off MCLK or the PMU rail.

## Input/output route parameters

The route implementation matches original instructions at `mem:0x082c2460` for
all 4096 combinations. The route mask is a software argument, not a register offset.

| Route bit | Effect |
|---|---|
| 0 | Enable ADC/input path (not the PCM playback enable) |
| 1,2 | Select output power values in reg 03 when bit 9 clear |
| 3 | PGA variant `3d/3d` |
| 6 | Include PGA bit 7 |
| 7 | PGA mask `c0`, reg 08 low nibble `f` |
| 8 | Suppress mixer mute bit 7 on 10/11/0f |
| 9 | Retain reg 03 shadow |
| 10 | PGA `5f/20`, reg 09 mask `a3`/value `21`, different reg 02 power bits |
| 11 | PGA `20/5f`, reg 09 mask `ac`/value `84`, reg 0a value `30` |

The reg 02 value starts with `a0` when bit 10 is set, else `40` for bit 11, else
`e0`. With input bit 0 clear, OR `0e`; with it set, bit 10 ORs `04` and bit 11
ORs `02`. Reg 02 update mask is always `ee`. Write order: reg 09 before 02, then
03, 0b, 0c, 0a, 08, 10, 11, 0f. For bit 9 clear, reg 03 shadow becomes
`((flags&2)?a0:f0)|((flags&4)?0a:0f)`; flags 0 forces `ff`. For bit 9 set, an
unset shadow `ff` first becomes `af`. PGA mask is `c0` with bit 7 set, otherwise
`ff`. Bit 6 ORs `80` into both PGA values. A route with any of bits 3/10/11 calls
the 400-unit settle service after the register writes.

Output-power method `mem:0x082bb518` takes selector 1 for reg 02 mask `ee`
(on value 0, off `0e`), selector 2 for reg 03 high nibble (on `a0`, off `f0`), or
selector 4 for its low nibble (on `0a`, off `0f`). It updates the shadow before
writing; unsupported selectors still issue zero-mask updates. Monitor method
`mem:0x082bb45a` disables via route 6 then PGA masks `3f` values 0, or enables by
writing PGA values 8 under masks `3f` then selecting route `10f`.

Ordinary input source 0 selects `209`, source 1 `601`, source 2 `a01`; source 3
uses radio I2S1. Playback worker `mem:0x082b1296` selects `12`. Microphone bias wiring and the source-number labels are not fully
reconstructed. Input disable selects `200` after buffer retirement. Source 1 also
invokes accessory setup `mem:0x0819f354`. The 400-unit delayed operation tied to
input changes is not fully interpreted.

Headphone volume is `clamp(-67 + (v*70>>8) + trim,-60,12) & 0x7f`; line volume is
`-60+(v*61>>8)+trim`. The diagnostics formulas and the reg 05 rate table are
identical (`diag DxeCS42L58.pe+0xc5c,+0xa8c,+0xb5c,+0x18c4`). `v` is 0..255; trim is in firmware level units (dB-sized
steps). Mute is separate bit 7.

## Unknown

Die identification, analog pinout, input bias, physical level calibration, and
unused codec registers. `TestAudioIpod` routing tables (`_RouteInputsToOutputs`) not decoded. Radio/headphone/speaker policies are software choices.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Bus, reset, MCLK, rail and register sequence | OSOS addresses above | observed |
| Diagnostics driver: reset 10 us pulse on `0xe0004` then reg 04 = 0, reg 07 bit 0 bracket, extra registers 0d/0e, 12/13, 18/19, 20 | `diag DxeCS42L58.pe+0x166c,+0x820,+0xa24,+0xde8` | observed |
