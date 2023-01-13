# ESFM Demystified

This projects aims to reverse-engineer and document the register map of ESFM,
the proprietary FM synthesizer found on ESS AudioDrive sound cards.

ESFM is often considered a "clone" of the once-ubiquitous and well-documented
Yamaha OPL3 (YMF262).  Many such clones exist, and all produce far inferior
sound quality to the original part.

ESFM is different.  It is indeed fully compatible with the OPL3, and in most
cases sounds virtually identical to it.  But what really sets it apart is its
"native" mode, where it offers many more features, none of which were ever
publicly documented.

To my knowledge, the only existing programs that are able to use these features,
are two basic MIDI drivers; one for Windows 95, and another for Miles Sound
System v3.  Both of these can only load from a small bank of hard-coded presets,
and do not provide any editing capabilities.  Hopefully, the release of this
document will eventually change that.

Currently ESFM is known to have 72 oscillators, or "operators", *twice* as many
as the OPL3.  These are configured as 18 separate 4-op channels.  On two
channels, the envelope generators can be triggered individually for each
operator pair.

Each operator has *individual frequency controls*.  Likewise for LFO depth.
Each also has a configurable envelope delay.  And there are no pre-set 4-op
configurations, as on the OPL3.  Instead, ESFM features separate controls for
output and modulation level on each operator - one operator can both produce
sound, and modulate another operator, simultaneously.

You know you want to use it now...  Let's figure out how!

## Accessing

The following 8-bit I/O ports are used in ESFM mode:

|     Port   | R/W | Description
| ----------:|:---:|:-----------
| `base + 0` |  R  | Status
| `base + 0` |  W  | Reset
| `base + 1` | R/W | Data
| `base + 2` |  W  | Index low
| `base + 3` |  W  | Index high

Where `base` is either `0x388` (Adlib base), or `0x220`, `0x240`, etc (SB base).

Registers are accessed by first writing a 16-bit index to the low and high
index ports (it is not possible to write these simultaneously).  Then register
values may be written through the data port.  Some delay is needed between
accessing the index and data ports, but the minimum required delay is currently
not known.  Between writing the low and high index registers, no delay is
necessary.

```c
    out(base + 2, index & 0xff);
    out(base + 3, index >> 8);
    delay();
    out(base + 1, data);
    delay();
```

Similar to the YMF71x chips, it is possible to read back register values from
the data port.

Any write to the "reset" port returns to OPL3-compatible mode - it does not
appear to clear any registers, or stop sound output.

### Status port

```
    ╔═══════╦═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╗
    ║ P↓ B→ ║   7   │   6   │   5   │   4   │   3   │   2   │   1   │   0   ║
    ╠═══════╬═══════╪═══════╪═══════╪═══════╧═══════╧═══════╧═══════╧═══════╣
    ║ base  ║  IRQ  │  FT1  │  FT2  │                   ?                   ║
    ╚═══════╩═══════╧═══════╧═══════╧═══════════════════════════════════════╝
```

The status port layout appears to be the same as in OPL3 mode.

Bit 5 (`FT2`) is set when timer 2 overflows.

Bit 6 (`FT1`) is set when timer 1 overflows.

Bit 7 (`IRQ`) is set when either of the timers overflow, and their respective
mask bits are not set.  This bit is not connected to any IRQ line.

### Initialization

With the chip in OPL2 or OPL3 mode, native ESFM mode is enabled by setting the
most significant bit in OPL3 register `0x105`:

```c
    out(base + 2, 0x05);
    delay();
    out(base + 3, 0x80);
    delay();
```

If ESFM mode is already active, this only writes the index ports, so this
sequence has no effect.

## Registers

The total address space is 11 bits - valid indices are `0x000` to `0x7ff`.
Currently, the following registers have been discovered:

|      Index      | Purpose
|:---------------:|:----------------------
| `0x000`-`0x23f` | Operator registers
| `0x240`-`0x253` | Key-on registers
| `0x402`-`0x404` | Timer registers
|     `0x408`     | Configuration register
|     `0x501`     | Test register

Registers in the range `0x400`-`0x5ff` are duplicated in `0x600`-`0x7ff`.

### Operator registers

```
    ╔═══════╦═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╗
    ║ R↓ B→ ║   7   │   6   │   5   │   4   │   3   │   2   │   1   │   0   ║
    ╠═══════╬═══════╪═══════╪═══════╪═══════╪═══════╧═══════╧═══════╧═══════╣
    ║   0   ║  TRM  │  VIB  │  EGT  │  KSR  │             MULT              ║
    ╟───────╫───────┴───────┼───────┴───────┴───────────────────────────────╢
    ║   1   ║      KSL      │                  ATTENUATION                  ║
    ╟───────╫───────────────┴───────────────┬───────────────────────────────╢
    ║   2   ║            ATTACK             │             DECAY             ║
    ╟───────╫───────────────────────────────┼───────────────────────────────╢
    ║   3   ║            SUSTAIN            │            RELEASE            ║
    ╟───────╫───────────────────────────────┴───────────────────────────────╢
    ║   4   ║                            FNUM 0-7                           ║
    ╟───────╫───────────────────────┬───────────────────────┐               ║
    ║   5   ║         DELAY         │         BLOCK         │    FNUM 8-9   ║
    ╟───────╫───────┬───────┬───────┼───────┬───────────────┴───────┬───────╢
    ║   6   ║ TRMD  │ VIBD  │   R   │   L   │          MOD          │   ?   ║
    ╟───────╫───────┴───────┴───────┼───────┴───────┬───────────────┴───────╢
    ║   7   ║          OUT          │     NOISE     │          WAVE         ║
    ╚═══════╩═══════════════════════╧═══════════════╧═══════════════════════╝
```

Each operator is composed of a block of 8 registers.  These are all ordered
sequentially, from `0x000` to `0x23f`.  The index for any given operator
register is calculated as follows:

```
    index = 32 * channel + 8 * operator + register
```

#### Operator register 0

This register is the same as `0x20`-`0x35` on OPL3.

Bits 0-3 (`MULT`) set the frequency multiplier.  Same as OPL3, with values 0,
11, 13 and 14 mapping to ½×, 10×. 12× and 15×, respectively.

Bit 4 (`KSR`) scales the envelope rate by pitch.

Bit 5 (`EGT`) determines the envelope generator type.  When set, the envelope
generator pauses after the decay stage until key-off.

Bit 6 (`VIB`) enables vibrato (pitch LFO).

Bit 7 (`TRM`) enables tremolo (amplitude LFO).

#### Operator register 1

This register is the same as `0x40`-`0x55` on OPL3.

Bits 0-5 (`ATTENUATION`) set the output level in steps of -0.75 dB.

Bits 7-6 (`KSL`) scales the output level by pitch.  Note that the bit order is
reversed.

#### Operator register 2

This register is the same as `0x60`-`0x75` on OPL3.

Bits 0-3 (`DECAY`) set the envelope decay rate.

Bits 4-7 (`ATTACK`) set the envelope attack rate.

#### Operator register 3

This register is the same as `0x80`-`0x95` on OPL3.

Bits 0-3 (`RELEASE`) set the envelope release rate.

Bits 4-7 (`SUSTAIN`) set the envelope sustain level (inverted).

#### Operator register 4

On the OPL3, this is a per-channel register in `0xa0`-`0xa8`, instead of
per-operator.

Bits 0-7 (`FNUM 0-7`) are the low 8 bits of the frequency number.

#### Operator register 5

Bits 0-1 (`FNUM 8-9`) are the high 2 bits of the frequency number.

Bits 2-4 (`BLOCK`) set the frequency number exponent.  Together with `FNUM`,
this makes up a floating-point number that determines the oscillator frequency,
which is calculated as:

```
    f = MULT * FNUM * 2**(BLOCK - 20) * 49716
```

Bits 5-7 (`DELAY`), when non-zero, add a delay of `2**(DELAY + 8)` samples to
the key-on trigger, delaying both the envelope and phase generators.  Key-off
is not affected.

| `DELAY` | Samples | Milliseconds
| -------:| -------:| ------------:
|       0 |       0 |          0.0
|       1 |     512 |         10.3
|       2 |    1024 |         20.6
|       3 |    2048 |         41.2
|       4 |    4096 |         82.4
|       5 |    8192 |        164.8
|       6 |   16384 |        329.6
|       7 |   32768 |        659.1

#### Operator register 6

Bit 0 (`?`) does not appear to serve any purpose.

Bits 1-3 (`MOD`) determines how much this operator is modulated by the operator
before it, in steps of 6 dB.  On the first operator of each channel, this field
sets the feedback level.

| `MOD` | Modulator input
| -----:| ---------------:
|     0 |           -∞ dB
|     1 |          -36 dB
|     2 |          -30 dB
|     3 |          -24 dB
|     4 |          -18 dB
|     5 |          -12 dB
|     6 |           -6 dB
|     7 |            0 dB

Bits 4 and 5 (`L`/`R`) enable output to the left and right speaker channels,
respectively.  Both these bits are set on power-on.

Bit 6 (`VIBD`) determines pitch LFO depth.

Bit 7 (`TRMD`) determines amplitude LFO depth.

#### Operator register 7

Bits 0-2 (`WAVE`) select the waveform, same as on OPL3.

Bits 3-4 (`NOISE`) appear to be used for percussion-mode sounds.  These bits
only have any effect on the 4th operator in a channel.  When bit 4 is set, the
frequency setting of the 3rd operator also affects sound produced by the 4th,
even when the 3rd operator itself has no envelope set.

| `NOISE` | OPL3 rhythm-mode sound
| -------:|:----------------------
|       0 | Melodic
|       1 | Snare drum
|       2 | Hi-hat
|       3 | Top cymbal

Bits 5-7 (`OUT`) set the direct speaker output level for this operator, in
steps of 6 dB.

| `OUT` | Output level
| -----:| ------------:
|     0 |        -∞ dB
|     1 |       -36 dB
|     2 |       -30 dB
|     3 |       -24 dB
|     4 |       -18 dB
|     5 |       -12 dB
|     6 |        -6 dB
|     7 |         0 dB

### Key-on register

```
    ╔═══════╦═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╗
    ║ R↓ B→ ║   7   │   6   │   5   │   4   │   3   │   2   │   1   │   0   ║
    ╠═══════╬═══════╧═══════╧═══════╧═══════╧═══════╧═══════╪═══════╪═══════╣
    ║   0   ║                       ?                       │   ?   │  KEY  ║
    ╚═══════╩═══════════════════════════════════════════════╧═══════╧═══════╝
```

The key-on bits for each channel are in separate registers, from `0x240` to
`0x253`.  The last two channels use two key-on registers each, so that the
envelope generators for each pair of operators can be triggered individually.
These should not be considered four independent 2-op channels, however.  The
third operator still receives modulation input from the second, and it does not
appear to be possible to reconfigure this as feedback.

|      Index      | Channel
|:---------------:|:----------------
| `0x240`-`0x24f` | 0 - 15
|     `0x250`     | 16, operator 0-1
|     `0x251`     | 16, operator 2-3
|     `0x252`     | 17, operator 0-1
|     `0x253`     | 17, operator 2-3

On each of these registers:

Bit 0 (`KEY`) triggers key-on for the associated channel, starting the envelope
generator and resetting the signal phase.

Bit 1 (`?`) is writable, but its purpose is unknown.

### Timer registers

```
    ╔═══════╦═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╗
    ║ R↓ B→ ║   7   │   6   │   5   │   4   │   3   │   2   │   1   │   0   ║
    ╠═══════╬═══════╧═══════╧═══════╧═══════╧═══════╧═══════╧═══════╧═══════╣
    ║ 0x402 ║                             TIMER1                            ║
    ╟───────╫───────────────────────────────────────────────────────────────╢
    ║ 0x403 ║                             TIMER2                            ║
    ╟───────╫───────┬───────┬───────┬───────────────────────┬───────┬───────╢
    ║ 0x404 ║  RST  │  MT1  │  MT2  │           ?           │  ST2  │  ST1  ║
    ╚═══════╩═══════╧═══════╧═══════╧═══════════════════════╧═══════╧═══════╝
```

These registers are the same as `0x02`-`0x04` in the OPL3.  Both timers count
up from their programmed initial counts and set their corresponding flags in
the status port on overflow.

#### Timer register `0x402`

Bits 0-7 (`TIMER1`) set the initial count for timer 1.

#### Timer register `0x403`

Bits 0-7 (`TIMER2`) set the initial count for timer 2.

#### Timer register `0x404`

Bit 0 (`ST1`) starts timer 1, with a 80μs tick interval.

Bit 1 (`ST2`) starts timer 2, with a 320μs tick interval.

Bit 5 (`MT2`) masks the timer output so that `FT2` does not trigger `IRQ` in
the status port.

Bit 6 (`MT1`) masks the timer output so that `FT1` does not trigger `IRQ` in
the status port.

Bit 7 (`RST`) resets all timer flags in the status port.  This bit does not
stick.

### Configuration registers

```
    ╔═══════╦═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╗
    ║ R↓ B→ ║   7   │   6   │   5   │   4   │   3   │   2   │   1   │   0   ║
    ╠═══════╬═══════╪═══════╪═══════╧═══════╧═══════╧═══════╧═══════╧═══════╣
    ║ 0x408 ║   ?   │  NTS  │                       ?                       ║
    ╚═══════╩═══════╧═══════╧═══════════════════════════════════════════════╝
```

#### Configuration register `0x408`

Bit 6 (`NTS`) determines how envelope rate scaling is calculated.  When set,
`FNUM` bit 9 is used.  When clear, `FNUM` bit 8 is used.
Note: this matches the description in the OPL3 datasheet, but is *inverted*
from how it is actually implemented on the OPL3.

### Test registers

```
    ╔═══════╦═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╗
    ║ R↓ B→ ║   7   │   6   │   5   │   4   │   3   │   2   │   1   │   0   ║
    ╠═══════╬═══════╪═══════╪═══════╪═══════╪═══════╧═══════╪═══════╪═══════╣
    ║ 0x501 ║   ?   │   X   │   ?   │   X   │       ?       │   X   │   ?   ║
    ╚═══════╩═══════╧═══════╧═══════╧═══════╧═══════════════╧═══════╧═══════╝
```

#### Test register `0x501`

Bit 1 produces a severely distorted sound.

Bit 4 reduces the output level by about -3dB.

Bit 6 disables sound output.

Setting bits 1 and 6 together produces a loud popping noise.
