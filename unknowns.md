# Open questions

Everything in these pages comes from what the firmware does. That leaves three
kinds of gap, listed separately below because they need different evidence:

1. **Registers written with unknown meaning.** The firmware stores a value and
   never reads it back or branches on it, so the pages record the value and
   the order but not what the bits mean. More firmware paths (other modes,
   the diagnostics image) or a data sheet would settle these.
2. **Parts not identified.** The firmware talks to a chip through registers but
   never prints a part number, so the exact device is inferred from the
   register map or a module name.
3. **Physical facts no firmware can give.** Voltages, currents, frequencies,
   pinouts, timing margins. Only a measurement, a schematic or a data sheet
   settles these.

"Not found" in a page means not found in the named images or routines, not
that the hardware lacks the feature.

## 1. Registers and fields with unknown meaning

| Block | What is open |
|---|---|
| [CPU/memory](cpu-memory.md) | SRAM capacity: declared 448 KiB and probed at one address; a 320 KiB mirror and anything above `0x2206ff40` not tested |
| [Bus fabric](bus-fabric.md) | The seven startup words at `3ff00104/0x77c/0x220/0x100` and `3810008c`, `38501000 = 2`, `3e000008 = 1f`, `3d500400 = 7`: values and order known, block identity and bit meanings not |
| [SDRAM](sdram.md) | Units of the timing fields, the meaning of the setup literals written once, and why the IRAM re-init writes different timing literals than PreEfi |
| [Clocks](clocks-reset.md) | Unexplained CLKCON bits, the `+70 = 0x8000` write, the per-ID CLKCON2..5 fields, the unit of the PLL-speed whitelist and of the lock delay, the `clkmon --off` path |
| [Interrupts](interrupts.md) | Which pad feeds which GPIO-interrupt group (no complete map), trigger type per line, whether timers 4..7 also raise VIC line 8 |
| [Timers](timers-watchdog.md) | Timer CON bits other than the ones the firmware sets, watchdog modes other than the reset write |
| [DMAC2](dma2.md) | Channel control bits 16/17/18 (the ROM copy runs with bit 16 clear and still advances), error register `+10` handling |
| [I2C](i2c.md) | The unnamed init writes (`+14 = 1`, `+18 = 0`, `+28 = 0` in the disk image only) and the error paths |
| [NAND / FMSS](nand-fmss.md), [instruction set](nand-fmss-isa.md) | `FMSS_C10/C58/C3C/C50`, start-word bits other than bit 6, transfer codes `0x20/0x10020/0x20020/0x30020/0x100e0/0x033338e0`, `FMC_18/28/5C`, `ECC_808`, `FMC_DMA_CTRL = 7`, whether a WAIT timeout vectors through `FMSS_C4C` |
| [ECC](ecc.md) | The BCH polynomial and code parameters (the wrapper only sets sizes and modes) |
| [AES](aes.md) | Mode and key-size combinations the firmware never uses |
| [SHA](sha.md) | Unused command bits, fault bits |
| [Public-key accelerator](pka.md) | `+0x04` (unused), `+0x18` (no caller), `+0x08` bit 3, `+0x14` bits 0/1, the per-slot flag word `+0x10`; whether the multiply is modular with the modulus in slot 0 (inferred) |
| [PRNG](prng.md) | Which of the three ready bits means what; what `+00 = 0x60` and the seed word at `+08` do |
| [Chip ID](chipid.md) | Encoding of the revision fields and of the 7-bit field in word `08`; fuses beyond the die-ID words |
| [USB core](usb-controller.md) | `GGPIO` integration bits 21/16, PCGCCTL behavior, the stage-specific setup-size literals; host, isochronous and OTG modes are never used |
| [USB PHY](usb-phy.md) | Units of the tuning words; `+0x48` beyond the values 2/4/0 used for DP/DM measurement |
| [PMU](pmu.md) | Rail nibble fields at `0b..0f`; reg `00` (chip ID, value unknown); event-mask polarity of `07..09`; reg `32` pin 7 target; reg `35` (times 120, unit unknown); reg `34` (`7c`/`0`, never read by any image); whether reg `33` bits 7:2 are charge current (label only) |
| [ADC](adc.md) | Reg `44 = 72` fields; channel code 0; the channel-0 capture through `b3/b9/ba` with flag bit 8 |
| [RTC](rtc.md) | The two uses of bit 6 (reg `50` read mask versus reg `56` write in the bootloader), the alarm IRQ bit, read atomicity across a rollover |
| [Backlight](backlight.md) | Single-wire block: register `24` period units, status bits, registers `00/04/10` beyond the observed values |
| [Click wheel](clickwheel.md) | Control bits other than 21/22, the `0x3a980` timing word, flag bits 1/2 of `+14`, controller registers other than 0..2, the update-file layout, the provider of protocol `99722de2…` |
| [Accelerometer](accelerometer.md) | Why the first recovery write goes to slave `0x19`, sample-rate and low-power encodings, interrupt polarity |
| [SPI / NOR](spi-nor.md) | The literal `0x40618`, register `+0c = 6`, the all-ones writes to `+3c/+40`, CPOL/CPHA fields |
| [UART](uart.md) | RX, interrupt and flow-control registers (only init and TX are exercised); which UART the fifth DMA endpoint belongs to |
| [I2S](audio-i2s.md) | TX/RX control bitfields beyond the literals `0x03100099`, `0x0b000019`, `0x1000` |
| [Codec](audio-codec.md) | Registers the firmware writes but never reads (`0d/0e`, `12/13`, `18/19`, `20`), the `TestAudioIpod` routing tables |
| [Headphone controller](headphone-interface.md) | Polarity of status reg `0c` bit 3 (OSOS reads 1 = inserted, the diagnostics print 0 = detect), the remote/microphone protocol beyond detection |
| [FM radio](fm-radio.md) | Commands `80/81` beyond "antenna switch", the patch stream, property `1404` (10 in OSOS, `0x14` in the diagnostics), RF units |
| [Camera interface](camera-interface.md) | Format modes other than 1/2/3, auxiliary block masks (`+04 = 0x98000001`), input signal polarity |
| [Sensors](camera-sensors.md) | Meaning of most of the 240 vendor register writes; why the diagnostics 7739 table is a different register set |
| [Video encoder](video-encoder.md) | Bitstream engine bits `+20/+50/+dc/+f0`, picture engine `+168..+184`, mode-engine words `+04[31:26]` and `+08`, what `0x550174` accumulates, host `904/908/910` beyond their formulas, the slice-limit unit |
| [H.264 decoder](video-decoder.md) | Status bits beyond the ones polled, the per-entry type byte that adds `0x8080` to a reference-list entry, which sub-block the `0x3e0` reset form resets |
| [JPEG encoder](jpeg-encoder.md) | Pixel formats other than the one used, control fields written as constants |
| [Compositor](display-compositor.md) | Per-layer nibbles in `010` (diagnostics) versus background use (OSOS), filter-threshold classes, fields named by bit position only |
| [LCD controller](lcd-controller.md) | Registers `18/20/68/78/7c/88`, every LCDCON bit |
| [Scaler](scaler.md) | Status bit 1 polarity, `0c` bit 0, pairing of the source-origin halves; no frame has been run through it |
| [TV output](tvout.md), [scaler](tv-scaler.md), [mixer](tv-mixer.md), [encoder](tv-encoder.md) | Most of the 398 written values: known as values and order, not as fields; format enums; which clock selectors (`06/44/45` on, `47` off) map to which gate bits |

## 2. Parts not identified from the firmware

| Device | What is known | What is not |
|---|---|---|
| PMU | D1759 family (driver name), I2C `0x73`, register map as used | exact part, thresholds (termination voltage, taper, recharge, thermal, timers) |
| Audio codec | CS42L55-compatible register map, I2C `0x4a` | exact part and revision, analog pinout |
| Speaker amplifier | command-only I2C `0x6f`, 36 gain codes; module named LM48557 | exact part, dB scale, wiring |
| Headphone/accessory controller | CD3282-style register map, I2C `0x39` | exact part, pinout |
| FM radio | Si47xx command set, I2C `0x63` | part and firmware revision |
| Accelerometer | LIS331DLM register map, WHO_AM_I `0x12`, I2C `0x09` | retail part suffix, mounted revision |
| Click wheel controller | Cypress (`0x104`) or Synaptics (`0x1288`), read at run time | which one a given unit carries; its firmware |
| Camera sensor | OV7692 or OV7739 by ID bytes | board population per revision |
| NAND | 18 geometry rows; one unit identified (Samsung, 4 CE) | other populations in the field |
| SDRAM | 64 MiB reported by the firmware | vendor, part, bus width |
| Serial NOR | SPI read/ID/status path in the ROM | whether any NOR is populated |
| LCD panels | five IDs with full command scripts | panel part numbers |
| Bus-fabric blocks | addresses and writes | block identities |

## 3. Physical and electrical facts

Not derivable from firmware; listed so they are not mistaken for decoded facts:

- Supply voltages beyond the PMU codes (the codes give 1200 mV core, 3000 mV
  LDO1/2/5/7, 1800 mV LDO9; actual rails are unmeasured), charge currents
  (the selector is a budget, not a measured current), battery thresholds as
  measured on a cell.
- Clock frequencies of every source (216/108/54 MHz and 274.67/137.33 MHz are
  derived from the PLL words, not measured), oscillator tolerance, PLL lock
  time.
- Pinouts: package pins, connector pins, pull strengths, drive levels, the
  panel data width, the wheel link's clock rate and edge polarity.
- Timing margins: NAND bus setup/hold, SDRAM stability, cache retention in
  sleep, self-refresh current, I2C bus speeds per divider.
- Analog paths: codec gains and biases, speaker impedance and power, headphone
  detection thresholds, TV output levels, backlight LED current and the
  luminance curve, sensor electrical limits.
- Anything measured on a device appears in the pages as "device" evidence.
