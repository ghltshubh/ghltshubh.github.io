---
layout: post
title:  "Lens: Explaining Any Word on the Page, On-Device"
date: 2026-07-13 12:00:00
description: An ML engineer with no web-dev background ships a Chrome extension in a weekend with Claude. What it does, what I learned about the browser platform along the way, and what the on-device model's latency forced the design to become.
tags: chrome-extension on-device-ai gemini-nano manifest-v3 typescript llm
categories: web on-device-ai machine-learning
---

> Select a word or phrase on any page and get a short, context-aware explanation (who, what, when, where, why, how) from a model that runs on your own machine. Nothing leaves the device. I build ML systems and had never touched browser-extension development. I didn't know the vocabulary, let alone the platform. I paired with Claude and shipped it over a weekend, and this is what I learned building it.

---

## Table of Contents

- [The idea](#the-idea)
- [Building outside my domain](#building-outside-my-domain)
- [Three isolated pieces](#three-isolated-pieces)
- [The output contract comes first](#the-output-contract-comes-first)
- [One seam for two backends](#one-seam-for-two-backends)
- [What the latency forced](#what-the-latency-forced)
  - [Keep-warm, because the worker keeps dying](#keep-warm-because-the-worker-keeps-dying)
  - [Stream the lead line](#stream-the-lead-line)
  - [Only pay for context when it's needed](#only-pay-for-context-when-its-needed)
- [Where it landed](#where-it-landed)
- [Sources](#sources)

---

## The idea

Chrome 148 ships a language model in the browser: Gemini Nano, reachable through the Prompt API, running locally with no network call. [Lens](https://chromewebstore.google.com/detail/lens-%E2%80%94-5w1h-explainer/jieilmnnihphmgkeeppfmmacabofdlai) is a small extension built on it. You select text on a page, and a tooltip-sized card explains it in context. A concept gets *What / How / Why*. A person gets *Who*. It shows only the dimensions that actually apply, never six boxes of filler.

{% include figure.html path="assets/img/lens-hero.png" class="img-fluid rounded z-depth-1" zoomable=true alt="A web article with the phrase cellular respiration highlighted; a dark card floats beneath it with a one-line definition plus How and Why rows." caption="Select text, get a short context-aware explanation (who, what, when, where, why, how) from a model running on your own machine." %}

The privacy story is the whole point. Your reading never leaves the machine by default. But the on-device constraint is also what shaped every technical decision. Wiring a selection to a model to a card is easy. Doing it on a small model that's sometimes fast and sometimes very slow is where the design comes from.

## Building outside my domain

I should be upfront about where I'm standing. I work on ML and MLOps. I'd never built anything for the browser, and when I started I couldn't have told you what a service worker was, or a content script, or "Manifest V3." The terms in this post are ones I picked up over the weekend, mostly by asking Claude "why doesn't this work" and reading the answer. I brought the systems instincts, how to structure it and where the latency would be and what to measure, and it filled in the entire web-platform layer I was missing.

The ramp-up that normally gates a project like this, days of reading docs and getting the boilerplate wrong before you write anything real, was close to zero. The domain you happen to know is no longer the thing that decides what you can make. So the rest of this is written the way I actually found it: the surprises, in the order they surprised me, not a tour from someone who already knew the platform.

## Three isolated pieces

The first surprise was that a browser extension isn't one program. It's a few separate pieces, each running in its own sandbox, that can't share memory or call each other directly. (I learned this arrangement is what "Manifest V3" refers to.) It looked like a hassle at first. It turned out to be a clean way to organize the thing.

```
select text
    │
    ▼
[ content script ]  ──EXPLAIN port──▶  [ service worker ]  ──▶  [ LlmBackend ]
  capture + overlay     (streamed)         owns the model         local | cloud
  runs on the page                         one place, isolated
```

The **content script** is the piece that's allowed to touch the actual web page. It turns your selection into a capture (the term plus its sentence and paragraph) and draws the result in a little card. That card lives in a *Shadow DOM*, which I came to understand as an isolated corner of the page where the site's own styling can't reach in and wreck it. The content script never talks to the model.

The **service worker** is a background script Chrome starts when it's needed, and it's the only piece that talks to the model. Keeping the model in exactly one place is what lets the content script stay light and lets me swap the model backend without touching anything on the page.

The two pieces can't share memory, so they pass typed messages back and forth. For an explanation I used a *Port* (a connection that stays open) rather than a single request-and-reply, so the worker can send a partial answer first and the full one after. That detail matters later.

{% include figure.html path="assets/img/lens-how-it-works.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Three numbered cards: select text, get the answer showing the explanation card, and you're in control showing a research-mode toggle and a keyboard shortcut." caption="The whole interaction: select, read, and a toggle that keeps the on-device model warm." %}

## The output contract comes first

Before any of the plumbing, the thing to nail down is the shape of an explanation. The Prompt API takes a `responseConstraint`, a JSON schema the output is forced to match, so the contract is both a runtime constraint and a TypeScript type:

```ts
interface Explanation {
  type: "person" | "place" | "concept" | "process" | /* … */ "term";
  dimensions: Dimension[];                     // only the ones that apply, most-relevant first
  answers: Partial<Record<Dimension, string>>; // one short sentence each
  confidence: "high" | "medium" | "low";
  note?: string;                               // disambiguation flag
}
```

Everything else in the codebase depends on this. The overlay reads `dimensions[0]` as the lead line and hides the rest behind an expander. The parser is strict where it matters (the `type` and `confidence` enums) and lenient where the small model is flaky. A dimension it lists but never answers just gets dropped rather than throwing.

One quirk that matters: Gemini Nano is strong on conceptual dimensions and hallucination-prone on factual specifics like dates, places, and names. So the prompt tells it to *omit rather than guess*, and `confidence` is a real signal the rest of the system uses, not decoration.

## One seam for two backends

There's a single interface between "the app" and "the model." Everything above it (capture, overlay, messaging) is written against this and nothing else:

```ts
interface LlmBackend {
  status(): Promise<BackendStatus>;
  ensureReady(onProgress?: (p: number) => void): Promise<void>;
  explain(input: CapturePayload, onPartial?: (p: Partial) => void): Promise<Explanation>;
}
```

`LocalBackend` implements it over Gemini Nano. A second `CloudBackend` implements the same interface over OpenRouter, so anyone who wants a stronger model can bring their own key. Cloud mode is strictly opt-in and off by default. The seam meant adding it changed one file and nothing else. The content script has no idea which backend answered.

## What the latency forced

I benchmarked the on-device model directly. Three numbers set the whole design:

| Cost | Measured |
|------|----------|
| Time to first token, short input | ~0.4 s |
| Decode throughput | ~30 tok/s |
| First token on a long, novel selection | ~9 s |
| Cold session create (model into memory) | ~16 s (one-time) |

Two things stand out. Startup is bimodal: a warm session answers in well under a second, a cold one pays a big model-load tax. The other is that prefill dominates. First-token latency scales with how much text you feed in, while decode speed barely moves. Each of those turned into a piece of architecture.

### Keep-warm, because the worker keeps dying

This is the thing that tripped me up most, and it took me a while to even realize it was happening. That background service worker isn't always running. Chrome shuts it down after about 30 seconds of inactivity, and the model session dies with it. So the next lookup pays the full 16-second cold start all over again. I couldn't find a way to just tell it to stay alive.

The workaround I landed on is a **research mode**: a toggle that, while it's on, keeps that open connection (the Port) alive from the page. As long as a connection is open, Chrome keeps the worker alive, which keeps the model session warm, which turns that 16-second cold start into a sub-second lookup. Flip it off and the model is released and the extension gets out of the way. It felt like a hack when I found it. As far as I can tell, it's the accepted way to do this.

### Stream the lead line

At ~30 tok/s, a full multi-dimension card is a few seconds of decode, long enough to feel like a hang. But the *lead* answer finishes first. So the worker streams the response, pulls the lead out of the partial JSON the moment its string closes, and pushes it to the card over the port before the rest is done:

```
worker streams the JSON token by token
   │
   ▼  (the "what" value closes here)
send EXPLAIN_PARTIAL → card shows the lead, pulsing, while the rest finishes
```

Total time is unchanged. The *wait* mostly disappears.

### Only pay for context when it's needed

Prefill scales with input, so the paragraph of context Lens sends for disambiguation is exactly what makes a lookup slow, and most terms don't need it. The local backend is therefore two-pass:

```
select text
   │
   ▼
pass 1 — sentence only → model → confidence
   │
   ├─ high / medium → done  (common, fast path)
   └─ low → pass 2: add the paragraph → model → done
```

A confident answer never pays for the paragraph, so a large one can't slow down a normal lookup. Only genuine ambiguity, the model saying `confidence: "low"` about itself, triggers the second, context-heavy pass. Each pass runs on a throwaway `clone()` of the base session, which also stops lookups from accumulating conversation history in one long-lived session.

## Where it landed

Lens is [live on the Chrome Web Store](https://chromewebstore.google.com/detail/lens-%E2%80%94-5w1h-explainer/jieilmnnihphmgkeeppfmmacabofdlai). Select text, get a context-aware explanation, entirely on your device by default, with an opt-in bring-your-own-key cloud mode for when you want a bigger model.

{% include figure.html path="assets/img/lens-private.png" class="img-fluid rounded z-depth-1" zoomable=true alt="A promo graphic reading Private by design, On your device, with three points: no network calls, instant and local, no account." caption="The default: everything on-device, nothing leaving the machine." %}

The AI feature itself was the easy part: a constrained prompt against a built-in model. Almost all the work was reacting to one small model being slow and to a background script that keeps shutting itself off: keep the session warm on purpose, show the first useful token as soon as it's ready, and don't send context you don't need. Those are latency-and-lifecycle problems, the kind I actually have instincts for. The web-platform specifics I didn't have (the extension model, the isolation between pieces, the APIs) were the part Claude carried.

That split is the whole point for me. The general engineering was mine to reason about. The domain knowledge I was missing was one question away. And the gap between the two, which used to mean weeks of ramp-up before I could ship anything, was a weekend. I don't think that's specific to me or to this project. If you can think clearly about a problem, the stack you happen not to know is no longer the wall it used to be.

## Sources

- [`simple-chromium-ai`](https://github.com/kstonekuan/simple-chromium-ai). The small TypeScript wrapper Lens uses to reach Chrome's built-in Prompt API (`LanguageModel`), and the clearest example of how to actually open a session and prompt Gemini Nano on-device.
- [Prompt API explainer](https://github.com/webmachinelearning/prompt-api). The web-standards proposal behind `LanguageModel`, including the `responseConstraint` JSON-schema output.
- [Chrome built-in AI docs](https://developer.chrome.com/docs/ai/prompt-api). Availability, the one-time model download, and hardware requirements.

---

*If you're an ML or data person eyeing a build outside your usual stack, or you've wrestled with the same browser-extension quirks I did, I'd be glad to compare notes.*
