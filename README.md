# About this fork

> **This is an unofficial fork** of [alnitak/flutter_soloud](https://github.com/alnitak/flutter_soloud), created and maintained by [cgalvar](https://github.com/cgalvar). It is **not endorsed by or affiliated with** the upstream project or its author. All credit for the plugin goes to the original author and contributors (Marco Bavagnoli and the flutter_soloud / SoLoud / miniaudio communities). The original `LICENSE` (MIT) is kept untouched. The original README follows below this section.

**Base version:** `4.1.7` as published on pub.dev. (The `v4.1.7` git tag differs from the published package in one line of `soloud_miniaudio.cpp`; this fork follows pub.dev. See the first commit of the branch.)

## What the patch does

Android only, and only when the AAudio backend is in use with the low-latency profile (`lowLatency: true`, the default). After the output stream starts, the patch shrinks the stream's buffer to **2 bursts** using `AAudioStream_setBufferSizeInFrames`. If the stream underruns, it grows the buffer again by one burst at a time (never above the size the stream was opened with, and never shrinking it again for that stream). It re-applies this to every new stream (start, route change/reroute, resume, `changeDevice`). It is the same idea as the `LatencyTuner` in Google's Oboe library.

It lives entirely in `src/soloud/src/backend/miniaudio/soloud_miniaudio.cpp`. There is no new Dart API and no change of defaults. If the AAudio symbols are not available, it silently does nothing.

## Why

I use flutter_soloud for a sound whose volume follows stylus motion (a pencil drag), so the delay between motion and sound is audible. With `lowLatency: true` the AAudio stream already runs on the MMAP path, but the stream kept a large queue of frames between the app and the hardware. On my test device the queue held 2160 frames (45 ms at 48 kHz) while the burst size was 240 frames. Changing the `bufferSize` passed to `SoLoud.instance.init()` (240 vs 256) did not change it: that value is the callback period, not this queue. Neither miniaudio nor the plugin exposes `AAudioStream_setBufferSizeInFrames`, so the queue could not be controlled from Dart.

## Measured result

| | Queued in the app stream |
|-|-|
| Stock 4.1.7 | 2160 frames = 45 ms |
| With this patch (initial stream, short session) | 720 frames = 15 ms, 0 xruns |

How it was measured: `adb shell dumpsys media.aaudio`, reading `Write#` minus `Read#` for the app's output stream, at 48 kHz, burst 240, capacity 3840. One device (Huawei MRDI-W09, HarmonyOS 4.3 / Android 12, API 31), one short session. This is **only the app-side queue**, not end-to-end latency to the speaker (it does not include the mixer period, the HAL/DSP, or the DAC and speaker path). Other devices may behave differently.

Not yet validated on a device (implemented, **device validation pending**): behavior after a route change (reroute), and the growth of the buffer after underruns. No numbers are claimed for them.

<!-- FORK-STATUS: reroute and xrun adaptation are implemented, device validation pending. Update this line after testing. -->

## Who may benefit

Apps where sound must react quickly to input: instruments, drawing/stylus or touch feedback, games, rhythm apps. If you play music, long samples or UI sounds where 30 ms does not matter, you will not notice anything and do not need this fork.

## Tradeoffs

- Less safety margin: the stream has about 15 ms of buffer instead of 45 ms, so a slow audio callback or a busy CPU can underrun sooner (the glitch is a click or dropout).
- The patch reacts to underruns by growing the buffer, so it can drift back toward the old latency on devices that underrun a lot. It never shrinks again until the stream is recreated.
- Slightly more wakeups and CPU for the same audio (smaller queue means the hardware is fed in smaller steps), so a small battery cost is possible. I have not measured it.
- Tested on one device only.
- Opt-out: pass `lowLatency: false` to `init()`; the patch is then inactive (this also selects the conservative AAudio profile, with its own tradeoffs; see the original docs).

## How to use it

In your `pubspec.yaml` (pin a commit so that the code cannot change under you):

```yaml
dependencies:
  flutter_soloud:
    git:
      url: https://github.com/cgalvar/flutter_soloud.git
      ref: aaudio-low-latency-buffer   # better: a full commit hash
```

Then `flutter pub get` and rebuild the Android app (it contains native code, so a full rebuild is needed).

## How to verify it yourself

Play a sound, then run:

```sh
adb shell dumpsys media.aaudio
```

Find your app's output stream and compute `Write#` minus `Read#` (frames queued). Divide by the sample rate to get milliseconds. Compare against the same app using the stock package. Run for a while and check that `xruns` stay at (or near) zero.

## Updating from upstream

This fork is a branch on top of the published 4.1.7 code, not a rolling mirror. To move to a newer upstream release, add `https://github.com/alnitak/flutter_soloud.git` as a remote, check out the new tag and cherry-pick the patch commit (it touches a single file). I do not promise to keep it up to date. If you can, prefer upstream: the best outcome would be for this to be solved there or in miniaudio.

---

A low-level audio plugin for Flutter.

[![Pub Version](https://img.shields.io/pub/v/flutter_soloud?logo=dart)](https://pub.dev/packages/flutter_soloud)
[![style: very good analysis](https://img.shields.io/badge/style-very_good_analysis-B22C89.svg)](https://pub.dev/packages/very_good_analysis)

||Linux|Windows|Android|macOS|iOS|Web|
|-|:-:|:-:|:-:|:-:|:-:|:-:|
|Support|💙|💙|💙|💙|💙|💙|
|Minimum Version|Any|Any|21+|10.15+|13.0+|iOS 16.4+<br>Safari 16.4+<br>Chrome 91+<br>Edge 91+<br>Firefox 89+|

## Overview

A high-performance audio plugin designed primarily for games and immersive applications, providing low latency and advanced features.

## Key Features

- ⚡ Low latency, high performance audio
- ⏱️ Sample-accurate scheduled playback: `playClocked` for sub-millisecond spaced playback regardless of buffer size, and `playScheduled` for score/manifest-style scheduling of whole batches of sounds on the engine's own clock (with optional scheduled stop/fade). Perfect for metronomes, music sequencers, rhythm games and precisely timed audio cues
- 🎮 3D positional audio with Doppler effect
- 🔄 Gapless looping with half-open `[start, end)` loop regions
- 🔄 Stream audio with auto-pause for buffering, support for PCM, MP3, WAV, Ogg with Opus, Vorbis and FLAC containers
- 📥 Pull-buffer streaming: the engine requests encoded data on demand (MP3, WAV, FLAC, Ogg Opus/Vorbis/FLAC), with seek support and callbacks for buffering, metadata, duration and data requests — ideal for network streams and custom data sources
- 🚌 Mixing buses: group voices (music, SFX, UI...) into sub-mixes with their own volume, filters and visualization
- 📊 Get audio wave and/or FFT audio data in real-time (useful for visualization)
- 🎛️ Rich effects system (reverb, echo, limiter, parametric equalizer, pitch shift, etc.)
- ⚙️ Faders for attributes (e.g. fade out for 2 seconds, then stop)
- 🎚️ Oscillators for attributes
- 🌊 Waveform generation and visualization
- 🔊 Multiple voices, playing different or even the same sound multiple times
- 🎵 Support for MP3, WAV, OGG, and FLAC
- 🔴 Capture the master mixer output as a stream for recording, processing, or streaming (with different PCM formats and Opus, Vorbis, FLAC, WAV encoded stream formats)
- ⏱️ Read audio data samples from a file with a given time range
- 🌊 Generate waveforms in real-time with various types (sine, square, saw, triangle, etc.)

Whether you are building a game (3D positional SFX, mixing buses, low-latency playback) or any other kind of audio app — music tools, metronomes, radio/streaming apps, visualizers, recorders — the plugin exposes the low-level control you need.

## Getting Started
- Watch the Flutter [Package of the Week](https://www.youtube.com/watch?v=2t6Bt04EyLw) video.
- Especially for web use, please look at the [setup guide docs](https://docs.page/alnitak/flutter_soloud_docs/get_started/setup).

If you are looking for a package to visualize audio using shaders or CustomPainter, please check out [audio_flux](https://pub.dev/packages/audio_flux). It uses this plugin for output and [flutter_recorder](https://pub.dev/packages/flutter_recorder) for input.

Also, if you are building using Swift Package Manager (SPM), please check out [iOS and macOS Configuration](https://docs.page/alnitak/flutter_soloud_docs/get_started/setup#ios-and-macos-configuration).

## Documentation

- [Full Documentation](https://docs.page/alnitak/flutter_soloud_docs)
- [API Reference](https://pub.dev/documentation/flutter_soloud/latest/)

## Simple Example

```dart
import 'package:flutter_soloud/flutter_soloud.dart';

void example() async {
  final soloud = SoLoud.instance;
  await soloud.init();

  await soloud.playSource(asset: 'assets/sound.mp3');
  // or
  final sound = await soloud.loadAsset('assets/sound.mp3');
  final handle = soloud.play(sound);
  
  [...]

  soloud.deinit();
}
```

## Apps & Games Using flutter_soloud

A showcase of apps and games built with this plugin:

| App/Game | Developer | Description |
|----------|-----------|-------------|
| GPhil</br>[web](https://app.g-phil.app/) [macOS](https://apps.apple.com/it/app/gphil/id6740543718) [Windows](https://apps.microsoft.com/detail/9pkkz2p2dldg?ocid=webpdpshare)| Vyacheslav Gryaznov | Innovative app designed for musicians to play instrumental concertos with flexible virtual orchestral accompaniment.|
| [Forcebar](https://forcebar.xyz) | Doug Todd | Forcebar is a pure reflex game. |
| [RadioVisualizer](https://radiovisualizer.com) | Marco Bavagnoli | Stream over 35,000 live radio stations from every corner of the globe. |
| Stellar Bastion</br>[web](https://www.crazygames.com/game/stellar-bastion) [Android](https://play.google.com/store/apps/details?id=com.coconutisland.stellar_bastion) [iOS](https://apps.apple.com/us/app/stellar-bastion/id6761073618) | Coconut Island Apps | 2D Tower Defense game. |
| Mortigen</br>[web](https://koldo92.github.io/mortigen/) [Android](https://play.google.com/store/apps/details?id=com.ler.mortigen) [iOS](https://apps.apple.com/us/app/mortigen/id6761758806) | Luis Enrique Ruiz | Roguelite survival shooter. |
| SUMOJI</br>[web](https://straspool.eu/sumoji/) [Android](https://play.google.com/store/apps/details?id=eu.straspool.sumoji) [iOS](https://apps.apple.com/us/app/sumoji/id6751641875) | Valentin Martinet | Fun Emoji-based Sudoku. |
| GuanDan</br>[web](https://guandan.app/) [macOS](https://apps.apple.com/us/app/%E6%8E%BC%E8%9B%8B-guandan/id6757966323) [Windows](https://apps.microsoft.com/detail/9pbv1xp2lc50) [Android](https://play.google.com/store/apps/details?id=org.rockstudio.guandan) [iOS](https://apps.apple.com/us/app/%E6%8E%BC%E8%9B%8B-guandan/id6757966323) | [yangyuan](https://github.com/yangyuan) | GuanDan (掼蛋) is a popular four-player Chinese card game. |

*Want to add your app? Feel free to open a PR!*

## License

The Dart plugin is covered by the MIT license. For information about the underlying SoLoud engine license, see the [documentation](https://docs.page/alnitak/flutter_soloud/get_started/license).
