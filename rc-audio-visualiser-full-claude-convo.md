I'm thinking of making an audio visualiser app in flutter (think 'Electric Sheep'). First though, I need to be \*\*certain\*\* that I can actually tap into audio that's playing on the device. I think this may be a showstopper so need validated, up-to-date info. No guesses, and please use the internet

Good — I now have solid, sourced answers per platform. Here's the honest breakdown:

___

## Can Flutter tap into audio playing on the device?

**It depends heavily on the platform. This is a genuine showstopper on iOS. Here's the full picture:**

___

### 🤖 Android — Possible, with caveats

The `AudioPlaybackCapture` API was introduced in Android 10, giving apps the ability to copy the audio being played by other apps — the audio equivalent of screen capture. [Android Developers](https://developer.android.com/media/platform/av-capture)

However, there are real constraints:

To capture audio, your app must have the `RECORD_AUDIO` permission, must display the `MediaProjectionManager.createScreenCaptureIntent()` prompt (which the user must approve), and both the capturing and playing apps must be in the same user profile. [Android Developers](https://developer.android.com/media/platform/av-capture)

Critically, **the source app can opt out**. The audio from an app can only be captured if that app's player sets its usage to `USAGE_MEDIA`, `USAGE_GAME`, or `USAGE_UNKNOWN`, and its capture policy is set to `AudioAttributes.ALLOW_CAPTURE_BY_ALL`. [Android Developers](https://developer.android.com/media/platform/av-capture) So Spotify, Apple Music, etc. can (and typically do) block this entirely.

There's a Flutter package — `internal_audio_recorder` — that wraps internal audio capture on Android and outputs PCM [Pub.dev](https://pub.dev/packages/internal_audio_recorder), but it's lightly maintained and subject to the same restrictions.

**In practice on Android:** you can capture audio from many apps (games, browsers, your own app), but music/streaming services will likely be silent.

___

### 🍎 iOS — Effectively impossible

This is the showstopper you feared. iOS audio capture is restrictive without native channel bridging. [Medium](https://medium.com/@reach.subhanu/flutter-for-audio-driven-apps-real-time-sound-visualization-audio-processing-and-voice-41f2d2cccb73) More bluntly:

In 2025, no iPhone app — built-in or third-party — can capture the audio of another app playing on the device. Apple mutes all internal audio during screen recordings to enforce DRM. That includes the default iOS screen recorder and every app on the App Store. [Hollyland](https://www.hollyland.com/blog/tips/screen-record-apple-music)

There is **no public API** on iOS for one app to tap the audio stream of another. The microphone is the only input available to third-party apps, and iOS only allows one app to access the microphone at a time [MacRumors](https://www.macrumors.com/2025/06/23/ipados-26s-local-capture-feature-tested/) (this has been a long-standing platform constraint).

So on iOS, your visualiser could only react to **microphone input** (i.e. ambient sound picked up by the mic), not to audio playing internally on the device.

___

### 🖥️ Desktop (macOS/Windows/Linux) — Workable

The `desktop_audio_capture` Flutter package supports system audio capture on macOS (13+), Windows, and Linux, with features including audio data streaming, decibel monitoring, and device management. [Pub.dev](https://pub.dev/packages/desktop_audio_capture)

___

## Summary Table

| Platform | Can capture internal audio? | Notes |
| --- | --- | --- |
| **Android** | ⚠️ Partially | Android 10+, user must approve, source apps can opt out |
| **iOS** | ❌ No | Blocked by Apple at OS level, no workaround in App Store |
| **macOS 13+** | ✅ Yes | Via `desktop_audio_capture`, needs Screen Recording permission |
| **Windows** | ✅ Yes | Via `desktop_audio_capture` |
| **Linux** | ✅ Yes | Via PulseAudio/PipeWire |

___

## Recommendation

If your Electric Sheep-style visualiser is targeting **mobile**, this is a significant architectural constraint. Your real options are:

1.  **Build it as a self-contained experience** — your app plays the music itself (via `just_audio`), so you own the audio stream and can trivially access it for FFT/visualisation. No capture needed.
2.  **Microphone-only on mobile** — react to ambient sound. Simpler, but less reliable.
3.  **Target desktop first** — macOS/Windows give you real system audio capture with no friction.

Option 1 is by far the most practical path for a mobile audio visualiser — many of the best ones work this way (the app _is_ the player).

If I went that way (option 1), what other significant hurdles might I meet?

Good, that's enough to give a thorough, grounded answer. Here's the honest picture:

___

## The significant hurdles ahead (Option 1: your app owns the audio)

### 1\. 🔌 Getting the raw PCM data out of your player

This is the first technical snag. `just_audio` — the gold-standard Flutter audio player — **does not expose a raw PCM stream**. It offers `just_waveform` for extracting a pre-computed waveform from a file, but that's static, not live. [Pub.dev](https://pub.dev/packages/just_audio) For a real-time visualiser reacting to what's actually playing _right now_, you need sample-level data on every frame.

Your realistic options are:

-   **`flutter_soloud`** — the most capable choice. Its `getAudioTexture2D` returns an array of 512×256 floats containing FFT data and wave data on every tick, specifically designed to feed shaders. [Flutter Awesome](https://flutterawesome.com/flutter-audio-plugin-using-soloud-library-and-ffi/)
-   **`flutter_sound`** — can record to a Dart stream of PCM Float32 or Int16 data, letting you process live audio in Dart. [Vercel](https://fs-doc.vercel.app/tau/guides/guides_live_streams.html) But it's a recorder-focused package, not a player with PCM tap.
-   Roll your own via a **platform channel** — write native Swift/Kotlin code that taps an `AVAudioEngine` or `AudioTrack` node and pipes buffers back to Dart. Doable, but non-trivial.

___

### 2\. ⚡ FFT performance — keep it off the main thread

Running Fast Fourier Transforms in Dart on every audio frame is expensive. Real-time audio latency may be high without low-level optimizations, and advanced audio filters require native integration or FFI. [Medium](https://medium.com/@reach.subhanu/flutter-for-audio-driven-apps-real-time-sound-visualization-audio-processing-and-voice-41f2d2cccb73) The prescribed solution is to use Dart `Isolate`s for FFT computation so you don't block the UI thread. You should also throttle UI updates — avoid redrawing waveforms every microsecond. [Medium](https://medium.com/@reach.subhanu/flutter-for-audio-driven-apps-real-time-sound-visualization-audio-processing-and-voice-41f2d2cccb73)

There are pure-Dart FFT libraries (`fftea` is well-regarded), but if you're doing complex per-frame analysis at 60fps on a phone, you'll need to be deliberate about this pipeline.

___

### 3\. 🎨 The "Electric Sheep" visual side — shaders are the path, but there's a learning curve

For anything genuinely psychedelic and generative, you'll want **fragment shaders**, not Flutter widget trees. The good news is Flutter supports this properly now. GLSL shaders with the `.frag` extension can be declared in `pubspec.yaml`, compiled by the Flutter toolchain to the appropriate backend format, and loaded at runtime via `FragmentProgram.fromAsset`. [Flutter](https://docs.flutter.dev/ui/design/graphics/fragment-shaders)

Shaders run directly on the GPU, which allows for efficient and fast graphical processing. [Flutter Agency](https://www.mtc-flutter.com/flutter-blog/exploring-shaders-in-flutter-my-journey-from-knowing-to-implementing) You pass your FFT frequency data in as **uniforms** each frame, and the shader does the per-pixel work on the GPU — that's the right architecture.

The catch: since many operations are required for each frame, the CPU and GPU can be under stress, leading to overheating of a mobile device. Sending an image to the shader via `setImageSampler()` is particularly expensive. [Flutter Awesome](https://flutterawesome.com/flutter-audio-plugin-using-soloud-library-and-ffi/)

Also, Flutter's GLSL support has **restrictions** — notably, vertex shaders are not supported, only fragment shaders [Flutter](https://docs.flutter.dev/ui/design/graphics/fragment-shaders) — which constrains some generative techniques you'd find on Shadertoy.

___

### 4\. 🔥 Thermal throttling on mobile

This is under-discussed but very real for this kind of app. Continuous shader rendering + audio processing + FFT = sustained high CPU/GPU load. On phones, this leads to thermal throttling — the device clocks down to stay cool and your framerate tanks. You'll need to build in quality-scaling (lower resolution render target, simpler shaders) and test aggressively on mid-range hardware, not just the latest flagship.

___

### 5\. 🎵 Audio-visual sync

There's a subtle but annoying problem: the audio callback giving you PCM data and the display refresh (vsync) tick operate on **different clocks**. If you naively push FFT data to the shader on every audio callback, you'll get tearing or stuttering in the visuals. The standard solution is a small ring buffer: the audio pipeline writes FFT results into it, and the render loop reads from it at vsync. Getting this right takes some care.

___

### 6\. 📱 Background behaviour & audio focus

If your app is a visualiser that's also the player, you'll need to handle audio focus correctly — pausing when a call comes in, resuming after, integrating with the system media controls. You should experiment with background playback, audio focus, and waveform visualization. [Vibe-studio](https://vibe-studio.ai/insights/flutter-audio-playback-recording-in-mobile-apps) The `audio_service` package handles most of this, but it adds architectural complexity.

___

## Summary

| Hurdle | Severity | Solution |
| --- | --- | --- |
| Getting live PCM from the player | 🔴 High | Use `flutter_soloud` or native channel |
| FFT performance | 🟡 Medium | Dart `Isolate` + throttled UI updates |
| Shader authoring (GLSL) | 🟡 Medium | Learnable, good docs & examples now exist |
| Thermal throttling | 🟡 Medium | Adaptive quality, real-device testing |
| Audio/visual sync | 🟡 Medium | Ring buffer between audio and render loops |
| Background audio + focus | 🟢 Low | `audio_service` package handles most of it |

The biggest actual risk is the **PCM access gap** — `flutter_soloud` is probably your best bet there as it's the only Flutter-native package that cleanly ties audio playback to real-time FFT data for shader use. Everything else is solvable with good architecture.

in terms of timing/performance... my thinking was vaguely: \* Find a way to isolate 'instruments' or 'audio tracks' (not literally, but something similar) via analysis during the first few seconds of sound, or on major switches like verses/choruses \* Think more in terms of BPM than FPS.... so the actual calculation or seeding of more complex stuff can happen at a lower frame rate, guided more by the audio than the display Does this even make sense?

Yes, it makes a lot of sense — and you're actually converging on some well-established ideas in the demoscene and audio-reactive graphics world. Let me reason through it properly rather than just validate you.

**The "instrument isolation" idea**

What you're describing is essentially **source separation** or at minimum **spectral decomposition** — splitting a mixed signal into perceptually distinct streams. True source separation (like isolating drums from vocals) is a hard ML problem. Facebook's Demucs model does it well but it's heavyweight and runs slower-than-realtime on a phone, so that's probably out.

However, you don't need _literal_ track separation for a visualiser. What you can do very practically is **frequency band decomposition** — split the FFT output into bands (sub-bass, bass, mids, highs) and treat each band as its own "voice" driving a different visual element. This is fast, real-time, and perceptually effective. A kick drum will light up the bass band reliably; hi-hats will hit the highs. It won't be perfect but it'll feel like the visuals "hear" the instruments.

The slightly more sophisticated version of this is **onset detection** — detecting when a new note, beat, or significant sonic event occurs in a given band. That's very computable in real-time and gives you musically meaningful trigger events rather than a continuous stream of numbers.

**The BPM-not-FPS framing**

This is the smart insight. What you're describing is a two-tier architecture:

-   **Fast tier** (~every audio buffer, ~10ms): raw FFT, band energy, onset detection. Runs in an Isolate, produces a stream of musical "events."
-   **Slow tier** (beat-rate, phrase-rate, section-rate): higher-level analysis — BPM, detected phrase boundaries, energy trends. This seeds or transitions the generative visual "state."

Then the **shader renders at full FPS** but it's doing cheap interpolation and procedural work within a state that was set by the slow tier. The GPU isn't recalculating what the visuals should "mean" every frame — it's just beautifully animating toward a target that the audio analysis set at beat-time.

This is essentially how the demoscene has always worked — sync to the music's structure, not to the display clock. There's even a well-known tool called **Rocket** (a "demo sync" system) built exactly around this philosophy, where visual parameters are keyframed against beats rather than time.

**Where it gets genuinely hard**

The tricky part in your scheme is the "major switches like verses/choruses" detection — that's **musical structure segmentation**, which is a harder problem than beat tracking. BPM and beat tracking are well-solved (the `aubio` library does this excellently; there's no mature Flutter port but you could call it via FFI or a native channel). Detecting that you've just crossed from verse to chorus in real-time is much less reliable — it typically requires looking at several seconds of history and involves heuristics around energy, spectral centroid change, and repetition detection. It's doable but it will sometimes be wrong, and on a phone with limited compute, you'd need a lightweight approximation.

A pragmatic simplification: rather than detecting song sections, detect **energy-level transitions** and **significant spectral shifts**. These correlate well with section changes without requiring true structural analysis. When the overall energy drops and comes back up (intro → verse → chorus pattern), that's a detectable signal even with simple methods.

**So — does it make sense overall?**

Yes. The architecture you're intuiting is sound:

1.  Real-time FFT → band energies + onset events (fast Isolate)
2.  Beat tracker → beat clock that everything else syncs to
3.  Phrase/section heuristics → seeds visual "scenes" or parameter transitions
4.  Shader renders at 60fps but interpolates within a musically-informed state space

The main calibration you'll need to do is managing the **latency** between audio analysis and visual response. Audio buffers arrive slightly behind real playback time, so you'll need to compensate — typically by shifting your analysis window slightly ahead or accepting a small but consistent visual delay. For an ambient, Electric-Sheep-style aesthetic (rather than a tight DJ tool), a 50–100ms lag is probably imperceptible.

It's an ambitious project but the architecture is coherent.

what about using the microphone to listen to whatever's playing?

This is actually a pretty elegant sidestep — and it's worth thinking through honestly.

**What it solves**

It completely bypasses the internal audio capture problem on iOS. The mic hears whatever's playing through the speakers, you get a real PCM stream, and your FFT/beat-tracking pipeline doesn't care where the audio came from. Architecturally, nothing changes.

**The real problems**

The fundamental issue is that a phone microphone in a real environment is a terrible audio source for this purpose. You get room acoustics, reverb, background noise, and frequency colouration from both the speaker and the mic's own response curve. Your bass band will be unreliable because phone speakers roll off hard in the low end — the kick drum energy you were planning to track may barely register. The BPM tracker will still work reasonably well since rhythm survives a lot of degradation, but fine spectral detail will be muddy.

There's also the user experience problem. The mic has to stay active and the phone can't be pocketed or covered. On iOS, having the mic open prevents other apps from using it. It also feels fragile and slightly weird as a product — "point your phone at your speaker" is not a compelling UX.

**Where it actually works well**

If the user is wearing headphones and the phone is playing music, the mic picks up almost nothing useful. But if the phone _is_ the speaker — playing music out loud — then the mic and speaker are on the same device, which brings up an interesting point: on Android you can sometimes route the mic input to pick up the speaker output with surprisingly low noise in a quiet room, because the physical coupling is consistent. It's not clean but it can be good enough for beat tracking and rough band analysis.

The more honest version of this approach is that it works fine as a **fallback mode** — for when the user wants to visualise music from a different source entirely (a vinyl record, a TV, someone else's phone). Framed that way it becomes a feature rather than a workaround.

**The better answer for your use case**

Since you're already committed to Option 1 (your app owns playback), you don't need the mic at all — you have clean PCM access. The mic path only makes sense if you want to support "external source" mode as an added feature, which is genuinely useful and worth the code complexity.

Short answer: it works well enough to be a fun extra, works poorly enough that you wouldn't want it as your primary path.

give me a very high level intro to ffts as they'd relate to an app lke this

An FFT — Fast Fourier Transform — solves one fundamental problem: a raw audio signal is a single stream of numbers representing air pressure over time, and that tells you very little that's musically useful. What you actually want to know is _which frequencies are present, and how loud is each one, right now_. The FFT is the mathematical operation that converts from one representation to the other.

**The core idea**

Any sound wave, no matter how complex, can be decomposed into a sum of simple sine waves at different frequencies and amplitudes. A piano chord is just several sine waves added together. A snare hit is a chaotic mess of them. The FFT takes a chunk of your raw PCM samples — say, 1024 or 2048 numbers — and tells you the amplitude of each frequency component within that chunk. The output is essentially a detailed snapshot of the frequency spectrum at that moment in time.

Visually, imagine the waveform display on a recording app (the wiggly line) versus the equaliser display (the bars at different frequencies). The FFT is what converts the former into the latter.

**What the output looks like**

If you feed 2048 samples in, you get roughly 1024 frequency "bins" out — each bin corresponding to a range of frequencies, from near-zero up to half your sample rate (this ceiling is called the Nyquist frequency, and it's a hard physical limit). At a 44,100 Hz sample rate you get bins covering 0 to 22,050 Hz, which covers the full range of human hearing.

The amplitude of each bin tells you how much energy is present at that frequency right now. Silence produces bins near zero everywhere. A pure sine wave produces one bin spiking and everything else flat. Real music produces a constantly shifting landscape of peaks and valleys.

**The musically useful bits**

The raw 1024 bins are far more granularity than you need for a visualiser. In practice you group them into bands that correspond to how humans perceive sound — and crucially, how music is structured:

-   **Sub-bass** (20–60 Hz): the physical thump, felt more than heard. Kick drum body.
-   **Bass** (60–250 Hz): bass guitar, kick drum punch, the foundation of rhythm.
-   **Low-mids** (250–2000 Hz): most melodic instruments live here — guitars, piano, vocals.
-   **High-mids** (2–6 kHz): presence, attack transients, the "crack" of a snare.
-   **Highs** (6–20 kHz): cymbals, air, hi-hats, the shimmer on top.

For your app, these five bands (or however many you choose) become the "instruments" you mentioned — not literally isolated tracks, but perceptually distinct voices that you can wire to different visual elements.

**The timing reality**

You don't run one FFT on the whole song — you run it continuously on overlapping windows of samples as the audio plays. A typical window might be 2048 samples, advancing by 512 samples each time (this overlap is important for temporal smoothness, and the technique is called a Short-Time Fourier Transform or STFT). At 44,100 Hz sample rate, 512 samples is about 12 milliseconds — so you're getting a fresh frequency snapshot roughly 86 times per second. That's faster than your display refreshes, which is why the "slow tier / fast tier" architecture you described earlier makes sense: the FFT runs faster than the screen, and you smooth or interpret its output before handing it to the shader.

**One gotcha — the window function**

Raw FFTs on a rectangular chunk of samples produce artefacts at the edges (because the maths assumes the signal repeats, and the discontinuity at the boundary creates phantom frequencies). In practice you multiply your sample window by a smoothing curve — called a Hann or Hamming window — before running the FFT. Every audio analysis library does this for you, but it's worth knowing it exists because it affects the frequency resolution of your output.

**What this means for your architecture**

The FFT gives you the raw material. What you build on top of it — onset detection, beat tracking, band energy smoothing, section-change heuristics — is all just progressively more musical interpretation of those frequency snapshots. The shader at the end doesn't see raw FFT data; it sees musically-meaningful numbers like "bass energy this beat", "high-frequency novelty this bar", "estimated BPM". The FFT is the foundation everything else stands on.

About FFT maths (ie how the input becomes the output). I'm not a mathematician but I'm really interested. Is there any point in trying to explain it to me, or should I not even bother?

Honestly? It's absolutely worth trying, and it's more approachable than its reputation suggests. The maths notation looks intimidating but the underlying ideas are genuinely beautiful and you can get real intuition for them without being able to manipulate the equations yourself.

There are a few conceptual stepping stones and if any one of them doesn't click, everything after it gets harder. So rather than launch into an explanation now, let me just check where you're starting from — it'll let me pitch it at exactly the right level rather than either boring you or losing you.

Three quick questions:

\[User dismissed — do not proceed, wait for next instruction\]

No problem — whenever you're ready, just ask and we'll pick it up.

Yeah I just don't like the 'survey questions' format

Fair enough, sorry about that. Just tell me roughly where you're at with sine waves and complex numbers and I'll take it from there.

I am a coder so I encounter sine waves. I like a visualisation I saw once of a coloured patch on a bicycle wheel, then the wheel turns until you see it face-on like only the thinnest view, and you see the patch move up and down (the x axis has been 'removed' by the rotation of the wheel) complex numbers, not so sure I think my brain is \*\*reasonably\*\* mathematical, I mean I've programmed for several decades and there must be some amount of crossover between programming logic/algorithms and maths-thinking. But my maths education is poor and yeah, those symbols!

That bicycle wheel visualisation is actually one of the best possible starting points — whoever made that knew what they were doing. The patch on the rim traces a sine wave as the wheel turns. The wheel _is_ the sine wave, just seen from the side. Hold that image, because it's going to do a lot of work.

And you're right about the programming crossover — the FFT is fundamentally an algorithm, and when you see it described that way it's far more approachable than the sigma notation suggests. Knuth wrote about it, it has a loop structure, it has a elegant divide-and-conquer trick at its heart. You'll recognise the shape of it.

Complex numbers are the one genuinely new idea you'll need, but here's the thing — you already have the intuition for them from that bicycle wheel, you just don't know it yet. The "trick" that makes complex numbers useful here isn't about imaginary square roots at all, it's about a cleaner way to talk about rotation. Which is exactly what your wheel is doing.

So the path I'd take you on is roughly:

-   Your wheel, but now imagine _two_ patches at different speeds — what does the combined signal look like from the side? That's what a chord sounds like as a waveform.
-   The question the FFT answers: given only the side view (the waveform), how do you figure out what wheels are spinning, and how fast?
-   Why complex numbers are just "the wheel seen from above" rather than the side — a 2D position, not a weird imaginary thing.
-   How the FFT's divide-and-conquer trick makes this calculation fast enough to be real-time.

Want me to just go for it?

first: I know even less about complex numbers than you imagine, like I don't even have a broad working definition?

OK good to know — let's fix that first because it'll make everything else cleaner.

You know how numbers live on a line? You've got zero in the middle, positive numbers stretching off to the right, negative numbers to the left. Every number you normally deal with lives somewhere on that line.

Complex numbers are what happens when you say — what if numbers could also go _up and down_, not just left and right? Instead of a line, you now have a 2D plane. Every complex number is just a point on that plane: some distance left/right (the "real" part, your normal number) and some distance up/down (the "imaginary" part, which is a terrible name that has caused centuries of unnecessary confusion).

So a complex number isn't really one number — it's a **pair of coordinates**. Like an x,y position on a graph. That's genuinely all it is at the level we need.

The reason they got saddled with the name "imaginary" is historical baggage — mathematicians encountered them while solving equations and felt embarrassed about them, as though they weren't "real". But for our purposes, forget all that. Think of a complex number as just a point, or equivalently an arrow from the origin to that point, living in a 2D space.

Now here's why this connects to your bicycle wheel: that coloured patch on the rim? At any moment in time, its position can be described as exactly one of these arrows — some distance left/right, some distance up/down. As the wheel turns, the arrow _rotates_. Complex numbers, it turns out, are the most natural mathematical language for describing rotation. That's the connection, and that's why they're at the heart of the FFT.

Want to go further, or does that need to settle first?

So I could almost get away with replacing 'complex number' with 'pair of coordinates' to get a broad overview? But their relationship to the origin is important (although I guess that's true of coordinates anyway, they have to relate to some origin too). In this example does it make sense to think of the origin as the central hub of the wheel? The arrow base being at that point, head at the patch? No because that would just mean the arrow is spinning with the wheel right?

Yes to the first part — "pair of coordinates" will get you surprisingly far, and yes the relationship to the origin matters in exactly the way it does for any coordinate system.

Your wheel instinct is almost right but you spotted the problem yourself. If the arrow spins with the wheel, it's always the same length pointing in a rotating direction — that describes the _wheel_ perfectly, but it doesn't give you the _wave_.

The key move is this: the wave is what you get when you **project** that spinning arrow onto a single axis. You're collapsing the 2D rotation down to 1D by just watching one component — the up/down position of the patch, ignoring left/right. That's the sine wave you see from the side.

The complex number picture keeps _both_ components — it watches the full 2D rotation without collapsing it. So where the wave throws away information (the left/right component), the complex representation keeps it. And that turns out to be crucial for the FFT, because the information you'd lose by just looking at the wave from the side is exactly what you need to untangle multiple wheels spinning at different speeds.

Which sets up the key question beautifully: if you're only given the side view — the mixed-up wave of multiple wheels — how do you figure out which wheels are spinning and how fast? That's precisely what the FFT solves, and the reason complex numbers are the natural tool is that they let you work with the full 2D rotation rather than the flattened 1D shadow of it.

Ready to go at that next bit?

This is good but I don't think I've grasped it well enough yet, might need some more analogies. If we stick with the patches, and imagine an animated point on a graph: \* single patch, we just see the point move up and down (or left/right), with that nice sine timing? \* 2 patches, as above but now the other axis is doing the same thing (offset?)?

You're close but not quite — let me untangle it.

With a single patch, yes — project onto one axis and you get a sine wave. The point bounces up and down with that smooth sine timing. Perfect.

But two patches isn't about the other axis doing its own thing. Let's keep everything projected onto the _same_ axis — because that's what a microphone does, it only "sees" one axis, the air pressure going up and down. There's no second axis available in a real sound wave.

So: two patches on the _same_ wheel, at different positions around the rim. Project both onto the vertical axis, and watch what the _combined_ up-down position looks like. It's not a clean sine wave anymore — it's two sine waves added together, making a more complex wobbly shape. If the patches happen to be opposite each other they might partially cancel. If they're close together they reinforce. The resulting shape depends entirely on the sizes of both patches (amplitude) and where they sit around the rim (phase).

Now make it even more interesting: two patches on _different sized wheels_, spinning at different speeds, both projected onto the same vertical axis, their contributions just added together. That combined signal is your audio waveform. A messy, complicated wiggle that contains both wheels' information, collapsed into a single bouncing point.

The FFT's job is to look at that single bouncing point and work backwards — figuring out how many wheels there are, how fast each is spinning, and how big each patch is.

Does that feel clearer before we move on?

I need to go slower. Back to the animated points, how about: \* single point, as described above \* 2 patches, now we have 2 points on the graph (as an intermediary step) \* next step, for each 'frame' of animation, the 2 points are reduced to a single one... by like averaging? something more complex?

Great instinct to slow down, and you're on exactly the right track. The answer to your last question is simpler than averaging — it's just **adding**.

For each frame, you take the vertical position of point 1 and the vertical position of point 2, and you add them together. That single number is the combined signal.

So if at some moment patch 1 is at height +0.6 and patch 2 is at height +0.3, the combined signal reads +0.9. If patch 1 is at +0.6 and patch 2 is at -0.6, they cancel perfectly and the combined signal reads 0.

That's it. No weighting, no averaging, just addition. And this is physically accurate — it's exactly how sound works in air. Two sound waves arriving at a microphone just add their pressure contributions together at every instant. The microphone doesn't know there were two sources, it just measures the total.

So your three steps become:

-   **One patch**: one animated point bouncing up and down, tracing a clean sine wave
-   **Two patches**: two animated points, each bouncing at their own rate and amplitude
-   **Combined signal**: at every frame, add the two heights together to get a single number — that number is what the microphone records

The combined signal will look like a messier, less regular bounce — sometimes the two reinforce, sometimes they partially cancel, depending on where each wheel happens to be at that moment.

Sitting comfortably with that before we go further?

this may be a distracting/silly/pointless question, but how do we 'know' that in your example the 0.9 is 0.6 + 0.3, and not 0.4 + 0.5? or do we not need to know?

Also does this all happen in the positive direction ie no negative numbers?

Not silly at all — it's actually the _central_ question. And the answer is: **we don't know, and that's exactly the problem the FFT exists to solve.**

All the microphone ever sees is 0.9. It has no idea it came from 0.6 + 0.3 versus 0.4 + 0.5 versus 0.1 + 0.8. That information is gone the moment the waves add together. The FFT is the mathematical detective work that looks at how that combined number _changes over time_ and reasons backwards to figure out which wheels were spinning. It can't do it from a single frame — it needs a whole window of samples, which is why you feed it a chunk of audio rather than one number at a time.

So your instinct was pointing directly at the heart of the problem. Good sign.

___

On negative numbers — yes, absolutely they're involved. The patches spend half their time below the centreline, so the vertical position swings between roughly -1 and +1. When one patch is at +0.8 and another is at -0.8 they cancel almost completely to near zero. This is actually important musically — it's how certain combinations of frequencies can create "quiet" moments in a waveform even when both wheels are spinning at full size. It's also why audio signals look like they're oscillating around zero rather than always being positive.

wierd so it's like an individual slice/moment/sample of sound can effectively be 0/silent... but when combined with other samples it can somehow represent part of an audible moving wave?

Yes, and this is one of those moments where the reality of how sound works is genuinely counterintuitive.

A single sample is just a number. It has no meaning in isolation — you can't hear it, it has no pitch, it represents nothing musical by itself. Pitch and timbre only _emerge_ from the pattern across thousands of samples over time. A single frame of 0 could be part of a loud chord or part of silence — there's no way to tell from that one number alone.

A good analogy: take a photograph and look at a single pixel. It's just a colour. It tells you nothing about whether that photo is a portrait or a landscape or a picture of a cat. The meaning only exists in the _relationships_ between pixels across the whole image.

Or think about it from the other direction — if you took any one frame of your animated bouncing point and froze it, you couldn't tell how fast the wheel was spinning, or even if it was spinning at all. The speed — the frequency — only reveals itself through _change over time_.

This is actually why the FFT needs a chunk of samples rather than one at a time. You're not asking "what frequency is this number?" — that question is meaningless. You're asking "what pattern of frequencies best explains how this sequence of numbers is changing?" And you need enough samples to see at least one full cycle of the slowest wheel you care about, otherwise you can't reliably detect it.

It's a bit like needing to watch a clock for at least a full minute before you can be sure the minute hand is actually moving.

ok and musically/audibly, what does a full spin of the wheel represent?

One full spin of the wheel is one complete cycle of a sine wave — and that maps directly to **one cycle of a frequency**. The number of full spins per second is the frequency in Hz.

So:

-   A wheel spinning 440 times per second produces the note A4 — the standard orchestral tuning note, the one used to tune instruments
-   A wheel spinning 261 times per second produces middle C
-   A wheel spinning 20 times per second is at the edge of human hearing — you'd feel it more than hear it
-   A wheel spinning 20,000 times per second is about the upper limit of hearing for a young person

The faster the wheel spins, the higher the pitch. That's it. Frequency _is_ spin rate, Hz _is_ spins per second. They're the same thing described two different ways.

And amplitude — the size of the patch's contribution, how far up and down it swings — that's the volume of that particular frequency. A big patch on a slow wheel is a loud low note. A tiny patch on a fast wheel is a quiet high-pitched tone.

So when you have a real piece of music and you run the FFT on a chunk of it, what you're getting back is essentially a list of wheels — here's how fast each one is spinning (frequency/pitch) and here's how big each patch is (amplitude/volume) at this particular moment in time. Which is exactly the information your band-splitter needs to say "lots of energy in the bass band right now, kick drum just hit."

Good catch — I used that word loosely and it deserves unpacking.

It's not really a moment. As we established, a single sample tells you nothing. The FFT output represents a _window_ of time — that chunk of, say, 2048 samples we mentioned earlier. At 44,100 samples per second, that's about 46 milliseconds of audio.

So when I said "at this particular moment" I was being sloppy. It's more like "during this particular window." The FFT gives you the frequency content _averaged across that whole window_. If a note starts halfway through the window, the FFT will see it but weakly — it only occupied half the window. If a note rings clearly through the entire window, it shows up strongly.

This creates an interesting tension that's fundamental to signal processing, with an actual name — the **time-frequency uncertainty principle** (deliberately named after the quantum mechanics one, because the maths is analogous):

-   A longer window gives you better frequency resolution — you can distinguish two very close pitches — but worse time resolution, because you're smearing across a longer slice of time
-   A shorter window gives you better time resolution — you can pinpoint _when_ something happened — but worse frequency resolution, because you haven't seen enough cycles to accurately measure slow wheels

For your app this is a real design decision. If you want to accurately detect bass frequencies (slow wheels, needing many cycles to measure), you need a longer window. If you want to react sharply to a snare hit, you want a shorter window. In practice people often run multiple FFT sizes simultaneously for exactly this reason.

how does the FFT 'know' for a sample if it is made up of 2 or 8 wheels or whatever? Is that ability tied to the window size?

Really good question, and the answer is slightly surprising: **it doesn't choose**. It doesn't decide "there are 3 wheels here." It always gives you the _same number_ of wheels back, regardless of what's actually in the signal.

If you feed in 2048 samples, you get 1024 frequency bins back. Every single time. All 1024 bins always have a value. The FFT isn't detecting how many wheels exist — it's measuring the energy at every possible spin rate simultaneously, whether anything is spinning there or not.

Most of those bins will come back near zero — "nothing spinning at this speed." A few will come back with significant values — "something spinning here." But the FFT itself doesn't make that judgement. That's your job afterwards, looking at the output and deciding which bins are meaningfully above zero and which are just noise.

So it's less like a detective identifying suspects, and more like a bank of 1024 incredibly sensitive microphones, each tuned to one specific frequency, all listening at once. Most of them report near-silence. A few report real energy. You look at the results and draw your own conclusions.

And yes, this is tied to window size — but not in the way you might expect. The window size determines _which_ frequencies you can measure and _how precisely_. Specifically:

-   The number of bins you get is always half your window size
-   The lowest frequency you can detect is determined by how long the window is — you need to fit at least one full cycle in
-   The spacing between bins (your frequency resolution) gets finer with a longer window

So a longer window doesn't help you find _more_ wheels — it helps you distinguish wheels that are spinning at _very similar speeds_.

so I could run eg 3 FFTs, each aiming to capture specific frequency groups (bands?)?

Yes, exactly — and this is a legitimate real-world technique. But the way you'd typically do it is slightly different from running three completely separate FFTs.

The more common approach is to run **one FFT** with a window size tuned to a reasonable middle-ground, and then _group the output bins_ into your bands afterwards. Since you get 1024 bins back covering the full frequency range, you just decide which bins belong to bass, which to mids, which to highs, and sum or average the energy within each group. That's fast and simple.

But your instinct about running multiple FFTs points at something real — the time-frequency tradeoff we talked about. Bass frequencies genuinely _need_ a longer window to measure accurately, because the wheels are spinning slowly and you need to see enough cycles. High frequencies are fine with a short window because their wheels spin fast and complete many cycles even in a small chunk of time.

So the sophisticated version of your idea — called a **multi-resolution analysis** — does something like:

-   Long window FFT for the low bands, good frequency resolution, accepts slower time response
-   Medium window FFT for the mids
-   Short window FFT for the highs, snappy time response, accepts fuzzier frequency resolution

This maps quite nicely onto musical reality. It doesn't matter if your bass detection is 50ms behind — bass frequencies are slow-moving anyway. But you want your hi-hat detection to be razor sharp because transients at high frequencies are over in milliseconds.

There's even a formal version of this idea called a **wavelet transform**, which essentially does continuous multi-resolution analysis rather than picking three fixed window sizes. It's more complex to implement but conceptually it's just the logical conclusion of what you're describing.

For your app, honestly, a single FFT with grouped bins is probably the right starting point. You'd only reach for multi-resolution if you found that bass detection felt sluggish or hi-hat response felt smeared — which you might, but it's an optimisation rather than a necessity from day one.

yeah that's where I was going with the idea, the 3 (or whatever) FFTs having different 'accuracies' (of timing I guess then?) to help with performance Are wavelet transforms more computationally expensive?

Yes, generally more expensive — but the comparison is a bit nuanced.

The FFT is famously fast. The "Fast" in Fast Fourier Transform refers to a specific algorithmic breakthrough (that divide-and-conquer trick I mentioned earlier) that reduced the computation from something that scaled horribly with window size to something very manageable. It's one of the most optimised algorithms in existence, with decades of hardware and software tuning behind it. Extremely well-understood, heavily optimised libraries everywhere.

Wavelet transforms don't have quite the same universal "fast" shortcut, though efficient implementations exist. They're also more complex to implement correctly and harder to reason about — there are many different wavelet "shapes" to choose from (unlike the FFT which has one definition), and picking the right one for audio is a non-trivial decision.

For your specific use case though, I'd reframe the question slightly. The multi-resolution FFT approach you were describing — three FFTs at different window sizes — is probably _more_ expensive than one single FFT, not less. You're doing three transforms instead of one.

The performance win in your original thinking wasn't really about FFTs at all — it was about the **downstream work**. The insight was that you don't need to re-seed your complex visuals at 60fps, you can do heavy visual computation at beat-rate or phrase-rate. That's where the real savings are, and that holds regardless of whether you use one FFT, three FFTs, or wavelets.

The FFT itself, even running continuously, is cheap enough on modern hardware that it's unlikely to be your bottleneck. What you do with the results is where the interesting performance decisions live.

right. yes I keep saying 3 FFTs but I get your point about the grouped bins so tell me a bit more about the mathematical magic?

Right, so here's where it gets genuinely beautiful.

The core question is: given a window of mixed-up samples — all those wheels added together and collapsed to a single bouncing point — how do you separate them back out?

The insight is surprisingly intuitive once you see it. Imagine you suspect there's a wheel spinning at 440Hz (the A note). You want to know if it's really there, and if so how big its patch is. Here's what you do:

You take your own imaginary reference wheel, spinning at exactly 440Hz. Then you compare it against the signal, sample by sample, by multiplying them together and adding up the results across the whole window.

If there really is a 440Hz wheel in the signal, your reference wheel and that component will be **in sync** — they'll reinforce each other across the whole window, and the sum will be a large number.

If there's no 440Hz component, your reference wheel will sometimes be pointing the same way as the signal and sometimes the opposite way — over the whole window these will cancel out, and the sum will be near zero.

This process — multiply two signals together sample by sample and sum the results — has a name: the **dot product**, or in this context the **inner product**. You probably know dot products from vector maths. This is the same operation, just applied to signals rather than spatial vectors.

And here's the magic: you repeat this for every frequency bin. Every possible spin rate gets its own reference wheel, and you measure how much the signal "agrees with" each one.

The naive way to do this is exactly as described — for each of your 1024 bins, spin a reference wheel at that frequency, multiply against every sample, sum it up. That's 1024 × 2048 multiplications. Expensive, and this naive version is called the **Discrete Fourier Transform** or DFT.

The FFT is just a clever observation that a huge amount of that work is **redundant**. Many of those reference wheels share mathematical structure with each other — specifically, a wheel spinning at frequency 2x completes exactly twice as many cycles as one at frequency x, so half its values are predictable from the slower one.

The FFT exploits this by splitting the window in half, computing the transform of the even-numbered samples and the odd-numbered samples separately, then combining the results. And each of those halves can be split again. And again. All the way down to windows of size 1, which are trivially their own transform.

This is the divide-and-conquer you'd recognise from sorting algorithms — it's the same shape as merge sort. And like merge sort, it reduces the work from n² to n log n. For 2048 samples that's the difference between about 4 million operations and about 22,000. That's why it's fast enough to run in real time.

The reference wheels are where complex numbers come in properly, and it's actually very concrete. Each reference wheel at each moment in time is a point on a circle — some amount of "horizontal" and some amount of "vertical". That's your complex number. Just a 2D position on the unit circle. Multiplying two complex numbers together is equivalent to _adding their angles_ — which is exactly what you need to advance a spinning wheel forward in time. Complex multiplication _is_ rotation, which is why complex numbers are the natural language for this.

So the full picture is: the FFT is a highly optimised way of asking "how much does my signal agree with a reference wheel at each possible frequency?" — using the dot product to measure agreement, complex numbers to represent the spinning reference wheels, and a recursive divide-and-conquer trick to avoid doing the same work multiple times.

The output — your 1024 bins — is just the answer to that question at each frequency. Big number means strong agreement, meaning that wheel is really spinning in your signal. Near zero means no agreement, meaning that frequency isn't present.

That's genuinely all it is.

the bins contain something like an amplitude for a frequency? please say yes

Yes.

Well — almost yes, and the qualification is small enough that you can ignore it for now if you want.

Each bin contains a **complex number** — that 2D coordinate pair. The reason is that the reference wheel comparison tells you two things: how strong the frequency is, _and_ where in its cycle it currently sits (its phase — basically, where around the rim the patch is at the start of the window).

But for a visualiser, you almost certainly don't care about phase. You just want the amplitude — how loud is this frequency right now. And getting that from the complex number is trivial: it's just the distance from the origin to that 2D point. Pythagoras — square root of (a² + b²). One simple calculation per bin and you have your amplitudes.

So in practice, for your purposes: yes. The bins contain an amplitude for a frequency. Everything else is a detail you can safely set aside.

Good instinct — knowing what to ignore is half of engineering.

So you now have a working mental model of the whole pipeline, which is not nothing:

-   Sound is wheels spinning at different speeds, added together into one signal
-   A window of samples is a brief recording of that combined signal
-   The FFT spins reference wheels at every possible frequency and measures how much each one agrees with the signal
-   Out come 1024 amplitudes, one per frequency bin
-   You group those bins into bands
-   Those bands drive your visuals

That's a genuinely solid foundation. When you get into actual implementation and start seeing terms like "Hann window", "magnitude spectrum", "spectral flux" — you'll have enough intuition to know roughly where they fit in that picture.

Want to go back to the app architecture, or is there something else about the FFT you want to poke at first?

I think I need a short gentle coma first

Ha. Fully earned. It's a lot to absorb even when it clicks nicely.

When you surface, we've still got the interesting stuff to get into — beat tracking, onset detection, how you'd actually wire FFT output to shader uniforms, and the architecture of that two-tier fast/slow system you intuited earlier. Good foundations for all of it now.

Go rest the maths brain.

ok one big mistake I made with this, please tell me how important it is: I've been talking as if all sound is sine waves!

It's not a mistake at all — it's actually the correct starting assumption, and it's not just a simplification.

This is a theorem: **any** periodic signal, no matter how complex or harsh or jagged, can be represented as a sum of sine waves. It's called Fourier's theorem, after the same person the transform is named after. It's not an approximation, it's mathematically exact.

A square wave — the harshest, most un-sinusoidal shape you can imagine — is provably equal to an infinite sum of sine waves at specific frequencies and amplitudes. A sawtooth wave, same thing. A snare hit, a vowel sound, a distorted guitar — all of them, sine waves added together.

So when you were thinking in terms of wheels, you weren't making a naive assumption that music is smooth and gentle. You were correctly identifying the fundamental building blocks of _all_ sound. The wheels aren't a metaphor for simple sounds — they're the literal mathematical constituents of every sound that exists.

This is actually why the FFT is so powerful and so universal. It works on any signal at all, not just "nice" ones, because any signal at all is secretly just wheels.

Fourier's original insight — that this was true — was considered almost scandalous when he proposed it in the early 1800s. Other mathematicians didn't believe him for a while. The idea that something as jagged and ugly as a square wave could be made entirely from smooth sine waves seemed absurd.

He was right though.

what does a complex number really look like (how is it written?)

It's written like this:

**3 + 4i**

That's it. The 3 is the horizontal coordinate (real part), the 4 is the vertical coordinate (imaginary part), and the i is just a label meaning "this number is on the vertical axis." You can read it as simply "3 across, 4 up."

The i has a formal mathematical definition (it's √-1, which is where all the "imaginary" baggage comes from) but for your mental model you can completely ignore that. It's just a tag that says "this is the vertical component, don't mix it up with the horizontal one."

So when you see:

-   **3 + 4i** — a point 3 across, 4 up
-   **\-2 + 1i** — a point 2 to the left, 1 up
-   **5 + 0i** — a point sitting right on the horizontal axis, which is just a normal number
-   **0 + 3i** — a point sitting right on the vertical axis

And the distance from the origin — the amplitude you'd extract for your FFT bin — is just Pythagoras: √(3² + 4²) = √(9 + 16) = √25 = **5**.

That's a complex number. Genuinely not more complicated than that to look at.

so it's basically like (3, 4)?

Yes. Exactly that.

The notation "3 + 4i" versus "(3, 4)" is just historical accident. Mathematicians developed complex numbers before they thought of them as coordinates on a plane, so they wrote them in the additive form. If complex numbers had been discovered later, after coordinate geometry was already established, they'd probably just be written as pairs and none of the "imaginary" confusion would ever have happened.

In fact in code, that's often exactly how they're represented — a struct with a real field and an imaginary field, which is just x and y with different names.

The whole edifice of "imaginary numbers" that puts people off mathematics for life is, at the level you need it, just a 2D coordinate with an unfortunate naming history.

out of interest, "at the level I need it" <- what else is it used for at other levels? and why 'imaginary'?

**Why imaginary**

It came from a specific algebraic problem: what is the square root of a negative number? If you square any normal (positive or negative) number, you always get a positive result. So √-1 seems to have no answer — it shouldn't exist. Early mathematicians needed it to appear in their equations to solve certain cubic polynomials, but felt deeply uncomfortable about it, like they were using a nonsense placeholder that happened to give right answers. Leibniz called it "an amphibian between existence and nonexistence." They called it imaginary as a sort of embarrassed shrug — "we know this isn't a real number, but bear with us."

Then about 150 years later, Gauss and others realised these "imaginary" numbers were just points on a 2D plane all along, and the embarrassment was unnecessary. But the name had stuck.

**What else they're used for**

At deeper levels, complex numbers turn out to be woven into the fabric of reality in ways that are genuinely hard to explain without the full maths:

-   **Quantum mechanics** — the equations that describe how particles behave fundamentally require complex numbers. Not as a convenience, but necessarily. A particle's quantum state is a complex number, and without them the maths simply doesn't work.
-   **Electrical engineering** — AC circuits (which is all mains electricity) are analysed almost entirely using complex numbers because alternating current is literally a sine wave, and complex numbers handle the phase relationships between voltage and current elegantly.
-   **Fluid dynamics** — complex analysis (the deep maths of complex numbers) can solve certain 2D fluid flow problems almost magically, turning hideously complicated differential equations into much simpler ones.
-   **Control systems** — designing the autopilot in a plane, or the stability system in a car, involves complex numbers throughout.

The deep reason they keep appearing everywhere is something mathematicians find almost philosophical — complex numbers are in some sense the "completed" version of ordinary numbers, the natural home where algebra becomes fully consistent and beautiful. There's a famous result called the **Fundamental Theorem of Algebra** which says that every polynomial equation has a solution if you allow complex numbers — something that's simply not true if you restrict yourself to real numbers.

Euler's identity — often called the most beautiful equation in mathematics — connects complex numbers to pi and to natural logarithms in a way that seems like it absolutely should not work:

**e^(iπ) + 1 = 0**

Five fundamental constants, one equation. That's the level at which complex numbers get genuinely deep. But for your FFT pipeline, (3, 4) is all you need.

**i** you know now — it's √-1, the vertical axis label, the thing that makes complex numbers 2D.

**e** is a number, approximately 2.71828, called Euler's number. It's one of those constants that keeps appearing everywhere in mathematics uninvited, the way π does.

The easiest way to get a feel for it is through growth. If you have something growing continuously — not in discrete jumps but smoothly, at every instant — e turns out to be the natural base for describing that growth. Compound interest, population growth, radioactive decay, the way a capacitor charges — they all involve e because they're all continuous processes where the rate of change is proportional to the current value.

The deeper reason e is special is that it's the unique number where the function eˣ is its own derivative — the rate of change of eˣ is eˣ itself. That self-referential property makes it the natural language for anything involving continuous change, which turns out to be most of physics.

So in Euler's identity:

-   **e** represents continuous growth/change
-   **i** represents rotation in the 2D complex plane
-   **π** represents a half turn (180 degrees) of a circle

And what the identity is really saying is: if you grow continuously in the _rotational_ direction (multiply the exponent by i), for a duration of π, you end up exactly at -1 on the number line — half a turn around the unit circle, landing on the left side.

Which is just your bicycle wheel again, in the most compressed possible notation.

I want to check my understanding: \* A bigger (radius) wheel would indicate higher amplitude? \* A faster spin indicates higher frequency (tone)? \* I feel like there's some kind of inverse relationship here somewhere but can't place it (and might just be wrong)?

First two: **yes, exactly right.** You've got those nailed.

The inverse relationship you're sensing — you're not wrong, there is one, and I think you're half-remembering the time-frequency uncertainty principle we touched on earlier. Which is:

-   A **longer window** gives you **finer frequency resolution** but **coarser time resolution**
-   A **shorter window** gives you **finer time resolution** but **coarser frequency resolution**

Those two quantities — time precision and frequency precision — are in a trade-off. You can't have both simultaneously. Squeeze one and the other expands. That's the inverse relationship baked into the mathematics of waves.

There might also be another inverse you're sensing, which is more physical: **low frequencies have long wavelengths, high frequencies have short wavelengths.** Frequency and wavelength are inverses of each other — double the frequency and you halve the wavelength. That's why bass frequencies need a longer window to measure accurately — their cycles are physically longer, so you need more time to see enough of them.

Or you might be thinking of something else entirely — is either of those the relationship that felt like it was lurking?

Something I'm not getting when you talk about a big patch meaning high volume (amplitude). Where is this in the number?

Good question — let me make it concrete.

Remember the complex number is a 2D point, like (3, 4). The amplitude is the distance from the origin to that point — the length of the arrow. √(3² + 4²) = 5 in that case.

So the amplitude isn't a separate thing stored somewhere — it _is_ the arrow length. A big patch on the wheel means the arrow is long. A small patch means the arrow is short. Silent means the arrow has zero length — the point sits right on the origin.

But I realise I may have created some confusion by mixing two different uses of "amplitude" in our conversation, so let me untangle them:

-   **The wheel patch** — its size represents how loud that particular frequency is in the signal. Big patch, loud frequency.
-   **The FFT output bin** — a complex number whose _distance from the origin_ represents that same loudness.

They're describing the same thing from two different angles. The patch size is the intuitive picture. The arrow length is how that same information is encoded in the mathematical output.

So when you extract your 1024 amplitudes from the FFT output, you're just computing the arrow length for each of the 1024 complex numbers. 1024 Pythagoras calculations. Those lengths are your bar heights in a visualiser — long arrow means a tall bar at that frequency.

Does that connect it up?

but it sounds like amplitude is part of the size of the wheel? one of the axes?

You've caught a genuine ambiguity in how I've been talking — good.

Let me be more precise. The wheel has two separate properties:

-   **Radius** — how big the wheel is, how far the patch is from the hub. This is the amplitude. This is what becomes the arrow length in the complex number.
-   **Spin rate** — how fast it rotates. This is the frequency.

The patch isn't contributing to the _size_ of the wheel — the patch is just a marker on the rim so you can see it rotating. It sits at the rim regardless. I've been using "big patch" as loose shorthand for "big wheel" and that was sloppy of me.

So the complex number arrow length maps to the **radius of the wheel**, not the size of any patch. A loud low note is a large slow wheel. A quiet high note is a small fast wheel.

And to your question about axes — the complex number's two components (the x and y, the real and imaginary parts) aren't separately "amplitude" and "frequency." They're both part of encoding the amplitude together, via Pythagoras. The frequency is encoded in _which bin_ the number sits in — bin 1 means slow wheel, bin 1024 means fast wheel. The complex number inside that bin then tells you the radius (via its length) and the phase (via its angle).

Cleaner now?

yeah that was really confusing. It's not loose shorthand, tbf it makes no sense and made me thing there was an extra aspect to the maths I was missing. Surely the 'patch' must be conceptualised as a point? It has no size, only location?

You're completely right and I apologise — it was a bad metaphor that I kept using past its useful life. A point on the rim is exactly what it should have been from the start. It has location, no size.

So to restate cleanly:

-   The **wheel radius** = amplitude. Bigger wheel, louder frequency.
-   The **spin rate** = frequency. Faster spin, higher pitch.
-   The **point on the rim** = just a marker so you can track where the wheel is in its rotation. It has no size, it contributes nothing to the maths. It's just there so you can see the wheel moving.

And the complex number in each FFT bin is simply a snapshot of where that point currently is — its (x, y) position on the rim at this moment. The distance from the origin to that point is the wheel's radius, which is the amplitude. The angle tells you the phase. That's all.

The patch was a helpful way to make the wheel visible in the original bicycle wheel visualisation, but I should have dropped it the moment we moved into the maths. It added a phantom "size" property that doesn't exist and confused things unnecessarily. That's on me.

ok so if we say the y is the amplitude is the x the phase? or is it the angle (requiring both axes) that is the phase?

It's the angle — requiring both axes.

Neither x nor y alone is the amplitude or the phase. They're both just components of the 2D position, and you have to combine them to get the meaningful quantities:

-   **Amplitude** = distance from origin = √(x² + y²) — uses both
-   **Phase** = angle of the arrow = arctan(y/x) — also uses both

So x and y aren't independently meaningful in a musical sense. They're like the legs of a right triangle — individually they're just numbers, but together they define both the length and the angle of the hypotenuse.

A useful way to think about it: you could describe the same point either as:

-   **(x, y)** — rectangular form, what the FFT actually outputs
-   **(amplitude, phase)** — polar form, what you actually care about

Same point, two different ways of describing it. Converting between them is just Pythagoras and arctan. For your visualiser you only ever need the amplitude, so you do the Pythagoras calculation and throw away the phase entirely.

ok this is all breaking down now? if I look at a sine wave in an audio editor, y \*\*is\*\* the amplitude?

You're absolutely right, and I've tied myself in knots. Let me step back.

In an audio editor, looking at a waveform:

-   **x** is time
-   **y** is amplitude at that moment

That's the raw signal. Completely straightforward.

The complex number in an FFT bin is a **different space entirely** — it's not the same x and y axes as the waveform. I should have been clearer that we moved into a completely different coordinate system when we started talking about FFT output.

In the FFT output space:

-   You have one complex number per frequency bin
-   That complex number's position — its (x, y) — encodes two things **about that frequency**: how loud it is and where in its cycle it currently sits
-   Neither axis maps to anything as simple as "amplitude" or "phase" directly — you have to extract those via Pythagoras and arctan as I described

The waveform y-axis and the FFT bin's y-axis are completely unrelated things that I kept letting bleed into each other. The waveform lives in time-vs-pressure space. The FFT output lives in a completely different space where each point represents a frequency's current state.

That's the source of the confusion and it's my fault for not flagging the gear change clearly. Does that separate the two pictures out?

no you've made me worry that my entire undestanding is wrong now

No — your understanding is solid. I'm the one who got confused, not you.

Everything you built up is correct:

-   Wheels spinning at different speeds = different frequencies ✓
-   Wheel radius = amplitude = loudness of that frequency ✓
-   Spin rate = frequency = pitch ✓
-   FFT measures how much of each frequency is present ✓
-   Output is one complex number per bin ✓
-   Distance from origin of that complex number = amplitude of that frequency ✓

The only thing that went wrong just now is I got tangled up when you correctly pointed out that in a waveform, y is amplitude. That's true. But that's a different picture from the FFT output — the waveform is the _input_ to the FFT, the complex numbers are the _output_. Two different spaces, two different pictures.

Your waveform intuition is right. Your FFT output intuition is right. They just don't share the same axes, and I muddied that badly by not keeping them clearly separated.

You haven't misunderstood anything. Put the worry down.

it's the different concepts of the waveform and the fft bins. Once I get the complex numbers from the fft bins, all I do is one formula to get the amplitude, the frequency is inherent in the bin it's in? and there's another formula if I want the phase?

Yes. Perfectly stated.

-   **Frequency** — just look at which bin it is. Bin number tells you the frequency. No calculation needed.
-   **Amplitude** — one formula: √(x² + y²). Done.
-   **Phase** — one formula if you want it: arctan(y/x). You probably won't bother.

That's the complete FFT output, fully decoded. Everything else we discussed was just building the intuition for why those three things are what they are.

You've got it.

I've never really understood phase. I've read descriptions of it, even listened to examples A B of in phase and out of phase. Honestly I can tell the difference, but only barely and because they are side by side. I could probably listen to out of phase music all day and not know

Phase is genuinely the least musically intuitive of the three, and your experience of it is completely normal. It's not like frequency (obviously higher/lower) or amplitude (obviously louder/quieter) — phase has no direct perceptual quality you can point at and name.

In terms of the wheel, phase is simply: **where around the rim is the point right now, at the start of the window?** Two wheels of identical size and spin rate but different phases are completely indistinguishable to your ear in isolation. They sound identical. The difference only appears when they interact with other frequencies.

Where phase actually matters:

**Cancellation** — two identical waves perfectly out of phase cancel to silence. This is real and dramatic. It's how noise-cancelling headphones work — they generate a wave with the same frequency and amplitude as the noise but flipped phase, and the two cancel.

**Stereo imaging** — your brain uses tiny phase differences between your left and right ear to locate sounds in space. This is why "out of phase" stereo sounds weird and spaceless — your brain is getting contradictory location information.

**Speaker/mixing engineering** — phase problems between microphones or tracks can cause certain frequencies to partially cancel, making a mix sound thin or hollow. Engineers care about it a lot.

But for a listener in a normal situation? You're right. You'd never notice. And for your visualiser, you can comfortably ignore it entirely, which is exactly what you said you'd do.

How does it relate to real sound in the real world?

This is where it gets interesting. In the real world, phase is almost never something you consciously hear — but it's constantly affecting what you hear without you knowing.

The most concrete example: you're in a room listening to a speaker. The sound reaches you directly, but it also bounces off the walls, floor, ceiling. Those reflections are the same frequencies as the direct sound but they've travelled further, so they arrive slightly later — which means they're at a different point in their cycle when they reach your ear. Different phase.

Some frequencies arriving from the reflection will be in phase with the direct sound and get louder. Others will be partially out of phase and get quieter. This is why rooms sound different. It's why a concert hall sounds rich and a bathroom sounds boxy and a cupboard sounds dead. You're not hearing phase directly — you're hearing the _consequence_ of phase interactions across hundreds of reflections, which reshapes the frequency content of the sound reaching your ears.

It's also why standing in different spots in a room changes what you hear. Move a metre to the left and the path lengths of all those reflections change, shifting which frequencies reinforce and which cancel. Bass frequencies are particularly vulnerable to this because their long wavelengths mean small room dimensions can create dramatic cancellations — this is why bass can boom in one corner of a room and almost disappear in another.

So phase in the real world is less a thing you hear and more a thing that silently shapes everything you hear, all the time, through its interactions with physical space.

For your app — genuinely irrelevant. But it's not an abstract mathematical curiosity either. It's in every room you've ever listened to music in.

I'd always thought that was more about timing, eg reflected sound travelling further and arriving later

That's exactly right — and here's the thing: **phase and timing are the same thing**, just described differently.

A reflection arriving later means the wave has travelled further, which means when it arrives it's at a different point in its cycle. "Different point in its cycle" is just another way of saying "different phase." The two descriptions are identical, just one is physical (it travelled further, it arrived later) and one is mathematical (it's offset in its cycle).

The reason we use phase rather than timing is that "later" is ambiguous without knowing the frequency. A delay of 1 millisecond means:

-   Almost nothing for a 10Hz wave — it's barely moved in its cycle
-   A quarter cycle offset for a 250Hz wave
-   A full cycle offset for a 1000Hz wave — which means it's back in phase, as if no delay happened at all

So the same physical delay has completely different consequences depending on the frequency. Phase captures that relationship cleanly — it describes the delay _relative to the cycle length_ of each frequency, which is what actually determines whether frequencies reinforce or cancel.

Timing is the physical cause. Phase is the mathematical way of expressing what that timing means for each frequency individually.

They're the same phenomenon, two different languages for describing it.

but sometimes a relected sound could arrive many cycles later, surely that signal to the brain has more information than just the phase difference?

Yes, absolutely — and this is where the physics gets richer than the simple phase picture.

A reflection arriving many cycles later isn't just a phase offset anymore — it's a **distinct echo**. And your brain treats it completely differently depending on how late it arrives:

-   **Under about 30ms** — your brain fuses it with the direct sound. You don't hear two sounds, you hear one sound with a changed tonal colour (because of the phase interactions we discussed). The Haas effect — your brain attributes the sound to the first arrival and ignores the later one as a separate event.
-   **30–100ms** — you start to hear it as a distinct thickening or slap, but still not a clean separate echo. This is the "room sound" range, what reverb pedals and plugins are simulating.
-   **Over about 100ms** — now you hear it as a genuine separate echo. Your brain has given up trying to fuse it with the original and treats it as a new sound event.

So the brain is doing something much more sophisticated than phase comparison — it's using timing to decide _how_ to process the reflection, and applying completely different perceptual machinery depending on the answer.

Phase relationships are only the whole story in that first sub-30ms window. Beyond that, timing carries richer information — about the size of the space, the distance of surfaces, the geometry of the room. Which is exactly how your brain builds a three dimensional model of the acoustic space you're in, entirely unconsciously, just from the pattern of reflections arriving at your ears.

Human" is genuinely remarkable that, isn't it

If I feed in 2048 samples and get back 1024 bins, how is one mapped to the other? eg is sample 1 and 2 mapped to the first bin? Guess not... but there must be a reason you get half the bins compared to the sample number

Your guess is right — it's not a simple pairing like that. Every single sample contributes to every single bin. It's not a carving-up of the samples, it's a transformation of the whole window simultaneously.

The reason you get half as many bins is a physical constraint called the **Nyquist theorem**, and it's actually very elegant.

To reliably detect a frequency, you need to see it complete at least one full cycle within your window. But more fundamentally, you need a minimum of **two samples per cycle** — one catching the wave near its peak, one near its trough. If you only have one sample per cycle, you can't distinguish that frequency from a lower one. You're not sampling fast enough to see it oscillating.

So if your sample rate is 44,100 Hz, the highest frequency you can possibly represent is 22,050 Hz — because that's the frequency where each cycle is exactly two samples long. Any faster and you'd need more samples per cycle than you have available.

That ceiling is the Nyquist frequency — exactly half your sample rate. And since your bins cover from zero up to that ceiling, and your window has 2048 samples, you get 1024 bins covering that range. Half the samples, because half the sample rate is the hard physical limit of what's detectable.

This isn't a limitation of the FFT algorithm — it's a fundamental truth about sampling itself. It's why audio CDs use 44,100 Hz — just comfortably above twice the upper limit of human hearing, with a little headroom. Anything higher would be wasted, anything lower would lose audible frequencies.

so it's kind of 'trying to be' 2048, but due to needing 2 samples per bin, half is the best it can do?

Yes, exactly that — though I'd tweak the framing slightly.

It's not that it's _trying_ to be 2048 and failing — it's that 2048 samples can only ever contain 1024 independent pieces of frequency information. The other 1024 would be above the Nyquist ceiling, describing frequencies that literally cannot exist in a signal sampled at this rate. They're not missing — they were never there to find.

It's a bit like asking how many distinct colours are in a black and white photograph. The answer isn't "it tried to have colour but couldn't" — colour simply isn't a dimension that exists in that image.

There's actually a satisfying symmetry to it: your 2048 samples describe 1024 frequencies, each with two pieces of information (the x and y of the complex number — amplitude and phase). So the information content balances perfectly:

-   **In**: 2048 numbers
-   **Out**: 1024 complex numbers = 1024 × 2 = 2048 numbers

No information is created or destroyed. The FFT is just a perfect reorganisation of the same information — from time domain into frequency domain. Which is actually one of the beautiful things about it. It's fully reversible. You can run it backwards (an inverse FFT) and get your original samples back perfectly.

so it's kind of 'trying to be' 2048, but due to needing 2 samples per bin, hazsza

Ha. Go have that coma, you've earned it.

You said:

186 The more common approach is to run \*\*one FFT\*\* with a window size tuned to a reasonable middle-ground, and then \_group the output bins\_ into your bands afterwards. Since you get 1024 bins back covering the full frequency range, you just decide which bins belong to bass, which to mids, which to highs, and sum or average the energy within each group. That's fast and simple.

but my current understanding would say you don't go for a middle ground, you go based on the size of the lowest frequency you need (because that's the window size?)?

Really good catch, and you're essentially right — the lowest frequency you care about should drive your window size choice.

The "middle ground" framing I used earlier was loose. What you actually trade off is:

-   **Window too short** — you can't see enough cycles of low frequencies to measure them reliably. Your bass bins become untrustworthy.
-   **Window too long** — your time resolution suffers. A kick drum that lasts 20ms gets smeared across a 100ms window and loses its sharpness.

So it's not really a neutral middle ground — it's a window size that's long enough to reliably capture your lowest frequency of interest, while short enough to maintain useful time resolution.

In practice for music, 2048 samples at 44,100 Hz gives you a lowest detectable frequency of around 21Hz — which comfortably covers the full range of human hearing. And the window is about 46ms, which is short enough to track musical events reasonably well.

So 2048 has become a kind of industry default not because it's a vague compromise, but because it happens to satisfy both constraints adequately for general music analysis — low enough floor, short enough window. It's not arbitrary, it just looks that way until you understand the constraints driving it.

Your instinct was right. The lowest frequency you need is the real constraint. 2048 just happens to be the answer that constraint produces for standard audio.

cool so 2048 wasn't just an example, it basically/usually would be the number to use

Yes, for music visualisation specifically it's pretty much the standard starting point. Not a magic number, but a very well-worn one that the community has converged on because it hits the sweet spot for music's frequency range and typical musical event timings.

You'd only deviate if you had a specific reason:

-   **4096** if you needed finer frequency resolution — distinguishing very close pitches, or more precise bass bin separation
-   **1024** if you needed snappier time response and didn't mind losing the very bottom of the bass range
-   **512** for something like a hi-hat trigger detector that needs to be razor sharp in time

But for a general music visualiser covering the full frequency range with reasonable musical responsiveness? 2048 is where most people start and many people stay.

whn the fft maths 'compares' with its frequencies and multiplies samples, how come it doesn't get confused by the same frequency being out of phase? I imagine the conpared samples cancelling each other out and making the fft 'think' that frequency wasn't well represented?

Brilliant question — and this is exactly where the complex number representation earns its place.

If the FFT used a simple 1D reference wave for comparison, you'd be absolutely right. A frequency perfectly out of phase with the reference would cancel completely and the FFT would report zero energy — wrong answer, that frequency is clearly present.

But the reference isn't a 1D wave. It's a spinning wheel — a complex number tracing a circle. And this is the key: the FFT compares against **both** the sine and cosine versions of each frequency simultaneously. One is the wheel seen from the side, the other is the same wheel seen from the front — exactly 90 degrees apart.

So for each frequency bin, you're actually doing two comparisons:

-   How much does the signal agree with the sine version? → gives you the y component
-   How much does the signal agree with the cosine version? → gives you the x component

If the signal's frequency is perfectly out of phase with the sine reference, it cancels to zero — but it aligns perfectly with the cosine reference and comes back strongly there. The energy doesn't disappear, it just shows up in the other component.

Then when you compute √(x² + y²) to get the amplitude, both components contribute. The total length of the arrow is the same regardless of where around the circle the point sits.

Phase changes the _angle_ of the arrow but never its _length_. And amplitude is the length. So phase cannot fool the amplitude measurement — which is exactly what you need.

Yes — that's the moment isn't it.

That's also why complex numbers aren't just a convenient notation for the FFT — they're _necessary_. A 1D comparison would be phase-dependent and therefore broken. The 2D complex comparison is phase-independent by construction. The maths isn't using complex numbers because mathematicians like complexity, it's using them because one dimension genuinely isn't enough to do the job correctly.

Want to be notified when Claude responds?
