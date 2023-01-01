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
| `base + 0` |  R  | Status?
| `base + 0` |  W  | Reset
| `base + 1` |  W  | Data
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

Any write to the "reset" port returns to OPL3-compatible mode - it does not
appear to clear any registers, or stop sound output.

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
Currently, only registers `0x000` to `0x253` have a known purpose.

Each operator is composed of a block of 8 registers.  These are all ordered
sequentially, from `0x000` to `0x23f`.  The index for any given operator
register is calculated as follows:

```
    index = 32 * channel + 8 * operator + register
```

The key-on bits for each channel are in separate registers, from `0x240` to
`0x253`.  The last two channels use two key-on bits each, so that the envelope
generators for each pair of operators can be triggered individually.  It is not
known if it is possible to configure the modulation level on the third operator
as feedback, to produce a true 2-op mode.

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

#### Operator register 0

This register is the same as on OPL3.

Bits 0-3 (`MULT`) set the frequency multiplier.  Same as OPL3, with values 0,
11, 13 and 14 mapping to ½×, 10×. 12× and 15×, respectively.

Bit 4 (`KSR`) scales the envelope rate by pitch.  It is not known whether ESFM
has an `NTS` bit to configure how this scaling is calculated.

Bit 5 (`EGT`) determines the envelope generator type.  When set, the envelope
generator pauses after the decay stage until key-off.

Bit 6 (`VIB`) enables vibrato (pitch LFO).

Bit 7 (`TRM`) enables tremolo (amplitude LFO).

#### Operator register 1

This register is the same as OPL3.

Bits 0-5 (`ATTENUATION`) set the output level (inverted).

Bits 7-6 (`KSL`) scales the output level by pitch.  Note that the bit order is
reversed.

#### Operator register 2

This register is the same as OPL3.

Bits 0-3 (`DECAY`) set the envelope decay rate.

Bits 4-7 (`ATTACK`) set the envelope attack rate.

#### Operator register 3

This register is the same as OPL3.

Bits 0-3 (`RELEASE`) set the envelope release rate.

Bits 4-7 (`SUSTAIN`) set the envelope sustain level (inverted).

#### Operator register 4

On the OPL3, this is a per-channel register, instead of per-operator.

Bits 0-7 (`FNUM 0-7`) are the low 8 bits of the frequency number.

#### Operator register 5

Bits 0-1 (`FNUM 8-9`) are the high 2 bits of the frequency number.

Bits 2-4 (`BLOCK`) set the frequency number exponent.  Together with `FNUM`,
this makes up a floating-point number that determines the oscillator frequency,
which is calculated as:

```
    f = MULT * FNUM * 2**(BLOCK - 20) * 49716
```

Bits 5-7 (`DELAY`), when non-zero, add a delay of `2**(8 + n)` samples to the
envelope generator.

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
before it.  On the first operator of each channel, this field sets the feedback
level.

Bits 4 and 5 (`L`/`R`) enable output to the left and right speaker channels,
respectively.

Bit 6 (`VIBD`) determines pitch LFO depth.

Bit 7 (`TRMD`) determines amplitude LFO depth.

#### Operator register 7

Bits 0-2 (`WAVE`) select the waveform, same as on OPL3.

Bits 3-4 (`NOISE`) appear to be used for percussion-mode sounds.  These bits
only have any effect on the 4th operator in a channel.  When bit 4 is set, the
frequency setting of the 3rd operator also affects sound produced by the 4th,
even when the 3rd operator itself has no envelope set.

| `NOISE` | Sound type
| -------:|:----------
|       0 | Normal
|       1 | Snare drum?
|       2 | Top cymbal?
|       3 | Hihat?

Bits 5-7 (`OUT`) set the direct speaker output level for this operator.  How
this value corresponds with decibel levels is currently unknown.

### Key-on register

```
    ╔═══════╦═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╤═══════╗
    ║ R↓ B→ ║   7   │   6   │   5   │   4   │   3   │   2   │   1   │   0   ║
    ╠═══════╬═══════╧═══════╧═══════╧═══════╧═══════╧═══════╧═══════╪═══════╣
    ║   0   ║                           ?                           │  KEY  ║
    ╚═══════╩═══════════════════════════════════════════════════════╧═══════╝
```

On each of these registers, the first bit enables the envelope generator for
the associated channel.  The purpose of the other bits is currently unknown.
