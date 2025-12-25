---
layout: post
title: "Speculative decoding for faster LLM inference (and what EAGLE changes)"
description: "Draft+verify, rejection sampling, and the EAGLE / EAGLE-2 / EAGLE-3 line"
---

# Abstract

Speculative decoding (often presented as *speculative sampling*) accelerates autoregressive generation by splitting decoding into two roles: a **draft** stage that cheaply proposes multiple future tokens, and a **verification** stage where the **target** model scores the drafted span in a single **batched forward pass** (producing logits for each drafted position) and accepts a prefix of those draft tokens (stopping at the first rejection). When implemented with the standard accept/reject correction, the generated text is **distribution-identical** to sampling directly from the target model.

This post walks through the mechanics that matter in real inference stacks—what is computed, what is discarded, and what drives speedup—and then summarizes what the EAGLE family changes in the drafting model: **token-level** drafting (classic speculative sampling) versus **feature-level** drafting (EAGLE), plus the draft-tree and training upgrades introduced in **EAGLE‑2** and **EAGLE‑3**.

# Naming: “v1 / v2 / v3” (my mental model)

The papers don’t label themselves “v1/v2/v3”. I’m using it as a shorthand for the evolution:

- **v1**: draft+verify with a *linear* token draft (classic speculative sampling).
- **v2**: draft a *tree* (branching width/depth) so the verifier can often accept a deeper continuation and waste less work past the first rejection.
- **v3**: improve the draft model itself (EAGLE feature drafting; EAGLE‑3 training-time test moves toward direct token prediction + multi-layer feature fusion) so the verifier accepts longer runs and wastes less verification work.

In these terms, the EAGLE line is basically “v2+v3 on top of v1”: EAGLE adds feature-level drafting (v3-ish) and a draft-tree mechanism (v2-ish), EAGLE‑2 focuses on better trees, and EAGLE‑3 focuses on better training for the draft model.

# A simpler walkthrough (first principles)

The crux of speculative decoding is not “guessing tokens faster”. It’s reducing how often the **target** model has to run the expensive parts of a transformer forward pass.

At step *t*, the expensive work is:

- **Attention**: forming Q/K/V projections and attending over all previous tokens (mostly a memory + kernel launch problem at decode-time).
- **MLP blocks**: the per-layer feed-forward compute.
- **KV cache update**: writing the new token’s K/V so future tokens can attend to it.

Once a token’s KV is written, it is reused for the rest of the generation. That’s why inference cost scales with how many KV entries the **target model** had to compute—both the ones you keep (accepted tokens) and the ones you later discard (rejected suffix).

Classic speculative sampling (the draft+verify pattern) changes the schedule:

1) a cheap **draft** proposes multiple tokens ahead,
2) the **target** runs one batched forward over the drafted span to compute probabilities,
3) an accept/reject rule keeps the output distribution identical to the target model.

The basic rule in verification is simple: you scan the draft left-to-right, accept tokens until the first rejection, and then **discard all later draft tokens**. That “first rejection boundary” is where you fall back to a corrected sample from the target distribution and continue from there.

So you still pay for target attention/MLP **for the drafted span**, but you pay it in fewer, chunkier passes with better utilization and fewer sequential “one-token” iterations.

EAGLE then changes *what the draft predicts*: instead of drafting tokens with a smaller LLM, it drafts **second-to-top-layer features** (the hidden state before the LM head) and uses the target LM head to turn those features into token distributions. The goal is higher acceptance rates with a lighter draft, while keeping verification lossless.

Terminology note: people sometimes say “compute speculation” or “speculating KV/hidden states”. At a minimum, speculative decoding already has a compute-reuse boundary: the verifier computes KV for a speculative span, you **keep** the KV for accepted tokens, and you **drop** the KV suffix after the first rejection. Importantly, EAGLE’s “features” are for producing a better draft; the verifier still runs to validate tokens and build its own KV during verification. Some research goes further and tries to reuse *draft-produced* internal representations; the EAGLE papers are best understood as strengthening the **drafting model** (feature-level drafting + better trees + better training) while keeping lossless verification against the target distribution.

# 1. Baseline: autoregressive decoding (prefill + decode)

Most inference engines split generation into:

1) **Prefill**: run the prompt through the model once to build KV cache.
2) **Decode**: generate one token at a time, updating the KV cache each step.

## Pseudocode

```python
kv = model.prefill(prompt_ids)

token = prompt_ids[-1]
while not stop(token):
    logits, kv = model.decode_one(token, kv)
    token = sample(logits)  # greedy = argmax(logits)
```

The loop is exact and simple, but it’s latency-bound: each new token needs another pass through the target model.

# 2. v1: Token-level speculative sampling (draft + verify)

Classic speculative sampling adds a **draft model** `q` and keeps the original **target model** `p` as the authority. The draft proposes `k` tokens and their token distributions; the target then verifies those `k` positions by running a single **batched** forward over the drafted span (producing logits for each drafted position).

Let the draft propose tokens `t̂_{j+1:j+k}` with distributions `q_{j+1:j+k}`. The target runs a batched forward over the drafted span to obtain `p_{j+1:j+k}`. Tokens are accepted left-to-right; token `t̂_{j+i}` is accepted with probability:

`min(1, p_{j+i}(t̂_{j+i}) / q_{j+i}(t̂_{j+i}))`

If a token is rejected, you resample that position from the corrected distribution:

`norm(max(0, p_{j+i} - q_{j+i}))`

This accept/reject rule is what preserves exact sampling equivalence with the target model.

Verification is also strictly sequential: you accept draft tokens until the first rejection, and then discard the remaining draft suffix. (If you accept all `k` draft tokens, many implementations additionally sample one more token from the target’s “next-token” distribution that is already available from the same verification pass.)

## Pseudocode

```python
# draft stage
draft_tokens, q_dists = draft.generate(prefix, k)  # tokens + per-step distributions

# verification stage (one batched forward over the draft span)
p_dists = target.probabilities(prefix, draft_tokens)  # p_{j+1:j+k}

accepted = []
for i, tok in enumerate(draft_tokens):
    u = uniform_0_1()
    if u < min(1.0, p_dists[i][tok] / q_dists[i][tok]):
        accepted.append(tok)
        continue

    # rejection => discard the remaining draft tokens and resample at this position
    tok = sample(normalize(relu(p_dists[i] - q_dists[i])))
    accepted.append(tok)
    break
```

For **greedy decoding** (temperature = 0), acceptance reduces to a simpler condition (“does the draft token equal the target argmax?”). For sampling (temperature > 0), you need the accept/reject correction above to be distribution-exact.

# 3. What drives speedup in practice

Speculative sampling doesn’t magically make the target model cheaper per token; it changes the *shape* of the work.

The target model is invoked less often (one verification pass per drafted chunk), and the verification pass itself has better hardware utilization because it processes multiple decode steps together. The speedup is dominated by:

- **Average accepted length** (how many draft tokens survive before rejection).
- **Draft overhead** (how expensive it is to generate the draft tokens/distributions).
- **Waste** (how much verification compute is thrown away past the first rejection).

When the draft is accurate and cheap, you accept long runs and amortize target passes well. When the draft is inaccurate or expensive, speculative sampling collapses back toward baseline.

# 4. v2: Draft trees (more candidates, less waste)

v1 has a hard failure mode: the first rejection invalidates the entire remaining suffix of the draft. That means any work spent drafting or verifying tokens beyond the first rejection is wasted.

Tree-based drafting is a way to spend the same “draft budget” more intelligently. Instead of proposing a single chain of length `k`, the draft produces a small **tree** of candidates (depth = “how far ahead”, width = “how many branches”). The verifier then commits a **deeper accepted continuation** (accepting tokens along a valid branch), and discards everything else.

The key intuition is that branching makes it less likely that one early mistake ruins the entire speculative chunk, so you often accept more tokens per iteration and waste less verification work.

It’s still important to keep the “output is a single sequence” invariant in mind: the tree is just a way to propose *more* candidate continuations for the verifier to score; the verifier ultimately commits one sequential prefix and moves on.

EAGLE uses **tree attention** to generate a draft tree with depth `m` while producing more than `m` tokens in `m` forward passes. You can think of it as “many drafted candidates per iteration”, with the accept/reject correction still guaranteeing the target distribution.

EAGLE‑2 then improves the tree itself. The key observation is that acceptance rates are **context-dependent**, not just position-dependent, so a static tree wastes budget in the wrong places. EAGLE‑2 uses a **context-aware dynamic draft tree**, guided by the draft model’s confidence (treated as a proxy for acceptance probability), to allocate branching where acceptance is likely.

# 5. v3: Better draft models (features + training)

Tree structure (v2) helps, but it still depends on having a good draft model. v3 is about making the draft model *more predictive per unit compute*, so you accept longer runs and waste less verification work.

## KV-cache view (reuse vs recompute)

One concrete way to understand the runtime behavior is to track KV:

- During verification, the **target** model computes logits *and* KV entries for the speculative span.
- If you accept a token, you keep the target’s KV for that token (so you never recompute it later).
- When the first rejection happens, you drop the speculative KV for that rejected position and everything after it, sample a corrected token, and resume decoding with fresh KV from that boundary onward.

This reuse/drop behavior exists in v1 and v2 too; the reason “v3” matters in practice is that better drafting increases the accepted prefix length, so more of the verifier’s verification work becomes reusable KV and less becomes discarded suffix.

EAGLE’s core “v3-ish” move is **feature-level drafting**. In the paper’s notation, a “feature” is the **second-to-top-layer hidden state** (the vector right before the LM head). Instead of using a smaller transformer to draft next-token distributions directly, EAGLE predicts the next feature vector and then uses the **target model’s LM head** to obtain a token distribution.

Feature-level autoregression has a subtle ambiguity: if you don’t condition on the sampled token, the next feature is not a single target—it’s an implicit mixture over multiple possible next tokens. EAGLE resolves this by feeding the draft model an **advanced token sequence** (the token sequence shifted by one time step), so the feature predictor is conditioned on realized token outcomes rather than a mixture.

Implementation-wise, you can think of EAGLE as adding a lightweight feature predictor in front of the target LM head: predict the next second-to-top-layer feature vector, then project it through the LM head to obtain a draft distribution to verify.

EAGLE‑3 then targets how well the draft improves with more training data. The paper attributes EAGLE’s limited scaling to a “feature prediction constraint” and introduces a **training-time test** draft architecture that:

- switches to **direct token prediction** while simulating multi-step generation during training, and
- replaces reliance on only top-layer features with **multi-layer feature fusion** (low/mid/high-level features from the target model).

EAGLE‑3 reports speedups up to ~6.5× (plus throughput gains in SGLang at larger batch sizes).

In practice, these “better draft model” ideas should also compose with draft-tree ideas (v2): the tree changes how you spend the draft budget, while v3 changes how good each drafted step is.

# References (papers)

- EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty — arXiv:2401.15077 <https://arxiv.org/abs/2401.15077>
- EAGLE-2: Faster Inference of Language Models with Dynamic Draft Trees — arXiv:2406.16858 <https://arxiv.org/abs/2406.16858>
- EAGLE-3: Scaling up Inference Acceleration of Large Language Models via Training-Time Test — arXiv:2503.01840 <https://arxiv.org/abs/2503.01840>
- Leviathan et al.: Fast Inference from Transformers via Speculative Decoding — arXiv:2211.17192 <https://arxiv.org/abs/2211.17192>
- Chen et al.: Accelerating Large Language Model Decoding with Speculative Sampling — arXiv:2302.01318 <https://arxiv.org/abs/2302.01318>
