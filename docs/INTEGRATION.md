# SHAC Integration Guide

**Integrate SHAC encoding/decoding into DAWs, media players, and audio tools**

This guide helps developers add SHAC export and playback capabilities to existing audio software.

---

## Overview

SHAC integration typically involves:

1. **Export from DAW** - Add SHAC as an export format option
2. **Import for playback** - Support loading and playing .shac files
3. **Spatial positioning UI** - Interface for placing sources in 3D space
4. **Real-time preview** - Optional interactive navigation during creation

---

## DAW Integration (Export)

### Integration Points

Most DAWs have plugin architectures for custom export formats:

- **Reaper:** Custom export script or ReaScript
- **Ableton Live:** Max for Live device or Python script
- **Logic Pro:** Audio Unit or script automation
- **Pro Tools:** AudioSuite plugin
- **FL Studio:** Python script or custom plugin

### Export Workflow

```
User creates multi-track project
  ↓
Select tracks to include as SHAC layers
  ↓
Specify 3D position for each track
  ↓
Set ambisonic order (default 3)
  ↓
Encode each track to ambisonics
  ↓
Write SHAC file
```

### Implementation Example (Pseudocode)

```python
# DAW Export Plugin Pseudocode

def export_shac_from_daw(project, output_path):
    """Export current DAW project to SHAC file"""

    # Get selected tracks
    tracks = project.get_selected_tracks()

    # Get project settings
    sample_rate = project.get_sample_rate()
    duration = project.get_duration()

    # Initialize SHAC writer
    from shac.codec import SHACFileWriter
    writer = SHACFileWriter(order=3, sample_rate=sample_rate)

    # Process each track
    for track in tracks:
        # Bounce track to mono audio
        audio = track.bounce_to_mono(duration)

        # Get spatial position (from plugin UI or track metadata)
        position = get_track_position(track)  # Returns (x, y, z)

        # Encode to ambisonics
        from shac.codec.encoders import encode_mono_source, convert_to_spherical
        spherical_pos = convert_to_spherical(position)
        ambisonic_audio = encode_mono_source(audio, spherical_pos, order=3)

        # Add to SHAC file
        writer.add_layer(
            layer_id=track.name,
            ambisonic_audio=ambisonic_audio,
            metadata={
                'position': list(position),
                'type': track.get_type(),  # 'vocals', 'drums', etc.
                'gain': track.get_volume()
            }
        )

    # Write file
    writer.write_file(output_path)
    return output_path
```

### UI Components Needed

**Track Selection:**
- Checkbox list of project tracks
- "Select All" / "Deselect All" buttons

**Spatial Positioning:**
- 3D viewport (top-down and side view)
- Click-and-drag to position tracks
- Numeric input fields for precise X, Y, Z coordinates
- Presets: "Circle", "Line", "Random", "Manual"

**Export Settings:**
- Ambisonic order dropdown (1-7, default 3)
- Sample rate (inherit from project or specify)
- Output file path selection

**Example UI Layout:**
```
┌─────────────────────────────────────┐
│  SHAC Export                        │
├─────────────────────────────────────┤
│  Tracks:                            │
│  ☑ Vocals        [X: 0.0  Y: 0.5 │
│  ☑ Guitar        [X: -2.0 Y: 0.0 │
│  ☑ Bass          [X: 2.0  Y: 0.0 │
│  ☑ Drums         [X: 0.0  Y: 0.0 │
│                                     │
│  [3D Viewport]                      │
│  ┌───────────┐   ┌───────────┐     │
│  │ Top View  │   │ Side View │     │
│  │           │   │           │     │
│  └───────────┘   └───────────┘     │
│                                     │
│  Settings:                          │
│  Order: [3 ▼]   Sample Rate: 48kHz │
│                                     │
│  [Cancel]  [Export SHAC]            │
└─────────────────────────────────────┘
```

---

## Media Player Integration (Playback)

### Integration Points

For media players like VLC, foobar2000, or custom apps:

- **File format registration** - Associate .shac extension
- **Decoder plugin** - Parse and decode SHAC files
- **Spatial audio renderer** - Convert ambisonics to stereo/multichannel
- **Navigation controls** - WASD or virtual joystick for movement

### Playback Workflow

```
User opens .shac file
  ↓
Decode header and layers
  ↓
Create audio buffers
  ↓
Initialize spatial audio engine
  ↓
Render binaural audio based on listener position
  ↓
Output to headphones/speakers
  ↓
Handle navigation input (update position/rotation)
```

### Implementation Example (JavaScript)

```javascript
// Media Player SHAC Plugin

class SHACPlayerPlugin {
    constructor(audioContext) {
        this.audioContext = audioContext;
        this.decoder = new SHACDecoder();
        this.engine = new SpatialAudioEngine(audioContext);
    }

    async loadFile(file) {
        // Read file
        const arrayBuffer = await file.arrayBuffer();

        // Decode
        const shacData = await this.decoder.decode(arrayBuffer);

        // Load into spatial engine
        await this.engine.loadSHACData(shacData);

        // Setup navigation controls
        this.setupControls();

        return {
            duration: shacData.header.samples / shacData.header.sampleRate,
            sampleRate: shacData.header.sampleRate,
            layers: shacData.layers.length
        };
    }

    play() {
        this.engine.play();
    }

    pause() {
        this.engine.pause();
    }

    stop() {
        this.engine.stop();
    }

    setupControls() {
        // Keyboard navigation (WASD)
        document.addEventListener('keydown', (e) => {
            const speed = 0.1;  // meters per keypress
            const pos = this.engine.listenerPosition;

            switch(e.key) {
                case 'w': this.engine.setListenerPosition(pos.x, pos.y, pos.z + speed); break;
                case 's': this.engine.setListenerPosition(pos.x, pos.y, pos.z - speed); break;
                case 'a': this.engine.setListenerPosition(pos.x - speed, pos.y, pos.z); break;
                case 'd': this.engine.setListenerPosition(pos.x + speed, pos.y, pos.z); break;
                case 'q': this.engine.setListenerPosition(pos.x, pos.y + speed, pos.z); break;
                case 'e': this.engine.setListenerPosition(pos.x, pos.y - speed, pos.z); break;
            }
        });

        // Mouse rotation
        let isDragging = false;
        document.addEventListener('mousedown', () => isDragging = true);
        document.addEventListener('mouseup', () => isDragging = false);
        document.addEventListener('mousemove', (e) => {
            if (isDragging) {
                const rot = this.engine.listenerRotation;
                this.engine.setListenerRotation(
                    rot.azimuth + e.movementX * 0.01,
                    rot.elevation + e.movementY * 0.01
                );
            }
        });
    }

    getCurrentPosition() {
        return this.engine.listenerPosition;
    }

    getCurrentRotation() {
        return this.engine.listenerRotation;
    }
}

// Usage in media player
const audioContext = new AudioContext();
const plugin = new SHACPlayerPlugin(audioContext);

// Load SHAC file
const file = await fetch('demo.shac').then(r => r.blob());
const info = await plugin.loadFile(file);

console.log(`Loaded SHAC: ${info.duration}s, ${info.layers} layers`);

// Play
plugin.play();
```

### UI Components for Playback

**Player Controls:**
- Play/Pause/Stop buttons
- Timeline scrubber
- Volume control

**Navigation Display:**
- 2D overhead map showing listener position
- Source positions as markers
- Current orientation indicator

**Position Info:**
- Current XYZ coordinates
- Distance to nearest source
- Active sources in range

**Example Player UI:**
```
┌────────────────────────────────────────┐
│  ▶ ⏸ ⏹    [====●========] 1:23 / 3:45  │
├────────────────────────────────────────┤
│  Navigation Map:                       │
│  ┌──────────────────────────────────┐  │
│  │         🔊 Vocals                 │  │
│  │                                   │  │
│  │  🔊 Guitar    👤 ← You            │  │
│  │                                   │  │
│  │         🔊 Drums                  │  │
│  └──────────────────────────────────┘  │
│                                        │
│  Position: X: 0.0  Y: 0.0  Z: 0.0     │
│  Rotation: 45°                         │
│                                        │
│  Controls: WASD = Move  Mouse = Look  │
└────────────────────────────────────────┘
```

---

## Python Integration

### Reaper ReaScript Example

```python
# Reaper ReaScript for SHAC Export

import reaper_python as rpr
from shac.codec import SHACFileWriter
from shac.codec.encoders import encode_mono_source, convert_to_spherical
import numpy as np

def export_reaper_project_to_shac():
    """Export current Reaper project to SHAC"""

    # Get project info
    sample_rate = int(rpr.GetSetProjectInfo(0, 'PROJECT_SRATE', 0, False))
    end_time = rpr.GetProjectLength(0)

    # Initialize SHAC writer
    writer = SHACFileWriter(order=3, sample_rate=sample_rate)

    # Get all tracks
    num_tracks = rpr.CountTracks(0)

    for i in range(num_tracks):
        track = rpr.GetTrack(0, i)

        # Get track name
        _, track_name = rpr.GetSetMediaTrackInfo_String(track, 'P_NAME', '', False)

        # Bounce track to numpy array
        audio = bounce_track_to_numpy(track, end_time, sample_rate)

        # Get position from track FX parameters or user input
        # (This would require custom FX or metadata)
        position = get_track_spatial_position(track)  # Returns (x, y, z)

        # Encode
        spherical_pos = convert_to_spherical(position)
        ambisonic_audio = encode_mono_source(audio, spherical_pos, order=3)

        # Add layer
        writer.add_layer(
            layer_id=track_name,
            ambisonic_audio=ambisonic_audio,
            metadata={
                'position': list(position),
                'type': 'track',
                'gain': rpr.GetMediaTrackInfo_Value(track, 'D_VOL')
            }
        )

    # Write file
    output_path = rpr.GetUserFileNameForRead('', 'Export SHAC', '.shac')
    if output_path:
        writer.write_file(output_path)
        rpr.ShowMessageBox(f'SHAC file exported: {output_path}', 'Success', 0)

def bounce_track_to_numpy(track, duration, sample_rate):
    """Bounce Reaper track to numpy array"""
    # Render track to temporary file
    temp_file = rpr.GetTemporaryFolder() + '/temp_bounce.wav'
    rpr.RenderTrackToFile(track, temp_file, duration)

    # Load as numpy array
    import soundfile as sf
    audio, sr = sf.read(temp_file)

    # Convert to mono if needed
    if audio.ndim > 1:
        audio = np.mean(audio, axis=1)

    # Resample if needed
    if sr != sample_rate:
        from scipy import signal
        audio = signal.resample(audio, int(len(audio) * sample_rate / sr))

    return audio.astype(np.float32)

# Run export
export_reaper_project_to_shac()
```

---

## C++ Integration

For native applications or high-performance tools:

### Header Parsing (C++)

```cpp
#include <fstream>
#include <vector>
#include <cstdint>

struct SHACHeader {
    uint16_t version;
    uint16_t order;
    uint16_t channels;
    uint32_t sampleRate;
    uint32_t bitDepth;
    uint32_t samples;
    uint16_t numLayers;
    uint16_t normalization;
};

bool parseSHACHeader(std::ifstream& file, SHACHeader& header) {
    // Read magic bytes
    char magic[4];
    file.read(magic, 4);
    if (std::string(magic, 4) != "SHAC") {
        return false;  // Invalid file
    }

    // Read header fields (little-endian)
    file.read(reinterpret_cast<char*>(&header.version), 2);
    file.read(reinterpret_cast<char*>(&header.order), 2);
    file.read(reinterpret_cast<char*>(&header.channels), 2);
    file.read(reinterpret_cast<char*>(&header.sampleRate), 4);
    file.read(reinterpret_cast<char*>(&header.bitDepth), 4);
    file.read(reinterpret_cast<char*>(&header.samples), 4);
    file.read(reinterpret_cast<char*>(&header.numLayers), 2);
    file.read(reinterpret_cast<char*>(&header.normalization), 2);

    // Validate
    if (header.version != 1) return false;
    if (header.channels != (header.order + 1) * (header.order + 1)) return false;
    if (header.bitDepth != 32) return false;

    return true;
}

// Usage
std::ifstream file("example.shac", std::ios::binary);
SHACHeader header;
if (parseSHACHeader(file, header)) {
    std::cout << "Sample rate: " << header.sampleRate << std::endl;
    std::cout << "Layers: " << header.numLayers << std::endl;
}
```

### Layer Parsing (C++)

```cpp
struct SHACLayer {
    std::string id;
    std::string metadata;  // JSON string
    std::vector<std::vector<float>> channels;  // [channel][sample]
};

bool parseSHACLayer(std::ifstream& file, const SHACHeader& header, SHACLayer& layer) {
    // Read ID length
    uint16_t idLength;
    file.read(reinterpret_cast<char*>(&idLength), 2);

    // Read metadata length
    uint32_t metadataLength;
    file.read(reinterpret_cast<char*>(&metadataLength), 4);

    // Read layer ID
    std::vector<char> idBuffer(idLength);
    file.read(idBuffer.data(), idLength);
    layer.id = std::string(idBuffer.begin(), idBuffer.end());

    // Read metadata
    std::vector<char> metadataBuffer(metadataLength);
    file.read(metadataBuffer.data(), metadataLength);
    layer.metadata = std::string(metadataBuffer.begin(), metadataBuffer.end());

    // Read audio data (interleaved float32)
    size_t numFloats = header.samples * header.channels;
    std::vector<float> interleavedData(numFloats);
    file.read(reinterpret_cast<char*>(interleavedData.data()), numFloats * sizeof(float));

    // De-interleave channels
    layer.channels.resize(header.channels);
    for (size_t c = 0; c < header.channels; ++c) {
        layer.channels[c].resize(header.samples);
        for (size_t s = 0; s < header.samples; ++s) {
            layer.channels[c][s] = interleavedData[s * header.channels + c];
        }
    }

    return true;
}
```

---

## Web Integration

### File Format Registration

Register .shac as a supported format:

```javascript
// Service Worker for PWA
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open('shac-player-v1').then((cache) => {
            return cache.addAll([
                '/shac-decoder.js',
                '/spatial-audio.js',
                '/index.html'
            ]);
        })
    );
});

// Register file handler
if ('launchQueue' in window) {
    window.launchQueue.setConsumer(async (launchParams) => {
        if (launchParams.files && launchParams.files.length > 0) {
            const file = launchParams.files[0];
            if (file.name.endsWith('.shac')) {
                await player.loadFile(file);
                player.play();
            }
        }
    });
}
```

### Drag-and-Drop Support

```javascript
// Enable drag-and-drop for .shac files
const dropZone = document.getElementById('drop-zone');

dropZone.addEventListener('dragover', (e) => {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'copy';
});

dropZone.addEventListener('drop', async (e) => {
    e.preventDefault();

    const files = e.dataTransfer.files;
    for (const file of files) {
        if (file.name.endsWith('.shac')) {
            await player.loadFile(file);
            player.play();
            break;
        }
    }
});
```

---

## Testing Your Integration

### Validation Checklist

- [ ] Can load and parse SHAC header correctly
- [ ] Can load all layers from test files
- [ ] Audio playback is artifact-free
- [ ] Spatial positioning is accurate
- [ ] Navigation controls work smoothly
- [ ] Distance attenuation functions correctly
- [ ] Rotation/orientation updates in real-time
- [ ] File size handling (test with 1GB+ files)
- [ ] Error handling for corrupted files
- [ ] Performance meets real-time requirements

### Test Files

Use these test cases:

1. **Single source** - Verify basic encoding/decoding
2. **Multiple sources** - Test layer management
3. **Circular arrangement** - Test spatial accuracy
4. **Large file (>1GB)** - Test memory handling
5. **Corrupted file** - Test error handling

---

## Performance Optimization

### Encoder Optimization

- **Parallelize source encoding** - Encode multiple tracks simultaneously
- **Vectorize math operations** - Use SIMD where available
- **Cache normalization factors** - Precompute and reuse
- **Show progress indicators** - Long encodes need user feedback

### Decoder Optimization

- **Stream large files** - Don't load entire file into memory
- **Use Web Workers** - Offload decoding to background threads
- **Implement buffer pooling** - Reuse audio buffers
- **Cache rotation matrices** - Avoid recalculating for common angles

---

## Licensing and Distribution

**SHAC codec is MIT licensed - free to integrate into commercial products.**

### Requirements:

1. **Include copyright notice** in your application
2. **Include MIT license text** in your documentation
3. **No additional fees** or royalties required

### Attribution Example:

```
This application uses the SHAC (Spherical Harmonic Audio Codec)
encoder/decoder, created by Clarke Zyz and Claude.

SHAC is open source and MIT licensed.
https://github.com/clarkezyz/Spherical-Harmonic-Audio-Codec
```

---

## Support and Community

**For integration questions:**
- Open an issue on GitHub
- Check documentation at [ENCODING.md](./ENCODING.md) and [DECODING.md](./DECODING.md)
- See [SPECIFICATION.md](../SPECIFICATION.md) for technical details

**Share your integration:**
If you've added SHAC support to your DAW or media player, let us know! We'd love to list it in the README.

---

## Next Steps

- **[ENCODING.md](./ENCODING.md)** - How to create SHAC files
- **[DECODING.md](./DECODING.md)** - How to parse and play SHAC files
- **[SPECIFICATION.md](../SPECIFICATION.md)** - Complete technical reference
- **[HISTORY.md](./HISTORY.md)** - Development story and evolution

---

**Ready to integrate SHAC into your project? Start with the [ENCODING.md](./ENCODING.md) or [DECODING.md](./DECODING.md) guides.**
