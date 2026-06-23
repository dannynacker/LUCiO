# LUCiO: Lucia Unified Control Interface for OctAVEs

**LUCiO** is an offline sourcecode-generation interface for creating Lucia/RX1-compatible `.lscf` files. It provides two complementary workflows:

```text
LUCiO-Audio      audio-driven sourcecode generation
LUCiO-Composer   typed/manual sourcecode composition
```

Both modes generate row-based Lucia sourcecodes in which oscillator frequency, luminance, and duty cycle are written into Lucia-compatible `.lscf` files.

LUCiO-Audio derives stimulation parameters from an uploaded audio file. LUCiO-Composer does not use audio; instead, users define a timed sequence directly by specifying duration, frequency, luminance, and duty-cycle values for OSC1–OSC4.

The current validated implementation uses a **dynamic-duty, header-preserving export mode**. This means that a GUI-authored Lucia sourcecode file is used as a validated binary template, while the sourcecode rows are programmatically replaced with newly generated stimulation parameters.

## Validated Lucia Template

LUCiO currently requires a GUI-authored dynamic-duty `.lscf` template file:

```text
d_spcwkdc1020904022_various.lscf
```

This file acts as a validated header/template carrier. LUCiO preserves the template’s internal Lucia header identity, replaces the row content, and recomputes the final Lucia XOR checksum.

Because the template header is preserved, generated sourcecodes initially appear inside Lucia under the template’s internal display name:

```text
spcwkdc1020904022
```

This is expected. After loading the generated sourcecode in Lucia, it can be saved/resaved through the GUI as a standard session or session configuration with the desired final name.

## Lucia Folder Setup

On the Lucia USB/storage device, generated sourcecode files should be placed in:

```text
user/sourcecodes/
```

In the Lucia sourcecode browser, the files are accessed under the custom category/folder:

```text
D
```

and are usually loaded/saved under:

```text
various
```

In practice, generated files are copied to:

```text
USB Lucia/user/sourcecodes/d_<name>_various.lscf
```

Then loaded in Lucia via:

```text
Session editor → D → various → spcwkdc1020904022
```

Because the internal display name is currently inherited from the validated template, it is best to copy and test **one generated `.lscf` file at a time**. Once loaded and verified, the file can be saved/resaved in Lucia under the desired final session/configuration name.

## Modes

### LUCiO-Audio

LUCiO-Audio converts an uploaded audio file into a row-based Lucia sourcecode. For each control window, the notebook estimates audio-derived stimulation parameters and writes them into OSC1–OSC4.

For each uploaded audio file, LUCiO-Audio exports:

```text
user/sourcecodes/d_<audio_name>_various.lscf
debug/<audio_name>_lucio_debug.csv
plots/<audio_name>_lucio_sequence_plot.png
plots/<audio_name>_lucio_sequence_plot.pdf
```

The debug CSV stores the row-level mapping used to generate the sourcecode, including raw audio frequency, folded SLS frequency, achieved Lucia frequency, RMS amplitude, luminance, raw duty estimate, final duty value, and row timing information.

### LUCiO-Composer

LUCiO-Composer generates Lucia sourcecodes from a typed/manual sequence rather than from audio. Users define a list of timed steps, specifying frequency, luminance, and duty cycle for OSC1–OSC4.

A Composer step can use fixed values:

```python
{
    "label": "alpha_lock",
    "duration": 20,
    "osc1": osc(freq=10, lum=40, duty=50),
    "osc2": osc(freq=10, lum=40, duty=50),
    "osc3": off(),
    "osc4": off(),
}
```

or author-level ramps:

```python
{
    "label": "rise",
    "duration": 30,
    "osc1": ramp(freq=[8, 15], lum=[10, 70], duty=[30, 75]),
    "osc2": ramp(freq=[15, 8], lum=[70, 10], duty=[75, 30]),
    "osc3": osc(freq=60, lum=10, duty=50),
    "osc4": osc(freq=60, lum=10, duty=50),
}
```

Lucia itself does not store native start/end ramp values per row. Instead, LUCiO-Composer expands `[start, end]` ramps into discrete 1-s Lucia rows before export. Each exported Lucia row contains one target state per oscillator.

For each scripted sequence, LUCiO-Composer exports:

```text
user/sourcecodes/d_<sequence_name>_various.lscf
debug/<sequence_name>_lucio_composer_debug.csv
plots/<sequence_name>_lucio_composer_plot.png
plots/<sequence_name>_lucio_composer_plot.pdf
```

## Control Architecture

LUCiO generates Lucia sourcecode rows rather than streaming real-time framewise stimulation. Each row defines the target state for the oscillators over a fixed control window.

The current validated settings are:

```text
Control step:              1.0 s
Parameter update rate:     1 Hz
Lucia row duration byte:   10 tenths = 1.0 s
Halogen:                   off / not yet implemented
```

The important distinction is that **1 Hz refers to the parameter update rate**, not the flicker frequency. Within each 1-s row, Lucia generates flicker at the encoded oscillator frequency. The current sourcecode format therefore supports row-based parameter control rather than high-rate framewise modulation.

In LUCiO-Composer, longer authored steps are expanded into multiple 1-s sourcecode rows. For example, a 30-s ramp becomes 30 discrete Lucia rows.

## Audio-to-Strobe Mapping

For each 1-s audio window, LUCiO-Audio extracts:

1. **Dominant Spectral Frequency**
   The dominant audio frequency is estimated within a configurable spectral band and folded by octave into the validated SLS stimulation range.

2. **SLS Oscillator Frequency**
   The folded SLS frequency is converted into Lucia oscillator cycle units. In the currently validated row grammar, frequency is represented as an integer number of cycles relative to a 2.5-s main-cycle representation, giving an approximate frequency resolution of 0.4 Hz.

3. **Luminance**
   Window-level RMS amplitude is normalized across the track and mapped to a bounded Lucia luminance range.

4. **Duty Cycle**
   Duty cycle is dynamically estimated from the within-window audio amplitude envelope. The default method is envelope occupancy: the proportion of the normalized 1-s envelope above a fixed threshold. This produces lower duty values for sparse/transient audio and higher duty values for sustained/dense audio.

The current default LUCiO-Audio mappings are:

```text
SLS frequency range:       4.12–30.87 Hz
Luminance range:           5–50
Duty cycle range:          10–90
Duty method:               envelope occupancy
Duty occupancy threshold:  0.35
```

## Composer Mapping

LUCiO-Composer writes frequency, luminance, and duty values directly from the scripted sequence.

Each oscillator can be controlled independently:

```text
OSC1 frequency / luminance / duty
OSC2 frequency / luminance / duty
OSC3 frequency / luminance / duty
OSC4 frequency / luminance / duty
```

Values can be fixed or ramped. Ramped values are expanded by LUCiO into discrete rows before export.

Composer parameters are clipped to the currently validated bounds:

```text
Frequency range:           0–60 Hz
Luminance range:           0–100
Duty cycle range:          10–90
```

A luminance value of 0 is used for dark/off states. The current validated exporter keeps oscillator blocks structurally active and uses luminance to determine whether visible output is present.

## Recommended Workflow

1. Open the LUCiO Colab notebook.
2. Choose either LUCiO-Audio or LUCiO-Composer.
3. Upload the validated dynamic-duty template:

```text
d_spcwkdc1020904022_various.lscf
```

4. For LUCiO-Audio, upload one or more audio files.
5. For LUCiO-Composer, edit the `SEQUENCE_NAME` and `STEPS` section.
6. Run the notebook to generate `.lscf`, debug CSV, and sequence plot outputs.
7. Download the generated ZIP.
8. Copy one generated `.lscf` file at a time into:

```text
USB Lucia/user/sourcecodes/
```

9. In Lucia, load:

```text
Session editor → D → various → spcwkdc1020904022
```

10. Verify playback.
11. Save/resave the loaded sourcecode as a normal Lucia session or session configuration using the desired final name.

## Current Validation Status

Validated in LUCiO v0.1:

```text
Full-song .lscf generation
Typed/manual .lscf generation
1-s row-based sourcecode control
Dynamic oscillator frequency
Dynamic oscillator luminance
Dynamic oscillator duty cycle
Independent OSC1–OSC4 control in Composer mode
All-four-oscillator output
Template-header-preserving dynamic-duty mode
Viewable in Lucia sourcecode browser
Playable in Lucia session editor
Resavable as a standard Lucia session/session configuration
```

Current limitations:

```text
Generated files initially inherit the template’s internal Lucia display name
Only one generated sourcecode should be copied/tested at a time
Halogen/wash-light mapping is not yet implemented
Control updates are row-based at 1 Hz, not framewise or streamed
Composer ramps are pre-expanded into discrete rows
Lucia rows store target states, not native start/end ramp values
Internal-name rewriting for dynamic-duty sourcecodes is not yet solved
```

Planned extensions:

```text
LUCiO v0.2: dynamic-duty-compatible internal-name/header rewriting
LUCiO-H: halogen/wash-light layer
LUCiO-M: multi-band or multi-feature audio mappings
LUCiO-Composer extensions: reusable sequence libraries and named presets
```

## Methods

LUCiO implements an offline sourcecode-generation interface for Lucia/RX1-compatible stroboscopic stimulation. The current implementation uses a validated GUI-authored dynamic-duty `.lscf` file as a header/template carrier. The template header is preserved, the sourcecode rows are programmatically regenerated, and the final XOR checksum is recomputed.

In LUCiO-Audio, audio files are segmented into 1-s windows, and each window is converted into a Lucia sourcecode row specifying oscillator frequency, luminance, and duty cycle. Dominant audio frequencies are folded by octave into the SLS stimulation range and encoded as Lucia oscillator cycle counts. Window-level RMS amplitude is mapped to luminance, while duty cycle is estimated from amplitude-envelope occupancy within each window.

In LUCiO-Composer, users define a timed sequence directly. Each authored step specifies fixed or ramped frequency, luminance, and duty-cycle values for OSC1–OSC4. Ramps are expanded into one discrete Lucia row per control step before export. Generated files are viewable and playable in Lucia and can be resaved through the GUI as standard session/session configuration files.

## Technical Notes

The validated `.lscf` structure used by LUCiO follows:

```text
Header:     132 bytes
Rows:       56 bytes each
Checksum:   final 1-byte XOR checksum
```

The final byte is recomputed as the XOR of all preceding bytes. In the current dynamic-duty mode, the header is preserved exactly from the validated GUI-authored template, and only the row sequence plus final XOR checksum are regenerated.

Each 56-byte row contains four oscillator blocks, global oscillator luminance/activation fields, timing/control bytes, and loop information.

Duty-cycle bytes are written directly as decimal percentage values in the valid Lucia GUI range of 10–90. For all-four-oscillator rows, these are located at row-relative offsets:

```text
OSC1 duty:  3
OSC2 duty:  11
OSC3 duty:  19
OSC4 duty:  27
```

The currently validated timing/control convention uses:

```text
row duration: 1.0 s
row[48]:      10
loops:        1
```

Frequency is encoded as an integer cycle count relative to the row’s main-cycle representation. In the currently validated template, this gives an approximate frequency resolution of 0.4 Hz.

## Credit

LUCiO was developed as part of the octAVEs-to-stroboscopic-light workflow for generating Lucia/RX1-compatible sourcecodes from audio-derived and manually authored stimulation features. The current implementation is reverse-engineered and validated empirically through GUI loading/playback tests and should be treated as a practical compatibility layer rather than an official Lucia/RX1 specification.

LUCiO was developed as part of the octAVEs-to-stroboscopic-light workflow for generating Lucia/RX1-compatible sourcecodes from audio-derived and manually authored stimulation features. The current implementation is reverse-engineered and validated empirically through GUI loading/playback tests and should be treated as a practical compatibility layer rather than an official Lucia/RX1 specification.
