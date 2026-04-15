# Tommy the Cat — Practice Loop

A static web app that lets you loop a drum + bass stem, mute either track
independently, and change tempo (180–230 BPM, i.e. ~86–110 % of the 210 BPM
original) while keeping the pitch unchanged.
Built to help you learn Primus' "Tommy the Cat" bass line by playing along
to the drums.

**Live demo:** [pmmathias.github.io/TommyTheCat](https://pmmathias.github.io/TommyTheCat/)
**Blog post explaining how it was built:** [ki-mathias.de → Tommy Practice](https://ki-mathias.de/en/tommy-practice.html)
**More posts on AI-assisted audio & code:** [ki-mathias.de](https://ki-mathias.de/en/)

---

## 1 · What this app does

| Feature | How |
|---|---|
| Two synced tracks (drums & bass) | Two `AudioWorkletNode`s sharing one `AudioContext` clock |
| Endless loop | Source buffer wraps around **inside** the SoundTouch worklet (modulo indexing) |
| Pitch-preserving tempo change | SoundTouch WSOLA algorithm running in an `AudioWorkletProcessor` |
| Per-track mute + volume | A `GainNode` per track |
| No backend | Four static files — drop them on any web host |

### Quick start

```bash
# clone, then serve the folder over http (file:// won't work for WAV fetch)
cd TommyTheCat
python3 -m http.server 8765
open http://localhost:8765/
```

### Hosting

Upload these four files to any static host (GitHub Pages, Netlify Drop,
Cloudflare Pages, S3 + CloudFront, …). No build step:

```
index.html              9 KB   UI + controls + AudioContext wiring
soundtouch-worklet.js  60 KB   SoundTouch WSOLA processor (patched)
drums.wav              3.2 MB  16-bar drums loop, 210 BPM
bass.wav               3.2 MB  16-bar bass   loop, 210 BPM
```

---

## 2 · Syncing recorded videos to a studio mix — the method

This repo is the companion to a blog post about **using Claude Code to
align multi-take home recordings with a studio mix using FFT
cross-correlation**.  The following is the recipe we actually used —
every number below comes from our own Tommy-the-Cat session so you can
see how the theory plays out in practice.

### 2.1 · The setup

| Source | Duration | Content |
|---|---|---|
| `tommy_mix.mp3` | 110.9 s | Studio mix of all instruments |
| `drums.mp4` (WhatsApp) | 129.6 s | Playing e-drums with metronome in headphones |
| `bass.mp4` (WhatsApp) | 117.1 s | Playing e-bass, analog pickup only |

Each video's own audio track is the source material that was later mixed
into the MP3.  So in principle every drum hit and every bass note in the
video should appear identically in the MP3 — but shifted in time,
because the user clicked record on the phone at a different moment than
the DAW.  That's the sync problem.

### 2.2 · Step one: FFT cross-correlation for the coarse offset

The classic solution.  Extract mono audio from each file, normalise,
then compute the cross-correlation in the frequency domain:

```python
correlation = scipy.signal.fftconvolve(mp3, video_audio[::-1], mode='full')
peak = int(np.argmax(np.abs(correlation)))
offset = (peak - len(video_audio) + 1) / sample_rate
# video_time + offset  =  mp3_time
```

Our result at this stage:

| Track | Offset | Peak strength (peak / mean) |
|---|---|---|
| drums | −26.166 s | 22× |
| bass  | −27.044 s | 128× |

The bass peak is **128× above the mean** — because the bass audio in
the video literally *is* the bass in the mix, just with different EQ.
Cross-correlation is at its best when the two signals share actual
waveform content.

### 2.3 · Pitfall #1 — bar-phase aliasing

When our first version of the code ran cross-correlation on
**onset-envelope features** (as opposed to raw samples), drums locked
onto **−78.5 s** with confidence 9×.  Playback had rolls that almost
landed but were off by ~3 s.  Tempo is 210 BPM, so 3.4 s = 2 bars — the
algorithm had picked a repetition-shifted local maximum.

**Fix:** switch to raw-sample cross-correlation **band-passed to the
instrument's range**:

- drums: band-pass 150–6000 Hz (snare body + hi-hat transients)
- bass: band-pass 40–300 Hz (fundamentals)

With that change the drum peak jumped to 22× confidence at −26.17 s.
Higher peak-to-mean ratio ⇒ less ambiguity about bar alignment.

### 2.4 · Pitfall #2 — "the flamenco strokes don't sit"

Even at 22× confidence drums still had a perceptible drift at specific
beats.  A **sliding-window local cross-correlation** told us why:

```
window:  5 s centred at MP3 times 10, 15, …, 105 s
search:  only ± 0.25 s around the global offset (to avoid re-aliasing)
```

This returned a **median local lag of +0.3 ms and std 84 ms for
drums**, but **median −118.5 ms for bass**.  In other words, the global
cross-correlation had found a peak that was *already* 118 ms off — a
tiny side-lobe, but audibly wrong.  We refined the bass offset to the
sliding-window median.

**Lesson:** the global peak is a statistical best-fit.  Always verify
with a local sliding check before trusting it.

### 2.5 · Sub-sample precision

The raw argmax is sample-accurate (~45 µs at 22 kHz).  For very tight
hits that's still not enough.  We added **parabolic interpolation**
around the peak:

```python
def parabolic(y, k):
    a, b, c = y[k-1], y[k], y[k+1]
    frac = 0.5 * (a - c) / (a - 2*b + c)
    return frac          # fractional offset from sample k
```

Adds maybe 10 lines of code; gives you sub-sample offsets for free.

### 2.6 · Pitfall #3 — WhatsApp's variable-frame-rate exports

After fixing the offset math, drums in the render still looked drifting
from their own audio.  `ffprobe` revealed:

```
drums.mp4 video: avg_frame_rate = 291450/9713 ≈ 30.006 fps  (nominally 30)
drums.mp4 audio: duration 129.567 s
drums.mp4 video: duration 129.507 s   ←  60 ms shorter than audio
```

This is variable-frame-rate (VFR) video, typical of mobile exports.
**Fix:** re-encode to strict 30 fps CFR **with `aresample=async=1`** to
keep A/V aligned:

```bash
ffmpeg -i drums.mp4 \
  -vf fps=30 \
  -af aresample=async=1 \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -ar 44100 \
  -vsync cfr \
  drums_clean.mp4
```

### 2.7 · Pitfall #4 — a 2 s V/A drift introduced by `setpts`

Our render started with 2 s of black before the video fades in.  The
first implementation added the intro with:

```
[drums]trim=...,setpts=PTS-STARTPTS+2/TB,fade=t=in:st=2:d=2:alpha=1
```

That line **shifts every frame 2 s forward** but leaves the audio
stream untouched.  Result: the rendered output had **drum audio leading
drum video by 1.87 s** — we measured it with an OpenCV motion vs audio
onset correlation on the output file itself:

```
drums lag = -1.867 s   conf=7.70×
bass  lag = -1.233 s   conf=6.37×
```

**Fix:** remove the `setpts` shift, extend the fade-in to cover the
black period instead (`fade=t=in:st=0:d=4:alpha=1`).  Re-measured:

```
drums lag = +0.133 s   conf=7.89×
bass  lag = -0.467 s   conf=5.98×
```

**Lesson:** when you manipulate timestamps on video streams, *always*
verify the final output with an independent A/V alignment measurement
— audio-to-audio sync proves nothing about whether the video frames
line up with the audio you actually render.

### 2.8 · Picking the right window to show

The song is 110 s long; we only wanted to show a ~50 s clip.  Which
section?

```python
# per-track RMS in half-second bins → find where the player is actually active
active_bass  = [6.8 s .. 66.5 s]      # limited by bass recording
active_drums = [17.0 s .. 108.9 s]
intersection = [6.8 s .. 66.5 s]
```

We kept a 1 s safety margin at each end and used **MP3[12 s … 65.5 s]**.
Bass hand motion peaks early in the recording (flamenco strokes
around MP3 t = 5–40 s), drums peak mid-song.  Cropping to the *active*
window also solved another user complaint: *"I don't want to see
myself walking to the bass in the first frames."*

### 2.9 · Verifying visually — the optical sync check

Audio cross-correlation tells you what the *audio* offset is.  It
doesn't tell you whether the *video frames* match.  We added an optical
sync step: for each video, compute a **per-frame motion envelope**
(pixel-diff on a cropped hand region at 30 fps) and correlate it
against the MP3 onset envelope in the same band:

```python
# Motion
frames = extract_frames(video, 30_fps, grayscale, crop_hands=True)
motion = np.abs(np.diff(frames, axis=0)).mean(axis=(1,2))

# Audio onsets
env = librosa.onset.onset_strength(y=bandpassed_mp3,
                                   sr=sr, hop_length=sr//30)

# Correlate the two envelopes
```

For drums this gave an independent confirmation:

| Source | Offset | Confidence |
|---|---|---|
| Audio-only xcorr | −26.166 s | 22× |
| Optical xcorr    | −26.133 s | 6× |

33 ms agreement — well within musical tolerance.  If the two methods
*disagreed*, it would mean either the MP3 tracks had been edited after
the video was recorded, or the video had internal A/V drift.

---

## 3 · Under the hood of the practice app

### 3.1 · Time-stretching with pitch preservation

A naïve `AudioBufferSourceNode.playbackRate = 0.7` will slow the audio
but drop the pitch by ~5 semitones — useless for ear training.

We use **SoundTouch** (WSOLA algorithm), wrapped in an
`AudioWorkletProcessor` from
[`@soundtouchjs/audio-worklet`](https://www.npmjs.com/package/@soundtouchjs/audio-worklet).
This preserves pitch up to very slow tempos; quality noticeably
degrades below ~70 %, which is why our slider caps there.

### 3.2 · Pitfall: AudioWorklet needs a ticking input

```js
const node = new AudioWorkletNode(ctx, 'soundtouch-worklet',
    { numberOfInputs: 1, outputChannelCount: [2] });
```

The worklet's `process(inputs, outputs)` reads
`inputs[0][0].length` as its quantum-size reference.  Without an input
stream connected, `inputs[0]` is empty and the worklet silently returns
every render call — you hear nothing.

**Fix:** connect a silent `ConstantSourceNode` just to drive the clock:

```js
const silent = ctx.createConstantSource();
silent.offset.value = 0;
silent.connect(node);
silent.start();
```

### 3.3 · Pitfall: seamless loops via buffer wrap-around

The original worklet's `process()` returns `false` when the source
runs out — which **permanently terminates the processor** per the
AudioWorklet spec.  Re-initializing from the main thread works but
creates a ~10 ms gap at every loop boundary.

Instead we patched the worklet's buffer source to **wrap** instead of
end:

```diff
- target[i*2]     = this.leftChannel[i + position]
- target[i*2 + 1] = this.rightChannel[i + position]
- return Math.min(numFrames, this.leftChannel.length - position);
+ const idx = (i + position) % this.leftChannel.length;
+ target[i*2]     = this.leftChannel[idx];
+ target[i*2 + 1] = this.rightChannel[idx];
+ return numFrames;                       // infinite source
```

SoundTouch now sees a continuous stream and its FFT-window overlap
keeps working across the loop point — seamless, zero gap.

### 3.4 · Pitfall: GarageBand adds trailing silence on export

Exporting a cycle region as WAV often includes a fraction of a second
of silence after the last audio.  For a 16-bar loop at 210 BPM the
intended length is **exactly 16 × 4 × 60/210 = 18.2857 s**; our first
export was 19.316 s, i.e. 1.03 s of silence.

Looping it caused a periodic stutter of *exactly that duration*.
Either trim the WAV to the computed bar length:

```bash
ffmpeg -i drums.wav -t 18.285714 -c:a pcm_s16le drums_trim.wav
```

…or move the end of your cycle region in GarageBand to the last note
onset (not the next bar line) before exporting.

---

## 4 · File-by-file tour

```
index.html
├─ <audio> nothing — all audio is routed through Web Audio nodes
├─ UI: one Play/Stop, one tempo slider + quick-buttons,
│      two per-track mute + volume rows
└─ <script>
   ├─ SoundTouchTrack class
   │  ├─ _createNode()   build a fresh AudioWorkletNode per track
   │  ├─ start()/stop()  connect / disconnect
   │  ├─ setTempo()      postMessage SET_PIPE_PROP  'tempo'
   │  └─ setVolume()/setMute()  GainNode control
   └─ wiring for DOM events

soundtouch-worklet.js
├─ PATCH 1: ProcessAudioBufferSource.extract() wraps with modulo
│          (the seamless-loop trick, §3.3)
├─ SoundTouch WSOLA algorithm   (upstream code)
├─ SimpleFilter                 (upstream code)
└─ registerProcessor('soundtouch-worklet', SoundTouchWorklet)

drums.wav, bass.wav
└─ 16-bar stereo PCM, 44.1 kHz, trimmed to the exact bar length
```

---

## 5 · Read more

- 📝 **[Full write-up on the blog](https://ki-mathias.de/en/tommy-practice.html)** —
  the story behind this repo, with diagrams of the FFT cross-correlation
  and the sliding-window verification method.
- 🏠 **[ki-mathias.de](https://ki-mathias.de/en/)** — more posts on
  building things with Claude Code, audio DSP, and home recording.

## 6 · Credits & licence

- **[SoundTouch](https://www.surina.net/soundtouch/)** by Olli Parviainen —
  WSOLA time-stretching, LGPL 2.1
- **[SoundTouchJS AudioWorklet](https://github.com/cutterbl/SoundTouchJS)**
  by Steve "Cutter" Blades — the worklet wrapper we patched
- **Primus — "Tommy the Cat"** — the reason we went through all of this.
- Built collaboratively with [Claude Code](https://claude.com/claude-code).
  Every FFT plot, every sync pitfall and every ffmpeg one-liner in this
  README came out of the session documented in the blog post.

Licence: MIT for the code in this repo.  SoundTouch upstream remains
LGPL 2.1.  WAV samples are for educational use.
