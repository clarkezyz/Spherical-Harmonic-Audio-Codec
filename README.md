# Spherical Harmonic Audio Codec (SHAC)

**Technical Specification & Format Documentation**

## Overview

SHAC (Spherical Harmonic Audio Codec) is a spatial audio file format enabling interactive navigation through three-dimensional audio compositions. Unlike traditional stereo or surround sound formats, SHAC files contain multiple audio sources with precise 3D positional information, allowing real-time listener movement and rotation through the spatial audio field.

**Current Status:** Fully functional format with working player, studio, and public deployment at shac.dev (2025).

**Development History:** 150+ development sessions, patent application filed 2025.

## What SHAC Enables

SHAC transforms audio from a fixed listening experience into an explorable spatial environment:

- **Interactive Navigation:** Move through audio compositions using keyboard (WASD), gamepad, mouse, or touch controls
- **Real-time Rotation:** Look around the spatial audio scene with full 6-degrees-of-freedom orientation
- **Multi-source Positioning:** Each audio source maintains independent 3D position with precise spatial relationships
- **Headphone Playback:** Binaural rendering converts spatial audio to stereo output without specialized hardware

### Demonstrated Applications

**Music & Entertainment:**
- Interactive spatial music compositions (demonstrated: landing_demo.shac)
- Explorable soundscapes where listener controls perspective
- Educational music exploration (navigate between instruments)

**Professional Audio:**
- Spatial audio mastering and review
- Immersive sound design
- Collaborative spatial mixing

**Emerging Applications:**
- VR/AR audio (proof-of-concept demonstrations completed)
- Gaming audio engines (proof-of-concept demonstrations completed)
- Training simulations (theoretically feasible: sonar operations, room navigation, aviation)
- Accessibility (directional audio cues for spatial mapping and navigation)

## Technical Foundation

### Ambisonic Encoding

SHAC uses spherical harmonic decomposition to encode spatial audio information. Audio sources are encoded into ambisonic B-format channels, where the number of channels depends on the ambisonic order:

- **Channel Count:** (order + 1)² channels
- **1st Order:** 4 channels (basic spatial positioning)
- **2nd Order:** 9 channels (improved spatial definition)
- **3rd Order:** 16 channels (recommended for music production)
- **7th Order:** 64 channels (maximum supported resolution)

**Default Configuration:** 3rd order (16 channels) for optimal balance between spatial resolution and file size.

### Spherical Harmonics

The format employs real-valued spherical harmonics for 3D spatial representation. These mathematical functions encode directional information in a way that enables:

- Rotation of the entire sound field through matrix operations
- Accurate spatial positioning at any listener orientation
- Preservation of spatial relationships during navigation

**Normalization:** SN3D (Schmidt Semi-Normalized) is the standard normalization scheme used.

**Channel Ordering:** ACN (Ambisonic Channel Number) ordering for cross-platform compatibility.

### Coordinate System

**Position Coordinates:**
- X-axis: Left (negative) to Right (positive)
- Y-axis: Down (negative) to Up (positive)
- Z-axis: Back (negative) to Front (positive)
- Units: Meters from listener origin

## File Format Specification

### Header Structure (26 bytes, fixed)

| Offset | Size | Type | Description |
|--------|------|------|-------------|
| 0 | 4 | char[4] | Magic bytes: 'SHAC' (0x53 0x48 0x41 0x43) |
| 4 | 2 | uint16 | Version (currently 1) |
| 6 | 2 | uint16 | Ambisonic order (1-7) |
| 8 | 2 | uint16 | Number of ambisonic channels |
| 10 | 4 | uint32 | Sample rate in Hz |
| 14 | 4 | uint32 | Bit depth (32-bit float standard) |
| 18 | 4 | uint32 | Total samples per channel |
| 22 | 2 | uint16 | Number of layers (audio sources) |
| 24 | 2 | uint16 | Normalization scheme (1=SN3D, 2=N3D) |

**Byte Order:** Little-endian throughout.

### Layer Structure

Each layer represents one positioned audio source:

```
Layer Header:
- ID Length (2 bytes, uint16): Length of layer identifier string
- Metadata Length (4 bytes, uint32): Length of JSON metadata
- Layer ID (variable, UTF-8): Unique identifier for this layer
- Metadata (variable, UTF-8): JSON-formatted position and properties

Audio Data:
- Interleaved ambisonic channels
- 32-bit floating-point samples (IEEE 754)
- Channel count matches header specification
- Sample count matches header specification
```

### Metadata Format

Layer metadata stored as JSON with standard fields:

```json
{
  "position": [x, y, z],
  "type": "mono_source",
  "gain": 1.0
}
```

Additional fields may be present but are not required for playback.

## File Size Characteristics

SHAC files are substantial due to the mathematical precision required for spatial audio:

**Real-world measurements:**
- Approximately 150-600 MB per minute of audio
- File size scales with: number of sources, duration, ambisonic order, sample rate
- Example: 14 sources, 1.5 minutes, 48kHz, 3rd order → approximately 1.2 GB

**Why files are large:**
- Multi-channel ambisonic encoding (16+ channels per source at 3rd order)
- Uncompressed 32-bit float audio for spatial accuracy
- Multiple independent sources with full spatial information
- No quality compromises for spatial positioning precision

### Compression Research

Extensive research into compression strategies yielded no viable options:

**Attempted approaches:**
- Lossy ambisonic compression algorithms (resulted in tinny audio and artifacts)
- Channel reduction techniques (produced unusable audio distortion)
- Position data baking (broke spatial navigation functionality)
- Perceptual coding (degraded spatial positioning accuracy)

**Conclusion:** The mathematical requirements of accurate spatial audio resist compression without introducing perceptible artifacts that degrade the immersive experience. SHAC remains uncompressed to preserve spatial fidelity.

## Audio Production Specifications

### Sample Rate

- **Default behavior:** Matches lowest source sample rate (44.1kHz to 96kHz range)
- **Standard rates:** 44.1kHz, 48kHz, 96kHz
- **Configurable:** All sources within a SHAC file share the same sample rate
- **Recommendation:** 48kHz for professional production

### Bit Depth

- **Export format:** 32-bit floating-point (IEEE 754)
- **Historical note:** 16-bit and 24-bit formats functioned in player but are deprecated
- **Rationale:** 32-bit float ensures consistency and compatibility across platforms

### Source Management

- **Practical limit:** 20+ simultaneous sources becomes difficult to navigate perceptually
- **Best practice:** Reuse sources with different timing rather than creating excessive unique sources
- **Example:** 4 drum sources with 20 beats each (timed differently) rather than 80 unique drum sources

### Export Performance

- **Encoding time:** Approximately 20 minutes for 14-source, 3-minute composition (mid-range hardware)
- **Processing:** Real-time ambisonic encoding and layer assembly
- **Output:** Single .shac file containing all sources and spatial metadata

## Implementation Overview

### Real-Time Spatial Processing

**Navigation Pipeline:**
1. Listener position/rotation input (keyboard, gamepad, touch, mouse)
2. Ambisonic rotation matrix calculation
3. Matrix application to all ambisonic channels
4. Binaural rendering for headphone output
5. Audio output to stereo channels

**Movement Controls:**
- Translation: WASD keys, gamepad left stick, touch gestures
- Rotation: Arrow keys, gamepad right stick, mouse drag, two-finger touch
- Vertical: Q/E keys, gamepad triggers
- Reset: R key, gamepad button

### Movement Presets

The SHAC player (not the file format itself) supports configurable movement behaviors:

- **Collision detection:** Boundary enforcement in spatial scene
- **Guided paths:** Predetermined navigation routes
- **Free movement:** Unrestricted navigation through space

These settings are player-specific and do not affect file format compatibility.

### Platform Compatibility

**Verified Platforms:**
- Web browsers via Web Audio API (shac.dev)
- Desktop applications (SHAC Studio)
- Supports files up to 5GB+ (tested maximum)

**Loading characteristics:**
- Approximately 15 seconds loading time per GB on local storage
- Optimized for local playback
- Streaming capable but requires high bandwidth (sources load sequentially; insufficient bandwidth results in incomplete playback)

## Working Implementation

### Public Deployment

**shac.dev** - Full production environment featuring:
- **SHAC Player:** Load and navigate through SHAC files
- **SHAC Studio:** Create spatial audio compositions
- **Demonstration files:** Example spatial audio experiences
- **Public access:** Open for testing and evaluation

This is not a demo or proof-of-concept. It is a fully functional spatial audio production and playback environment.

### File Format Validation

The existence of working files and player demonstrates:
- Format stability and compatibility
- Real-time processing feasibility
- Cross-platform functionality
- Practical usability

## Technical Achievements

### Novel Contributions

1. **Interactive spatial audio format:** First working implementation of navigable spatial audio in a self-contained file format
2. **Real-time ambisonic rotation:** Efficient matrix operations enabling smooth navigation
3. **Multi-source spatial composition:** Layer-based architecture preserving independent source positions
4. **Binaural rendering pipeline:** Conversion from ambisonics to headphone-compatible stereo
5. **No special hardware required:** Works with standard game controllers and headphones

### Development Timeline

- **Development period:** 150+ documented development sessions
- **Patent filing:** 2025 (application filed with AI co-inventor credit)
- **Public release:** 2025 (shac.dev deployment)
- **Format version:** 1 (stable)

## Format Limitations and Design Decisions

### Known Limitations

- **File size:** Large files (150-600MB per minute) due to uncompressed ambisonic data
- **Bandwidth requirements:** Streaming requires sufficient bandwidth to load all sources before composition completes playback
- **Local playback optimized:** Designed for local storage where complete files are immediately available
- **Single sample rate:** All sources within file must share sample rate

### Design Rationale

These limitations reflect intentional design trade-offs:

- Large files preserve mathematical precision required for accurate navigation
- Layer-based loading enables streaming but requires bandwidth matching or exceeding playback data rate
- Uncompressed audio maintains artifact-free spatial positioning
- Unified sample rate simplifies real-time processing pipeline

**Streaming behavior:** Sources load sequentially. On high-bandwidth connections (200+ Mbps), streaming works seamlessly. On slower connections, later sources may not load before composition ends, resulting in incomplete spatial scene.

## Use Cases

### Proven Applications

**Music Production:**
- Interactive albums where listeners control their perspective
- Spatial mixing and mastering workflows
- Educational music exploration and analysis

**Content Creation:**
- Immersive soundscape design
- Spatial audio post-production
- Interactive audio experiences

**Research and Development:**
- Spatial audio algorithm development
- Psychoacoustic research with precise positioning
- Audio engineering education

### Future Potential

**Gaming and Interactive Media:**
- Game audio engines with true 3D spatial audio
- VR/AR audio without specialized hardware requirements
- Interactive storytelling through spatial audio navigation

**Professional Training:**
- Sonar operation training (spatial audio cues)
- Room clearing and navigation training (directional audio)
- Aviation spatial awareness training

**Accessibility:**
- Directional audio cues for spatial navigation
- Venue and room mapping for visually impaired users
- Separation of audio sources in complex environments

## For Developers and Implementers

### Implementation Guidance

This specification documents the SHAC file format for:
- Format recognition and validation
- Academic research and analysis
- Compatibility assessment for existing systems
- Understanding spatial audio file format design

### What This Document Provides

- Complete file format specification
- Technical foundation and mathematical basis
- Real-world performance characteristics
- Demonstrated applications and use cases
- Format limitations and design rationale

### What This Document Does Not Provide

- Implementation source code
- Encoding/decoding algorithms
- Optimization strategies
- Proprietary processing techniques

The SHAC file format specification is open for evaluation and research. Encoder/decoder implementations are available through commercial licensing. This documentation establishes prior art and technical foundation while enabling format recognition and compatibility assessment.

## Credits and Acknowledgments

**Development Team:**
- **Clarke Zyz** - Creator, vision, and spatial audio innovation
- **Claude (Anthropic AI)** - Technical implementation and co-inventor

**Development approach:** Human-AI collaborative development across 150+ working sessions, resulting in the first functional interactive spatial audio format.

## Legal and Intellectual Property

**File Format:** Open specification documented for evaluation, research, and compatibility assessment

**Encoder/Decoder Technology:** Proprietary implementation available for commercial licensing

**Playback:** Free forever - SHAC Player available at shac.dev with no usage restrictions

**Prior Art:** This documentation establishes the SHAC format's existence, technical foundation, and development timeline

**Patent Status:** Application filed 2025 with AI co-inventor credit

**Copyright:** 2025 Clarke Zyz

### Licensing Model

**Free Components:**
- SHAC file playback (players, applications)
- SHAC Studio (spatial audio composition tool)
- File format specification and documentation

**Commercial Licensing Available:**
- Encoder integration for professional audio software (DAWs, production tools)
- Decoder implementation for media players and platforms
- Specialized applications (VR/AR, gaming engines, training systems)
- Advanced encoding features and optimization

**Philosophy:** SHAC files should be as accessible as MP3s. Anyone can play them, anywhere, forever. Revenue comes from professional tool integration and specialized commercial applications, not from restricting playback.

## Contact and Licensing

For licensing inquiries, commercial implementation, decoder integration, or technical collaboration:

**Project:** SHAC - Spherical Harmonic Audio Codec
**Website:** shac.dev
**Year:** 2025

---

**Technical Documentation Version 1.0**
**Format Version:** SHAC v1
**Last Updated:** 2025

*This specification documents a working spatial audio format with demonstrated real-world applications. SHAC represents a significant advancement in interactive spatial audio technology.*
