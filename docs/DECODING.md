# SHAC Decoding Guide

**How to parse and play SHAC files programmatically**

This guide covers implementing SHAC decoders for playback, visualization, or integration into media players and applications.

---

## Quick Start (JavaScript/Web)

```javascript
import { SHACDecoder } from './shac-decoder.js';
import { SpatialAudioEngine } from './spatial-audio.js';

// Create decoder
const decoder = new SHACDecoder();

// Load SHAC file
const response = await fetch('demo.shac');
const arrayBuffer = await response.arrayBuffer();

// Decode file
const shacData = await decoder.decode(arrayBuffer);

// Create spatial audio engine
const audioContext = new AudioContext();
const engine = new SpatialAudioEngine(audioContext);

// Load decoded data into engine
await engine.loadSHACData(shacData);

// Play
engine.play();

// Navigate (WASD controls)
engine.setListenerPosition(x, y, z);
engine.setListenerRotation(azimuth, elevation);
```

---

## File Parsing

### Header Parsing

The first 26 bytes contain the SHAC header:

```javascript
function parseHeader(buffer) {
    const view = new DataView(buffer);
    let offset = 0;

    // Magic bytes (4 bytes): 'SHAC'
    const magic = new Uint8Array(buffer, offset, 4);
    if (magic[0] !== 0x53 || magic[1] !== 0x48 ||
        magic[2] !== 0x41 || magic[3] !== 0x43) {
        throw new Error('Invalid SHAC file: magic bytes mismatch');
    }
    offset += 4;

    // Version (2 bytes, uint16)
    const version = view.getUint16(offset, true);  // little-endian
    if (version !== 1) {
        throw new Error(`Unsupported SHAC version: ${version}`);
    }
    offset += 2;

    // Ambisonic order (2 bytes, uint16)
    const order = view.getUint16(offset, true);
    offset += 2;

    // Number of channels (2 bytes, uint16)
    const channels = view.getUint16(offset, true);
    offset += 2;

    // Verify channel count matches order
    const expectedChannels = (order + 1) * (order + 1);
    if (channels !== expectedChannels) {
        throw new Error(`Channel mismatch: expected ${expectedChannels}, got ${channels}`);
    }

    // Sample rate (4 bytes, uint32)
    const sampleRate = view.getUint32(offset, true);
    offset += 4;

    // Bit depth (4 bytes, uint32)
    const bitDepth = view.getUint32(offset, true);
    if (bitDepth !== 32) {
        throw new Error(`Unsupported bit depth: ${bitDepth} (only 32-bit float supported)`);
    }
    offset += 4;

    // Total samples (4 bytes, uint32)
    const samples = view.getUint32(offset, true);
    offset += 4;

    // Number of layers (2 bytes, uint16)
    const numLayers = view.getUint16(offset, true);
    offset += 2;

    // Normalization (2 bytes, uint16)
    const normalization = view.getUint16(offset, true);
    offset += 2;

    return {
        version,
        order,
        channels,
        sampleRate,
        bitDepth,
        samples,
        numLayers,
        normalization,
        headerSize: offset  // Should be 26
    };
}
```

### Layer Parsing

After the header, parse each layer:

```javascript
function parseLayer(buffer, offset, header) {
    const view = new DataView(buffer);
    const textDecoder = new TextDecoder('utf-8');

    // ID length (2 bytes, uint16)
    const idLength = view.getUint16(offset, true);
    offset += 2;

    // Metadata length (4 bytes, uint32)
    const metadataLength = view.getUint32(offset, true);
    offset += 4;

    // Layer ID (UTF-8 string)
    const idBytes = new Uint8Array(buffer, offset, idLength);
    const layerId = textDecoder.decode(idBytes);
    offset += idLength;

    // Metadata (UTF-8 JSON)
    const metadataBytes = new Uint8Array(buffer, offset, metadataLength);
    const metadataString = textDecoder.decode(metadataBytes);
    const metadata = JSON.parse(metadataString);
    offset += metadataLength;

    // Validate metadata
    if (!metadata.position || metadata.position.length !== 3) {
        throw new Error(`Invalid metadata for layer ${layerId}: missing or invalid position`);
    }

    // Audio data (32-bit float, interleaved channels)
    const audioDataSize = header.samples * header.channels * 4;  // 4 bytes per float32
    const audioData = new Float32Array(buffer, offset, header.samples * header.channels);
    offset += audioDataSize;

    // De-interleave channels
    const channels = [];
    for (let c = 0; c < header.channels; c++) {
        const channel = new Float32Array(header.samples);
        for (let s = 0; s < header.samples; s++) {
            channel[s] = audioData[s * header.channels + c];
        }
        channels.push(channel);
    }

    return {
        id: layerId,
        metadata,
        channels,  // Array of Float32Arrays, one per ambisonic channel
        nextOffset: offset
    };
}
```

### Complete Decoder

```javascript
class SHACDecoder {
    async decode(arrayBuffer) {
        // Parse header
        const header = parseHeader(arrayBuffer);

        // Parse all layers
        const layers = [];
        let offset = header.headerSize;

        for (let i = 0; i < header.numLayers; i++) {
            const layer = parseLayer(arrayBuffer, offset, header);
            layers.push(layer);
            offset = layer.nextOffset;
        }

        return {
            header,
            layers
        };
    }

    reset() {
        // Clear cached data if needed
    }
}
```

---

## Spatial Audio Playback

### Ambisonic Rotation

When the listener rotates, apply rotation matrices to ambisonic channels:

```javascript
function rotateAmbisonics(input, azimuth, elevation, order) {
    // For order 1 (4 channels: W, Y, Z, X)
    if (order === 1) {
        const output = new Float32Array(4);
        const cosAz = Math.cos(azimuth);
        const sinAz = Math.sin(azimuth);
        const cosEl = Math.cos(elevation);
        const sinEl = Math.sin(elevation);

        // W channel (omnidirectional) - unchanged
        output[0] = input[0];

        // Rotate Y, Z, X channels
        output[1] = input[1] * cosAz - input[3] * sinAz;  // Y
        output[2] = input[2] * cosEl - input[0] * sinEl;  // Z
        output[3] = input[1] * sinAz + input[3] * cosAz;  // X

        return output;
    }

    // For higher orders, use Wigner D-matrices (more complex)
    // See SPECIFICATION.md for mathematical details
    return rotateHigherOrder(input, azimuth, elevation, order);
}
```

### Binaural Rendering

Convert ambisonic channels to stereo for headphone playback:

```javascript
function binauralRender(ambisonicChannels, order, hrtfData) {
    // Simplified binaural decode for order 1
    if (order === 1) {
        // HRTF coefficients for left ear
        const H_L = hrtfData.left;  // [h0_L, h1_L, h2_L, h3_L]

        // Ambisonic channels [W, Y, Z, X]
        const A = ambisonicChannels;

        // Left ear
        let left = 0;
        for (let i = 0; i < 4; i++) {
            left += H_L[i] * A[i];
        }

        // Right ear (symmetric or separate HRTF)
        const H_R = hrtfData.right;
        let right = 0;
        for (let i = 0; i < 4; i++) {
            right += H_R[i] * A[i];
        }

        return { left, right };
    }

    // For higher orders, more HRTF coefficients required
    return binauralRenderHigherOrder(ambisonicChannels, order, hrtfData);
}
```

### Distance Attenuation

Apply distance-based volume reduction:

```javascript
function calculateDistance(pos1, pos2) {
    const dx = pos1.x - pos2.x;
    const dy = pos1.y - pos2.y;
    const dz = pos1.z - pos2.z;
    return Math.sqrt(dx * dx + dy * dy + dz * dz);
}

function applyDistanceAttenuation(gain, distance, minDistance = 0.1) {
    // 1/r falloff with minimum distance to prevent division by zero
    const effectiveDistance = Math.max(distance, minDistance);
    return gain / effectiveDistance;
}
```

---

## Web Audio API Integration

### Complete Spatial Audio Engine

```javascript
class SpatialAudioEngine {
    constructor(audioContext) {
        this.audioContext = audioContext;
        this.layers = new Map();
        this.listenerPosition = { x: 0, y: 0, z: 0 };
        this.listenerRotation = { azimuth: 0, elevation: 0 };
        this.isPlaying = false;
    }

    async loadSHACData(shacData) {
        this.header = shacData.header;
        this.shacLayers = shacData.layers;

        // Create AudioBuffers for each layer
        for (const layer of this.shacLayers) {
            const audioBuffer = this.audioContext.createBuffer(
                this.header.channels,
                this.header.samples,
                this.header.sampleRate
            );

            // Copy channel data
            for (let c = 0; c < this.header.channels; c++) {
                audioBuffer.copyToChannel(layer.channels[c], c);
            }

            this.layers.set(layer.id, {
                buffer: audioBuffer,
                metadata: layer.metadata,
                source: null,
                gainNode: null
            });
        }
    }

    play() {
        if (this.isPlaying) return;
        this.isPlaying = true;

        // Create source and processing nodes for each layer
        for (const [id, layer] of this.layers) {
            // Create buffer source
            const source = this.audioContext.createBufferSource();
            source.buffer = layer.buffer;

            // Create gain node for distance attenuation
            const gainNode = this.audioContext.createGain();

            // Connect: source -> gain -> destination
            source.connect(gainNode);
            gainNode.connect(this.audioContext.destination);

            // Start playback
            source.start();

            // Store references
            layer.source = source;
            layer.gainNode = gainNode;
        }

        // Start update loop for spatial processing
        this.startSpatialProcessing();
    }

    stop() {
        for (const layer of this.layers.values()) {
            if (layer.source) {
                layer.source.stop();
                layer.source = null;
            }
        }
        this.isPlaying = false;
    }

    setListenerPosition(x, y, z) {
        this.listenerPosition = { x, y, z };
        this.updateSpatialAudio();
    }

    setListenerRotation(azimuth, elevation) {
        this.listenerRotation = { azimuth, elevation };
        this.updateSpatialAudio();
    }

    updateSpatialAudio() {
        // Update distance attenuation for each layer
        for (const [id, layer] of this.layers) {
            const sourcePos = layer.metadata.position;
            const distance = calculateDistance(
                this.listenerPosition,
                { x: sourcePos[0], y: sourcePos[1], z: sourcePos[2] }
            );

            const baseGain = layer.metadata.gain || 1.0;
            const attenuatedGain = applyDistanceAttenuation(baseGain, distance);

            if (layer.gainNode) {
                layer.gainNode.gain.setValueAtTime(
                    attenuatedGain,
                    this.audioContext.currentTime
                );
            }
        }

        // Rotation would be applied via ambisonic rotation matrices
        // (more complex - see full implementation in spatial-audio.js)
    }

    startSpatialProcessing() {
        // Update spatial audio at regular intervals
        const updateInterval = 1000 / 60;  // 60 Hz
        this.spatialUpdateTimer = setInterval(() => {
            if (this.isPlaying) {
                this.updateSpatialAudio();
            }
        }, updateInterval);
    }
}
```

---

## Python Decoder Example

```python
import struct
import json
import numpy as np

class SHACDecoder:
    def __init__(self):
        self.header = None
        self.layers = []

    def decode_file(self, filepath):
        """Decode a SHAC file and return header + layers"""
        with open(filepath, 'rb') as f:
            data = f.read()

        # Parse header
        self.header = self.parse_header(data[:26])

        # Parse layers
        offset = 26
        for i in range(self.header['num_layers']):
            layer, offset = self.parse_layer(data, offset)
            self.layers.append(layer)

        return {
            'header': self.header,
            'layers': self.layers
        }

    def parse_header(self, data):
        """Parse 26-byte SHAC header"""
        # Magic bytes
        magic = data[0:4]
        if magic != b'SHAC':
            raise ValueError('Invalid SHAC file: magic bytes mismatch')

        # Unpack header fields (little-endian)
        (version, order, channels, sample_rate, bit_depth,
         samples, num_layers, normalization) = struct.unpack(
            '<HHHIIIIHH', data[4:26]
        )

        # Validate
        if version != 1:
            raise ValueError(f'Unsupported version: {version}')

        expected_channels = (order + 1) ** 2
        if channels != expected_channels:
            raise ValueError(f'Channel mismatch: expected {expected_channels}, got {channels}')

        if bit_depth != 32:
            raise ValueError(f'Unsupported bit depth: {bit_depth}')

        return {
            'version': version,
            'order': order,
            'channels': channels,
            'sample_rate': sample_rate,
            'bit_depth': bit_depth,
            'samples': samples,
            'num_layers': num_layers,
            'normalization': normalization
        }

    def parse_layer(self, data, offset):
        """Parse a single layer starting at offset"""
        # ID length (2 bytes)
        id_length = struct.unpack('<H', data[offset:offset+2])[0]
        offset += 2

        # Metadata length (4 bytes)
        metadata_length = struct.unpack('<I', data[offset:offset+4])[0]
        offset += 4

        # Layer ID (UTF-8 string)
        layer_id = data[offset:offset+id_length].decode('utf-8')
        offset += id_length

        # Metadata (JSON)
        metadata_bytes = data[offset:offset+metadata_length]
        metadata = json.loads(metadata_bytes.decode('utf-8'))
        offset += metadata_length

        # Audio data (32-bit float, interleaved)
        num_floats = self.header['samples'] * self.header['channels']
        audio_data = np.frombuffer(
            data[offset:offset + num_floats * 4],
            dtype=np.float32
        )
        offset += num_floats * 4

        # Reshape to (samples, channels)
        audio_data = audio_data.reshape(
            (self.header['samples'], self.header['channels'])
        )

        return {
            'id': layer_id,
            'metadata': metadata,
            'audio': audio_data
        }, offset

# Usage
decoder = SHACDecoder()
shac_data = decoder.decode_file('example.shac')

print(f"Sample rate: {shac_data['header']['sample_rate']}")
print(f"Layers: {shac_data['header']['num_layers']}")
for layer in shac_data['layers']:
    print(f"  - {layer['id']}: {layer['metadata']['position']}")
```

---

## Optimization Techniques

### Memory Pooling

Reuse Float32Array buffers to reduce GC pressure:

```javascript
class AudioBufferPool {
    constructor() {
        this.pool = [];
        this.inUse = new WeakSet();
    }

    acquire(size) {
        // Find free buffer of right size
        for (const buffer of this.pool) {
            if (buffer.length === size && !this.inUse.has(buffer)) {
                this.inUse.add(buffer);
                return buffer;
            }
        }

        // Create new buffer
        const buffer = new Float32Array(size);
        this.pool.push(buffer);
        this.inUse.add(buffer);
        return buffer;
    }

    release(buffer) {
        this.inUse.delete(buffer);
        buffer.fill(0);  // Clear for reuse
    }
}

const bufferPool = new AudioBufferPool();
```

### Rotation Matrix Caching

Cache rotation matrices for common angles:

```javascript
class RotationCache {
    constructor(maxSize = 1000) {
        this.cache = new Map();
        this.maxSize = maxSize;
    }

    getKey(azimuth, elevation) {
        // Quantize angles to reduce cache size
        const azDeg = Math.round(azimuth * 180 / Math.PI);
        const elDeg = Math.round(elevation * 180 / Math.PI);
        return `${azDeg}_${elDeg}`;
    }

    get(azimuth, elevation, order) {
        const key = this.getKey(azimuth, elevation);
        return this.cache.get(key);
    }

    set(azimuth, elevation, matrix) {
        if (this.cache.size >= this.maxSize) {
            // Remove oldest entry (FIFO)
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
        }
        const key = this.getKey(azimuth, elevation);
        this.cache.set(key, matrix);
    }
}
```

### Streaming Large Files

For files >1GB, load layers progressively:

```javascript
async function streamSHACFile(url) {
    const response = await fetch(url);
    const reader = response.body.getReader();

    let receivedBytes = new Uint8Array();
    let headerParsed = false;
    let header = null;

    while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        // Append new bytes
        const newBytes = new Uint8Array(receivedBytes.length + value.length);
        newBytes.set(receivedBytes);
        newBytes.set(value, receivedBytes.length);
        receivedBytes = newBytes;

        // Parse header if not yet done
        if (!headerParsed && receivedBytes.length >= 26) {
            header = parseHeader(receivedBytes.buffer);
            headerParsed = true;
            console.log('Header parsed:', header);
        }

        // Parse layers as they arrive
        // (implementation depends on layer size calculation)
    }
}
```

---

## Troubleshooting

### "Magic bytes mismatch"
File is not a valid SHAC file or is corrupted. Verify:
- File was downloaded completely
- File extension is `.shac`
- File was created with SHAC encoder

### "Unsupported version"
Your decoder only supports version 1. File may be from future SHAC version.

### "Channel mismatch"
File may be corrupted. Verify:
- channels = (order + 1)²
- File size matches expected size

### "Out of memory"
File is too large to load entirely. Use streaming or load layers individually.

### Audio artifacts / glitches
- Check sample rate matches between decoder and audio context
- Verify buffer sizes are correct
- Ensure proper de-interleaving of channels

---

## Performance Targets

**Decoding:**
- Header parsing: <1ms
- Layer parsing: ~10ms per layer
- Total load time: ~15 seconds per GB (local storage)

**Playback:**
- Real-time performance: 1x playback speed minimum
- Target: 8-10x real-time for smooth navigation
- Latency: <50ms for navigation updates

---

## Reference Implementation

Full reference decoders available in:
- **JavaScript:** `/player/shac-decoder.js`
- **Spatial engine:** `/player/spatial-audio.js`
- **Python:** `/codec/core.py`

See repository for complete, production-tested code.

---

## Next Steps

- **[ENCODING.md](./ENCODING.md)** - How to create SHAC files
- **[INTEGRATION.md](./INTEGRATION.md)** - Integrate into DAWs and applications
- **[SPECIFICATION.md](../SPECIFICATION.md)** - Complete technical reference
- **[HISTORY.md](./HISTORY.md)** - Development story and evolution

---

**For questions or issues, see the main repository or open an issue on GitHub.**
