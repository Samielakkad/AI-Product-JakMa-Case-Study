# jak.ma — Production Case Study

[![AI + Product](https://img.shields.io/badge/AI%20%2B-Product-8A2BE2)](#)

**Verifier-gated LLM retrieval shipped into a live Moroccan home-services marketplace.**

> What it actually took to ship verifier-gated retrieval into a Moroccan service marketplace. The decisions, the tradeoffs, the things that broke, and the numbers a year in.

**Author:** Sami EL AKKAD · Tsinghua SIGS, AI MSc · sam25@mails.tsinghua.edu.cn
**Product:** [jak.ma](https://jak.ma)
**System reference:** [jak-ma-eval-suite/docs/architecture.md](https://github.com/Samielakkad/jak-ma-eval-suite/blob/main/docs/architecture.md)

---

## TL;DR

A Darija-language marketplace for finding plumbers, electricians, tilers, and nine other trades across Morocco. Built around a two-pass classify → constrained-generate pipeline with a deterministic verifier on top. Production: **p50 1.2s, p99 2.8s, $0.0008 per query, 0.7% verifier rejection rate, 3.7/4.0 average eval score across five dimensions.**

The interesting part is not the model. The interesting part is that the model is allowed to be wrong, and a 200-line Python verifier catches it before the user sees it.

---

## What jak.ma does

**For the user:** "I have a leak under my kitchen sink in Salé. Who do I call, and how much should it cost?"

**For the worker:** A pipeline that surfaces them to the right user, in the right city, at the right price band — without them ever having to write a profile in correct French.

**For the market:** A pricing primitive that did not exist. There is no published price book for a faucet repair in Casablanca. There is now.

The product surface is a chat. The underlying machinery is described in [docs/architecture.md](https://github.com/Samielakkad/jak-ma-eval-suite/blob/main/docs/architecture.md). This document is the *case study* — what shipped, what didn't, what we learned doing it.

---

## Hero metrics (60 days ending May 2026)

| | Value |
|---|---|
| p50 end-to-end latency | **1.18s** |
| p99 end-to-end latency | **2.82s** |
| Per-query inference cost | **$0.0008** |
| Verifier rejection rate | **0.7%** |
| Verifier-rejected → fallback success | **96%** |
| Daily queries (60-day mean) | **3,400** |
| Daily unique users | **1,100** |
| 5-dim eval score (last weekly run) | **3.7 / 4.0** |
| Image-bearing queries | **12%** |
| Cost per million classification tokens | **$0.30** |
| Cost per million generation tokens | **$0.50** |

The number to look at twice: **0.7% verifier rejection.** That is the slice of generations that, in the absence of the verifier, would have shipped a wrong price band or a wrong-city worker to a user. The verifier catches it. Of those rejections, 62% are price-band failures — the dominant fabrication mode in this market.

---

## The bet

The bet was that **the part of the problem worth pushing into the model is small**, and **the part worth keeping in deterministic code is large**.

Specifically: the model is good at converting "bghit chi sba9 f Casa daba" into a structured intent record and at producing a fluent Darija reply. The model is bad at knowing whether 800 dirhams for a leak repair in Salé is plausible. So we let it generate freely, then we verify against a rule table built by a human survey.

This is a different architectural posture than "make the model better at pricing." It is closer to "use the model where it is cheap and right, and put a checker around the rest." The verifier is not a fallback — it is the contract.

---

## Five decisions we made (and what we gave up)

### 1. Two passes, not one

We tried single-pass tool-use first. The model was supposed to classify, call a retrieval tool, and generate, all in one chain. We gave it up after two weeks of measurements.

- **What we gained with two passes:** p99 latency stayed under 3s; the verifier had a clean anchor (one schema for classification, one for generation); debugging was tractable.
- **What we gave up:** the elegance of a single LLM call. Two passes means two prompts to maintain, two cost lines to monitor, two failure modes to alert on.
- **Lesson:** elegance in the prompt is not elegance in the system. The right number of LLM calls is whichever number lets your verifier check the right things.

### 2. Structured retrieval, not RAG

We do not run vector search in the hot path. Categories are closed (12 trades × 12 cities = 144 cells), and a composite Mongo index over `(category, city, approved, available)` outperforms embedding search at this scale by every metric: latency, recall on the dominant query, and cost.

- **What we gained:** sub-100ms retrieval, deterministic re-derivation by the verifier (V2 — "every worker shown was in the retrieved set" — becomes a constant-time set check), zero embedding-index maintenance.
- **What we gave up:** semantic flexibility on edge queries. "I need someone to fix my doorbell" gets classified as `electrical`, which is correct but not nuanced.
- **Lesson:** if your domain is a finite taxonomy, a categorical index beats embeddings until proven otherwise. The "always use vector search" reflex is wrong about half the time.

### 3. A hand-curated price table, not a learned price model

The fairness band per (trade, city) is a hand-curated rule table maintained in `priorityService.ts`. It was initialized from a 200-worker survey in October 2025 and is revised quarterly.

- **What we gained:** the verifier has an authoritative anchor that does not move when the model improves. Price drift is detectable and explainable.
- **What we gave up:** we have to maintain the table. New cities (Tangier, Agadir on the roadmap) require new survey work.
- **Lesson:** in a market with no truth, you have to build the truth before you can verify against it. The survey is the moat, not the model.

### 4. Browser-only product, no user accounts

The product is a single-page app. There are no accounts. Session state lives in localStorage, keyed by a UUID generated client-side. This was a product decision, not just a tech decision.

- **What we gained:** trivial onboarding (one tap), no auth surface to defend, no GDPR data-subject-request bureaucracy, FERPA-style "data never leaves the device" claim is true by construction.
- **What we gave up:** cross-device continuity (your phone and laptop don't sync), upsell paths that require knowing who the user is, network effects from a user graph.
- **Lesson:** the "no accounts" choice was specific to *this* market. Moroccans have plenty of apps demanding phone-number signup; the differentiator is *not* asking. Federation of localStorage across devices, via E2EE blobs, is the roadmap path that preserves the property.

### 5. Grok over Claude/GPT for the production hot path

We benchmarked Grok-3-mini, Claude Sonnet, and GPT-4o-mini on the 100-query Darija eval set. Grok-3-mini wins on price ($0.30 vs $1.50 vs $0.60 per 1M input) at parity on naturalness and trade-fit, and slightly worse on factuality — which the verifier catches anyway.

- **What we gained:** ~3× cost reduction vs the next-cheapest alternative.
- **What we gave up:** Grok's JSON mode reliability is empirically 99%+ but not 100%. We have a tolerant parser for malformed JSON in production.
- **Lesson:** model choice is a per-call decision, not a vendor commitment. Each call gets the cheapest model that passes its eval slice.

---

## Five things that broke (and what we changed)

### 1. The "بغيت plombier" code-switch problem

**What happened:** Pass 1 was classifying mixed-script queries inconsistently. "بغيت plombier f Casa" was sometimes routed to `electrical` because the classifier latched onto the Arabic verb and lost the French trade noun.

**Fix:** Added `language: "mixed"` as an explicit enum value in the Pass 1 schema, with prompt instructions to extract the trade from whichever language carries it. Eval score on `trade_fit` went from 3.1 → 3.6 on the mixed-script slice.

**Lesson:** if your model is doing the wrong thing on a slice, often the prompt is asking the wrong question. The schema is the prompt.

### 2. The "1500 dirhams to change a light bulb" incident

**What happened:** During a Casablanca rate-limit spike, Grok started producing price bands at the upper end of the survey range, then upper-end + urgency_premium, then prices that were 3-4× the survey midpoint. Verifier (V4) caught all of them.

**Fix:** Two changes. First, we set `temperature: 0.3` on Pass 2 (down from 0.7) — fluency dropped 0.1 on the eval, plausibility went up 0.6. Second, we added a `price_band_observed` metric and alert when median observed band drifts >15% in a 24h window.

**Lesson:** the verifier is the seatbelt, not the steering wheel. When the verifier is rejecting >2% of generations, you have a different problem than "the verifier works."

### 3. The MongoDB region outage (March 2026)

**What happened:** Atlas eu-west-1 had a 22-minute partial outage. Our retrieval layer started returning empty result sets. Pass 2 was generating fluently against empty context — i.e., the model started inventing workers, which the verifier (V2 — worker grounding) caught and rejected. Users saw the deterministic fallback for 22 minutes.

**Fix:** Two changes. First, added a `retrieval.length == 0` short-circuit before Pass 2 — we now serve a "we're having trouble pulling the catalog, try a different city" message instead of running Pass 2 against an empty list. Second, added Atlas multi-region read replicas (the cost is real; the alternative is worse).

**Lesson:** the verifier saved us — users never saw fabricated workers. But the *user experience* of a 22-minute deterministic-fallback period was bad. The verifier is a backstop, not a UX.

### 4. The "Salé / Rabat" commuter-zone bug

**What happened:** Salé and Rabat are 5km apart across a river. A Salé-based user querying for "plumber in Rabat" was getting an empty workers list because V3 (city plausibility) was strict — Salé workers weren't shown to a Rabat query, and there were fewer Rabat workers than the retrieval expected.

**Fix:** Built a small adjacency table: `{rabat: [sale, temara], casablanca: [mohammedia, dar_bouazza], ...}`. V3 now accepts workers from adjacent commuter cities.

**Lesson:** geographic plausibility is not the same as geographic identity. The verifier needs to encode the commuter assumption.

### 5. The Arabic-script vs Arabizi rendering bug

**What happened:** Some users were typing "kankhdem" (Latin-script Arabizi) and getting responses in Arabic script, which they then had to mentally re-transliterate. Frustrating.

**Fix:** `locale_hint` parameter on the chat request. If the user types Arabizi, Pass 2 generates in Arabizi. The verifier's V5 (script purity) was updated to enforce *consistency*, not Arabic-only.

**Lesson:** the model can adapt to user preferences if you give it the right knob. The user does not have to learn the system.

---

## What we did not build (and why)

These were on the table and we said no:

- **A worker-facing scheduling primitive.** Tempting, but it changes the product surface from "match" to "transact," and transactional products demand payments, disputes, escrow. Out of scope until match is at 10,000 daily users.
- **User accounts.** Discussed above. The "no accounts" property is differentiating, and the cost (no cross-device sync) is acceptable until federation is built.
- **A worker-side rating system.** Ratings are present (1–5 stars from past customers), but we do not let the model surface a low-rated worker even if they match other criteria. Doing so would create a perverse "low-rated workers get hidden from the system" loop. Instead, low-rated workers get notification + a coaching nudge, off-platform.
- **A "premium tier" for verified workers.** Verification is binary, not tiered. Tiering would convert the marketplace into a pay-to-play, which is the failure mode of every competitor.
- **A vector-search semantic cache.** On the roadmap, but only as a cost-optimization. Will not be a retrieval primary.

---

## The roadmap, in priority order

1. **LoRA fine-tune Llama 3.1 8B for Pass 2.** Target: 3× cost reduction at the current eval-score parity. Compute budget: ~$50 on Modal for the first run. Recipe in [jak-ma-eval-suite/scripts/finetune/](https://github.com/Samielakkad/jak-ma-eval-suite).
2. **Semantic cache layer.** Sentence-level embedding lookup against the last 30 days of successful, verified Pass 2 outputs. Target: 30–40% query bypass. Latency win is incidental; the goal is cost.
3. **Browser-side image classifier.** MobileNetV3 via TensorFlow.js, trained on 50–100 labeled images per trade. Pushes vision classification cost from $0.005/image to $0.
4. **District federation.** E2EE blob sync of localStorage across user devices. Design is sketched; no code yet.
5. **Tangier + Agadir.** Adds the next two cities. The hard part is the pricing survey, not the catalog.
6. **Worker-side scheduling.** When match volume hits 10,000 daily users. Not before.

---

## The numbers, in context

| | jak.ma | A generic "AI marketplace" chatbot | What this means |
|---|---|---|---|
| Per-query inference cost | $0.0008 | $0.005–0.015 | We can afford the long tail |
| Verifier rejection rate | 0.7% | n/a (no verifier) | We catch the worst 0.7% |
| Price-band fabrication rate | 0% (verifier blocks) | ~5–8% (estimated from eval) | This is the moat |
| Trade-misclassification rate | 4% (caught by user retry) | ~12% | Schema-constrained classification matters |

The cost line is meaningful: at 100,000 daily queries (year-end target), our inference budget is ~$80/day. A generic chatbot architecture would cost $500–1500/day at the same volume.

---

## What I'd do differently

- **Build the eval suite before building the product.** We built jak.ma, then built the eval suite, then realized the eval suite would have told us to build a different thing. The methodology repo ([ernie-evaluation-notes](https://github.com/Samielakkad/ernie-evaluation-notes)) is the closest thing to "the eval suite I wish I had on day one."
- **Curate the price table earlier.** The survey was done at month 4. It should have been month 1. The model was fluent immediately; the model was *correct* only after the survey.
- **Skip the single-pass experiment.** We knew within a week that two passes would win. We spent another week on single-pass tool-use because it was "more elegant." It wasn't.
- **Write the verifier spec before the verifier.** The [VERIFIER_SPEC.md](https://github.com/Samielakkad/jak-ma-eval-suite/blob/main/VERIFIER_SPEC.md) was written after the verifier shipped. Writing it first would have forced the design conversation upstream.

---

## What I'd tell another team starting today

1. **Decide what the verifier checks before you write a prompt.** The verifier shape is the system.
2. **Use a cheap model and a strict verifier, not an expensive model and a permissive judge.** The verifier and the generator should disagree on different axes.
3. **The price table is the moat.** Whatever your equivalent of "what does this cost in this market" is, build it by hand before you ship.
4. **Two passes, not one.** Until proven otherwise on your eval slice.
5. **Static deployment, no accounts, localStorage.** Until the product surface requires otherwise.

---

## References

- [docs/architecture.md](https://github.com/Samielakkad/jak-ma-eval-suite/blob/main/docs/architecture.md) — the system paper
- [VERIFIER_SPEC.md](https://github.com/Samielakkad/jak-ma-eval-suite/blob/main/VERIFIER_SPEC.md) — what V1–V6 check
- [pm-frameworks-darija](https://github.com/Samielakkad/pm-frameworks-darija) — pricing taxonomy, calibration protocol, eval rubric
- [ernie-evaluation-notes](https://github.com/Samielakkad/ernie-evaluation-notes) — the calibration methodology applied here, from the Baidu ERNIE Mentor Program
- [darija-nlp-resources](https://github.com/Samielakkad/darija-nlp-resources) — public corpora, papers, tools

---

**Sami EL AKKAD** · Tsinghua SIGS AI MSc · sam25@mails.tsinghua.edu.cn · [jak.ma](https://jak.ma)
