# Sky: an FFmpeg player backend beside libVLC (Qt 6)

## Context

Seeking an online video with VLC 3 is not exact:

- with VLC's FFmpeg demuxer, a paused seek can leave a stale picture or land a few frames off;
- with VLC's own MP4 demuxer, any HTTP source ends about 3.6 s early. This was measured through
  a local proxy and a local HTTP server: it comes from VLC's HTTP path, not from the network.

VLC's input queue also overflowed under seek spam, and its stop can deadlock. None of this can
be fixed from Sky.

`WBackendFfmpeg` plays on our own FFmpeg, beside libVLC, on Qt 6 only:

- each `WBackendManager` picks its engine at runtime;
- VLC stays the default until FFmpeg matches it;
- Qt keeps its own FFmpeg for Qt Multimedia, untouched.

The priorities are exact seeks, decoding and rendering performance, portability, and clean
simple code shaped like `WBackendVlc`.

Phase 1 (video) and phase 3 (audio) are done; the other phases are outlined at the end.

## FFmpeg

- **The build:** a BtbN master build, LGPL, shared libraries. We pin the last build of a month,
  which BtbN keeps for two years (daily builds go after 14 days).
  - `3rdparty/generate.sh` sets `FFmpeg_version` and `FFmpeg_build`, and fetches it for win64
    and linux64, the platforms BtbN builds for.
  - The layout is `3rdparty/<platform>/FFmpeg/<version>/{include,lib,bin}`.
- **What it brings:** dav1d (software AV1), the hardware decoders (d3d11va, vaapi), Schannel
  for https, and MSVC import libraries.
- **Configure and deploy:**
  - `Sky/configure.sh` copies the headers to `include/ffmpeg` and the libraries to `lib`;
  - `Sky/deploy.sh` and `Sky-runtime/configure.sh` copy avcodec, avfilter, avformat, avutil,
    swresample and swscale.
- **Beside Qt's FFmpeg:** master is `avcodec-63` while Qt's is `avcodec-61`. The names differ,
  so both load in one process. On Linux, the sonames and the symbol versions differ the same
  way.
- **Build switches:**
  - `SkMedia.pro` and `Sky-runtime.pro` build with FFmpeg when Qt is 6 and `include/ffmpeg`
    exists;
  - otherwise they define `SK_NO_FFMPEG`, which `Sk.h` expands to `SK_NO_BACKENDFFMPEG` and
    `SK_NO_FFMPEGPLAYER`.

## The classes

### WFfmpegPlayer (`SkMedia/src/ffmpeg`)

It mirrors `WVlcPlayer`:

- the same interface (`setBackend` with setup, lock and unlock frame callbacks, `setSource`,
  `play(at)`, `pause`, `stop`, `seek`, `nextFrame`, `setSpeed`, `setVolume`, `deletePlayer`);
- the same events posted to the backend.

It owns the decoding, so it could also serve without a QML player.

- **Inputs:** one read thread per input, opened in parallel, each feeding a video queue, an
  audio queue or both. Queues are bounded to 4 s, and an input waits only once all its queues
  are full.
  - **`OutputMedia`:** the video, plus the audio from the separate audio URL (the input slave of
    VLC) when there is one, otherwise from the file itself. The audio URL can be any audio file
    FFmpeg reads, local or over https.
  - **`OutputVideo`:** the video alone, and the audio is never read.
  - **`OutputAudio`:** the audio alone.
- **Decode thread:** decodes up to 3 frames ahead and copies them from the GPU, so a slow
  frame never comes late. A copy from the GPU takes 11 to 13 ms for a 4K frame, with peaks
  past 30 ms.
- **Video thread:** converts the next frame ahead into the backend's buffer, then only publishes
  it when it's due.
- **Late frames:** behind the clock, a frame late by two frames is dropped before its copy from
  the GPU, never more than five in a row, and a frame whose next one is due already is skipped,
  so the video stays on the audio. A seek's frame and a step are never dropped.
- **Audio thread:** owns a `QAudioSink` in push mode, in its own event loop. Every 10 ms it
  decodes what the device can take, converts it with `swresample` to the device format (float
  when supported), and writes it into a 100 ms device buffer.
- **Speed:** at another speed than 1x the samples go through libavfilter's `atempo`, which
  keeps the pitch (chained below 0.5x). At 1x there's no filter, the samples go straight to the
  device. A speed change rebuilds the filter and goes on from where the audio decodes, without a
  seek.
- **Shared state:** a mutex and a wait condition guard what the threads share.
- **Events:**
  - events go through the player, which drops the ones of a previous source and forwards the
    rest;
  - the time of a frame is posted before the frame itself, so the frame comes to a backend
    already done loading.
- **Network:** FFmpeg's HTTP with reconnects, keep-alive for seeks, and a 20 s read timeout.
  Requests are of 10 MB at most (`request_size`): YouTube serves one long request at about twice
  the playback speed and 10 MB ones at full speed. A seek on a YouTube video at 60 s went from
  5 to 7.5 s to its first frame down to 1.5 s, with its audio.
  There's no reconnect for a source without byte ranges: it can't resume, and at the end of a
  stream sent without length FFmpeg would take the end for a cut and retry.
- **Sources that can't seek:** a server without byte ranges is read through `WFfmpegCache`, which
  keeps in memory what was read:
  - seeks back and forth land in it;
  - a file with its index at its end (an m4a with its `moov` last) plays too;
  - a seek to the end of a source without length is refused rather than read to the end, as a
    radio has no end.
- **Stop:** an interrupt callback on an abort flag makes any blocking read return at once, so a
  stop joins both threads right away.
- **Seek:**
  - the latest target wins, with no queue;
  - `av_seek_frame` goes back to the keyframe, then the player flushes and decodes forward;
  - it skips non-reference frames until 0.5 s before the target;
  - it shows the frame the target time falls in.

  Times are whole milliseconds and frames are 16 or 17 ms apart at 60 fps, so a frame that
  starts within 1 ms after the target counts as well. That way, a step of one frame duration
  lands on the next frame in either direction.
- **Decoding:** hardware first (d3d11va, VideoToolbox, vaapi). AV1 goes to FFmpeg's own AV1
  decoder on the GPU, since dav1d is software only. When the GPU can't decode a stream, its
  first frame fails, and the player decodes it in software from the last seek.
- **Frames:** hardware frames come down as NV12 or P010, and `sws_scale` writes I420 straight
  into the backend's planes. Full range frames are brought to the video range of the shader.
- **Playback:** frames are paced on their timestamps against a clock.
  - **With audio:** the audio leads. What we hear is the end of the decoded audio, minus what
    waits in the device and in the player, each millisecond of it being `speed` milliseconds of
    media. The audio thread moves the clock to it only when the two drift apart by more than
    20 ms, so the video stays smooth.
  - **Without audio:** a steady clock times the speed, restarted from the first frame shown
    after a play or a seek.
  - **Seeks:** they start the audio on their target, to the sample, once the video shows its
    frame, so both start together.
  - **End:** when the audio ends first, the clock goes on alone.
- **Volume:** VLC's curve.
- **Output:** as VLC, another output plays the source again from the current time, here with
  its last frame on screen meanwhile. turbopixel's `LayerPlayer` reads the audio once its sound
  goes on, until the next video, so turning the sound off is the volume alone.
- **Logs:** FFmpeg logs its fatal errors only. The player reports its own, FFmpeg's are mostly
  the ones of a source we cut, as a probe interrupted (dav1d complaining of a cut packet) or a
  TLS connection we close.

### WBackendFfmpeg (`SkMedia/src/media`)

It follows the structure of `WBackendVlc`, trimmed:

- the same reply and quality handling;
- the same double buffered I420 planes and setup event, with planes aligned to 32 bytes for
  swscale;
- the same state events.

Where it differs:

- **Pause:** it pauses the player at once, even while loading. A paused player shows the frame
  at its time and stays there.
- **Seek:** no coalescing in the backend, since the player keeps only the last seek.
- **`getFrame`:** `backendUpdateFrame` converts the current planes to RGB with swscale (BT.709,
  video range), so a frame can be taken at any time, paused or playing. `SK_SOFTWARE` converts
  on each frame and draws with `backendDrawFrame`, as VLC does.

### WBackendYuv

The I420 node, material and shader moved out of `WBackendVlc` unchanged (`WBackendYuvNode`).
Both backends render with them.

### WBackendManager

- **`engine`:** `EngineVlc` or `EngineFfmpeg`, exposed to QML as `BackendManager`.
- **Switching:** setting it stops the playback and swaps the inner backend. `setBackend` gives
  the new backend the manager's settings and a hook of its own. The source plays again on the
  new backend once played.
- **Render node:** `backendCreateNode` asks the inner backend for its node.

Sky-runtime registers `BackendManager` and `BackendFfmpeg`. turbopixel's `LayerPlayer` and
`ItemVideo` set `player.backend.engine = BackendManager.EngineFfmpeg`.

## Measured

All on 1080p60 AV1 (`itag=399`, 990.6 s), on a laptop RTX A1000 with d3d11va.

| Test | Result |
|---|---|
| Seek to 6 targets, a step back from each (from the time the player says, as the bar does), then the targets in reverse, against the frames the ffmpeg CLI decodes at those times | 18 of 18 identical (difference 0.00), on disk and on YouTube |
| Same time sought twice | same picture every time |
| Paused seek on YouTube | about 0.4 s to the frame |
| Playback, 16 s at the end of the YouTube video | 60 frames a second, to the last frame and `ended` (VLC's MP4 demuxer stopped at 987 s) |
| CPU while playing those 16 s | FFmpeg 2.5 to 2.7 s, VLC 7.9 s |
| Previous frame held on YouTube for 40 s, then quit | no freeze, exit in 0.4 s |
| One player switching VLC, FFmpeg, VLC while playing | each plays from its time, exit in 0.4 s |
| YouTube with its separate audio URL, from 975 s | sound from the start to `ended` (Windows audio meter of the process), the time on the wall clock |
| A file with its own audio: pause, seek paused, play, 2x, back to 1x | silent paused, sound at play and at 2x and back at 1x, `ended` on time |
| Speeds 2x, 0.5x, 0.25x and back to 1x, on that file and on YouTube | media at 2.00, 0.50, 0.25 and 1.00 s a second, sound at each, no stall at the changes |
| YouTube start at 960 s | 0.7 s to resolve the page, 0.6 s to open both inputs, 0.15 s to seek, 0.8 s to fetch and decode up to the exact frame, then audio and video together |

4K60 streaming from YouTube against VLC, the frame times as they reach the player over 20 s of
playback, in alternate runs (the laptop heats up):

| 4K60 | VLC 3.0.23 | FFmpeg engine |
|---|---|---|
| SDR, AV1 (`itag=401`): spacing jitter (sd) | 2.2 to 2.5 ms | 1.0 to 1.2 ms |
| SDR: late frames / dropped | 0 to 0.1% / 0% | 0% / 0% |
| SDR: CPU | 218 to 249% (dav1d, software) | 41 to 51% (d3d11va) |
| HDR, AV1 10 bits (`itag=701`): jitter (sd) | 4.5 to 5.2 ms | 1.1 ms, one run 5.0 ms |
| HDR: late frames / dropped | 2.5 to 3.6% / 0%, 6.1% dropped when hot | 0% / 0%, one run 4.1% / 1.9% |
| HDR: CPU | 479 to 711% | 93 to 110% |

Neither engine maps HDR to SDR yet: the 10 bits frame is shown through the BT.709 shader, so
its colors are flat.
| `OutputVideo` playback (turbopixel's players) | no sound, the audio is never read |
| A local video with an https audio file as its input slave: ogg (byte ranges), then mp3, m4a with its index last, opus, flac and aac from a server without byte ranges, a seek back mid play | each plays with sound from its start, the seek back lands and the sound goes on |

Backward seeks on YouTube take 0.2 to 1.5 s each, since each one fetches again from its
keyframe.

## Later phases

- **2. Rendering:** NV12 and P010 shader variants (2 planes, no conversion), then zero copy on
  D3D11. FFmpeg would decode on the device Qt's RHI renders with, and the texture would go to
  the scene graph as is, which removes the copy from the GPU, the main cost at 4K. HDR to SDR
  tone mapping in that shader. VideoToolbox to Metal would work the same way on Apple.
- **4. Portability:** FFmpeg builds of the same commit for macOS, iOS and Android
  (VideoToolbox, MediaCodec), lifting `SK_NO_FFMPEG` there.
- **5. Parity:** track selection, subtitles, HLS live, a packet cache for backward seeks on a
  stream, then a decision on the default engine.
