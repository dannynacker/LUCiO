# LUCiO: Lucia Unified Control Interface for OctAVEs

**LUCiO** is an offline audio-to-stroboscopic-sourcecode pipeline for generating Lucia/RX1-compatible `.lscf` files from audio-derived control parameters. It converts an input audio file into a row-based Lucia sourcecode sequence in which audio features are mapped to oscillator frequency, luminance, and duty cycle.

The current validated implementation uses a **dynamic-duty, header-preserving export mode**. This means that a GUI-authored Lucia sourcecode file is used as a validated binary template, while the row sequence is programmatically replaced with audio-derived stimulation parameters.

## Validated Lucia Template

LUCiO currently requires a GUI-authored dynamic-duty `.lscf` template file:

```text
d_spcwkdc1020904022_various.lscf
```

This file acts as a validated header/template carrier. The pipeline preserves the template’s internal Lucia header identity and replaces only the sourcecode row content, then recomputes the final Lucia XOR checksum.

Because the template header is preserved, generated sourcecodes initially appear inside the Lucia software under the template’s internal display name:

```text
spcwkdc1020904022
```

This is expected. After loading the generated sourcecode in Lucia, it can be saved/resaved through the GUI as a standard session or session configuration with the desired final name.

## Lucia Folder Setup

On the Lucia USB/storage device, sourcecode files should be placed in:

```text
user/sourcecodes/
```

In the Lucia sourcecode browser, the files are accessed under the custom category/folder:

```text
D
```

and are usually saved/loaded under:

```text
various
```

In practice, generated files are copied to:

```text
USB Lucia/user/sourcecodes/d_<audio_name>_various.lscf
```

Then loaded in Lucia via:

```text
Session editor → D → various → spcwkdc1020904022
```

Because the internal display name is currently inherited from the validated template, it is best to copy and test **one generated `.lscf` file at a time**. Once loaded and verified, the file can be saved/resaved in Lucia under the desired final session/configuration name.

## Control Architecture

LUCiO generates Lucia sourcecode rows rather than streaming real-time framewise stimulation. Each row defines the target state for all four oscillators over a fixed control window.

The current validated settings are:

```text
Control step:              1.0 s
Parameter update rate:     1 Hz
Lucia row duration byte:   10 tenths = 1.0 s
Oscillators:               OSC1–OSC4 mapped identically
Halogen:                   off / not yet implemented
```

The important distinction is that **1 Hz refers to the parameter update rate**, not the flicker frequency. Within each 1-s row, Lucia generates flicker at the encoded oscillator frequency. The current sourcecode format therefore supports coarse-grained audio feature tracking rather than high-rate framewise modulation.

## Audio-to-Strobe Mapping

For each 1-s audio window, LUCiO extracts:

1. **Dominant Spectral Frequency**
   The dominant audio frequency is estimated within a configurable spectral band and folded by octave into the validated SLS stimulation range.

2. **SLS Oscillator Frequency**
   The folded SLS frequency is converted into Lucia oscillator cycle units. In the currently validated row grammar, frequency is represented as an integer number of cycles relative to a 2.5-s main-cycle representation, giving an approximate frequency resolution of 0.4 Hz.

3. **Luminance**
   Window-level RMS amplitude is normalized across the track and mapped to a bounded Lucia luminance range.

4. **Duty Cycle**
   Duty cycle is dynamically estimated from the within-window audio amplitude envelope. The default method is envelope occupancy: the proportion of the normalized 1-s envelope above a fixed threshold. This produces lower duty values for sparse/transient audio and higher duty values for sustained/dense audio.

The current default mappings are:

```text
SLS frequency range:       4.12–30.87 Hz
Luminance range:           5–50
Duty cycle range:          10–90
Duty method:               envelope occupancy
Duty occupancy threshold:  0.35
```

## Output Files

For each uploaded audio file, LUCIO-D exports:

```text
user/sourcecodes/d_<audio_name>_various.lscf
debug/<audio_name>_lucio_debug.csv
plots/<audio_name>_lucio_sequence_plot.png
plots/<audio_name>_lucio_sequence_plot.pdf
```

The `.lscf` file is copied to the Lucia sourcecode directory. The debug CSV stores the row-level mapping used to generate the sourcecode, including raw audio frequency, folded SLS frequency, achieved Lucia frequency, RMS amplitude, luminance, raw duty estimate, final duty value, and row timing information. The plots provide a compact visual summary of the generated sequence.

## Recommended Workflow

1. Open the LUCiO Colab notebook.
2. Upload the validated dynamic-duty template:

```text
d_spcwkdc1020904022_various.lscf
```

3. Upload one or more audio files.
4. Run the notebook to generate `.lscf`, debug CSV, and sequence plot outputs.
5. Download the generated ZIP.
6. Copy one generated `.lscf` file at a time into:

```text
USB Lucia/user/sourcecodes/
```

7. In Lucia, load:

```text
Session editor → D → various → spcwkdc1020904022
```

8. Verify playback.
9. Save/resave the loaded sourcecode as a normal Lucia session or session configuration using the desired final name.

## Current Validation Status

Validated in LUCiO v0.1:

```text
Full-song .lscf generation
1-s row-based sourcecode control
Dynamic oscillator frequency
Dynamic oscillator luminance
Dynamic oscillator duty cycle
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
Internal-name rewriting for dynamic-duty sourcecodes is not yet solved
```

Planned extensions:

```text
LUCIO v0.2: dynamic-duty-compatible internal-name/header rewriting
LUCIO-H: halogen/wash-light layer
LUCIO-M: multi-oscillator or multi-band audio mappings
```

## Methods

LUCiO implements an offline sourcecode-generation interface for Lucia/RX1-compatible stroboscopic stimulation. Audio files are segmented into 1-s windows, and each window is converted into a Lucia sourcecode row specifying oscillator frequency, luminance, and duty cycle. Dominant audio frequencies are folded by octave into the SLS stimulation range and encoded as Lucia oscillator cycle counts. Window-level RMS amplitude is mapped to luminance, while duty cycle is estimated from amplitude-envelope occupancy within each window. The resulting `.lscf` file preserves a validated GUI-authored dynamic-duty header and replaces the row sequence with audio-derived stimulation parameters. Generated files are viewable and playable in Lucia and can be resaved through the GUI as standard session/session configuration files.

## Technical Notes

The validated `.lscf` structure used by LUCIO-D follows:

```text
Header:     132 bytes
Rows:       56 bytes each
Checksum:   final 1-byte XOR checksum
```

The final byte is recomputed as the XOR of all preceding bytes. In the current dynamic-duty mode, the header is preserved exactly from the validated GUI-authored template, and only the row sequence plus final XOR checksum are regenerated.

Each 56-byte row contains four oscillator blocks, global oscillator luminance/activation fields, timing/control bytes, and loop information. LUCIO-D currently writes identical frequency, luminance, and duty values to OSC1–OSC4.

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

## Citation / Credit

LUCiO was developed as part of the octAVEs-to-stroboscopic-light workflow for generating Lucia/RX1-compatible sourcecodes from audio-derived stimulation features. The current implementation is reverse-engineered and validated empirically through GUI loading/playback tests and should be treated as a practical compatibility layer rather than an official Lucia/RX1 specification.
