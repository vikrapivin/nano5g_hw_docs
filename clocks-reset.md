# Clock generation, gates and software reset

The mask ROM, PreEfi, restore OS and OSOS use different gate numbering and
different clock settings. A gate number only means something within its
stage's namespace.

Evidence: the PreEfi code (`bootloader mem:0x2200b17c`), the boot ROM, and the restore
image's gate table at `disk mem:0x0807d634`. A byte comparison against the
original images reproduces the table and constants;
it checks transcription only.

## Register map

Base `C=0x3c500000`, 32-bit accesses.

| Offset | Name | Fields |
|---|---|---|
| `+0x00` | CLKCON0 | low nibble clock divider/state; bits 13:12 source select (`1` = PLL0 in this path). Bits 15 and 30 appear in steady-state values; meaning unknown |
| `+0x04` | CLKCON1 | polled fields 21:17, 13:9, 5:1. Divider fields and bits 22/14/6 handled by `bootloader ClockAndReset.pe+0xe24`; domain naming not decoded |
| `+0x08/0c/10/14` | CLKCON2..5 | peripheral clock select/divide; boot literals below; consumers such as LCD/audio configure further fields. The diag camera driver writes `+0x14` as a whole word for the sensor clock: `0x0003ffff` (7739) or `0x00008000` (7692), then waits 100 ms (`diag IspDriver.pe+0x24b8`) |
| `+0x20/24/28` | PLL0/1/2 parameters (the diag Clock protocol also exposes a PLL3 id, see Diagnostics clock service) | PLL0 `P` bits 29:24, `M` bits 15:8 (8 bits), `S` bits 2:0 per the firmware decoder (`ClockAndReset.pe+0xcda..0xce8`). Only combinations in use are qualified |
| `+0x30/34/38` | PLL lock counts | PLL0 gets `0x1c20` in ROM, `0x28488` in PreEfi; time unit unknown |
| `+0x3c` | PLL1/2 source config | PreEfi writes zero; `clk_source_hz` reads it; fields not recovered |
| `+0x40` | PLL lock status | PLL0 bit 0 polled set; other lock bits exist, transitions not measured |
| `+0x44` | PLL mode | ordered values below; bit 0 enable and bit 16 select/bypass inferred from sequence |
| `+0x48/4c/58/68/6c` | five gate words | set bit disables a clock, cleared bit enables it; several bits can belong to one logical gate |
| `+0x50` | software reset | `0xa5` written by restore `0x08070c04` and the SecCore reset stub |
| `+0x64` | clock-test output select | Monitor 0 value `0x20000 \| bit`, monitor 1 `0x10000 \| bit`; source bits below (`diag TestMonitor.pe+0xd38..0xe48`, `+0xe46..0xe48`) |
| `+0x70` | unnamed clock/config word | PreEfi writes `0x8000`; bit meaning unknown |

PLL formula (`bootloader ClockAndReset.pe+0x5f6`, called by `+0xca8`): 24,000,000-Hz
input, `M * (fin / (P << S))` in integer arithmetic. Rates below are nominal,
not measured. PreEfi does not read the speed-grade bit before choosing its
constants; `bootloader ClockAndReset.pe+0x92c` has a separate path that halves
P when chip word `0x3d100008` bit 0 is clear.

## Reset and bring-up sequences

ROM `mem:0x200000a4`:

1. Mask both VICs; clear side acknowledgements; write watchdog `0xa5`.
2. Gate words `+48/+4c/+58/+68/+6c` =
   `001f1fe7 ffabafff 000001f5 0003d7ff 06ffffff`.
3. `+44=0`, `+00=0`, `+20=0x06006c01`, `+30=0x1c20`.
4. `+44=1`; wait `+40 & 1`; `+44=0x10001`.
5. `+08=0x80008000`, `+0c=0x80008000`, `+10=0x8000`, `+14=0x8000`.
6. `+04=0x00404240`, `+00=0x40001000`.

Resulting firmware frequency variables: 216 MHz / 108 MHz / 54 MHz
(FCLK/HCLK/PCLK). The diag Clock protocol names clock id `0x11` `sclk` and id `0x12` `fclk`
(`fclk_cpu` / `fclk_dsp` are ids 0 / 1); in the bootloader provider HCLK / PCLK / ACLK divide the id `0x11` value
and id `0x12` equals id `0x11`, divided by a CLKCON1 field when bit 30 is set (`diag TestClock.pe+0x10dc`,
`bootloader ClockAndReset.pe+0xe24` cases 15 / 16). They also feed software-computed timer/SPI dividers.

PreEfi `mem:0x2200b17c`:

1. Gate words `+48/+4c/+58/+68/+6c` =
   `001fa7e5 fc2ae96d 000001f3 000380ff 06dcf779`.
2. `+00=0x8001`; wait low nibble `1`.
3. `+04=0`; wait fields 21:17 and 13:9 zero.
4. `+44=7`, then `0`; `+3c=0`.
5. `+20=0x0900ce01` (P=9, M=206, S=1), `+30=0x28488`.
6. `+44=1`, then `0x10001`; wait `+40 & 1`.
7. `+04=0x00404040`; wait fields 21:17, 13:9, 5:1 zero.
8. `+00=0x40009000`; wait low nibble zero.
9. `+08=0x80008000`, `+0c=0xc000c000`, `+10=0xc000`,
   `+14=0x80008000`, `+70=0x8000`.
10. Write `0x38501000=2` (meaning unknown).

Nominal result: FCLK about 274.67 MHz, HCLK/PCLK about 137.33 MHz. Low-battery
PreEfi `0x2200afbc` can instead select `CLKCON0=0x40001003`, `CLKCON1=0` and
PMU `0x14=0x0d`, a coupled clock/power transition. The PLL0 table in
`059_ClockAndReset.pe` at module offset `0x1164` holds `(P,M,S)` choices
`(6,133,0)`, `(6,108,0)`, `(6,100,0)`, `(6,100,2)`, `(6,133,3)`, `(8,121,4)`.
Safe sustained limits are not known.

## Gate namespaces

The bootloader's own table (`ClockAndReset.pe+0x12fc`, 75 entries of the same
six-word format, used by its gate query below ID 0x4b) equals the restore
table for IDs 0..56, inserts an extra entry 57 (`+48` mask `0x4000`, `+68`
mask `0x800`), and carries restore entries 57..73 as IDs 58..74 with
identical masks. Restore IDs 57 and above must not be used with the
bootloader protocol.

ROM `0x2000169c/0x200018b8` decodes gate indices as:

```text
n < 0x15  : +48 bit n
n < 0x35  : +4c bit (n-0x15)
n < 0x3c  : +58 bit (n-0x35)
n < 0x48  : +68 bit (n-0x3c)
otherwise : +6c bit (n-0x48)
```

It also changes paired bits: ROM gate `0x0e` changes `+68 bit11`;
`0x1a,0x2c,0x2d,0x2e,0x2f,0x30,0x31,0x1e,0x32,0x33,0x34,0x19,0x1b,0x17,0x20,0x24`
change `+6c bits0..15` respectively; gates `0x3e,0x36,0x39,0x37,0x3a,0x3b`
change `+6c bits18..23` respectively. This is visible in the original helper;
`gate=word*32+bit` does not hold.

## Restore gate table

Restore OS `0x0802c5b4` uses an ID/mask table. The 74 rows below were read
from the original image bytes: `disk mem:0x0807d634`,
stride `0x18`, six little-endian words `{id,mask48,mask4c,mask58,mask68,mask6c}`.
A zero mask leaves that word unchanged. Unnamed IDs are kept.

| ID | +48 mask | +4c mask | +58 mask | +68 mask | +6c mask |
|---:|---|---|---|---|---|
| 0 | `0x00100000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 1 | `0x00010000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 2 | `0x00008000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 3 | `0x00002000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 4 | `0x00001000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 5 | `0x00000800` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 6 | `0x00000400` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 7 | `0x00000200` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 8 | `0x00000100` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 9 | `0x00000080` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 10 | `0x00000040` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 11 | `0x00000020` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 12 | `0x00000010` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 13 | `0x00000008` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 14 | `0x00000004` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 15 | `0x00000002` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00010000` |
| 16 | `0x00000001` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 17 | `0x00000000` | `0x00000000` | `0x00000100` | `0x00000000` | `0x04000000` |
| 18 | `0x00000000` | `0x80000000` | `0x00000000` | `0x00000000` | `0x00000400` |
| 19 | `0x00000000` | `0x40000000` | `0x00000000` | `0x00000000` | `0x00000200` |
| 20 | `0x00000000` | `0x20000000` | `0x00000000` | `0x00000000` | `0x00000100` |
| 21 | `0x00000000` | `0x00000200` | `0x00000000` | `0x00000000` | `0x00000080` |
| 22 | `0x00000000` | `0x00000000` | `0x00000040` | `0x00000000` | `0x00800000` |
| 23 | `0x00000000` | `0x00000000` | `0x00000020` | `0x00000000` | `0x00400000` |
| 24 | `0x00000000` | `0x10000000` | `0x00000000` | `0x00000000` | `0x00000040` |
| 25 | `0x00000000` | `0x08000000` | `0x00000000` | `0x00000000` | `0x00000020` |
| 26 | `0x00000000` | `0x04000000` | `0x00000000` | `0x00000000` | `0x00000010` |
| 27 | `0x00000000` | `0x02000000` | `0x00000000` | `0x00000000` | `0x00000008` |
| 28 | `0x00000000` | `0x01000000` | `0x00000000` | `0x00000000` | `0x00000004` |
| 29 | `0x00000000` | `0x00800000` | `0x00000000` | `0x00000000` | `0x00000002` |
| 30 | `0x00000000` | `0x00000020` | `0x00000000` | `0x00000000` | `0x00000001` |
| 31 | `0x00000000` | `0x00200000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 32 | `0x00000000` | `0x00080000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 33 | `0x00000000` | `0x00040000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 34 | `0x00000000` | `0x00020000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 35 | `0x00000000` | `0x00004000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 36 | `0x00000000` | `0x00002000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 37 | `0x00000000` | `0x00001000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 38 | `0x00000000` | `0x00000000` | `0x00000010` | `0x00000000` | `0x00100000` |
| 39 | `0x00000000` | `0x00000000` | `0x00000002` | `0x00000000` | `0x00080000` |
| 40 | `0x00000000` | `0x00008000` | `0x00000000` | `0x00000000` | `0x00008000` |
| 41 | `0x00000000` | `0x00000800` | `0x00000000` | `0x00000000` | `0x00004000` |
| 42 | `0x00000000` | `0x00000004` | `0x00000000` | `0x00000000` | `0x00002000` |
| 43 | `0x00000000` | `0x00010000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 44 | `0x00000000` | `0x00000400` | `0x00000000` | `0x00000000` | `0x00000000` |
| 45 | `0x00000000` | `0x00000080` | `0x00000000` | `0x00000000` | `0x00000000` |
| 46 | `0x00000000` | `0x00000100` | `0x00000000` | `0x00000000` | `0x00000000` |
| 47 | `0x00000000` | `0x00000040` | `0x00000000` | `0x00000000` | `0x00001000` |
| 48 | `0x00000000` | `0x00000010` | `0x00000000` | `0x00000000` | `0x00000800` |
| 49 | `0x00000000` | `0x00000008` | `0x00000000` | `0x00000000` | `0x00000000` |
| 50 | `0x00000000` | `0x00000002` | `0x00000000` | `0x00000000` | `0x00000000` |
| 51 | `0x00000000` | `0x00000001` | `0x00000000` | `0x00000000` | `0x00000000` |
| 52 | `0x00000000` | `0x00000000` | `0x00000004` | `0x00000000` | `0x00200000` |
| 53 | `0x00000000` | `0x00000000` | `0x00000001` | `0x00000000` | `0x00000000` |
| 54 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00004000` | `0x00000000` |
| 55 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00002000` | `0x00000000` |
| 56 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00001000` | `0x00000000` |
| 57 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000400` | `0x00000000` |
| 58 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000200` | `0x00000000` |
| 59 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000100` | `0x00000000` |
| 60 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000080` | `0x00000000` |
| 61 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000040` | `0x00000000` |
| 62 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000020` | `0x00000000` |
| 63 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000010` | `0x02000000` |
| 64 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000008` | `0x00000000` |
| 65 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000004` | `0x00040000` |
| 66 | `0x00000000` | `0x00000000` | `0x00000080` | `0x00030000` | `0x00000000` |
| 67 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000002` | `0x00000000` |
| 68 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000001` | `0x00000000` |
| 69 | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00020000` |
| 70 | `0x00020000` | `0x00000000` | `0x00000000` | `0x00008000` | `0x00000000` |
| 71 | `0x00080000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 72 | `0x00040000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| 73 | `0x001fffff` | `0xff8fffff` | `0x000001f7` | `0x0003ffff` | `0x07ffffff` |

Names from traced restore consumers (call sites of `clkgate_set`
`0x0802c5b4`): IDs 4/5 DMAC1/0, 7 ECC, 9 AES, 11 FMC, 14 USB OTG, 15 LCD,
16 SHA1, 18..21 UART3..0, 35 chip ID, 36 PKE, 47/48 I2C1/0, 49 USB PHY,
51 PRNG. IDs 37 GPIO, 50 wheel and 52 single-wire are named by mask position
only; no restore `clkgate_set` call passes them. Some block names come from
related-chip headers; the masks are authoritative. Read the sleep, cache/core,
timer, SPI, UART, memory and media gates with their subsystem's traced
consumer, not by table order.

Disable is the reverse mask operation (set bits); boot helpers commonly
restore a saved gate word. Do not gate a block while its DMA, interrupt or bus
transaction is active; the minimum quiescence time is not known for every
block.

## Diagnostics clock service

Protocol GUID `3a6e3065-cb91-4db1-9ae1-d0ee9b990043`, provider `bootloader ClockAndReset.pe`
(not the PMU register protocol `3d4aa229`). Client `diag TestClock.pe`; the table at
`+0x10dc` has 23 rows of 12 bytes (id, UTF-16 name pointer, ASCII name pointer).

| ID | Clock | ID | Source / PLL |
|---:|---|---:|---|
| 0 | fclk_cpu | 0 | OSC |
| 1 | fclk_dsp | 1 | PLL0 |
| 2 | hclk | 2 | PLL1 |
| 3 | dclk | 3 | PLL2 |
| 4 | pclk | 4 | PLL3 |
| 5 | nclk | 5 | Clock_24MHz |
| 6 | mclk (= mclk0) | 6 | Clock_27MHz |
| 7 | uclk | 7 | "OSC Times 2" |
| 8 | amcclk | 8 | Clock_32K |
| 9 | ck24o | | |
| 0xa | ck27o | | |
| 0xb | mclk1 | | |
| 0xc | mclk2 | | |
| 0xd | camclk | | |
| 0xe | vclk (= vclk0) | | |
| 0xf | vclk1 | | |
| 0x10 | admclk | | |
| 0x11 | sclk | | |
| 0x12 | fclk | | |
| 0x13 | aclk | | |
| 0x14 | mphy_clk | | |

Source table `diag TestClock.pe+0x1094`, parser `+0xf20`: command names `pll0..pll3`, `24mhz`, `27mhz` map to 1..6; anything else = OSC.

Protocol slots used (`diag TestClock.pe+0xb24..0xdb4`): +0xc reset; +0x10 GetPllSpeed(pll, &Hz); +0x14 SetPllSpeed(pll, value);
+0x18 GetClockSpeed(clk, &Hz); +0x20 Configure(clk, source, div); +0x24 (clk, 1) follow-up after Configure; +0x28 monitor enable (mon, 0/1);
+0x2c monitor select (mon, source 1..6). `clocksetspeed` is a stub returning `0x80000002`. Slot +4 is the gate call (`clock protocol slot +4, id, 1/0`; ChipId uses id `0x23`).

SetPllSpeed accepts only these values (others rejected, result 0): 1, 22, 23, 24, 45, 49, 54, 67, 68, 82, 100, 110, 113, 123, 133,
135, 162, 166, 192, 200, 208, 216, 221, 360, 400, 432, 532 (`diag TestClock.pe+0xfec..0x1090`). The usage text says Hz; the values are small integers; unit unknown.

The `mclk` command (`diag TestMonitor.pe+0xbfc..0xc50`, strings `+0x15de`): source 0 OSC /4 "6 MHz", 1 PLL0 /49 "12 MHz",
2 PLL1 /4 "6 MHz", 3 PLL2 /4 "6 MHz". It first sets PLL1 to 113 (source 2) or PLL2 to 54 (source 3) via SetPllSpeed, then
Configure(clk `0xc` = MCLK2, source, div) and (`0xc`, 1).

Clock-test register `+0x64` source bits (option table `+0x11f4`; pads GPIO 1.0 / 1.1, function 6, [GPIO](gpio-pinmux.md)):

| Source | Monitor 0 | Monitor 1 |
|---|---|---|
| 24M | bit 0 | bit 8 |
| 12M | bit 1 | bit 9 |
| 48M | bit 2 | bit 10 |
| USBPHY | bit 4 | bit 12 |
| PLL0 | bit 5 | bit 13 |
| PLL1 | bit 6 | bit 14 |
| PLL2 | bit 7 | bit 15 |
| 32K | no bit | no bit |

Bootloader id-to-register map behind the protocol (`bootloader ClockAndReset.pe+0xe24`, switch on id-2, table `+0xe40`):
ids 0, 1, 3, 7, 8, 9, 0xa, 0xd, 0x10 return 0 (unsupported); 2 hclk, 4 pclk, 0x13 aclk from CLKCON1 (bit 22 / `[21:17]`, bit 14 / `[13:9]`, bit 6 / `[5:1]`);
0x11 from CLKCON0 (source `[13:12]`, divider `[3:0]+1`); 0x12 from CLKCON1 bit 30 and `[29:25]`; 5, 6, 0xb, 0xc from CLKCON3 / CLKCON2 / CLKCON4 fields;
0xe, 0xf, 0x14 from CLKCON2 `+8` fields. Field meanings not decoded.

Gate ids in the diag namespace: the camera driver enables `0x47` then `0x49` (off in the same order) and the sleep code uses `0x47`
(arg 0 on enter, 1 on wake) and `0x49` (1) with GPIO 16.2 (`diag IspDriver.pe+0x894`, `SystemPowerManagement.pe+0x27e..0x2bc`);
the PWM module enables `0x1e`; the camera driver also uses `0x40`. Above id 56 the diag, restore and bootloader tables differ, so the physical bits of
`0x40 / 0x47 / 0x49` are not asserted (restore row 0x47 = `+48` bit 19, row 0x49 = the all-bits row, so the diag numbering is not simply the restore table).
The diag LCD driver clears `0x3c500048` bit 14 and `0x3c500068` bits 14, 11, 10, 9 (mask `0xb1ff`) before touching the controller (`diag Clcd.pe+0x5b4..0x640`).

Idle clock state written by the IRAM sleep routine before WFI (`diag IramRoutines.pe+0xbb8..0xd0c`): it saves
`+00, +04, +08, +0c, +10, +14, +44, +48, +4c, +58, +68, +6c, +70`; writes `+48 = ffff7fe7`, `+4c = fffbefff`, `+58 = 000001f7`,
`+68 = ffffcfff`, `+6c = 06ffffff`, `+70 = 8000`, `+14 = 80008000`, `+10 = 0000c000`, `+0c = c000c000`, `+08 = 80008000`,
`+04 = 007e7e7e`, `+00 = 4000800f`, `+44 = 7` then `0`; sets `0x39700000` bits 25:24; executes WFI (`c7,c0,4`); restores all.

Software reset in the diag (`SystemReset` -> ClockAndReset slot 3, `bootloader ClockAndReset.pe+0x5a8`): `0x3c800000 = 0x00100000`, then `+50 = 0xa5`, then spin ([timers](timers-watchdog.md)).

## Unknown

PLL operating envelope, oscillator tolerance, PLL lock-time unit, reset
values, clock tree, SetPllSpeed whitelist unit, per-id CLKCON2..5 fields, the `clkmon --off` path and the meaning of `+0x64` bits 17 / 16 as monitor 0 / 1 enable (inferred from the register composition only). Unexplained CLKCON bits, the `+70=0x8000` write and
`0x38501000=2` are recorded as written. A firmware frequency option does not
prove support on all board revisions and voltages.

## Finding ledger

| Finding | Evidence | Type |
|---|---|---|
| ROM and PreEfi clock sequences differ | ROM `0x200000a4`; `bootloader mem:0x2200b17c` | observed |
| Gate enable clears bits | ROM `0x2000169c`, restore `0x0802c5b4` | observed |
| Paired gates and distinct namespaces | ROM helper plus 74-row restore table | observed |
| Nominal frequencies | PLL parameters, `bootloader ClockAndReset.pe+0x5f6/+0xca8/+0xe24` | inferred from firmware; not measured |
| Software reset key | restore `0x08070c04`, SecCore `0x220226b0` | observed `+50=0xa5`; reset-domain coverage unknown |
| Diag clock IDs 0..0x14, PLL/source IDs 0..8, protocol slots | `diag TestClock.pe+0x10dc/+0x1094/+0xb24..0xdb4` | observed |
| Clock-test register `+0x64` source bits | `diag TestMonitor.pe+0xd38..0xe48` | observed; enable bits inferred |
| Idle clock words written before WFI | `diag IramRoutines.pe+0xbb8..0xd0c` | observed |
| Bootloader id-to-register map | `bootloader ClockAndReset.pe+0xe24` | observed; fields not decoded |
