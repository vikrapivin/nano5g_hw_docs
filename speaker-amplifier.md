# Built-in speaker amplifier

The speaker has its own control device at **7-bit I2C address `0x6f`** (wire
address `0xde`). OSOS sends **single command bytes with no register prefix**. PCM
stays on codec I2S2 TX; the speaker has no PCM FIFO or DMA endpoint of its own.
The part and the codec-to-amplifier analog wiring are unconfirmed. The
diagnostics module `DxeLM48557` issues the same 03/02/01 sequence
(entry `DxeLM48557.pe+0x390`), so the name LM48557 is inferred from the module
name, not a device read. `TestAudioIpod` calls the part the "Piezo speaker amp"
("Turning on/off Piezo speaker amp", `diag TestAudioIpod.pe+0x4070..0x40de`,
string `+0x4118`); `TestI2CStress` calls the same transfer "Write Pizeo Register".
The diagnostics send each command byte twice in one transfer, `[v, v]`, length 2
(`diag DxeLM48557.pe+0x494`, `bootloader DxeSmbus.pe+0x4dc`).

## Bus and firmware state

`osos mem:0x082bdc88` reads system-descriptor word `+0x84`: model upper 16 bits
must equal `001c`; revision lower 16 below 4 selects I2C0, otherwise I2C1. This
is not a GPIO index. `mem:0x082b5204` installs enable-GPIO value `0xc8`, a
sentinel meaning **skip GPIO** (not pin 200). The diagnostics always use I2C1
(`diag DxeLM48557.pe+0x240`) and, at module load, call DevicePowerManagement
device `0x40` state 6, which drives GPIO `0xe0003` (port 14.3, pin `0x73`) high
(cfg `0xf`); state 7 drives it low (`diag DxeLM48557.pe+0x2f0`,
`DevicePowerManagement.pe+0x338,+0x2ec`, literal `+0x330`).

Descriptor at `mem:0x08676664`: state pointer at `+04`; methods at `+08` init,
`+0c` shutdown, `+10` read shutdown, `+14` mute, `+18` read mute, `+1c` volume,
`+20` read volume, `+24` raw level, `+28` read level. State is eight bytes at
`mem:0x0867665c`. These are firmware RAM layouts, not amplifier registers.

## Command protocol

| Operation | Command / sequence | Meaning |
|---|---|---|
| Initialize | `03`, wait 3 ms, `02`, wait 3 ms, `01`, `80` | `mem:0x082c0bf8`; stops on transport failure. Diagnostics: `03`, Stall 2500 us, `02`, Stall 1000 us, `01` (`diag DxeLM48557.pe+0x3bc/+0x3d8`) |
| Control | `(muted==1 ? 2 : 0) \| (shutdown==0 ? 1 : 0)` | bit 1 mute request; bit 0 operating request. Diagnostics: state byte bit 0 = enabled, bit 1 = mute; mute writes `(shadow&0xfb) \| (mute?2:0)`; disable writes `shadow&0xfa` (`00` or `02`) (`diag DxeLM48557.pe+0x390,+0x348`) |
| Muted shutdown | `02` | Common port shutdown request |
| Gain | `80 \| gain_codes[index]` | 21 settings below; gain units unknown. Diagnostics accept level `0x00..0x1f` and `0x3c..0x3f` (36 levels) and write `0x80 \| (level&0x3f)`; levels `0x20..0x3b` are rejected (`diag DxeLM48557.pe+0x458`). The OSOS table uses `00`, `10..1f`, `3c..3f` |

For a non-sentinel enable GPIO, initialization configures output low, waits 1 ms,
then drives high before the command sequence. The factory descriptor skips it.
Init is `osos mem:0x082c0bf8`; transport/error handling is in the audio reference.

| Index | Command | Firmware level value |
|---:|---:|---:|
|0|`80`|−80|
|1|`90`|10|
|2|`91`|12|
|3|`92`|14|
|4|`93`|16|
|5|`94`|18|
|6|`95`|20|
|7|`96`|22|
|8|`97`|24|
|9|`98`|26|
|10|`99`|28|
|11|`9a`|30|
|12|`9b`|32|
|13|`9c`|34|
|14|`9d`|36|
|15|`9e`|38|
|16|`9f`|40|
|17|`bc`|42|
|18|`bd`|44|
|19|`be`|46|
|20|`bf`|48|

The level column has no known unit (not shown to be dB). For volume 0..255,
index = `volume*21/256`, with nonzero volume forced to at least index 1. The audio
method at `mem:0x082b1354` limits speaker volume to 240, so ordinary calls reach
index 19 at most. Enable method `mem:0x082b6524` initializes, selects volume 160
(index 13, command `9c`), and marks the output enabled. Disable asks for shutdown 1.
Firmware caches mute/shutdown before the write and does not roll back cached gain
on failure, so a repeated request may skip the retry.

## Unknown

`TestClicker -s time` is a separate speaker-drive test: complementary 1 kHz drive on GPIO `0x10000` and `0x10001` (port 1.0/1.1), cfg `0xe` low / `0xf` high, Stall 1 us between edges and 500 us half period, `time*1000` iterations; `-h` = `0x10000` high + `0x10001` low, `-l` = opposite; `-b` buzzer through the PWM protocol (Pwm base `0x3c700000`, clock id `0x1e`) (`diag TestClicker.pe+0xc84..0xeee`).

Register-read protocol, fault/status input, maximum power, speaker impedance,
thermal behavior, analog pinout.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| Command-only I2C and revision-selected bus | `mem:0x082bdc88`, `0x082b5204` | observed |
| Init and gains | `mem:0x082c0bf8`, `0x082c0ca8` | observed |
| Diagnostics path: I2C1, port 14.3 enable, `[v,v]` writes, 2500/1000 us waits, 36-level gain | `diag DxeLM48557.pe+0x240,+0x2f0,+0x3bc,+0x458,+0x494` | observed |
| `TestClicker` GPIO 1.0/1.1 drive and PWM buzzer | `diag TestClicker.pe+0xc84..0xeee` | observed |
| Analog characteristics | none recovered | unknown |
