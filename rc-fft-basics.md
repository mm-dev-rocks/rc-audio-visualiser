___


### 5\. 🎵 Audio-visual sync

There's a subtle but annoying problem: the audio callback giving you PCM data and the display refresh (vsync) tick operate on **different clocks**. If you naively push FFT data to the shader on every audio callback, you'll get tearing or stuttering in the visuals. The standard solution is a small ring buffer: the audio pipeline writes FFT results into it, and the render loop reads from it at vsync. Getting this right takes some care.

___

What you're describing is essentially **source separation** or at minimum **spectral decomposition** — splitting a mixed signal into perceptually distinct streams. True source separation (like isolating drums from vocals) is a hard ML problem. Facebook's Demucs model does it well but it's heavyweight and runs slower-than-realtime on a phone, so that's probably out.

However, you don't need _literal_ track separation for a visualiser. What you can do very practically is **frequency band decomposition** — split the FFT output into bands (sub-bass, bass, mids, highs) and treat each band as its own "voice" driving a different visual element. This is fast, real-time, and perceptually effective. A kick drum will light up the bass band reliably; hi-hats will hit the highs. It won't be perfect but it'll feel like the visuals "hear" the instruments.

The slightly more sophisticated version of this is **onset detection** — detecting when a new note, beat, or significant sonic event occurs in a given band. That's very computable in real-time and gives you musically meaningful trigger events rather than a continuous stream of numbers.




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


# FFT maths 



And you're right about the programming crossover — the FFT is fundamentally an algorithm, and when you see it described that way it's far more approachable than the sigma notation suggests. Knuth wrote about it, it has a loop structure, it has a elegant divide-and-conquer trick at its heart. You'll recognise the shape of it.

-   Your wheel, but now imagine _two_ patches at different speeds — what does the combined signal look like from the side? That's what a chord sounds like as a waveform.
-   The question the FFT answers: given only the side view (the waveform), how do you figure out what wheels are spinning, and how fast?
-   Why complex numbers are just "the wheel seen from above" rather than the side — a 2D position, not a weird imaginary thing.
-   How the FFT's divide-and-conquer trick makes this calculation fast enough to be real-time.




first: I know even less about complex numbers than you imagine, like I don't even have a broad working definition?

You know how numbers live on a line? You've got zero in the middle, positive numbers stretching off to the right, negative numbers to the left. Every number you normally deal with lives somewhere on that line.

Complex numbers are what happens when you say — what if numbers could also go _up and down_, not just left and right? Instead of a line, you now have a 2D plane. Every complex number is just a point on that plane: some distance left/right (the "real" part, your normal number) and some distance up/down (the "imaginary" part, which is a terrible name that has caused centuries of unnecessary confusion).

So a complex number isn't really one number — it's a **pair of coordinates**. Like an x,y position on a graph. That's genuinely all it is at the level we need.

Think of a complex number as just a point, or equivalently an arrow from the origin to that point, living in a 2D space.

Now here's why this connects to your bicycle wheel: that coloured patch on the rim? At any moment in time, its position can be described as exactly one of these arrows — some distance left/right, some distance up/down. As the wheel turns, the arrow _rotates_. Complex numbers, it turns out, are the most natural mathematical language for describing rotation. That's the connection, and that's why they're at the heart of the FFT.


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

 — it's just **adding**.

For each frame, you take the vertical position of point 1 and the vertical position of point 2, and you add them together. That single number is the combined signal.

So if at some moment patch 1 is at height +0.6 and patch 2 is at height +0.3, the combined signal reads +0.9. If patch 1 is at +0.6 and patch 2 is at -0.6, they cancel perfectly and the combined signal reads 0.

 this is physically accurate — it's exactly how sound works in air. Two sound waves arriving at a microphone just add their pressure contributions together at every instant. The microphone doesn't know there were two sources, it just measures the total.

So your three steps become:

-   **One patch**: one animated point bouncing up and down, tracing a clean sine wave
-   **Two patches**: two animated points, each bouncing at their own rate and amplitude
-   **Combined signal**: at every frame, add the two heights together to get a single number — that number is what the microphone records


All the microphone ever sees is 0.9. It has no idea it came from 0.6 + 0.3 versus 0.4 + 0.5 versus 0.1 + 0.8. That information is gone the moment the waves add together. The FFT is the mathematical detective work that looks at how that combined number _changes over time_ and reasons backwards to figure out which wheels were spinning. It can't do it from a single frame — it needs a whole window of samples, which is why you feed it a chunk of audio rather than one number at a time.


On negative numbers — yes, absolutely they're involved. The patches spend half their time below the centreline, so the vertical position swings between roughly -1 and +1. When one patch is at +0.8 and another is at -0.8 they cancel almost completely to near zero. This is actually important musically — it's how certain combinations of frequencies can create "quiet" moments in a waveform even when both wheels are spinning at full size. It's also why audio signals look like they're oscillating around zero rather than always being positive.


A single sample is just a number. It has no meaning in isolation — you can't hear it, it has no pitch, it represents nothing musical by itself. Pitch and timbre only _emerge_ from the pattern across thousands of samples over time. A single frame of 0 could be part of a loud chord or part of silence — there's no way to tell from that one number alone.

This is actually why the FFT needs a chunk of samples rather than one at a time. You're not asking "what frequency is this number?" — that question is meaningless. You're asking "what pattern of frequencies best explains how this sequence of numbers is changing?" And you need enough samples to see at least one full cycle of the slowest wheel you care about, otherwise you can't reliably detect it.

It's a bit like needing to watch a clock for at least a full minute before you can be sure the minute hand is actually moving.


One full spin of the wheel is one complete cycle of a sine wave — and that maps directly to **one cycle of a frequency**. The number of full spins per second is the frequency in Hz.

So:

-   A wheel spinning 440 times per second produces the note A4 — the standard orchestral tuning note, the one used to tune instruments
-   A wheel spinning 261 times per second produces middle C
-   A wheel spinning 20 times per second is at the edge of human hearing — you'd feel it more than hear it
-   A wheel spinning 20,000 times per second is about the upper limit of hearing for a young person

The faster the wheel spins, the higher the pitch. That's it. Frequency _is_ spin rate, Hz _is_ spins per second. They're the same thing described two different ways.

And amplitude — the size of the patch's contribution, how far up and down it swings — that's the volume of that particular frequency. A big patch on a slow wheel is a loud low note. A tiny patch on a fast wheel is a quiet high-pitched tone.

So when you have a real piece of music and you run the FFT on a chunk of it, what you're getting back is essentially a list of wheels — here's how fast each one is spinning (frequency/pitch) and here's how big each patch is (amplitude/volume) at this particular moment in time. Which is exactly the information your band-splitter needs to say "lots of energy in the bass band right now, kick drum just hit."


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




# BINS

Each bin contains a **complex number** 
how strong the frequency is, _and_ its phase — basically, where around the rim the patch is at the start of the window).

But for a visualiser, you almost certainly don't care about phase. 
You just want the amplitude — how loud is this frequency right now. And getting that from the complex number is trivial: it's just the distance from the origin to that 2D point. Pythagoras — square root of (a² + b²). One simple calculation per bin and you have your amplitudes.

So in practice, for your purposes: yes. The bins contain an amplitude for a frequency. Everything else is a detail you can safely set aside.










So you now have a working mental model of the whole pipeline, which is not nothing:

-   Sound is wheels spinning at different speeds, added together into one signal
-   A window of samples is a brief recording of that combined signal
-   The FFT spins reference wheels at every possible frequency and measures how much each one agrees with the signal
-   Out come 1024 amplitudes, one per frequency bin
-   You group those bins into bands
-   Those bands drive your visuals



When you surface, we've still got the interesting stuff to get into — beat tracking, onset detection, how you'd actually wire FFT output to shader uniforms, and the architecture of that two-tier fast/slow system you intuited earlier. Good foundations for all of it now.


# SINE WAVES

It's not a mistake at all — it's actually the correct starting assumption, and it's not just a simplification.

This is a theorem: **any** periodic signal, no matter how complex or harsh or jagged, can be represented as a sum of sine waves. It's called Fourier's theorem, after the same person the transform is named after. It's not an approximation, it's mathematically exact.

A square wave — the harshest, most un-sinusoidal shape you can imagine — is provably equal to an infinite sum of sine waves at specific frequencies and amplitudes. A sawtooth wave, same thing. A snare hit, a vowel sound, a distorted guitar — all of them, sine waves added together.

So when you were thinking in terms of wheels, you weren't making a naive assumption that music is smooth and gentle. You were correctly identifying the fundamental building blocks of _all_ sound. The wheels aren't a metaphor for simple sounds — they're the literal mathematical constituents of every sound that exists.

This is actually why the FFT is so powerful and so universal. It works on any signal at all, not just "nice" ones, because any signal at all is secretly just wheels.

Fourier's original insight — that this was true — was considered almost scandalous when he proposed it in the early 1800s. Other mathematicians didn't believe him for a while. The idea that something as jagged and ugly as a square wave could be made entirely from smooth sine waves seemed absurd.

He was right though.

