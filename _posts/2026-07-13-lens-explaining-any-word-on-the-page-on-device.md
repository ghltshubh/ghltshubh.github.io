---
layout: post
title:  "Lens: Explaining Any Word on the Page, On-Device"
date: 2026-07-13 12:00:00
description: A Chrome extension that explains the text you select using Chrome's built-in Gemini Nano, why it's built as three isolated pieces, and what the on-device model's latency forced the design to become
tags: chrome-extension on-device-ai gemini-nano manifest-v3 typescript llm
categories: web on-device-ai machine-learning
---

> Select a word or phrase on any page and get a short, context-aware explanation — who, what, when, where, why, how — from a model that runs on your own machine. Nothing leaves the device. Most of the interesting decisions came from the model being slow, not from the AI itself.

---

## Table of Contents

- [The idea](#the-idea)
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

Chrome 148 ships a language model in the browser — Gemini Nano, reachable through the Prompt API, running locally with no network call. [Lens](https://chromewebstore.google.com/detail/lens-%E2%80%94-5w1h-explainer/jieilmnnihphmgkeeppfmmacabofdlai) is a small extension built on it: you select text on a page, and a tooltip-sized card explains it in context. A concept gets *What / How / Why*; a person gets *Who*. It shows only the dimensions that actually apply, never six boxes of filler.

The privacy story is the whole point — your reading never leaves the machine by default — but the on-device constraint is also what shaped every technical decision. Wiring a selection to a model to a card is easy. Doing it on a small model that is sometimes fast and sometimes very slow is where the design comes from.

## Three isolated pieces

A Manifest V3 extension is naturally split into contexts that can't share memory, and Lens leans into that instead of fighting it.

```
select text
    │
    ▼
[ content script ]  ──EXPLAIN port──▶  [ service worker ]  ──▶  [ LlmBackend ]
  capture + overlay     (streamed)         owns the model         local | cloud
  runs on the page                         one place, isolated
```

The **content script** is the only part that touches the page. It turns a selection into a capture — the term plus its sentence and paragraph — and renders the result in a Shadow DOM card so page CSS can't bleed in. It never talks to the model.

The **service worker** is the only part that talks to the model. Keeping the LLM in exactly one place means the content script stays light, the model session is shared, and swapping the backend touches nothing on the page.

They talk over typed messages. Explanation runs over a long-lived `Port` rather than a one-shot message, specifically so the worker can push a partial result before the full one — more on that below.

## The output contract comes first

Before any of the plumbing, the thing to nail down is the shape of an explanation. The Prompt API takes a `responseConstraint` — a JSON schema the output is forced to match — so the contract is both a runtime constraint and a TypeScript type:

```ts
interface Explanation {
  type: "person" | "place" | "concept" | "process" | /* … */ "term";
  dimensions: Dimension[];                     // only the ones that apply, most-relevant first
  answers: Partial<Record<Dimension, string>>; // one short sentence each
  confidence: "high" | "medium" | "low";
  note?: string;                               // disambiguation flag
}
```

Everything else in the codebase depends on this. The overlay reads `dimensions[0]` as the lead line and hides the rest behind an expander. The parser is strict where it matters (the `type` and `confidence` enums) and lenient where the small model is flaky — a dimension it lists but never answers just gets dropped rather than throwing.

One quirk worth stating plainly: Gemini Nano is strong on conceptual dimensions and hallucination-prone on factual specifics — dates, places, names. So the prompt tells it to *omit rather than guess*, and `confidence` is a real signal the rest of the system uses, not decoration.

## One seam for two backends

There's a single interface between "the app" and "the model." Everything above it — capture, overlay, messaging — is written against this and nothing else:

```ts
interface LlmBackend {
  status(): Promise<BackendStatus>;
  ensureReady(onProgress?: (p: number) => void): Promise<void>;
  explain(input: CapturePayload, onPartial?: (p: Partial) => void): Promise<Explanation>;
}
```

`LocalBackend` implements it over Gemini Nano. A second `CloudBackend` implements the same interface over OpenRouter, so anyone who wants a stronger model can bring their own key. Cloud mode is strictly opt-in and off by default; the seam meant adding it changed one file and nothing else. The content script has no idea which backend answered.

## What the latency forced

I benchmarked the on-device model directly. Three numbers set the whole design:

| Cost | Measured |
|------|----------|
| Time to first token, short input | ~0.4 s |
| Decode throughput | ~30 tok/s |
| First token on a long, novel selection | ~9 s |
| Cold session create (model into memory) | ~16 s (one-time) |

Two things stand out. Startup is bimodal — a warm session answers in well under a second, a cold one pays a big model-load tax. And *prefill dominates*: first-token latency scales with how much text you feed in, while decode speed doesn't care. Each of those turned into a piece of architecture.

### Keep-warm, because the worker keeps dying

MV3 service workers are killed after ~30 seconds idle, and the model session dies with them — so the next lookup pays the cold tax again. "Always warm" isn't a thing you can just ask for. So Lens has a **research mode**: a toggle that, while on, holds a keep-alive `Port` open from the page. An open port keeps the worker alive, which keeps the session warm, which turns a 16-second cold start into a sub-second lookup. When it's off, the model is released and the extension gets out of the way.

### Stream the lead line

At ~30 tok/s, a full multi-dimension card is a few seconds of decode — long enough to feel like a hang. But the *lead* answer finishes first. So the worker streams the response, pulls the lead out of the partial JSON the moment its string closes, and pushes it to the card over the port before the rest is done:

```
worker streams the JSON token by token
   │
   ▼  (the "what" value closes here)
send EXPLAIN_PARTIAL → card shows the lead, pulsing, while the rest finishes
```

Total time is unchanged; the *wait* mostly disappears.

### Only pay for context when it's needed

Prefill scales with input, so the paragraph of context Lens sends for disambiguation is exactly what makes a lookup slow — and most terms don't need it. So the local backend is two-pass:

```
select text
   │
   ▼
pass 1 — sentence only → model → confidence
   │
   ├─ high / medium → done  (common, fast path)
   └─ low → pass 2: add the paragraph → model → done
```

A confident answer never pays for the paragraph, so a large one can't slow down a normal lookup. Only genuine ambiguity — the model saying `confidence: "low"` about itself — triggers the second, context-heavy pass. Each pass runs on a throwaway `clone()` of the base session, which also stops lookups from accumulating conversation history in one long-lived session.

## Where it landed

Lens is [live on the Chrome Web Store](https://chromewebstore.google.com/detail/lens-%E2%80%94-5w1h-explainer/jieilmnnihphmgkeeppfmmacabofdlai). Select text, get a context-aware explanation, entirely on your device by default, with an opt-in bring-your-own-key cloud mode for when you want a bigger model.

The AI part was the easy part — a constrained prompt against a built-in model. The engineering was almost all a response to one small model being slow and one platform tearing down its own workers: keep the session warm on purpose, stream the first useful token, and don't send context you don't need. None of those are AI problems. They're the same latency-and-lifecycle problems you get anywhere the compute is scarce and local, which is increasingly where the interesting models are going to run.

## Sources

- [`simple-chromium-ai`](https://github.com/kstonekuan/simple-chromium-ai) — the small TypeScript wrapper Lens uses to reach Chrome's built-in Prompt API (`LanguageModel`). The clearest example of how to actually open a session and prompt Gemini Nano on-device.
- [Prompt API explainer](https://github.com/webmachinelearning/prompt-api) — the web-standards proposal behind `LanguageModel`, including the `responseConstraint` JSON-schema output.
- [Chrome built-in AI docs](https://developer.chrome.com/docs/ai/prompt-api) — availability, the one-time model download, and hardware requirements.

---

*If you're building on the browser's built-in models, or fighting MV3's worker lifecycle, I'd be glad to compare notes.*
