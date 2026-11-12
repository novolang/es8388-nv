# Changelog

## 0.2.0 — 2026-11-12

Moves off its stand-ins onto devices-nv 0.4.0 and embedded-hal-nv 0.4.0. The release depends on both.

- `codec_set_gain` and `codec_gain` take `CodecHeadphone` (the pair OUT1),
  `CodecLine` (the pair OUT2), `CodecDac` and `CodecAdc` (the digital volumes).
  They replace the inherent `set_out_volume`, `set_dac_volume`, `dac_volume`,
  `set_adc_volume` and `adc_volume`. `CodecSpeaker` answers `None`.
- `codec_select_output` powers OUT1 or OUT2 up or down; it replaces
  `set_outputs`, which set both pairs in one call.
- `codec_set_path_mute` and `codec_path_muted` mute and read the ADC for
  `CodecInput` and `CodecAdc` and the DAC for `CodecOutput` and `CodecDac`;
  they replace `set_adc_mute`. 0.1.0 had no way to read the ADC mute back.
- The framings `AudioRightJustified`, `AudioDspA` and `AudioDspB` replace the
  inherent `set_rj_format` and `set_dsp_format`; they are written with the
  format, and the handle is configured for the same framing. `AudioPdm` is
  refused.
- `slot_bits` is passed to the I2S handle, so the data sheet's 64 bit clocks at
  16 bits are `slot_bits` 32.
- A record literal of `AudioFormat` adds `slot_bits`.

## 0.1.0

The first release: a driver for Everest Semiconductor's ES8388 stereo audio
codec on an I2C control bus and a board's I2S handle.

- `Es8388`, the type `attach` builds from the board's I2C bus, the wiring and
  the I2S handle, implementing devices-nv's `AudioCodec`, with its own methods
  for the digital volumes, the ADC mute and the PGA of each channel, the
  analogue input and the slot each ADC fills, the output drivers and their
  volumes, the output mixers, the mono mix, the ALC, and the DSP/PCM and
  right-justified formats. The part has no identification register, so there
  is no `Identifiable`.
- The clock: `clock_plan` finds the 5-bit MCLK to LRCK ratio code of the data
  sheet's Table 2 for 8 to 96 kHz from the 12.288, 11.2896, 18.432 and
  16.9344 MHz clocks, their doubles with MCLKDIV2 and the 12 and 24 MHz USB
  clocks, and the double-speed bit above 50 kHz; a slave takes the ratios of
  Table 1. Tests pin it at the rows of the tables and at 48 kHz from
  12.288 MHz, ratio 256, code 00010.
- The serial port in I2S and left-justified formats through `AudioCodec`, and
  the right-justified and DSP/PCM mode A and B through `set_rj_format` and
  `set_dsp_format`; 16, 24 and 32 bits. `codec_set_format` configures and
  enables the board's I2S handle first.
- The output gain of `AudioCodec` is the four output drivers, -45 to +4.5 dB
  in 1.5 dB steps, and the input gain is the PGA of both channels, 0 to
  +24 dB in 3 dB steps, each never above the request.
- The reset, the base state, the power-up and the power-down as named steps
  (`step_reset`, `step_open`, `step_source`, `step_ratio`, `step_serial_port`,
  `step_power_up`, `step_power_down`), the power-up and power-down in the
  order of the ES8388 User Guide, section 10. A power-up leaves the DAC muted,
  as `AudioCodec` requires.
- `AlcConfig`, `alc_voice` and `alc_generic`, and `alc_set`, which writes
  Registers 18 to 22 and refuses a field outside its register.
- The suite runs over a double of the part and of the I2S handle and covers
  every line of `src/`.
- The device program `examples/line_in.nv`. Built for the nRF52840-DK it is
  14,644 B of flash and for the nRF54LM20-DK 16,124 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries an ES8388, and no
  board package has an I2S block. The Nordic DKs drive I2C with TWIM0 and
  TWIM22, so the device program's images carry the reading path; a reply from
  the codec over either bus is NOT VERIFIED. The device program is given a
  stand-in handle.
- The register map is from the data sheet, Revision 5.0, July 2018, and the
  sequences from the user guide dated 2011-06-17. The two disagree on the ALC
  target and maximum gain codes; the data sheet is taken. The user guide's
  sequence writes reserved bits in CTRL2 and in registers 0x28 and 0x29; they
  are written as the user guide gives them.
- The 100 ms charge of the reference capacitors in the power-up is the Linux
  driver's; neither Everest document gives a time.
- The manifest names devices-nv and embedded-hal-nv by the registry range
  `^0.3.3`, the first releases with `AudioCodec` and `I2s`.
- Not implemented: the shelving filters, the stereo enhancement, the
  de-emphasis, the low-power bits, the DC offset, the ADC filter bits, the
  18 and 20 bit word lengths, the SPI port, a clock that is not a multiple of
  the rate, and `AudioCodecAsync`.
- A local named `codec` breaks the embedded tier check
  (`a-local-named-like-a-stdlib-module-makes-the-embedded-tier-check-read-its-method-calls-as-module-calls`);
  the device program calls its codec `part`.
- The linter's `char-literal` rule reads register numbers as characters. It is
  wrong for a register map, and those sites are left.
