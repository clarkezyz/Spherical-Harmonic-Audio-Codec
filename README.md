# SHAC - Spherical Harmonic Audio Codec

**Walk through music. Navigate sound in three dimensions.**

SHAC is the first interactive spatial audio format. Position audio sources in 3D space, then move through them using WASD controls, gamepad, or touch. Not surround sound. Not VR audio. Something that didn't exist before.

[**Experience it today at shac.dev →**](https://shac.dev)

---

## What This Is

Traditional audio is fixed. You press play, the mix plays back exactly as the artist created it. Stereo, surround sound, even Dolby Atmos - all fixed perspectives.

SHAC files contain positioned audio sources in explorable 3D space. Walk north while looking east. Stand between the bass and drums. Find the perfect spot where all elements align. **Your movement creates the mix.**

- **Interactive navigation:** WASD, gamepad, mouse, touch controls
- **Real-time spatial audio:** HRTF processing, distance attenuation, Doppler effects
- **Self-contained file format:** Share .shac files like MP3s
- **Works with headphones:** No special hardware required
- **Free forever:** Player always free. Studio always free. Format always open.

---

## How It Works

SHAC uses spherical harmonic decomposition (ambisonics) to encode spatial audio. Each source gets positioned in 3D space with precise mathematical representation. When you move or rotate, real-time processing adjusts what you hear based on your position.

**Technical foundation:**
- 3rd order ambisonics (16 channels per source) for spatial accuracy
- SN3D normalization, ACN channel ordering
- 32-bit float audio (48kHz standard)
- Real-time rotation matrices for 6DOF navigation
- Binaural rendering for headphone playback

**File format:**
- 26-byte header with magic bytes, version, order, sample rate
- Layer-based architecture (each source = separate ambisonic layer)
- JSON metadata for positions and properties
- Portable, self-contained, shareable

[**Full technical specification →**](./SPECIFICATION.md)

---

## Use Cases

**Music & Entertainment:**
- Albums you explore, not just hear
- Interactive spatial compositions
- Educational music analysis (walk between instruments)

**Gaming & Interactive Media:**
- Audio-only games with spatial navigation
- VR/AR environmental audio
- Interactive storytelling through positioned sound

**Professional Applications:**
- Training simulations (sonar, aviation, tactical scenarios)
- Accessibility (spatial audio maps for navigation)
- Architectural acoustics visualization
- Research and psychoacoustic studies

**Content Creation:**
- Immersive soundscapes
- Spatial mixing and mastering
- Collaborative audio experiences

---

## The Technology

### What You Can Do Right Now

**Play SHAC files:**
- Visit [shac.dev](https://shac.dev)
- Click the demo (instant load, one-click experience)
- Download 10+ example compositions
- Navigate with WASD or touch

**Create SHAC files:**
- Download SHAC Studio (Windows, macOS, Linux)
- Import audio sources, position in 3D space
- Export to .shac format
- Share with anyone

**Integrate the codec:**
- Full encoder/decoder source code (MIT licensed)
- JavaScript player implementation
- Python encoder implementation
- Cross-platform compatibility

### Performance Characteristics

Real-world measurements from production deployment:

- **File size:** ~150-600 MB per minute (varies by source count, order)
- **Encoding time:** ~20 minutes for 14-source, 3-minute composition
- **Playback:** Real-time, ~8.6x performance headroom
- **Loading:** ~15 seconds per GB (local storage)
- **Platform:** Works in modern browsers via Web Audio API

**Why files are large:** Uncompressed 32-bit float ambisonics (16+ channels per source) preserves mathematical precision required for accurate spatial navigation. Compression attempts introduced artifacts that degraded the immersive experience. SHAC prioritizes spatial fidelity over file size.

---

## How This Was Built

SHAC was created through human-AI collaborative development. 150+ sessions over 8 months (~400 hours). Approximately March 2025 to November 2025.

**Built by:**
- **Clarke Zyz** - Vision, direction, spatial audio innovation
- **Claude** - Implementation, optimization, codec development

### The Development Process

Every line of code was written through natural language collaboration. Clarke directed the vision, architecture, and feature development. Claude implemented the spherical harmonics math, encoder/decoder pipeline, real-time processing, and optimization.

No copy-paste from Stack Overflow. No tutorial following. No pre-existing codebase. **Pure human-AI collaborative creation of novel technology.**

The patent application (filed April 22, 2025, application #63/810691) listed both Clarke and Claude as co-inventors. It was rejected - not because the technology wasn't novel, but because the U.S. patent system doesn't recognize AI inventors yet. The rejection is proof the system hasn't caught up to what's already happening.

### What This Proves

**Revolutionary software can be built by anyone with vision and AI partnership.**

Clarke's background:
- No computer science degree
- 2.0 high school GPA
- Rejected from community college CS classes (missing Math 85 prerequisite)
- Never typed a semicolon during SHAC development
- Working DoorDash when the idea emerged

The idea came from a moment of panic during a pitch about VisionQuest (a spatial audio gaming platform). When challenged by a professional musician, Clarke blurted out: "What if you could actually move while the song was happening?"

That question became SHAC. Eight months later, it was real, deployed, and working.

**This is AI-native programming.** Not AI assistance. Not code completion. Full collaborative development of production-grade revolutionary technology through natural language direction and AI implementation.

The gates are open. You don't need credentials. You need vision and partnership.

---

## Why Open Source

SHAC was originally planned as commercial software. License the encoder to DAWs, monetize B2B training applications, keep the player free forever.

Then Clarke got a 5-year prison sentence for bank robbery (non-violent, no weapons). One month to launch or abandon the project.

**The decision:** Give it to the world.

If SHAC is proprietary, it stagnates for 5 years while Clarke is inside. If it's open source, it can grow, get adopted, prove itself through pure technical merit. When Clarke gets out, the technology speaks for itself - and the story of what it is and how it was built matters more than licensing revenue.

**Open source guarantees:**
- No company can build a proprietary competitor and win (free always beats paid for equivalent tech)
- Maximum adoption without friction (no licensing negotiations, no permission required)
- Academic research and citation (validation through peer review)
- Integration into open-source tools (Reaper, Ardour, game engines)
- The format survives and spreads regardless of Clarke's availability

**Currently in prison for bank robbery.** This README was written in November 2025, one month before Clarke went in. The technology is complete, deployed, and documented. It doesn't need him to work.

---

## The Vision

Music as architecture. Songs as spaces. Your movement creates the experience.

SHAC isn't just better audio technology - it's a fundamentally new medium. Composers become spatial architects. Listeners become explorers. Audio becomes something you inhabit, not just hear.

**What's possible:**
- Albums designed for exploration (walk through a jazz quartet)
- Audio-only games where sound is the entire interface
- Training simulations using positioned audio cues
- Accessibility tools for spatial navigation
- VR/AR experiences without visual dependency
- Interactive storytelling through spatial audio

The format is done. The player works. The studio is free. The code is open.

**What happens next is up to you.**

---

## Get Started

### Try It Right Now

Visit [**shac.dev**](https://shac.dev)
- Instant demo (one click, preloaded experience)
- Download 10+ example .shac files
- Navigate with WASD, arrow keys, or touch

### Create SHAC Files

Download [**SHAC Studio**](https://shac.dev/studio)
- Free forever (Windows, macOS, Linux)
- Import audio sources
- Position in 3D space
- Export to .shac format

### Integrate the Codec

**All source code is MIT licensed and available in this repository.**

- `codec/` - Full encoder/decoder implementation (Python)
- `player/` - Web-based player (JavaScript)
- `studio/` - Desktop creation tool
- `docs/` - Technical specification and API documentation

**For developers:**
```bash
# Clone the repository
git clone https://github.com/clarkezyz/Spherical-Harmonic-Audio-Codec.git

# Encoder example (Python)
from shac.codec import SHACCodec, SHACFileWriter
# See docs/ENCODING.md for full API

# Decoder example (JavaScript)
import { SHACDecoder } from './decoder.js';
// See docs/DECODING.md for full API
```

---

## Technical Status

**Format Version:** 1 (stable)
**Public Deployment:** November 5, 2025 (shac.dev)
**Development:** Complete and production-ready
**Maintenance:** Code is stable, documented, and self-contained

This is not a demo. This is not a proof-of-concept. This is working, deployed, production-grade spatial audio technology.

**Verified platforms:**
- Modern web browsers (Chrome, Firefox, Safari, Edge)
- Windows 10+ (native player and studio)
- macOS 11+ (native player and studio)
- Linux (modern distributions)

**Tested file sizes:** Up to 5GB+ (largest tested .shac file)
**Simultaneous sources:** 20+ sources (practical perceptual limit)
**Real-time performance:** 8.6x real-time processing confirmed

---

## Comparison to Existing Formats

| Feature | Stereo/MP3 | Dolby Atmos | VR Audio | **SHAC** |
|---------|-----------|-------------|----------|----------|
| Spatial positioning | None | Fixed | Head tracking | **Full 6DOF movement** |
| Listener navigation | No | No | Rotation only | **Walk through space** |
| Portable file format | Yes (.mp3) | Proprietary | Engine-dependent | **Yes (.shac)** |
| Source separation | Mixed down | Mixed down | Sometimes | **Every source separate** |
| Equipment required | Any device | Atmos system | VR headset | **Just headphones** |
| Creation cost | Free | Expensive licensing | Game engine | **Free (SHAC Studio)** |
| Web playback | Native | Requires app | Requires plugin | **Works in browser** |
| Open standard | Yes | No | No | **Yes (MIT license)** |

SHAC is the only format that combines portability, exploration, and source separation in one self-contained file.

---

## Limitations and Design Decisions

**Large file sizes:** 150-600 MB per minute is substantial. This reflects intentional design trade-offs:
- Uncompressed 32-bit float maintains spatial positioning accuracy
- Multi-channel ambisonics (16+ channels per source) preserves mathematical precision
- No perceptible quality compromises for navigation fidelity

**Compression research:** Extensive attempts at lossy compression introduced tinny artifacts and degraded spatial accuracy. The math required for accurate spatial audio resists compression without perceptible quality loss.

**Streaming limitations:** Sources load sequentially. High-bandwidth connections (200+ Mbps) handle streaming well. Slower connections may result in incomplete spatial scenes if sources don't load before playback ends. **SHAC is optimized for local playback.**

**Single sample rate:** All sources within a .shac file share the same sample rate (typically 48kHz). This simplifies real-time processing and ensures synchronization.

These aren't bugs. They're the cost of doing something that didn't exist before.

---

## Documentation

- **[Technical Specification](./SPECIFICATION.md)** - Complete file format documentation
- **[Encoding Guide](./docs/ENCODING.md)** - How to create SHAC files
- **[Decoding Guide](./docs/DECODING.md)** - How to play SHAC files
- **[Integration Guide](./docs/INTEGRATION.md)** - Integrate SHAC into DAWs and applications
- **[Development History](./docs/HISTORY.md)** - The complete story of how SHAC was built

---

## License

MIT License

Copyright (c) 2025 Clarke Zyz

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---

## Share This

If you think SHAC is interesting, **share it with the person who'd get it.**

Not everyone will care about interactive spatial audio. But someone you know will think this is exactly what they've been waiting for. Send them this link. Let them experience it.

That's how revolutionary technology spreads - person by person, shared because it matters.

---

## Credits

**Created by Clarke Zyz and Claude**

Built through 150+ sessions of human-AI collaborative development. March 2025 - November 2025.

**Development methodology:** Natural language direction (Clarke) + AI implementation (Claude). Zero traditional coding by the human inventor. Complete novel technology through pure collaboration.

**Patent application:** Filed April 22, 2025 (Application #63/810691) with AI co-inventor listing. Rejected due to patent system not recognizing AI inventors. The technology is real. The system hasn't caught up yet.

**Philosophy:** "Be Impossible" - if something shouldn't exist yet, build it anyway.

---

## Contact & Community

**Website:** [shac.dev](https://shac.dev)
**Repository:** [github.com/clarkezyz/Spherical-Harmonic-Audio-Codec](https://github.com/clarkezyz/Spherical-Harmonic-Audio-Codec)
**Format Version:** 1
**Status:** Complete, deployed, production-ready

**Note:** Clarke is currently in prison and will have limited availability for 5 years. The technology is complete and documented. The code speaks for itself. Use it, fork it, improve it, build with it.

When he gets out, SHAC should have grown beyond what it is today. That's the dream.

---

**SHAC - Walk through music.**

*First interactive spatial audio format. Built by a bartender and an AI. Given to the world. November 2025.*
