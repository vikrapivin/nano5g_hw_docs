# Nano 5G reconstructed hardware reference

Start with the [overview and component map](overview.md). One page per
block or external device: register offsets, bit fields, clocks, GPIO signals,
DMA and interrupts, initialization and shutdown sequences, with the firmware
address each fact comes from. Values whose meaning is unknown are listed as
such.

- **System:** [CPU/memory](cpu-memory.md), [SDRAM](sdram.md),
  [clocks/reset](clocks-reset.md), [bus fabric](bus-fabric.md),
  [interrupts](interrupts.md), [timers/watchdog](timers-watchdog.md),
  [GPIO](gpio-pinmux.md), [I2C](i2c.md), [PL080 DMA](dma.md), [DMAC2](dma2.md).
- **Storage/data engines:** [NAND/FMSS](nand-fmss.md), [FMSS instruction set](nand-fmss-isa.md), [NAND chips](nand-chips.md),
  [ECC](ecc.md), [SPI/NOR](spi-nor.md), [AES](aes.md), [SHA](sha.md),
  [PRNG](prng.md), [chip ID](chipid.md), [public-key accelerator](pka.md).
- **USB/power/input:** [USB core](usb-controller.md), [PHY](usb-phy.md),
  [PMU/charger](pmu.md), [ADC](adc.md), [RTC](rtc.md), [backlight](backlight.md),
  [clickwheel](clickwheel.md), [accelerometer](accelerometer.md), [UART](uart.md).
- **Audio:** [I2S](audio-i2s.md), [codec](audio-codec.md),
  [speaker amplifier](speaker-amplifier.md), [headphone/accessory](headphone-interface.md),
  [FM radio](fm-radio.md).
- **Imaging/display:** [camera sensors](camera-sensors.md),
  [capture interface](camera-interface.md), [video encoder](video-encoder.md),
  [H.264 decoder](video-decoder.md), [JPEG encoder](jpeg-encoder.md),
  [compositor/scaler](display-compositor.md), [scaler/colour converter](scaler.md), [LCD controller](lcd-controller.md),
  [panels](lcd-panels.md), [TV pipeline](tvout.md),
  [TV scaler](tv-scaler.md), [TV mixer](tv-mixer.md), [TV encoder](tv-encoder.md).

[Open questions](unknowns.md): what is still unknown per block.

## Images and citation forms

The facts come from five firmware images of the iPod nano 5G: the mask boot ROM
(`bootrom`), the N33 bootloader (`bootloader`), the DFU-stage loader
(`wtf`), the restore/disk-mode OS (`disk`) and the iPod OS (`osos`), plus the
EFI modules of the diagnostics image (`diag`). A citation names the image and
the mapped address of the instruction or table (`osos mem:0x081337cc`,
`disk mem:0x08044054`, `bootrom mem:0x20001e34`, `bootloader mem:0x2200b650`
for the bootloader's execute-in-place part) or, for an EFI module, the module
and the offset from the start of that PE file (`bootloader NandReadOnly.pe+0x5f28`,
`diag Tvout.pe+0x0f04`). `+0x…` alone means an offset inside the register block
or structure under discussion. Each page ends with a finding ledger that gives
the evidence class of every finding (observed in code or data, dynamic analysis,
device-observed, inferred, unknown; see the [overview](overview.md)).
