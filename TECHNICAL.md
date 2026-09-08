# DeathEngine — Technical Notes

This is the deep-dive: how every stage works, what each control actually does
under the hood, measured CPU cost, and the full test suite. For install
instructions and a feature overview, see the [main README](../README.md).

---

A single-plugin modern metal rig. Guitar DI in, finished rhythm tone out — gate,
tightener, six pedals, high-gain preamp, power amp, cab, multiband control, width
and limiter, all in one window, all sharing one set of buffers.

It is not a model of a 5150, a Fortin, or anything else. It is a purpose-built
high-gain architecture aimed at one thing: modern thall / djent / industrial
rhythm tones that stay huge without turning to mush.

```
Input trim → Gate → Pick Attack → Downtune
    → ┌────────── 2x/4x/8x oversampled ──────────┐
      │ Sub → Grind → Chainsaw                   │      ← PRE pedals
      │ Tighten → Preamp → Filth                 │
      └──────────────────────────────────────────┘
    → Cab → Clarity Engine → Post EQ
    → Glitch → Delay → Reverb                           ← POST pedals
    → Width → Output → Limiter
```

Three pages, switched with the **AMP / PRE / POST** buttons in the header. The
meters, spectrum and preset bar stay put on all three. 46 factory presets in seven
categories, and you can save your own.

---

To install it, copy the `.vst3` folder into `C:\Program Files\Common Files\VST3`.

The standalone `.exe` needs no installation. Run it, hit **Options → Audio/MIDI
Settings**, pick your interface, and play.

---

## The controls that aren't obvious

Most of it is a normal amp. These five are the ones worth reading about.

### Tight

What a Tube Screamer in front of a high-gain amp is actually doing, as one knob.

Cutting low end *before* the clipper stops bass energy from modulating the whole
distortion — that is where chug mush comes from. Restoring some *after* keeps the
guitar from going thin. Turning this up moves the cut, the restore, the drive, the
clipper's knee and the transient emphasis together, because in practice you always
move them together anyway.

- **0%** — transparent, the amp sees your full low end
- **35%** — classic TS-in-front tightness
- **70%** — Vildhjarta-esk chug separation
- **100%** — clinical, percussive, almost gated-sounding attack

### Definition

Gain, but along the axis that actually matters.

The preamp is four cascaded stages, and each stage highpasses before it distorts.
Definition sweeps those corners together with the clipping knee. Low, the amp
distorts everything down to 8 Hz — huge, dark, woolly. High, the last stage only
sees content above ~270 Hz, so the low end reaches the power amp intact instead of
smeared through four nonlinearities.

- **Low** — huge, dark, dirty
- **High** — tight, articulate, modern

### Filth

The whole back half of the amp behind one knob. It is a macro, not a distortion
amount — six mechanisms move together, in the proportions they move in on a real
amp being pushed harder:

- power tube clipping gets a harder knee and more odd-order content
- crossover (class-B) distortion appears around the zero crossing
- the output transformer saturates, low end first
- highs roll off as the transformer runs out of headroom
- the speaker's cone breakup region starts distorting on its own
- power supply ripple modulates the signal when the amp is working hard

Low is a polished modern rig. The middle is where Humanity's Last Breath lives.
The top is industrial — falling apart on purpose.

**Sag** and **Bite** trim the supply droop and the crossover notch independently,
if the macro's proportions aren't what you want.

### Downtune

Polyphonic pitch shifting, ±24 semitones, voiced for guitar.

A plain phase vocoder shifted this far turns a guitar into mush for four specific
reasons, and each control addresses one:

| Control | Problem it solves |
|---|---|
| *(always on)* peak locking | Phasiness. Spectral peaks move as rigid blocks, so each partial keeps its shape instead of each bin drifting independently. |
| **Attack** | Smeared pick attacks. A spectral flux detector spots transients and re-locks the synthesis phase on those frames. |
| **Formant** | Lost body. Shifting the spectrum moves the guitar's and cab's resonances with the notes — tape-slowdown, not lower tuning. This measures the spectral envelope, shifts only the whitened spectrum, and puts the original envelope back. |
| **Clarity** | Fizz. Above ~4 kHz a distorted guitar is mostly inharmonic noise where the ear takes no pitch information. Everything above this corner stays unshifted. |
| **Sub Lock** | Hands some of the original low band back, so the low end doesn't move with the notes. |

Measured accuracy is within **1.6 cents** across the whole ±24 semitone range.

There is no honest way to make high-quality polyphonic shifting free, so latency
is a quality choice:

| Shift Quality | FFT | Latency @ 48 kHz |
|---|---|---|
| Live | 1024 | 768 smp — 16 ms |
| Studio | 2048 | 1536 smp — 32 ms |
| Ultra | 4096 | 3584 smp — 75 ms |

Latency is reported to the host, so delay compensation handles it on playback.
With **Engage** off, the shifter adds **zero** latency — it is fully bypassed, not
just muted.

### Clarity Engine

Five bands (`<90 | 90–250 | 250–900 | 900–3k | >3k`), each with its own compressor.

The premise: modern thall guitars are usually *less* distorted than people assume.
What makes them enormous without turning to mush is that each region of the
spectrum is held in place independently. One wideband compressor can't do that,
because whichever band is loudest ducks all the others with it.

**Amount** is a macro over the five band controls: 50% means "use the per-band
values as dialled", 100% doubles them, 0% switches the dynamics off and leaves
only the band trims.

The band split is a Linkwitz-Riley tree with allpass compensation on the lower
branches, so with everything flat the five bands sum back to the input with a flat
magnitude response. It is genuinely transparent when it isn't doing anything —
verified in the test suite.

### ARGENT

A voicing, not a "sound like Mick" button. It applies a fixed set of
offsets on top of whatever you have dialled in, pushing the whole chain toward the
industrial end: more weight underneath, less mid, a much dirtier power amp, the
cab resonance moved down, and the low band clamped hard so all that weight stays
put. **ARGENT Depth** scales the whole move, and past 50% it also forces the
Industrial preamp voice.

Your own settings are untouched — turn it off and you're exactly back where you were.

---

## Pedals

Six of them, on two pages. Where each one sits in the chain is not cosmetic — it
is most of why they work.

### PRE — in front of the amp

These run **inside the oversampled section**, so Grind's hard clipping aliases no
more than the preamp's does.

**SUB** — a synth octave under the riff. An analog-style flip-flop divider, not a
pitch tracker: the input is lowpassed to isolate the fundamental, squared up by a
Schmitt trigger, and halved. Dividing by two is exact by construction, and it
latches within one cycle instead of needing a window of audio. The tradeoff is
that dividers only track one note — which is correct here, because this exists to
put weight under single-note chugs. The square is heavily lowpassed on the way
out, so what you hear is closer to a sine than to a buzzy octave pedal.

**GRIND** — a boost, not an overdrive. Pulls the low end out before the amp sees
it, clips hard enough to flatten pick dynamics into a consistent wall, and pushes
a narrow band around 3 kHz so the attack survives the amp's own compression. That
last part is what separates it from a screamer: a screamer humps the mids broadly
around 700 Hz, which reads as "honk"; this reads as "grind".

**CHAINSAW** — a two-stroke engine keyed to your right hand, from an embedded
loop. Two things stop it sounding like a sound effect pasted over a guitar.

**Track** stops it droning: the saw only speaks while you play, and it revs
*harder* the harder you hit, like leaning a real saw into wood. **Rev** goes down
to a twelfth of the loop's natural speed, where it stops being a chainsaw and
becomes a slow mechanical grind under the riff.

**Sit** is the one that actually matters. On the records this is chasing you
*feel* the chainsaw far more than you hear it — turning Blend down does not
achieve that, it just gives you a quiet chainsaw that still pokes out. Sit splits
the saw into three bands, measures the guitar's energy in the same three bands,
and ducks each one independently. Where the guitar is dense the saw disappears;
in the gaps between chugs it swells back. At 100% that is about 6 dB of average
ducking and a lot more than that in whichever band the guitar is currently
occupying.

It also sits before the amp on purpose — running the saw through the same preamp,
power amp and cab as the guitar is what glues them together.

*Order is Sub → Grind → Chainsaw.* Sub goes first because the divider needs a
signal that has not been clipped yet. Grind then boosts guitar and sub together,
welding the sub to the riff. Chainsaw goes last so the saw hits the amp at full
level rather than being boosted into it.

### POST — after the cabinet

**GLITCH** — rhythmic destruction on a tempo grid. The reason production glitch
effects work and random ones do not is that they are quantised: events can only
start on a grid division, so whatever happens lands with the drums. **Chaos** sets
the probability that any grid slot fires — 100% is unlistenable on purpose; the
useful range is 15–40%, where it stays a performance rather than an effect. Five
behaviours, or Random to pick per event:

| Mode | What it does |
|---|---|
| Stutter | Repeats the last slice, subdividing faster as Depth rises |
| Reverse | Plays the slice backwards |
| Crush | Bit and sample-rate reduction |
| Tape Stop | Playback rate ramps to zero across the slot |
| Gate | Hard rhythmic chopping |

**DELAY** — tempo-syncable stereo delay with a ping-pong feedback path. Two
details make it usable on a distorted rhythm guitar rather than just adding wash:
the feedback path is band-limited on *both* ends (without the highpass, every
repeat stacks another chug's worth of 80 Hz and the low end turns to porridge),
and delay time is smoothed rather than snapped, so moving it pitch-bends the tail
like tape instead of clicking. With **Sync** on anything but Free, the Time knob
greys out — Sync overrides it.

**REVERB** — a 4-line feedback delay network, not a Freeverb comb bank. An FDN
with an orthonormal mixing matrix builds echo density much faster for the same
number of delay lines, which is what stops a reverb sounding grainy behind fast
palm mutes. Two choices exist specifically because this sits after a high-gain
amp: **Low Cut** is on the reverb's *input*, not its output (letting a distorted
guitar's low-mid energy into a tank is the fastest way to turn a heavy mix into
fog), and the delay lines are slowly modulated, because a static FDN has fixed
modes and a held chord will find them.

The wet path is level-matched by measurement: at full **Mix** a sustained source
comes back at about 1.5× the dry level. Only two of the four delay lines reach
each output and the tank is fed at half gain, so without compensating for that the
Mix control ran out of travel long before the reverb ran out of usefulness.

*Order is Glitch → Delay → Reverb.* Glitch is first so the delay and reverb catch
the pieces it throws; putting it last just sounds like someone unplugging a cable.

---

## Cabinets

Two slots, plus the parts of a mic'd cab that aren't in an IR at all: proximity
effect, distance rolloff, room reflections, cone resonance and air.

Each slot picks independently from:

- **Built-in Model** — an analytic 4×12: six biquads and a 4th-order lowpass. The
  lowpass matters most; the reason an amp without a cab sounds like a wasp in a
  jar is almost entirely the speaker's brick wall around 5 kHz, not the
  resonances below it. Slot A is voiced darker and slot B brighter, so the **A/B**
  blend does something useful even with no IR anywhere.
- **K1LLR · GR4V3 · SM0G · F4LL3N** — four IRs embedded in the plugin binary.
- **User IR** — anything you load with the **LOAD** button, which switches the
  slot to User IR automatically so the picker never disagrees with what is
  actually loaded. **X** clears it and reverts to the built-in model.

IRs are trimmed, normalised, resampled to the session rate and capped at 4096
samples. User IR paths are saved with the session and with user presets.

---

## Cost

Measured on my pc, one instance, one core, 48 kHz / 512 samples:

| Setting | CPU | Realtime factor |
|---|---|---|
| Eco (2×), amp only | 6.6 % | ×15 |
| Studio (4×), amp only | 9.6 % | ×10 |
| Insane (8×), amp only | 17.7 % | ×6 |
| Studio (4×) + shifter (Live) | 12.2 % | ×8 |
| Studio (4×) + shifter (Ultra) | 15.0 % | ×7 |
| **Studio (4×), all six pedals on** | **11.8 %** | ×8 |
| Studio (4×), pedals + shifter | 14.4 % | ×7 |
| Insane (8×), literally everything | 30.4 % | ×3 |

All six pedals together cost about **2%** on top of the amp. None of them is
expensive, and the ones that would be — Grind's clipper — are the only ones that
run oversampled.

Where the savings come from:

- **Only what aliases is oversampled.** The pre pedals, tightener, preamp and
  Filth run at 2–8×; the cab, clarity engine, post pedals, EQ and limiter run at
  base rate. Delay and reverb are linear — oversampling them would cost four
  times as much for no difference at all.
- **Bypassed modules cost nothing.** Every pedal returns immediately when it is
  off, rather than processing and mixing to zero.
- **Nothing is modelled at component level.** We are chasing a sound, not
  recreating a specific circuit, so a stage is a highpass, a gain, a shaped
  nonlinearity and a lowpass — not a nodal solve.
- **Coefficients only recook when a control moves.** Rebuilding every biquad on
  every 64-sample block would cost more than several of the modules it feeds.
- **Fast approximations in the hot loops.** The phase vocoder's `atan2`/`sin`/`cos`
  and the clarity engine's `log2`/`exp2` are polynomial approximations accurate to
  ~0.1%, which puts their error around 90 dB down.

**Oversampling quality:** Eco and Studio use a polyphase IIR half-band filter
(cheap, 4 and 6 samples latency). Insane switches to an equiripple FIR — linear
phase, no IIR ringing, 65 samples latency, and roughly double the CPU.

---

## Presets

### Factory

46 of them across seven categories. The preset menu opens into submenus, so you
pick a category and hover.

| Category | | What's in it |
|---|---|---|
| **Modern Metal** | 8 | The everyday tones — rhythm, lead, crunch, wide, big |
| **Argent** | 6 | Sub-driven industrial weight, chainsaw and octave layers |
| **Thall** | 7 | Dark, dissonant, bouncy, low — including a downshifted one |
| **Djent** | 6 | Tight and percussive, from clinical to gated |
| **Industrial** | 6 | Mechanical, glitched, corroded |
| **Crazy** | 8 | The experimental end. None of these are subtle |
| **Bass** | 5 | Same engine, very different voicing, for bass DI |


Most of them use the pedals: `Hell Gate` and `Machine Room` run the chainsaw
under the riff, `Titan Step` and `Sub Terror` lean on the octave divider,
`Assembly Line` and `Broken Transmission` are built around the glitch grid, and
`Cathedral` and `Chainsaw Church` push the reverb hard.

### Your own

**SAVE** writes the current state to a `.depreset` file; the preset menu lists
them under a *User* heading below the factory ones. **DEL** removes the loaded
one, and an **EDITED** flag appears next to the preset name as soon as any control
moves away from what was loaded.

```
%APPDATA%\DeathEngine\Presets\
```

They're plain XML, so they can be copied between machines, backed up, put in a
shared folder, or hand-edited — none of which is true of presets buried in a
binary blob. A saved preset is exactly the tree the host receives in
`getStateInformation`, so it captures everything: every parameter, both cab slots,
and any user IR paths.

---

## Honest notes

- **The tone stack is voiced, not simulated.** It behaves like a passive stack
  where it counts — the mid control moves the bass corner and its own width and
  centre — but it is not a bilinear transform of a specific component network. The
  plugin's whole premise is genre-first rather than circuit-first, so this was a
  deliberate choice.
- **The phase vocoder loses about 2 dB on broadband noise** at unity ratio. That
  is inherent: it estimates one frequency per bin, and for noise that estimate is
  random, so frames overlap-add slightly out of phase. On tonal material — which
  is what a guitar is — it round-trips at unity within 0.3%.
- **Stereo width is manufactured.** A DI is mono. The doubler is a real second
  "take" (20–26 ms delay with its own drift and tone), not a Haas trick, and
  everything below **Mono Below** is forced back to mono, because wide low end is
  the fastest way to make a heavy mix fall apart. (If you're using a VST guitar/bass, turn off width to get their actual stereo trough)
- **The Sub only tracks one note.** It is a divider, not a polyphonic tracker —
  on a chord it picks something and holds it. That is the right trade for what it
  is for (weight under single-note chugs), but it is not a pitch shifter.
- **Glitch is deterministic.** Its RNG is seeded identically every time, so a
  render is reproducible — but two instances with the same settings will glitch
  in the same places.
- **`/arch:AVX2` is on by default** (2013+ CPUs). 
- **JUCE licensing.** This builds against JUCE under its default terms with the
  splash screen enabled. If you want to remove it or ship this commercially, check
  the JUCE licence first — that is a decision about your project, not a build flag
  to flip quietly.

---

## Layout

```
IRs&SFX/                 embedded: 4 cab IRs + the chainsaw loop
Source/
  Params.h/.cpp          parameter IDs, ranges, layout
  PluginProcessor.*      chain assembly, oversampling, ARGENT voicing, state
  PluginEditor.*         window layout, page switching, preset UI
  dsp/
    DSPCommon.h          waveshapers, biquads, envelopes, fast maths
    Resources.*          access to the embedded IRs and chainsaw loop
    NoiseGate.*          program-dependent gate + pick attack enhancer
    Tightener.*          the Tight stage
    Preamp.*             four cascaded gain stages + tone stack
    FilthEngine.*        power amp, sag, transformer, speaker breakup
    Downtune.*           peak-locked phase vocoder
    CabSection.*         two slots: factory IR / user IR / analytic model
    ClarityEngine.*      five-band dynamics
    PostChain.*          post EQ, width/doubler, limiter
    Pedalboard.*         the two pedal chains and their order
    pedals/
      Grind.*            clipping boost
      Chainsaw.*         envelope-keyed sample blend
      SubSynth.*         flip-flop octave divider
      Glitch.*           tempo-quantised destruction
      Delay.*            tempo-syncable ping-pong delay
      Reverb.*           4-line FDN
  gui/
    Theme.*              colours, fonts, LookAndFeel
    Widgets.*            knobs, meters, spectrum, cab slots
    Panels.*             the amp page sections
    PedalWidgets.*       stompbox chassis, footswitch, animation hook
    PedalPanels.*        the six pedals and the two pedal pages
  presets/
    Presets.*            the 16 factory presets
    PresetManager.*      user preset files
Tests/
  DspTests.cpp           module-level numerical checks
  HostTests.cpp          processor-level checks + CPU benchmark
```
