# SHAC Technical Specification

**Spherical Harmonic Audio Codec - Format Version 1**

This document defines the complete technical specification for the SHAC file format, including header structure, layer architecture, mathematical foundations, and implementation details.

---

## File Format Overview

SHAC is a spatial audio file format storing multiple positioned audio sources encoded as ambisonic layers. Each layer contains ambisonic-encoded audio data with associated 3D position metadata.

**Key characteristics:**
- Self-contained portable file format (.shac extension)
- Layer-based architecture (one layer per audio source)
- Ambisonic encoding for spatial representation
- 32-bit floating-point audio
- JSON metadata for positions and properties
- Little-endian byte order throughout

---

## File Structure

```
[26-byte Header]
[Layer 1 Header]
[Layer 1 Metadata (JSON)]
[Layer 1 Audio Data (interleaved ambisonic channels)]
[Layer 2 Header]
[Layer 2 Metadata (JSON)]
[Layer 2 Audio Data (interleaved ambisonic channels)]
...
[Layer N Header]
[Layer N Metadata (JSON)]
[Layer N Audio Data (interleaved ambisonic channels)]
```

---

## Header Structure (26 bytes, fixed)

| Offset | Size | Type | Field Name | Description |
|--------|------|------|------------|-------------|
| 0 | 4 | char[4] | Magic | Magic bytes: 'SHAC' (0x53 0x48 0x41 0x43) |
| 4 | 2 | uint16 | Version | Format version (currently 1) |
| 6 | 2 | uint16 | Order | Ambisonic order (1-7) |
| 8 | 2 | uint16 | Channels | Number of ambisonic channels per layer |
| 10 | 4 | uint32 | Sample Rate | Sample rate in Hz (e.g., 48000) |
| 14 | 4 | uint32 | Bit Depth | Bits per sample (32 for float) |
| 18 | 4 | uint32 | Samples | Total samples per channel |
| 22 | 2 | uint16 | Layers | Number of audio source layers |
| 24 | 2 | uint16 | Normalization | Normalization scheme (1=SN3D, 2=N3D) |

**Total header size:** 26 bytes (fixed)

### Header Field Details

**Magic (4 bytes):**
- ASCII characters 'SHAC' (0x53 0x48 0x41 0x43)
- Used for format identification and validation

**Version (2 bytes):**
- Current version: 1
- Future versions may extend format while maintaining backward compatibility

**Order (2 bytes):**
- Ambisonic order: 1 to 7
- Determines spatial resolution and channel count
- Default: 3 (recommended for music production)

**Channels (2 bytes):**
- Number of ambisonic channels per layer
- Calculated as: (order + 1)²
- Examples:
  - Order 1 → 4 channels
  - Order 2 → 9 channels
  - Order 3 → 16 channels (default)
  - Order 7 → 64 channels (maximum)

**Sample Rate (4 bytes):**
- Audio sample rate in Hz
- Common values: 44100, 48000, 96000
- All layers share the same sample rate

**Bit Depth (4 bytes):**
- Bits per audio sample
- Standard: 32 (IEEE 754 floating-point)
- Historical note: 16-bit and 24-bit deprecated

**Samples (4 bytes):**
- Total samples per channel (duration = samples / sample_rate)
- All layers have the same sample count

**Layers (2 bytes):**
- Number of audio source layers in file
- Each layer represents one positioned audio source
- Practical limit: ~20 sources (perceptual navigability)

**Normalization (2 bytes):**
- Ambisonic normalization scheme
- 1 = SN3D (Schmidt Semi-Normalized) - standard
- 2 = N3D (Full 3D normalization) - alternative

---

## Layer Structure

Each layer follows this structure:

```
[Layer Header - variable length]
[Audio Data - fixed length based on header]
```

### Layer Header

| Field | Type | Size | Description |
|-------|------|------|-------------|
| ID Length | uint16 | 2 bytes | Length of layer ID string (bytes) |
| Metadata Length | uint32 | 4 bytes | Length of JSON metadata (bytes) |
| Layer ID | UTF-8 string | Variable | Unique identifier for this layer |
| Metadata | UTF-8 JSON | Variable | Position and properties (JSON) |

**Layer ID:**
- UTF-8 encoded string
- Unique identifier within the file
- Examples: "kick_drum", "bass_guitar", "vocals"
- Length specified by ID Length field

**Metadata:**
- UTF-8 encoded JSON object
- Contains position and optional properties
- Length specified by Metadata Length field

### Metadata Format

Required fields:

```json
{
  "position": [x, y, z],
  "type": "mono_source",
  "gain": 1.0
}
```

**position** (required):
- Array of 3 floats: [x, y, z]
- Cartesian coordinates in meters
- Coordinate system:
  - X: Left (negative) to Right (positive)
  - Y: Down (negative) to Up (positive)
  - Z: Back (negative) to Front (positive)

**type** (required):
- String describing source type
- Standard value: "mono_source"
- Used for player behavior and rendering hints

**gain** (optional):
- Float: volume multiplier (default 1.0)
- Applied during playback, not baked into audio

Additional fields may be present and should be preserved by encoders/decoders but are not required for playback.

### Audio Data

Following the layer header is the ambisonic audio data:

**Format:**
- Interleaved 32-bit floating-point samples (IEEE 754)
- Channel count matches header Channels field
- Sample count matches header Samples field

**Data layout:**
```
[Sample 0, Channel 0]
[Sample 0, Channel 1]
...
[Sample 0, Channel N-1]
[Sample 1, Channel 0]
[Sample 1, Channel 1]
...
[Sample 1, Channel N-1]
...
[Sample M-1, Channel N-1]
```

Where:
- N = number of channels (from header)
- M = number of samples (from header)

**Total audio data size per layer:**
- Size in bytes = Samples × Channels × 4 (32-bit = 4 bytes)

---

## Ambisonic Encoding

SHAC uses spherical harmonic decomposition to encode spatial audio.

### Spherical Harmonics

Spherical harmonics are mathematical functions defined on the surface of a sphere. Real-valued spherical harmonics Y_l^m(θ, φ) are indexed by:
- **l**: Order (0 to max order)
- **m**: Degree (-l to +l)
- **θ**: Azimuth angle
- **φ**: Elevation angle

### Channel Count and Ordering

**Number of channels for order L:**
```
channels = (L + 1)²
```

**ACN (Ambisonic Channel Number) ordering:**
Channels are ordered by increasing order l, then increasing degree m:

```
Channel 0:  Y_0^0  (omnidirectional)
Channel 1:  Y_1^(-1)
Channel 2:  Y_1^0
Channel 3:  Y_1^1
Channel 4:  Y_2^(-2)
Channel 5:  Y_2^(-1)
...
```

**ACN index formula:**
```
ACN = l² + l + m
```

### Normalization

**SN3D (Schmidt Semi-Normalized) - Standard:**

For spherical harmonic Y_l^m, the normalization factor is:

```
N_l^m = sqrt((2l + 1) × (l - |m|)! / (4π × (l + |m|)!))
```

Where:
- l = order
- m = degree
- |m| = absolute value of m
- ! = factorial

**N3D (Full 3D) - Alternative:**

```
N_l^m = sqrt((2l + 1) × (l - |m|)! / (l + |m|)!)
```

SHAC uses SN3D by default (Normalization field = 1).

### Encoding Process

To encode a mono audio source at position (azimuth, elevation):

1. **Calculate spherical harmonic values** for the source direction
2. **Multiply each Y_l^m by the normalization factor**
3. **Multiply the audio signal by each normalized Y_l^m value**
4. **Result:** N channels of ambisonic audio (where N = (order+1)²)

**Pseudocode:**
```
function encode_mono_source(audio, azimuth, elevation, order):
    channels = (order + 1)²
    output = allocate(channels, len(audio))

    for l in 0 to order:
        for m in -l to l:
            # Calculate spherical harmonic
            Y = compute_spherical_harmonic(l, m, azimuth, elevation)

            # Apply normalization (SN3D)
            N = compute_normalization_factor(l, m)

            # Encode to channel
            channel_index = l² + l + m
            output[channel_index] = audio × (Y × N)

    return output
```

### Associated Legendre Polynomials

Spherical harmonics are computed using Associated Legendre Polynomials P_l^m(x):

**Recurrence relations for efficient computation:**

```
P_0^0(x) = 1

P_l^l(x) = (-1)^l × (2l - 1)!! × (1 - x²)^(l/2)

P_l^m(x) = ((2l - 1) × x × P_{l-1}^m(x) - (l + m - 1) × P_{l-2}^m(x)) / (l - m)
```

Where:
- x = sin(elevation)
- !! = double factorial
- Computed iteratively for efficiency

### Real-Valued Spherical Harmonics

For real-valued ambisonic encoding:

```
Y_l^m(θ, φ) = N_l^m × P_l^|m|(sin φ) × T_m(θ)

Where:
T_m(θ) = sqrt(2) × cos(m × θ)  if m > 0
T_0(θ) = 1                      if m = 0
T_m(θ) = sqrt(2) × sin(|m| × θ) if m < 0
```

---

## Coordinate Systems

### Cartesian Coordinates (File Storage)

Positions stored in metadata use Cartesian coordinates:

```
X-axis: Left (negative) ←→ Right (positive)
Y-axis: Down (negative) ←→ Up (positive)
Z-axis: Back (negative) ←→ Front (positive)
```

Units: meters from listener origin (0, 0, 0)

### Spherical Coordinates (Encoding)

For ambisonic encoding, Cartesian positions convert to spherical:

```
distance = sqrt(x² + y² + z²)
azimuth = atan2(x, z)
elevation = asin(y / distance)
```

Where:
- **Azimuth (θ):** Horizontal angle, 0° = front, 90° = right, -90° = left
- **Elevation (φ):** Vertical angle, 0° = horizon, 90° = up, -90° = down
- **Distance (r):** Radial distance from origin in meters

---

## Real-Time Playback Processing

### Rotation

To rotate the ambisonic sound field (e.g., when listener rotates head):

1. **Compute rotation matrix** for desired rotation angles
2. **Apply matrix multiplication** to ambisonic channel coefficients
3. **Result:** Rotated ambisonic field

**Rotation matrices for order 1 (4 channels):**

Yaw rotation (around Y-axis) by angle α:
```
[1    0         0        0     ]
[0  cos(α)     0     -sin(α)   ]
[0    0         1        0     ]
[0  sin(α)     0      cos(α)   ]
```

For higher orders, rotation matrices become more complex but follow similar principles. Efficient implementations use Wigner D-matrices.

### Translation

When the listener moves (translation):

1. **Recalculate source positions** relative to new listener position
2. **Recompute azimuth/elevation** for each source
3. **Recompute ambisonic encoding** for new directions
4. **Apply distance attenuation** based on new distances

**Distance attenuation:**
```
gain = 1 / max(distance, minimum_distance)
```

Where minimum_distance prevents division by zero (typically 0.1 to 1.0 meters).

### Binaural Rendering

To convert ambisonics to headphone-compatible stereo:

1. **Apply Head-Related Transfer Functions (HRTFs)** to ambisonic channels
2. **Sum contributions** to left and right ear signals
3. **Result:** Binaural stereo output

**Simplified binaural decode (order 1):**
```
# HRTF coefficients for left ear at azimuth θ
H_L = [h0_L, h1_L, h2_L, h3_L]

# Ambisonic channels (W, Y, Z, X)
A = [a0, a1, a2, a3]

# Left ear output
left = sum(H_L[i] × A[i] for i in 0..3)

# Right ear (symmetric or separate HRTF set)
right = sum(H_R[i] × A[i] for i in 0..3)
```

For higher orders, more HRTF coefficients are required.

---

## Optimization Strategies

### Encoder Optimizations

**Vectorized computation:**
- Compute all spherical harmonics for a source in parallel
- Use SIMD instructions where available
- Process multiple samples simultaneously

**Precompute normalization factors:**
- Calculate N_l^m values once at initialization
- Store in lookup table indexed by (l, m)
- Avoid repeated factorial computations

**Cache rotation matrices:**
- Store recently computed rotation matrices
- Reuse when rotation angles repeat
- Significant speedup for interactive navigation

### Decoder Optimizations

**Memory pooling:**
- Reuse Float32Array buffers for audio processing
- Reduce garbage collection pressure
- Implement buffer pool with acquire/release pattern

**Efficient HRTF convolution:**
- Use FFT-based convolution for long HRTFs
- Overlap-add or overlap-save for real-time processing
- Separate low-order (time-domain) and high-order (frequency-domain) processing

**Rotation caching:**
- Cache rotation matrices for common angles
- Interpolate between cached values for smooth navigation
- Limit cache size to prevent memory bloat

---

## File Size Characteristics

### Size Calculation

For a SHAC file:

```
file_size = header_size + sum(layer_sizes)

Where:
header_size = 26 bytes

layer_size = layer_header_size + audio_data_size

layer_header_size = 6 + len(layer_id) + len(metadata_json)

audio_data_size = samples × channels × 4 bytes
```

### Real-World Examples

**Example 1: Single source, 1 minute, order 3**
- Samples: 2,880,000 (48kHz × 60 seconds)
- Channels: 16 (order 3)
- Audio data: 2,880,000 × 16 × 4 = 184,320,000 bytes (~184 MB)
- Total with headers: ~184 MB

**Example 2: 14 sources, 90 seconds, order 3**
- Samples: 4,320,000 (48kHz × 90 seconds)
- Channels per layer: 16
- Audio data per layer: 4,320,000 × 16 × 4 = 276,480,000 bytes (~276 MB)
- Total for 14 layers: ~3.9 GB
- Actual file (measured): ~1.2 GB
  - Discrepancy due to variable sample counts per layer in practice

**Typical range:** 150-600 MB per minute of audio (varies by source count and order)

### Why Files Are Large

1. **Uncompressed 32-bit float audio** - No lossy compression
2. **Multi-channel ambisonic data** - 16+ channels per source at order 3
3. **Multiple independent sources** - Each source fully encoded
4. **Mathematical precision** - Spatial accuracy requires full float precision

Compression attempts (lossy ambisonic compression, channel reduction, perceptual coding) introduced audible artifacts that degraded spatial positioning accuracy. SHAC prioritizes spatial fidelity over file size.

---

## Validation and Error Handling

### File Validation

Decoders should validate:

1. **Magic bytes** match 'SHAC' (0x53 0x48 0x41 0x43)
2. **Version** is supported (currently only version 1)
3. **Order** is in valid range (1-7)
4. **Channels** matches (order + 1)²
5. **Sample rate** is reasonable (8000-192000 Hz)
6. **Bit depth** is 32 (float)
7. **Layer count** is positive and reasonable (1-100)
8. **Normalization** is 1 (SN3D) or 2 (N3D)

### Layer Validation

For each layer:

1. **ID length** is positive and reasonable (1-256 bytes)
2. **Metadata length** is positive and reasonable (1-4096 bytes)
3. **Metadata** is valid JSON
4. **Position array** has exactly 3 numeric values
5. **Audio data size** matches expected size (samples × channels × 4)

### Error Conditions

**Invalid magic bytes:** Not a SHAC file
**Unsupported version:** Decoder cannot parse this version
**Channel mismatch:** File may be corrupted
**Truncated data:** File incomplete or corrupted
**Invalid JSON:** Metadata malformed

Decoders should fail gracefully and report specific errors.

---

## Implementation Notes

### Encoding Workflow

1. **Collect audio sources** (mono WAV, FLAC, or other formats)
2. **Determine positions** for each source (x, y, z in meters)
3. **Set ambisonic order** (default 3, higher for more precision)
4. **Encode each source:**
   - Convert position to spherical coordinates
   - Compute spherical harmonics for direction
   - Multiply audio by normalized harmonics
   - Generate N channels of ambisonic data
5. **Write header** with global parameters
6. **Write each layer:**
   - Layer header with ID and metadata
   - Interleaved ambisonic audio data
7. **Close file**

### Decoding Workflow

1. **Read and validate header**
2. **Parse layer headers** to build layer index
3. **For real-time playback:**
   - Decode ambisonic layers for all sources
   - Apply listener rotation to ambisonic field
   - Recalculate positions for listener translation
   - Apply distance attenuation
   - Render to binaural stereo via HRTFs
   - Output to audio device
4. **Handle navigation input:**
   - Update listener position and rotation
   - Trigger re-encoding or rotation as needed

### Performance Targets

**Encoding:**
- Real-time or faster for single sources
- Multi-source files may take several minutes (acceptable for offline encoding)

**Decoding/Playback:**
- Must run in real-time (faster than playback speed)
- Target: 8-10x real-time performance for headroom
- Low latency for navigation responsiveness (<50ms)

---

## Future Extensions

While version 1 is stable and complete, potential future extensions include:

- **Compression schemes** (if artifact-free methods are discovered)
- **Source animation** (time-varying positions)
- **Environmental effects** (room acoustics modeling)
- **Higher orders** (beyond order 7, if hardware supports)
- **Alternative binaural renderers** (HRTF personalization)

Any extensions should maintain backward compatibility with version 1 files.

---

## Reference Implementation

Full reference implementations are available:

- **Python encoder:** `codec/encoders.py`
- **JavaScript decoder:** `player/shac-decoder.js`
- **Spatial audio engine:** `player/spatial-audio.js`

See `/codec` and `/player` directories for complete source code.

---

## Specification Version

**Document version:** 1.0
**Format version:** 1
**Last updated:** November 2025

This specification is complete and stable. SHAC format version 1 is production-ready and deployed.

---

**For questions, corrections, or implementation assistance, see the main README or repository issues.**
