# SHAC Encoding Guide

**How to create SHAC files programmatically**

This guide covers the practical aspects of encoding audio into SHAC format using the Python codec implementation.

---

## Quick Start

```python
from shac.codec import SHACCodec, SHACFileWriter
from shac.codec.encoders import encode_mono_source, convert_to_spherical
from shac.codec.math_utils import AmbisonicNormalization
import numpy as np

# Load your audio (mono, 48kHz recommended)
audio = load_audio_file("kick.wav")  # Returns numpy array

# Create SHAC writer
writer = SHACFileWriter(
    order=3,  # Ambisonic order (3 = 16 channels, recommended)
    sample_rate=48000,
    normalization=AmbisonicNormalization.SN3D
)

# Position the source (x, y, z in meters)
position = (2.0, 0.0, 3.0)  # 2m right, 0m vertical, 3m forward

# Convert to spherical coordinates
spherical_pos = convert_to_spherical(position)

# Encode to ambisonics
ambisonic_audio = encode_mono_source(
    audio,
    spherical_pos,
    order=3,
    apply_distance_gain=False
)

# Add layer to SHAC file
metadata = {
    'position': position,
    'type': 'kick_drum',
    'gain': 1.0
}
writer.add_layer("kick", ambisonic_audio, metadata)

# Write the file
writer.write_file("output.shac")
```

---

## Installation

```bash
# Install dependencies
pip install numpy scipy

# Import SHAC codec
# (assumes codec is in your Python path or installed package)
from shac.codec import SHACCodec, SHACFileWriter
```

---

## Core Concepts

### Audio Format Requirements

**Input audio must be:**
- **Mono** (single channel) - SHAC encodes mono sources to spatial positions
- **NumPy array** of float32 or float64 values
- **Sample rate** matching your target SHAC sample rate (48kHz recommended)
- **Normalized** to [-1.0, 1.0] range

If you have stereo audio:
```python
# Convert stereo to mono (mix to center)
stereo_audio = load_stereo("file.wav")  # Shape: (samples, 2)
mono_audio = np.mean(stereo_audio, axis=1)  # Average L+R

# Or preserve just left channel
mono_audio = stereo_audio[:, 0]
```

### Coordinate System

Positions use **Cartesian coordinates** in meters:

```
         +Y (up)
          |
          |
          |_____ +X (right)
         /
        /
      +Z (forward)
```

- **X:** Negative = left, Positive = right
- **Y:** Negative = down, Positive = up
- **Z:** Negative = back, Positive = forward
- **Origin (0, 0, 0):** Listener position

Examples:
- `(0, 0, 0)` - At listener (center)
- `(3, 0, 0)` - 3 meters to the right
- `(0, 2, 0)` - 2 meters above
- `(0, 0, 5)` - 5 meters in front
- `(-2, 0, 3)` - 2m left, 3m forward

### Ambisonic Order

Higher order = better spatial resolution, but larger files:

| Order | Channels | File Size Impact | Use Case |
|-------|----------|------------------|----------|
| 1 | 4 | Smallest | Basic spatial positioning |
| 2 | 9 | Medium | Improved localization |
| **3** | **16** | **Recommended** | Professional music production |
| 4 | 25 | Large | Very high precision |
| 7 | 64 | Maximum | Research/experimental |

**Default: Order 3** provides excellent spatial accuracy without excessive file size.

---

## API Reference

### SHACFileWriter

Main class for creating SHAC files.

```python
from shac.codec import SHACFileWriter
from shac.codec.math_utils import AmbisonicNormalization

writer = SHACFileWriter(
    order=3,
    sample_rate=48000,
    normalization=AmbisonicNormalization.SN3D
)
```

**Parameters:**
- `order` (int): Ambisonic order (1-7), default 3
- `sample_rate` (int): Audio sample rate in Hz (e.g., 48000)
- `normalization` (AmbisonicNormalization): SN3D (standard) or N3D

**Methods:**

#### `add_layer(layer_id, ambisonic_audio, metadata)`

Add an audio source layer to the SHAC file.

```python
writer.add_layer(
    layer_id="bass_guitar",
    ambisonic_audio=encoded_audio,  # Shape: (samples, channels)
    metadata={
        'position': (1.0, 0.0, 2.0),
        'type': 'bass',
        'gain': 0.8
    }
)
```

**Parameters:**
- `layer_id` (str): Unique identifier for this layer
- `ambisonic_audio` (np.ndarray): Encoded ambisonic audio, shape (samples, channels)
- `metadata` (dict): Position and properties (required: 'position', 'type', 'gain')

**Metadata fields:**
- `position` (list): [x, y, z] in meters (required)
- `type` (str): Source type description (required)
- `gain` (float): Volume multiplier, default 1.0 (optional but recommended)

#### `write_file(filename)`

Write the SHAC file to disk.

```python
writer.write_file("output.shac")
```

**Parameters:**
- `filename` (str): Output path (will add .shac extension if missing)

**Returns:** Path to the written file

---

### Encoding Functions

#### `encode_mono_source(audio, spherical_pos, order, apply_distance_gain)`

Encode a mono audio source to ambisonic format.

```python
from shac.codec.encoders import encode_mono_source

ambisonic_audio = encode_mono_source(
    audio=mono_audio,           # NumPy array, shape (samples,)
    spherical_pos=(azimuth, elevation, distance),
    order=3,
    apply_distance_gain=False
)
```

**Parameters:**
- `audio` (np.ndarray): Mono audio, shape (samples,)
- `spherical_pos` (tuple): (azimuth, elevation, distance)
  - azimuth: Horizontal angle in radians
  - elevation: Vertical angle in radians
  - distance: Radial distance in meters
- `order` (int): Ambisonic order (1-7)
- `apply_distance_gain` (bool): Apply 1/r attenuation (default False)

**Returns:** NumPy array, shape (samples, channels) where channels = (order+1)²

#### `convert_to_spherical(cartesian_pos)`

Convert Cartesian coordinates to spherical.

```python
from shac.codec.encoders import convert_to_spherical

# Cartesian position
position = (2.0, 1.0, 3.0)  # x, y, z in meters

# Convert to spherical
azimuth, elevation, distance = convert_to_spherical(position)
```

**Parameters:**
- `cartesian_pos` (tuple): (x, y, z) in meters

**Returns:** (azimuth, elevation, distance)
- `azimuth` (float): Horizontal angle in radians
- `elevation` (float): Vertical angle in radians
- `distance` (float): Radial distance in meters

---

## Complete Examples

### Example 1: Single Source

```python
import numpy as np
from shac.codec import SHACFileWriter
from shac.codec.encoders import encode_mono_source, convert_to_spherical
from shac.codec.math_utils import AmbisonicNormalization

# Generate test audio (1 second sine wave at 440 Hz)
sample_rate = 48000
duration = 1.0
t = np.linspace(0, duration, int(sample_rate * duration))
audio = np.sin(2 * np.pi * 440 * t).astype(np.float32)

# Create writer
writer = SHACFileWriter(
    order=3,
    sample_rate=sample_rate,
    normalization=AmbisonicNormalization.SN3D
)

# Position source
position = (2.0, 0.0, 1.5)  # 2m right, 1.5m forward
spherical_pos = convert_to_spherical(position)

# Encode
ambisonic_audio = encode_mono_source(audio, spherical_pos, order=3)

# Add layer
writer.add_layer(
    layer_id="sine_tone",
    ambisonic_audio=ambisonic_audio,
    metadata={
        'position': list(position),
        'type': 'test_tone',
        'gain': 1.0
    }
)

# Write file
writer.write_file("single_source.shac")
print("Created single_source.shac")
```

### Example 2: Multiple Sources (Drum Kit)

```python
import numpy as np
from shac.codec import SHACFileWriter
from shac.codec.encoders import encode_mono_source, convert_to_spherical

# Load drum samples (assume these return mono numpy arrays)
kick = load_audio("kick.wav")
snare = load_audio("snare.wav")
hihat = load_audio("hihat.wav")

# Ensure all samples are same length (pad if needed)
max_length = max(len(kick), len(snare), len(hihat))
kick = np.pad(kick, (0, max_length - len(kick)))
snare = np.pad(snare, (0, max_length - len(snare)))
hihat = np.pad(hihat, (0, max_length - len(hihat)))

# Create writer
sample_rate = 48000
writer = SHACFileWriter(order=3, sample_rate=sample_rate)

# Define drum positions (arrange in semi-circle)
drums = {
    'kick': (0.0, -0.5, 1.0),      # Center, slightly below
    'snare': (-1.0, 0.0, 1.2),     # Left
    'hihat': (1.0, 0.3, 1.2)       # Right, slightly elevated
}

# Encode and add each drum
for name, position in drums.items():
    audio = {'kick': kick, 'snare': snare, 'hihat': hihat}[name]

    spherical_pos = convert_to_spherical(position)
    ambisonic_audio = encode_mono_source(audio, spherical_pos, order=3)

    writer.add_layer(
        layer_id=name,
        ambisonic_audio=ambisonic_audio,
        metadata={
            'position': list(position),
            'type': f'drum_{name}',
            'gain': 1.0
        }
    )

# Write file
writer.write_file("drum_kit.shac")
print("Created drum_kit.shac with 3 sources")
```

### Example 3: Circular Arrangement

```python
import numpy as np
from shac.codec import SHACFileWriter
from shac.codec.encoders import encode_mono_source, convert_to_spherical

# Create sources arranged in a circle
num_sources = 8
radius = 3.0  # meters
sample_rate = 48000
duration = 2.0

# Generate audio for each source (different frequencies)
sources = []
for i in range(num_sources):
    frequency = 220 * (2 ** (i / 12))  # Musical intervals
    t = np.linspace(0, duration, int(sample_rate * duration))
    audio = np.sin(2 * np.pi * frequency * t).astype(np.float32)
    sources.append(audio)

# Create writer
writer = SHACFileWriter(order=3, sample_rate=sample_rate)

# Position sources in circle
for i in range(num_sources):
    angle = 2 * np.pi * i / num_sources
    x = radius * np.cos(angle)
    z = radius * np.sin(angle)
    y = 0.0  # Horizontal plane

    position = (x, y, z)
    spherical_pos = convert_to_spherical(position)

    # Encode
    ambisonic_audio = encode_mono_source(sources[i], spherical_pos, order=3)

    # Add layer
    writer.add_layer(
        layer_id=f"source_{i}",
        ambisonic_audio=ambisonic_audio,
        metadata={
            'position': [x, y, z],
            'type': 'tone',
            'gain': 1.0
        }
    )

# Write file
writer.write_file("circle_arrangement.shac")
print(f"Created circle_arrangement.shac with {num_sources} sources")
```

### Example 4: Loading Audio Files

```python
import numpy as np
import soundfile as sf  # pip install soundfile
from shac.codec import SHACFileWriter
from shac.codec.encoders import encode_mono_source, convert_to_spherical

def load_audio_file(filepath, target_sr=48000):
    """Load audio file and convert to target sample rate if needed"""
    audio, sr = sf.read(filepath, dtype='float32')

    # Convert stereo to mono
    if audio.ndim > 1:
        audio = np.mean(audio, axis=1)

    # Resample if needed
    if sr != target_sr:
        from scipy import signal
        num_samples = int(len(audio) * target_sr / sr)
        audio = signal.resample(audio, num_samples)

    return audio

# Create SHAC from WAV files
sample_rate = 48000
writer = SHACFileWriter(order=3, sample_rate=sample_rate)

# List of (filename, position) tuples
audio_files = [
    ("vocals.wav", (0.0, 0.5, 2.0)),
    ("guitar.wav", (-2.0, 0.0, 1.5)),
    ("bass.wav", (2.0, 0.0, 1.5)),
    ("drums.wav", (0.0, 0.0, 3.0))
]

# Load, encode, and add each file
for filename, position in audio_files:
    audio = load_audio_file(filename, sample_rate)
    spherical_pos = convert_to_spherical(position)
    ambisonic_audio = encode_mono_source(audio, spherical_pos, order=3)

    layer_id = filename.replace(".wav", "")
    writer.add_layer(
        layer_id=layer_id,
        ambisonic_audio=ambisonic_audio,
        metadata={
            'position': list(position),
            'type': 'instrument',
            'gain': 1.0
        }
    )

writer.write_file("band_mix.shac")
print("Created band_mix.shac from WAV files")
```

---

## Advanced Topics

### Distance Gain

The `apply_distance_gain` parameter controls 1/r attenuation:

```python
# Without distance gain (default)
ambisonic_audio = encode_mono_source(audio, spherical_pos, order=3,
                                     apply_distance_gain=False)
# Source volume is constant regardless of distance
# Use this when you want manual control over levels

# With distance gain
ambisonic_audio = encode_mono_source(audio, spherical_pos, order=3,
                                     apply_distance_gain=True)
# Source gets quieter as distance increases (1/r relationship)
# Use this for physically accurate spatial audio
```

**Recommendation:** Leave `apply_distance_gain=False` and control levels manually via the `gain` metadata field. This gives you more creative control over the spatial mix.

### Sample Rate Matching

All sources in a SHAC file must have the same sample rate. If your source audio has different sample rates:

```python
from scipy import signal

def resample_audio(audio, original_sr, target_sr):
    """Resample audio to target sample rate"""
    num_samples = int(len(audio) * target_sr / original_sr)
    resampled = signal.resample(audio, num_samples)
    return resampled.astype(np.float32)

# Example usage
audio_44k = load_audio("file_44100.wav")  # 44.1kHz
audio_48k = resample_audio(audio_44k, 44100, 48000)
```

### Layer Timing

SHAC files currently require all layers to have the same duration (sample count). To create layers with different timing:

**Pad shorter sources:**
```python
max_length = max(len(audio1), len(audio2), len(audio3))

audio1_padded = np.pad(audio1, (0, max_length - len(audio1)))
audio2_padded = np.pad(audio2, (0, max_length - len(audio2)))
audio3_padded = np.pad(audio3, (0, max_length - len(audio3)))
```

**Offset sources (add silence at start):**
```python
# Start audio2 2 seconds later
offset_samples = int(2.0 * sample_rate)
silence = np.zeros(offset_samples, dtype=np.float32)
audio2_offset = np.concatenate([silence, audio2])
```

### Normalization

Ensure audio is normalized to prevent clipping:

```python
def normalize_audio(audio, peak=0.9):
    """Normalize audio to peak amplitude"""
    max_val = np.abs(audio).max()
    if max_val > 0:
        audio = audio * (peak / max_val)
    return audio

audio = normalize_audio(audio, peak=0.9)
```

---

## Performance Considerations

### Encoding Time

Encoding is computationally intensive. For a 3-minute composition with 14 sources at order 3:
- **Expected time:** ~20 minutes on mid-range hardware
- **Scales with:** Number of sources × duration × ambisonic order²

**Tips for faster encoding:**
- Use order 2 or 3 (avoid order 7 unless absolutely necessary)
- Encode sources in parallel if hardware supports it
- Pre-process audio to minimize source count

### Memory Usage

Peak memory usage during encoding:
```
memory ≈ num_sources × samples × channels × 4 bytes
```

For 10 sources, 3 minutes, order 3, 48kHz:
```
memory ≈ 10 × (3 × 60 × 48000) × 16 × 4
       ≈ 10 × 8,640,000 × 16 × 4
       ≈ 5.5 GB
```

Process sources individually if memory is constrained.

---

## Troubleshooting

### "Audio array must be mono (1D)"
Your audio has multiple channels. Convert to mono:
```python
if audio.ndim > 1:
    audio = audio[:, 0]  # Take first channel
```

### "Sample count mismatch"
All layers must have the same number of samples. Pad to match:
```python
target_length = 48000 * 60  # 1 minute at 48kHz
audio = np.pad(audio, (0, max(0, target_length - len(audio))))
```

### "Invalid position"
Position must be [x, y, z] with 3 numeric values:
```python
position = [1.0, 0.0, 2.0]  # Valid
# position = [1.0, 0.0]  # Invalid (only 2 values)
```

### "File too large"
SHAC files are large by design. To reduce size:
- Use lower ambisonic order (2 instead of 3)
- Reduce source count (merge similar sources)
- Shorten duration
- Lower sample rate (44.1kHz instead of 48kHz)

---

## Best Practices

1. **Start with order 3** - Best balance of quality and file size
2. **Use 48kHz sample rate** - Industry standard
3. **Normalize audio** - Prevent clipping and distortion
4. **Position sources thoughtfully** - Too many sources close together = perceptual confusion
5. **Keep layer IDs meaningful** - Helps with debugging and future editing
6. **Add descriptive metadata** - Store extra info for future reference
7. **Test in the player** - Always verify spatial positioning is as expected

---

## Next Steps

- **[DECODING.md](./DECODING.md)** - How to play and parse SHAC files
- **[INTEGRATION.md](./INTEGRATION.md)** - Integrate SHAC into DAWs and tools
- **[SPECIFICATION.md](../SPECIFICATION.md)** - Complete technical specification
- **[HISTORY.md](./HISTORY.md)** - Development story and evolution

---

**For questions or issues, see the main repository or open an issue on GitHub.**
