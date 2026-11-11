# es8388-nv

A driver for Everest Semiconductor's ES8388, a low-power stereo audio codec
with two ADCs, two DACs, a microphone amplifier, a headphone amplifier and
analogue mixers. The part has an I2C or SPI control port and an I2S,
left-justified, right-justified or DSP/PCM audio port. The driver implements
the `AudioCodec` trait of
[devices-nv](https://novo-lang.org/packages/devices-nv) and configures the
board's `I2s` handle of
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) for the
format it sets. It uses the I2C port. The ES8388 has no identification
register, so the driver has no `Identifiable`.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
chip, and no board package has an I2S block yet. The driver has run only
against a double of the part and a double of the I2S handle, in the host
suite. The device program below builds for the nRF52840-DK and the
nRF54LM20-DK against a stand-in handle and has not been flashed.

Two documents are the sources, both from Everest: the data sheet "ES8388
Low Power Stereo Audio CODEC With Headphone Amplifier", Revision 5.0, July
2018 (pins, clock modes, control port, register map, serial formats), and the
"ES8388 User Guide" dated 2011-06-17 (the register sequences of its section
10). The data sheet has no power-up sequence and the user guide has no
register map beyond its sequences, so the driver takes each fact from the
document that states it; the places where the two differ are listed under
Tests.

## Install

```sh
novo pkg add es8388-nv
```

The device program is `examples/line_in.nv`. Make it the `main` of a package
that depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/line_in.nv` brings the codec up and passes its line input through
the board to its headphone output, printing one line per step over the
board's UART:

```
codec found
```

```novo norun:needs-pkg
use embedded_hal.{ I2s, I2sConfig, BoardI2c }
use dev_audio.{ AudioCodec, AudioFormat, AudioFraming, CodecPath, MilliDb }
use dev_bus
use es8388

// The board's I2S block goes here; this stand-in refuses to be configured.
@value
struct NoI2s
    id: Int

impl I2s for NoI2s
    fn configure(self, c: I2sConfig) -> Bool [hw]
        false
    fn sample_rate(self) -> Int [hw]
        0
    fn enable(self) -> Bool [hw]
        false
    fn disable(self) [hw]
        pass
    fn transmit(self, var buf: Vec[i32; 256]) -> Int [hw]
        -1
    fn receive(self, var buf: Vec[i32; 256], n: Int) -> Int [hw]
        -1
    fn xruns(self) -> Int [hw]
        0

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // The codec answers at 0x10 with CE low, on the first I2C bus, and the
    // board gives it a master clock of 12.288 MHz.
    let part = es8388.attach(BoardI2c { id: 0 }, dev_bus.wiring_i2c(0, es8388.address_of(false)),
                             NoI2s { id: 0 }, es8388.config_default(12288000))
    if not part.begin()
        hal.uart.write("no answer from the codec at 0x10\n")
        return
    let f = AudioFormat { sample_rate: 48000, bit_depth: 16, channels: 2,
                          framing: AudioFraming.AudioPhilips, codec_is_master: false }
    if part.codec_set_format(f) and part.codec_set_power(true)
        let set = part.codec_set_gain(CodecPath.CodecOutput, MilliDb { mdb: -12000 })
        if set != None
            // After the first buffer is queued the output is opened.
            let _ = part.codec_set_mute(false)
```

The full program, which also sets the input gain and moves the buffers, is
`examples/line_in.nv`. A board package that depends on this package builds it
as its `main`. `tests/device_build.sh` does so from a scratch package for the
two boards below and for the emulated board, where it runs the image and
checks that the program reports that the codec does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 14,644 B | 3,866 B | 3,236 B |
| `nrf54lm20-dk` | 16,124 B | 4,306 B | 3,236 B |

The numbers are from the build's memory report of this program, 2026-11-11,
toolchain 0.24.1. Flash also holds the runtime, the compiler's support
library and the program's tables. The build drops the functions the program
does not call, so a program that sets the ALC or the mixers is larger. The
arena is the runtime's heap and is reserved whether or not the program uses
it.

**The I2S block is absent hardware on both boards; the I2C bus is not.**
embedded-hal-nv has a trait for an I2S interface and no board handle for one,
and neither Nordic board package drives its I2S block: no board implements
`I2s` or `PdmIn` yet, and there is no `hal.i2s` hook. The Nordic DKs drive the
control port's I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA P0.26 and SCL
P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04), on the pins
`docs/hal/README.md` names under "The Nordic DKs' buses", so both images carry
the driver's reading path. The program is written against the `I2s` trait and
is given `NoI2s`, so on both images `begin` runs on the bus, finds no codec
and prints `no answer from the codec at 0x10`. A reply from the codec over a
DK's bus is NOT VERIFIED until the driver rig carries one. The one argument of
`attach` is what changes when a board package gives an I2S handle.

### The effects

Every function of the driver that reaches the bus or the I2S handle declares
`[hw]`, and nothing else. The driver performs no input or output of its own.
`attach` allocates one cell of the arena, because a struct with a generic
field cannot be a `@value`; the sequences build only the one-byte frames of
devices-nv's register functions, in the caller's frame. The module checks at
every tier, `embedded` included.

## What it is

The ES8388 has two 24-bit sigma-delta ADCs behind a microphone amplifier
(PGA, 0 to +24 dB), an optional ALC and noise gate, two 24-bit DACs, analogue
mixers, and four output drivers: LOUT1 and ROUT1, LOUT2 and ROUT2. It runs
from 8 to 96 kHz. The I2C address is 0x10 with the CE pin low and 0x11 with CE
high; **a high-to-low change on CE after power-up selects the SPI port**, so a
board that drives CE sets it once and leaves it. The registers can be read, so
the driver keeps no copy of them. The part has no chip ID.

In slave mode the part detects the ratio of its master clock to the word clock
itself, from 256, 384, 512, 768 and 1024 up to 50 kHz and 128, 192, 256, 384
and 512 above. In master mode the ratio is a 5-bit code in two registers.
`clock_plan` finds it, and sets MCLKDIV2 for the clocks that are twice the
table's:

| Master clock | Rate | Ratio | Code | MCLKDIV2 |
|---|---|---|---|---|
| 12.288 MHz | 8 kHz | 1536 | 01010 | 0 |
| 12.288 MHz | 16 kHz | 768 | 00110 | 0 |
| 12.288 MHz | 48 kHz | 256 | 00010 | 0 |
| 12.288 MHz | 96 kHz | 128 | 00000 | 0 |
| 11.2896 MHz | 44.1 kHz | 256 | 00010 | 0 |
| 18.432 MHz | 96 kHz | 192 | 00001 | 0 |
| 16.9344 MHz | 44.1 kHz | 384 | 00011 | 0 |
| 24.576 MHz | 48 kHz | 256 | 00010 | 1 |
| 12 MHz (USB) | 8 kHz | 1500 | 11011 | 0 |
| 12 MHz (USB) | 48 kHz | 250 | 10010 | 0 |

The rates above 50 kHz also set the double-speed bit. A clock that is not a
whole multiple of the rate is refused. That includes the data sheet's
8.0182 kHz, 8.0214 kHz and 44.118 kHz rows, which are rates of a clock that is
not a multiple of the rate, and 44.1 kHz from the 12 MHz USB clock. A slave
takes only the ratios of the first table.

## What the package contains

| Module | What is in it |
|---|---|
| `es8388` | `Es8388Config`, `Es8388Port`, the I2C framing, the register access, the clock plan, the named steps, the gain functions, `AlcConfig`, the type `Es8388` and its trait |

## How to choose an entry point

A program on a board builds an `Es8388Config` with `config_default(mclk_hz)`,
passes it with the board's I2C bus, the wiring and the I2S handle to
`attach`, and uses the trait on the result.

| Method | What it does |
|---|---|
| `attach(bus, w, i2s, cfg)` | the codec at `w.addr` and the handle that carries its samples; nothing is sent |
| `begin()` | the bus ready, the reset, CTRL1 checked against its default 0x06, DAC muted and both digital volumes at 0 dB |
| `codec_set_format(f)` | configures and enables the I2S handle, then sets the role, the clock ratio and the serial port |
| `codec_set_power(on)` | runs the user guide's start-up with the DAC muted, or its power-down, and disables the handle |
| `codec_set_mute(on)`, `codec_is_muted()` | the DAC's mute bit |
| `codec_set_gain(path, g)`, `codec_gain(path)` | the four output drivers, or the PGA of both channels, to the largest step at or below `g` |
| `set_dac_volume(g)`, `set_adc_volume(g)` | the digital volumes, 0 to -96 dB in 0.5 dB steps |
| `set_adc_mute(on)`, `set_pga(l, r)` | the ADC mute and a PGA for each channel |
| `set_input(sel)`, `set_adc_data(sel)` | LIN1, LIN2 or a differential pair; which ADC fills which slot |
| `set_outputs(o1, o2)`, `set_out_volume(o2, g)` | enable and set one output pair |
| `set_mixer(dac, line, line2, gain)`, `set_dac_mono(on)` | the output mixers and the mono mix |
| `set_alc(cfg)`, `alc_disable()` | the ALC and noise gate from an `AlcConfig` |
| `set_dsp_format(depth, b)`, `set_rj_format(depth)` | DSP/PCM mode A or B, and the right-justified format |
| `reg(r)`, `set_reg(r, v)`, `i2s_stop()` | one register, a write, and the handle's clocks stopped |

The steps are functions over `Es8388Port`, a trait a type holding an I2C bus
implements, so a test runs them over a double of the part: `step_reset`,
`step_open`, `step_source`, `step_ratio`, `step_serial_port`, `step_power_up`
and `step_power_down`. Their order is the one place to read the sequences:

| Step | Writes |
|---|---|
| reset | CTRL1 0x80 (SCPReset), then 0x06, the default |
| base state | DAC muted, ADC and DAC digital volumes 0 dB (the reset leaves -96 dB) |
| source | MASTERMODE: master bit, MCLKDIV2, bit clock inversion; BCLKDIV 0, so a master makes its bit clock from the clock table |
| ratio | ADCCTL5 and DACCTL2: the code and the double-speed bit |
| serial port | word length, format and polarity of both ports |
| power up | DAC muted; CHIPPOWER 0xF3; same LRCK; CTRL1 0x07, 100 ms, CTRL1 0x05; CTRL2 0x40; ADC power 0x00; DAC power 0x3C; the mixers to the DAC alone; CHIPPOWER 0x00 |
| power down | both converters muted; CHIPPOWER 0xF3; ADC power 0xFC; DAC power 0xC0 |

## The gains

`codec_set_gain` on the output path sets LOUT1, ROUT1, LOUT2 and ROUT2 to one
step, from -45 dB to +4.5 dB in 1.5 dB steps; 0 dB is code 30, the value the
user guide writes. On the input path it sets the PGA of both channels, from 0
to +24 dB in 3 dB steps; the user guide's +21 dB is 0x77. The digital volumes
are separate: `set_dac_volume` and `set_adc_volume`, 0 dB down to -96 dB in 0.5
dB steps. The gain a call sets is never above the request, and a request below
the range sets nothing. The reset leaves the output drivers at -45 dB and both
digital volumes at -96 dB; `begin` moves the digital volumes to 0 dB and leaves
the drivers quiet until the program sets a gain.

## The ALC

`AlcConfig` holds the fields of the data sheet's Registers 18 to 22 as codes:
the channel selection, the maximum and minimum PGA gain, the target level, the
hold, decay and attack times, the limiter mode, the zero-cross detector, the
window and the noise gate. `alc_voice()` and `alc_generic()` are the user
guide's recommended settings for microphone recording and for general
recording, in the data sheet's codes. `set_alc` writes the five registers and
refuses a field outside its register without writing anything.

## The rules a user needs

1. Wait for the supply, then call `begin`. The part answers its address when it
   is powered and its I2C pins are wired; `begin` cannot tell the part from
   another that takes the reset, because there is no ID.
2. Set the format before powering up. `codec_set_format` configures the I2S
   handle and enables it first, because the part needs its clocks running
   before its registers are final.
3. `codec_set_power(true)` leaves the DAC muted. Open it with
   `codec_set_mute(false)` once the first buffer is queued.
4. `codec_set_power(false)` keeps the clock registers; the next power-up needs
   no new format. It sets the mixers back to the DAC alone, so a program that
   set `set_mixer` sets it again.
5. The part frames two slots. A mono format is refused; send a mono signal in
   both slots, or use `set_adc_data` and `set_dac_mono`.
6. `AudioFraming` names the I2S and left-justified formats only. DSP mode and
   the right-justified format are `set_dsp_format` and `set_rj_format`.
7. The data sheet says the bit clock is normally 64 times the word clock.
   `I2sConfig` has one bit depth and no slot width, so a 16-bit format asks the
   handle for 32 bit clocks per frame, whether the part takes that is NOT
   VERIFIED, and a program that wants 64 asks for 32 bits.

## What is not included

- The DAC's bass and treble shelving filters (Registers 30 to 37), the
  stereo enhancement, the de-emphasis, the output-phase inversions and the DAC
  output scale.
- The low-power bits of Registers 5 and 6, the ADC filter and polarity bits of
  Register 14, the ADC ramp rate, the DC offset of Registers 43 and 44, and the
  tri-state of the ADC output.
- The word lengths of 18 and 20 bits, which `AudioFormat` cannot name.
- The SPI port.
- A master clock that is not a multiple of the rate, and the non-integer rates
  of Table 2.
- The settling wait of `AudioCodecAsync.codec_power_up`: the one wait the
  power-up has, the 100 ms charge of the reference capacitors, is taken in
  the synchronous step, and no timer-parking body is written.
- The power-amplifier enable pin of a module: it is the board's GPIO.

## Where the traits fall short

- `AudioCodec` has one output gain and one input gain and no input mute. The
  output gain here is the four output drivers; the DAC volume, the ADC volume
  and the ADC mute are the driver's own functions.
- `AudioFraming` has two framings; the part also has the right-justified and
  the DSP/PCM modes. The driver's own `set_rj_format` and `set_dsp_format`
  cover them.
- `I2sConfig` has one `bit_depth`, which is both the slot width and the sample
  width, so a codec that takes 24-bit data in a 32-bit slot, or 16-bit data
  in a 64-clock frame, cannot say so.
- A trait method gets a plain `self`, so the driver keeps no state between
  calls. The part's registers are readable, and every change is a
  read-modify-write, so the driver needs none.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `AudioCodec` trait this driver implements and the wiring record and register
  helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `I2cBus` and `I2s` traits and the board handles.
- [es8311-nv](https://novo-lang.org/packages/es8311-nv) drives Everest's mono
  codec, the same maker's smaller part.
- [wm8960-nv](https://novo-lang.org/packages/wm8960-nv) drives a stereo codec
  with the same trait.

## Tests

`novo test` runs the module over a double of the part, a register file on an
I2C bus that starts from the data sheet's defaults, resets to them on
SCPReset and whose log keeps every transaction, and a double of the I2S
handle: the framing of reads and writes, the reset, the clock plan at the
rows of Table 2 for 12.288, 11.2896, 18.432 and 16.9344 MHz, at the doubled
clocks and at the USB clock, and at every refusal, the plan for each format,
every step as an exact register log (the reset, the start, the format at 48
and 96 kHz, as master and slave, the power-up in the user guide's order
with its wait, and the power-down), the bits each write keeps, the mute,
both gains and the digital volumes at their steps and limits, the inputs, the
mixers, the ALC against the data sheet's codes, and the type over absent
hardware. Every refusal of a write at each position of the start, the format,
the power-up, the power-down, the output gain and the ALC is checked. The
suite covers every line of `src/`; `bash tests/coverage.sh` prints the
number.

**Hardware: NOT VERIFIED.** See the device program section.

The sources read for this driver, for their facts and not ported, are the two
Everest documents above, Espressif's `esp_codec_dev` ES8388 driver (the
successor of the esp-adf one) and the Linux kernel's `sound/soc/codecs/es8328.c`
for the ES8328, the same family. The fixes and differences kept are named in
`src/es8388.nv` where they are kept:

- The Linux driver's registers agree with the ES8388's up to 0x27 and from
  0x2E. From 0x28 to 0x2D the ES8328 has cross mixers and mono outputs where
  the ES8388 has the clock and offset registers, so nothing from that range
  is used.
- The reference capacitors are charged through the 5 kohm divider for 100 ms
  before the 50 kohm divider, as the Linux driver does ("charge caps"); the
  user guide goes to the 50 kohm divider at once.
- The digital volumes are set to 0 dB after the reset. The reset leaves them at
  -96 dB, and a recording made without setting them is silent.
- The DAC is muted before the converters are powered, and stays muted until
  the program unmutes it. esp_codec_dev unmutes at the end of its start.
- The mute bits are changed by a read-modify-write. The user guide writes
  0x30, 0x32, 0x34 and 0x36 whole, which sets bits the data sheet calls
  reserved.
- The ALC target and maximum gain follow the data sheet's codes. The user
  guide's recommended table gives 1100 for -4.5 dB and its sample code writes
  0xE2 beside "23.5 dB"; the data sheet gives 1000 and 101.
- esp_codec_dev's writes to 0x35, 0x37 and 0x39, "to disable the internal DLL
  to improve 8 kHz", are not made: no document names those registers.
- The double-speed bit is set above 50 kHz. esp_codec_dev and Linux leave it
  at single speed.
- The microphone gain is the same nibble for both channels, as esp_codec_dev
  does; the two channels can differ with `set_pga`.

## Licence

Apache-2.0.
