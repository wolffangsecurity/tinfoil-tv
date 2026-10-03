<img width="1280" height="640"  src="https://github.com/user-attachments/assets/8d1c0631-dd40-429e-baa8-9fd4b71959f7" />

Tinfoil is a client-side web application that simulates a vintage television receiver tuning into curated public-domain broadcasts. The application calculates synchronized broadcast positions locally and streams media files. A machine learning speech synthesis pipeline in `voiceactor/make_bumpers.py` uses the Chatterbox model to generate station identification voice tracks by cloning the vocal style of public-domain narrators.

<img width="1599" height="772" alt="image" src="https://github.com/user-attachments/assets/a5066485-ba23-4cc4-8502-16fe85970639" />
<img width="845" height="712" alt="image" src="https://github.com/user-attachments/assets/3ed85000-baa0-41e1-a6d2-0fcfcfdde17b" />


## Try it Out
- https://tinfoil-topaz.vercel.app/

## Features

* Synchronized broadcast clock: Calculates current program offsets across active channels using a fixed epoch in `src/broadcast.js`, ensuring all clients calculate the same playback timestamp without central server coordination.
* VHF rotary channel selector: Supports channel switching across 12 dial positions with detent rotation mapping defined in `CHANNEL_ANGLES` in `src/main.js`.
* CRT visual rendering: Uses HTML5 Canvas in `src/crtRenderer.js` to draw VHF static noise, horizontal tear lines, moving hum bars, and vertical picture rolls during channel transitions.
* Analog audio synthesis: Synthesizes white noise, bandpass filtering, 60 Hz power hums, 1000 Hz test tones, and mechanical switch clicks using the Web Audio API in `src/audio.js`.
* Station announcer playback: Decodes and plays pre-generated audio bumpers with automatic volume ducking of the active video stream in `src/audio.js`.
* Vintage TV guide catalog: Provides an interactive directory with text filtering in `src/main.js` and dedicated print layout rules in `src/style.css`.
* Defensive client architecture: Enforces schema validation, HTML entity encoding, input rate limits, and dataset integrity checks in `src/security/`.
* Offline audio bumper generator: Uses a Python pipeline in `voiceactor/make_bumpers.py` to extract reference audio with Demucs, run voice synthesis with Chatterbox, and apply bandpass filtering with FFmpeg.
* Build-time metadata ingest: Queries archive metadata APIs and verifies media streams with HTTP HEAD requests in `scripts/ingest.js`.
* Automated test suite: Runs 41 automated assertions verifying input validation, access boundaries, rate limits, and cryptographic integrity in `scripts/security_test.js`.

## How It Works

```text
+------------------------------------------------------+
|                   Client Browser                     |
|                                                      |
|  +------------------------------------------------+  |
|  |              TinfoilApp (main.js)              |  |
|  |  [Validation, Tuning Logic, TV Guide Catalog]   |  |
|  +------------------------------------------------+  |
|         |                     |                |     |
|         v                     v                v     |
|  +--------------+   +-------------------+  +-------+ |
|  |BroadcastClock|   |   CRTRenderer     |  |Audio  | |
|  | (broadcast)  |   |  (crtRenderer)    |  |(audio)| |
|  +--------------+   +-------------------+  +-------+ |
|         |                     |                |     |
|         +----------+----------+                |     |
|                    v                           v     |
|             +--------------+             +---------+ |
|             | <video> Tag  |             | Web     | |
|             | (HTTP Range) |             | Audio   | |
|             +--------------+             +---------+ |
+--------------------|---------------------------------+
                     | HTTP GET (Byte Range)
                     v
+------------------------------------------------------+
|            Internet Archive Media Server             |
|         https://archive.org/download/...             |
+------------------------------------------------------+
```

### Speech Model Generation Pipeline

```text
+------------------------------------------------------+
|            Public Domain Audio Reference             |
|         Prelinger Archives ("What Makes Us Tick")    |
+------------------------------------------------------+
                           |
                           v
+------------------------------------------------------+
|             Demucs Vocal Extraction                  |
|         Isolates clean speech stem                   |
+------------------------------------------------------+
                           |
                           v
+------------------------------------------------------+
|             Chatterbox TTS Model                     |
|         Clones voice style with text prompt:         |
|         "You are watching Channel 5, Mystery & Noir" |
+------------------------------------------------------+
                           |
                           v
+------------------------------------------------------+
|             FFmpeg Acoustic Post-Processing          |
|         High-pass (250Hz), low-pass (4200Hz),        |
|         compression (-16dB), loudness normalization  |
+------------------------------------------------------+
                           |
                           v
+------------------------------------------------------+
|             Static Asset Export                      |
|         public/bumpers/ch05_intro.mp3                |
+------------------------------------------------------+
```

### Request Lifecycle

1. The user turns the channel dial or selects a channel through keyboard input in `src/main.js`.
2. `RateLimiter.checkLimit()` in `src/security/rateLimiter.js` validates that dial interaction rate remains within safe UI thresholds.
3. `InputValidator.validateChannelNumber()` in `src/security/inputValidator.js` validates the selected channel against VHF detents (2–13).
4. The application triggers a 600 ms transition in `CRTRenderer.triggerTransition()` in `src/crtRenderer.js`, activating VHF static noise and vertical picture roll.
5. `BroadcastClock.getChannelState()` in `src/broadcast.js` computes the elapsed time modulo the total playlist duration from `Date.UTC(2026, 0, 1)`.
6. `RuntimeProtection.safeSetVideoSource()` in `src/security/runtimeProtection.js` validates the HTTPS media URL and assigns it to the video element.
7. The browser issues HTTP range requests to retrieve the media stream directly from the remote archive repository.
8. `TVAudioSystem.playStationAnnouncer()` in `src/audio.js` ducks the video audio track, plays the three-note chime sequence, and triggers the station bumper buffer.
9. `CRTRenderer.render()` completes the vertical roll settling phase and clears the static overlay once the stream locks.



## Key Modules

| File | Purpose |
|---|---|
| `src/main.js` | Main application controller managing UI state, DOM events, tuning logic, and catalog interactions. |
| `src/broadcast.js` | Broadcast schedule calculator computing deterministic channel offsets and commercial breaks. |
| `src/audio.js` | Web Audio API synthesizer for static noise, switch transients, power hum, chimes, and bumper playback. |
| `src/crtRenderer.js` | 2D canvas renderer generating dynamic VHF static, line tearing, hum bars, and SMPTE test patterns. |
| `src/security/` | Defensive client subsystem covering schema validation, HTML entity encoding, rate limits, and dataset integrity. |
| `src/style.css` | Stylesheet defining the television cabinet layout, scanlines, CRT screen distortion, and print media rules. |
| `src/data/channels.json` | Static dataset containing channel definitions, playlist schedules, metadata, and media URLs. |
| `scripts/ingest.js` | Build-time ingest script querying archive metadata and validating stream availability. |
| `scripts/security_test.js` | Automated test suite testing validation, rate limiting, and integrity controls. |
| `scripts/sync_bumpers.js` | Asset synchronization script copying generated bumper audio files into the public directory. |
| `voiceactor/make_bumpers.py` | Python pipeline for vocal isolation, Chatterbox speech cloning, and acoustic post-processing. |
| `voiceactor/bumpers.json` | Script configurations, channel associations, and parameter settings for voice bumper generation. |

