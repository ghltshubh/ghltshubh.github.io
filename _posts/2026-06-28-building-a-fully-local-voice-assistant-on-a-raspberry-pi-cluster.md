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
- [What it does](#what-it-does)
- [Choosing the pieces](#choosing-the-pieces)
- [Latency Lessons](#latency-lessons)
  - [The LLM sets the floor](#the-llm-sets-the-floor)
  - [Memory-mapping over NFS is a trap](#memory-mapping-over-nfs-is-a-trap)
  - [Piper instead of Kokoro](#piper-instead-of-kokoro)
  - [Cutting the round-trips](#cutting-the-round-trips)
  - [Warmup and the single slot](#warmup-and-the-single-slot)
  - [Vision was the other slow path](#vision-was-the-other-slow-path)
- [The Wake Word](#the-wake-word)
  - [Teaching it my voice](#teaching-it-my-voice)
  - [Teaching it to ignore "ok"](#teaching-it-to-ignore-ok)
- [Smaller things that cost time](#smaller-things-that-cost-time)
- [What's next](#whats-next)
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

Every stage on that line ran into the same wall, which is latency on a CPU that has no fast path for the matrix math these models do.

## What it does

Once it's awake, it does the things you'd expect. It answers general questions straight from the language model, pulls live weather from a real API, tells the time, sets timers that announce themselves when they're up, and looks things up on Wikipedia. If I ask what it sees, a camera grabs one frame and a small vision model describes it. Follow-up questions stay in context for a few minutes, so "what about tomorrow" after a weather question works without me saying the wake word again.

One design choice runs through all of it: the tools hand back finished sentences, not raw data. A weather lookup returns "It's 14 degrees and cloudy in Paris," ready to speak. That turned out to matter for speed, which is most of what the rest of this is about.

## Choosing the pieces

Nothing here was picked on reputation. Every stage was a bake-off on the actual Pis. I installed two candidates, measured them on the same inputs, kept the winner, and deleted the loser. faster-whisper against whisper.cpp for speech recognition. Qwen3-4B against smaller models for the brain. Piper against Kokoro for the voice. SmolVLM against Moondream for the camera. The numbers that settled each one are in the sections below.

The services themselves run as plain systemd processes, one per node, rather than as pods on the k3s cluster that the Pis also run. Plain processes were quicker to restart and far easier to debug when something hung. The cluster's other workloads were moved off the entry node so the voice pipeline has it to itself.

## Latency Lessons

Here is what each stage cost and how I got it down.

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

### Vision was the other slow path

The "what do you see" feature runs a small vision-language model, SmolVLM, on the same node as the wake word and speech recognition. The first version took over three minutes for a single image, which is no use to anyone. Most of that was the model slicing each image into tiles and running them separately; turning the slicing off brought it down to about 24 seconds per frame, with no real loss in the descriptions I get back for a "what's in front of me" question. There was also a version trap: the current major release of the image-processing library broke the model's resolution handling, so it's pinned a release back. The camera only wakes on demand. It opens, takes one frame, describes it, and lets go, so nothing in the house is streaming video.

---

## The Wake Word

This is the part I thought would take an afternoon. It took a week, and it's where the real lesson was.

The easy way to make a custom wake word used to be Picovoice Porcupine, but that's enterprise-only now. The open option, OpenWakeWord, ships a few built-in words but not "ok computer," so I had to train one. The setup is simple on paper: generate positive examples of the phrase, mix them against a large pile of negative audio, and train a small classifier on audio embeddings. Every problem I hit came down to what was in those examples.

### Teaching it my voice

The textbook approach synthesizes the positives with TTS. I generated about 2,700 takes of "ok computer" from a 904-speaker voice model, trained, and got 0.999 on the held-out data. Then I said "ok computer" into a real microphone and it scored 0.12. Synthetic voices in a clean pipeline don't sound like a person in a room, so the model had mostly learned to recognize TTS. Recording 35 takes of my own voice and mixing them in took it to 0.94.

It still missed when the vacuum or air purifier was running, because the noise pulled my wake word under the threshold and lowering the threshold brought the false fires back. The fix was to record a couple of minutes of that background noise and use it both ways: mixed into my clean takes to make "wake word in noise" examples, and as negatives so the noise alone never triggers. After that it caught me through the vacuum while the vacuum itself scored zero.

### Teaching it to ignore "ok"

The last problem was that it fired on "ok" by itself, which makes sense since "ok computer" starts with "ok." The standard fix is to record yourself saying near-misses ("ok," "okay," "ok cool," "computer" alone) and add them as negatives. I did that, retrained, and every recorded "ok" scored 0.00. Then I said "ok" out loud and it fired at 0.86.

The recorded "ok" scoring zero while the live "ok" fired took a while to make sense of. The model hadn't learned that "ok" isn't the wake word. It had memorized the few dozen specific recordings I gave it, and my live "ok," in a slightly different tone, was a waveform it had never seen. The deeper reason is that a network learns the cheapest feature that separates its training set and then stops. My negatives were random audio that never contained "ok," so "is there an 'ok' sound" was enough to tell positives from negatives, and the model never had a reason to check for "computer" at all. It was the same trap as the synthetic version, where a perfect held-out score hid a model that had learned the wrong thing.

So I made the shortcut stop working. I synthesized about 1,800 clips of "ok," "okay," and other near-misses from the same 904 speakers that say "ok computer" in the positive set, so the same voice now appears in both classes and the only thing separating them is the word "computer." Retrained with a batch of my own varied "ok" recordings added, "ok computer" scores 0.89 and live "ok" dropped from the 0.82–0.95 range down to 0.36–0.44, well under the threshold. The whole wake-word detour was really about the training data, never the model.

## Smaller things that cost time

Not everything was a clean lesson. Some of it was just time.

DHCP moved the Pis out from under me partway through. The cluster looked dead, with SSH timing out and the orchestrator unreachable, but the Pis were fine; DHCP had handed them new addresses and my `/etc/hosts` was pointing at the old ones. One stale address even answered pings, because another device on the network had inherited it. The giveaway was that the `.local` mDNS name resolved while the hardcoded IP didn't. The real fix is DHCP reservations and using the mDNS names rather than trusting `/etc/hosts` on a network that hands out addresses.

The WebSocket carrying the microphone audio dropped about every eleven seconds for a while. The cause was setting the ping interval to zero to "turn off" pings, which instead made the library spin pinging with a zero timeout. It needs to be a large value, not zero.

The latency chimes fed back into the wake word at one point. I'd added short "listening" and "got it" tones to cover the gap while the model thinks, and an early model woke itself on its own chime. They only became safe once I confirmed the model scores the chime tones at 0.00.

Voice-activity detection had to run at its most aggressive setting, or steady background noise reads as one continuous sentence and the recording never ends.

## What's next

A few things I want it to do that it can't yet.

The first is messaging and calls. I want to say "text Alex that I'm running late," or have it place a call, across the chat apps I actually use, without any of my account credentials sitting on the Pi. The assistant stays thin and hands the action off to something that holds the keys, so it still never sees them itself. Same rule as the rest of the build.

The second is music. Ask for a song and have it play from my own library first, fall back to the wider web when it's not something I own, and respond to the obvious controls like pause, skip, and volume. Music and speech share one speaker, so it should duck while it talks and pick back up after.

The rest is the unglamorous reliability work: timeouts and sensible fallbacks on the external lookups, a plain "I don't know" when there's nothing to say instead of a confident guess, and the supervision details that let it come back on its own after a node reboots.

## Where it landed

A fully-local voice assistant on four Raspberry Pis works. It answers questions in about 21 seconds, sets timers, pulls live weather from a real API, describes what the camera sees, and sends no audio anywhere. Most of the engineering was trades made to buy back usability from hardware that didn't have much to spare: Piper over Kokoro, the 4B over the 1.5B, `--no-mmap`, dropping the second LLM call.

The part I'll remember is the wake word, and it's a point about data rather than wake words specifically. The model kept learning the easiest thing that fit the training set instead of the thing I meant. First it learned to recognize TTS instead of speech. Later it learned to detect the "ok" onset instead of the whole phrase. Both versions scored well on held-out data and both failed the moment they met a real microphone, because the held-out data had the same blind spot. The only thing that ever fixed it was changing the data so the easy answer stopped working: recording the real voice, then the real noise, then the matched-speaker contrast. None of that depended on the hardware, which is the one part of this that would have been the same on a GPU.

---

*If you're running something local on hardware that probably shouldn't manage it, I'd be glad to hear about it.*
