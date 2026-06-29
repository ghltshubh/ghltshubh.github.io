---
layout: post
title:  "Building a Fully-Local Voice Assistant on a Raspberry Pi Cluster"
date: 2026-06-28 12:00:00
description: How I built an Alexa-equivalent that runs entirely on four Raspberry Pis, and what a wake word that kept firing on "ok" taught me about training data
tags: raspberry-pi voice-assistant machine-learning edge-ai self-hosting wake-word llm
categories: machine-learning edge-computing self-hosting
---

> An Alexa-equivalent running entirely on a 4-node Raspberry Pi cluster, with no cloud and no audio leaving the house. The wake word took longer than everything else combined.

---

## Table of Contents

- [The Goal](#the-goal)
- [The Hardware](#the-hardware)
- [The Pipeline](#the-pipeline)
- [Latency Lessons](#latency-lessons)
  - [The LLM sets the floor](#the-llm-sets-the-floor)
  - [Memory-mapping over NFS is a trap](#memory-mapping-over-nfs-is-a-trap)
  - [Piper instead of Kokoro](#piper-instead-of-kokoro)
  - [Cutting the round-trips](#cutting-the-round-trips)
  - [Warmup and the single slot](#warmup-and-the-single-slot)
- [The Wake Word](#the-wake-word)
  - [Porcupine went enterprise](#porcupine-went-enterprise)
  - [Synthetic only: 0.12](#synthetic-only-012)
  - [Recording the real voice: 0.94](#recording-the-real-voice-094)
  - [Surviving the vacuum](#surviving-the-vacuum)
  - [The "ok" problem](#the-ok-problem)
  - [Where it got interesting](#where-it-got-interesting)
  - [The fix that worked](#the-fix-that-worked)
- [Why length wasn't enough](#why-length-wasnt-enough)
- [Smaller things that cost time](#smaller-things-that-cost-time)
- [Where it landed](#where-it-landed)

---

## The Goal

I wanted something that worked like Alexa but ran on hardware I controlled. Answer questions, set timers, give the weather, describe what a camera is pointed at. The one firm requirement was that all of it stay local: no audio sent to a data center, no transcripts sitting on anyone else's server, no microphone that's "always listening" in the sense that really means always uploading.

What I had to run it on was a 4-node Raspberry Pi 4 cluster, 8GB per node, already running k3s for other things. Four Cortex-A72 boards, no GPU, no neural accelerator. That constraint is the whole story. Wiring speech-to-text to an LLM to text-to-speech is straightforward; doing it fast enough to be tolerable on four Pis is not, and getting a custom wake word to recognize one specific voice while a vacuum runs in the background turned out to be the part I underestimated by about a week.

## The Hardware

The pipeline is split across nodes so no single Pi has to do everything at once.

{% include figure.html path="assets/img/pi-cluster-rack.jpg" class="img-fluid rounded z-depth-1" zoomable=true alt="Four Raspberry Pi 4 boards stacked in a clear acrylic case, two cooling fans mounted on top, a network switch and power supply below, with ethernet and USB-C cables running between them." caption="The whole machine: four Raspberry Pi 4 (8GB) boards in an acrylic stack, two AC Infinity fans up top, gigabit switch underneath. Everything the assistant does happens inside this." %}

| Node | Role | Backend | Measured |
|------|------|---------|----------|
| node1 | Pipeline entry | Wake word + ASR + vision | ASR ~3.4s |
| node2 | LLM | Qwen3-4B Q4_K_M via llama.cpp | ~1.6 tok/s |
| node3 | TTS | Piper `en_US-lessac-medium` | RTF ~0.4 |
| master | k3s + NAS | control plane, NFS storage | — |

{% include figure.html path="assets/img/pi-cluster-boards.jpg" class="img-fluid rounded z-depth-1" zoomable=true alt="Close-up of the four stacked Raspberry Pi 4 boards showing their HDMI, USB-C power and USB ports, with a red power LED lit on the top board." caption="The four nodes up close. One runs wake word, speech recognition and vision; one runs the language model; one does text-to-speech; the fourth is the k3s control plane and NAS." %}

Models live on an NFS share so there's one copy to manage and each node mounts what it needs. The boards are lightly overclocked (2.0GHz on node1 and node2, 1.8GHz on node3, which wanted less voltage to stay stable) and idle in the mid-40s to low-50s Celsius.

An orchestrator on node1 holds it together. It runs the wake loop, uses voice-activity detection to find the end of a sentence, calls ASR then the LLM then TTS, and streams the spoken reply back out. For development I used a laptop bridge that stands in for the Pi's microphone, speaker, and camera, shipping audio to node1 over a WebSocket so I could iterate without wiring up real peripherals.

## The Pipeline

The path itself is unremarkable:

```
mic -> [wake word] -> [ASR] -> [LLM + tools] -> [TTS] -> speaker
        node1          node1    node2            node3
```

Every stage on that line ran into the same wall, which is latency on a CPU that has no fast path for the matrix math these models do. Here is what each stage cost and how I got it down.

## Latency Lessons

### The LLM sets the floor

The language model dominates everything else. The Pi 4's Cortex-A72 doesn't have the `dotprod` or `i8mm` instructions that quantized inference relies on, so a 4-bit 4B model runs at about 1.6 tokens per second. A two-sentence answer is fifteen to twenty seconds of generation no matter what else I optimize.

The obvious move is to drop to a smaller model, and I benchmarked a few. Qwen3-4B Q4_K_M routed a six-question tool-use test correctly six times out of six. Qwen2.5-3B got five out of six and invented a tool call on the one it missed. Qwen2.5-1.5B ran more than twice as fast at 2.41 tok/s, which was tempting until I saw it score four out of six, fabricating `get_weather` calls for questions that had nothing to do with weather. A faster model that picks the wrong tool isn't a win, so I kept the 4B and deleted the rest. On this hardware the model that clears your accuracy bar is the latency floor, and the optimization work happens around it.

### Memory-mapping over NFS is a trap

This one cost an afternoon. The weights sit on the NFS share, and llama.cpp memory-maps weights by default. That's fine on a local SSD and bad over a network, because the kernel then page-faults weight pages across the network on demand during inference. First-token latency was around 64 seconds.

```bash
llama-server -m qwen3-4b-q4_k_m.gguf --no-mmap -fa on -np 1 ...
```

`--no-mmap` loads the whole model into RAM at startup instead. We have 8GB and the model is about 2.5GB, so it fits, and after one slow load there are no more network faults. That alone took the cold first token from 64 seconds to 8. Memory-mapping is a local-disk optimization, and it quietly becomes a cost the moment the disk is a gigabit hop away.

### Piper instead of Kokoro

I started with Kokoro for TTS because it sounds good. On a Pi 4 its real-time factor is around 8, meaning eight seconds of compute per second of audio, which is unusable. Piper, with the `en_US-lessac-medium` voice, has a real-time factor around 0.4, so it synthesizes faster than playback. It sounds a little less natural than Kokoro, and that was an easy trade to make given the alternative was waiting longer for the sentence to be spoken than it took to generate.

### Cutting the round-trips

The first working version answered a tool query, "what's the weather in Paris," in 52 seconds. That involved two LLM passes, one to decide which tool to call and one to turn the tool's result into a sentence, plus synthesizing the entire reply before any of it played.

Three changes brought it to 21 seconds. The biggest was removing the second LLM pass: tools now return speech-ready strings, so `get_weather` hands back "It's 14 degrees and cloudy in Paris" directly instead of JSON for the model to rephrase. That deletes a full generation pass. The second was streaming the TTS, speaking each sentence as soon as the model finishes it rather than waiting for the whole answer, so the first words come out while the model is still working on the rest. The third was using faster-whisper's `tiny.en` with greedy decoding for ASR, which dropped recognition from about 5 seconds to 3.4 with no accuracy loss on my test phrases.

Because of the streaming, it feels quicker than 21 seconds suggests; you hear it start talking around the eight-second mark.

### Warmup and the single slot

llama.cpp caches the prompt prefix, and my system prompt plus tool definitions are a large fixed prefix that's identical on every turn. Two settings take advantage of that. Running with a single parallel slot (`-np 1`) keeps that prefix resident between turns instead of evicting it for parallelism I don't need. And a throwaway request fired at startup pays the cold-prefix cost before anyone is waiting; without it, the first real question took 167 seconds, and with it the first question is about 22 like every other.

---

## The Wake Word

This is the part I thought would take an afternoon. It took a week, and it's where the actual lesson is.

### Porcupine went enterprise

The easy way to make a custom wake word used to be Picovoice Porcupine: type your phrase, download a model. That's enterprise-only now. The open option is OpenWakeWord, which ships a few built-in words like `hey_jarvis` and `alexa` but not "ok computer," which is what I wanted, so I trained one.

The recipe is clean on paper. Generate positive examples of the phrase, mix them against a large pile of negative audio (a precomputed 2,000-hour feature set called ACAV100M), and train a small classifier on the embeddings. The model never touches raw audio; a frozen melspectrogram-to-embedding network turns roughly two seconds of audio into a fixed `(16, 96)` tensor, and a small network scores that tensor. That fixed-size detail comes back later.

### Synthetic only: 0.12

The textbook approach synthesizes the positives with TTS. I used a 904-speaker LibriTTS voice to generate about 2,700 takes of "ok computer" across every speaker and three speeds, trained, and got 0.999 on the held-out features. Then I said "ok computer" into a real microphone and it scored 0.12.

Synthetic voices in a clean digital pipeline don't sound like a person in a room, so the model had mostly learned to recognize TTS. A strong score on synthetic data said nothing about how it would do on real audio. The same gap returned later in a form that was harder to see.

### Recording the real voice: 0.94

I recorded about 35 real takes of myself saying the phrase through the actual microphone, augmented them with gain, pitch, time-stretch and noise into roughly a thousand positives, mixed those evenly with the synthetic set, and retrained. That got 0.94 on my real voice, and for a little while I thought I was done.

### Surviving the vacuum

Then I tested it with the air purifier and a vacuum running, since that's a normal state for the house, and the clean-trained model missed. The background noise pulled my wake word under the threshold, and dropping the threshold brought the false fires back.

The fix was to record about two minutes of my actual background noise and use it two ways in training: mixed into my clean takes at SNRs from -2 to 12 dB to make "wake word in noise" positives, and sliced into negatives so the noise on its own never triggers. That gave 0.96 clean, 0.86 in noise, and 0.00 on noise alone, which is a wide enough margin that a 0.5 threshold catches me through the vacuum while the vacuum scores zero. With aggressive voice-activity detection so steady noise doesn't read as an endless sentence, it held up well.

### The "ok" problem

One thing remained: it fired on "ok" by itself. Say "ok" to someone else in the room and the assistant woke up, which makes sense given that "ok computer" starts with "ok" and that's all the model had learned to look for.

The standard answer is adversarial negatives. You record yourself saying things that are close to the phrase but aren't it, like "ok," "okay," "ok cool," "ok so," and "computer" alone, and add them as a dedicated negative class. I recorded about 24 of those, clean and noise-augmented, retrained, and checked:

```
"ok computer" : 0.96
"ok" / "okay" : 0.00
```

Zero on every recorded "ok" I had. I shipped it.

### Where it got interesting

Then I said "ok" out loud and it scored 0.82. Again, 0.95. It fired.

This was confusing, because the model scored my recorded "ok" at 0.00 and synthetic "ok" at 0.001. I checked the deployed model's checksum against the trained file in case I'd shipped a stale model, and they matched. The model genuinely rejected every "ok" I could play at it and genuinely fired on every "ok" I could say at it.

It was the synthetic problem again in a different form. A 0.00 on recorded negatives doesn't mean the model won't fire on them live. The 24 adversarial takes were too few and too similar, so the model hadn't learned that "ok" isn't the wake word; it had memorized those specific recordings. My live "ok," in a slightly different tone at a slightly different moment, was a waveform it hadn't seen, and it went straight through.

### The fix that worked

The fix had two parts. The first was recording more "ok" and "okay" takes, but varied on purpose: rising like a question, falling and final, drawn out, clipped, loud, quiet, at different distances from the mic. The 56 takes I'd had were all roughly the same, and that uniformity was the problem.

The second part is the one that mattered. I went back to the 904-speaker LibriTTS voice and synthesized about 1,800 clips of "ok," "okay," "ok cool," "ok so," and "computer," using the same speakers that say "ok computer" in the positive set. Now the training data has the same voice saying both "ok computer" and "ok," and the only thing separating the two classes is the word "computer." There's no shorter description of the difference for the model to latch onto.

Fused together (real positives, synthetic positives, real adversarials, synthetic adversarials, the ACAV negatives, and the noise) and retrained:

```
"ok computer" : 0.889  (91% of takes fire)
"ok" / "okay" : 0.000
```

The offline numbers resemble the earlier version's, which is exactly why I'm treating the live microphone as the real test this time rather than the held-out score. The difference is in what the model was forced to learn, and that's something the offline number can't show you on its own.

## Why length wasn't enough

The obvious question, which was mine too, is that "ok" is shorter than "ok computer," so why doesn't the model just use length.

The information is there. The fixed `(16, 96)` window has speech energy across more frames for "ok computer" than for "ok," and the model could use that. It didn't, because a network learns the cheapest feature that separates its training set and then stops. In the earlier version the negatives were random audio that essentially never contains the sound "ok," so a single cue, the presence of an "ok" onset, separated every positive from the noise and drove the training loss to zero. The "computer" tail was useful information that the model had no reason to encode, because nothing in the data ever penalized firing on "ok" alone. What it actually learned was "ok is present," not "ok computer is present."

Matched-speaker contrast removes that shortcut. Once "ok" shows up in both a positive and a negative, the onset cue no longer separates the classes, and the only way left to drive the loss down is to encode the rest of the window, which is effectively whether the phrase finishes. The length signal gets learned when the data makes it necessary and not before.

It also explains the gap between live and recorded "ok." In a noisy room the silence after "ok" fills with vacuum and room energy in roughly the spot where "computer" would be, so a detector keyed on "ok plus some energy after it" fires. Forcing the model to require the actual "computer" pattern is the same fix as before.

## Smaller things that cost time

Not everything was a clean lesson. Some of it was just time.

DHCP moved the Pis out from under me partway through. The cluster looked dead, with SSH timing out and the orchestrator unreachable, but the Pis were fine; DHCP had handed them new addresses and my `/etc/hosts` was pointing at the old ones. One stale address even answered pings, because another device on the network had inherited it. The giveaway was that the `.local` mDNS name resolved while the hardcoded IP didn't. The real fix is DHCP reservations and using the mDNS names rather than trusting `/etc/hosts` on a network that hands out addresses.

The WebSocket carrying the microphone audio dropped about every eleven seconds for a while. The cause was setting the ping interval to zero to "turn off" pings, which instead made the library spin pinging with a zero timeout. It needs to be a large value, not zero.

The latency chimes fed back into the wake word at one point. I'd added short "listening" and "got it" tones to cover the gap while the model thinks, and an early model woke itself on its own chime. They only became safe once I confirmed the model scores the chime tones at 0.00.

Voice-activity detection had to run at its most aggressive setting, or steady background noise reads as one continuous sentence and the recording never ends.

## Where it landed

A fully-local voice assistant on four Raspberry Pis works. It answers questions in about 21 seconds, sets timers, pulls live weather from a real API, describes what the camera sees, and sends no audio anywhere. Most of the engineering was trades made to buy back usability from hardware that didn't have much to spare: Piper over Kokoro, the 4B over the 1.5B, `--no-mmap`, dropping the second LLM call.

The part I'll remember is the wake word, and it's a point about data rather than wake words specifically. The model kept learning the easiest thing that fit the training set instead of the thing I meant. First it learned to recognize TTS instead of speech. Later it learned to detect the "ok" onset instead of the whole phrase. Both versions scored well on held-out data and both failed the moment they met a real microphone, because the held-out data had the same blind spot. The only thing that ever fixed it was changing the data so the easy answer stopped working: recording the real voice, then the real noise, then the matched-speaker contrast. None of that depended on the hardware, which is the one part of this that would have been the same on a GPU.

---

*If you're running something local on hardware that probably shouldn't manage it, I'd be glad to hear about it.*
