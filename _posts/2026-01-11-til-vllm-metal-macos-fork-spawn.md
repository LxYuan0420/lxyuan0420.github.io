---
layout: post
title: "TIL: Why fork() can crash on macOS after importing Metal/ObjC libraries (and why spawn fixes it) — vllm-metal"
description: "A practical note on vLLM engine-core workers, macOS fork-safety, and the vllm-metal default to spawn"
---

## What happened

On macOS, I hit intermittent hard crashes when using `vllm-metal` via vLLM’s `LLM()` API: the engine-core worker fails to start, and the parent process often only reports a generic “engine core initialization failed”. The reason it looks generic is that the actual crash happens inside the child worker process. When you do catch the child’s output, it can surface as: `objc_initializeAfterForkError`.

If you’ve been staring at logs thinking “why is engine init flaky?”, this is one of those cases where the first failure signal is too high-level. The parent is doing the right thing (it tried to start the worker), but the worker is crashing during its own initialization, so the parent can only report that startup failed.

## Root cause: vLLM worker startup + fork + Objective‑C runtime

vLLM starts an engine-core worker process via Python multiprocessing. If the multiprocessing start method is `fork`, the child is created by forking the parent process and inheriting its memory state.

That inheritance is the problem on macOS when the parent has already imported libraries that touched the Objective‑C runtime. Apple explicitly warns that `fork()` after the Objective‑C runtime has been initialized is not fork-safe. In practical Python ML stacks, it’s common to touch the ObjC runtime indirectly during import when using libraries in the Metal/MPS/MLX/PyTorch ecosystem. If the parent process imports those first and then vLLM forks a worker, the child can crash immediately during worker initialization—sometimes as `objc_initializeAfterForkError`.

The important detail here is the ordering: it’s not that vLLM “uses Objective‑C”, it’s that the parent process can end up with Objective‑C runtime state initialized before vLLM launches its worker. With `fork`, the worker inherits that state midstream, and macOS treats that as undefined/unsafe territory.

## Why spawn works

The multiprocessing start method `spawn` starts a fresh Python interpreter process and imports modules from scratch. That avoids inheriting the parent process’s partially-initialized Objective‑C runtime state, which is what makes `fork()` unsafe in this situation. In other words: `spawn` prevents the crash by sidestepping the macOS “fork-safety” hazard entirely.

You can think of it as “clean room initialization”: the child starts from an empty process image, and the imports happen in the child, not in the parent and then copied into the child. That’s exactly what you want when the thing you’re trying to avoid is copying runtime initialization state across a `fork()`.

## This isn’t new: vLLM already does it for CUDA

This safety principle already exists in vLLM on CUDA: when CUDA is initialized, vLLM forces `spawn` because CUDA is also not fork-safe after initialization. vLLM logs this decision, so you can see when it detects the unsafe state and switches the worker start method accordingly.

The macOS + Metal/ObjC case is the same shape of problem: once the parent has initialized a runtime that isn’t fork-safe, forking a worker is a risky default.

## The fix in vllm-metal

The implemented fix is to make `vllm-metal` default `VLLM_WORKER_MULTIPROC_METHOD=spawn` on macOS when the Metal plugin activates, but only if the user hasn’t explicitly set `VLLM_WORKER_MULTIPROC_METHOD`. Explicit user overrides are respected.

This is intentionally a “default, not a mandate”: if you know what you’re doing and want to force a different method, you still can. But for the common case—where users just want `LLM()` to work reliably on macOS—the safer default is to avoid `fork` once the environment has touched Objective‑C runtime state.

This is a safe default: it’s not a behavior change for non-macOS platforms, and it’s not intended to solve unrelated networking/VPN/distributed issues (or anything else outside the macOS fork-safety crash class).

## How to reproduce/verify (macOS)

```bash
# safe:
python -c "from vllm import LLM; LLM(model='HuggingFaceTB/SmolLM2-135M', max_model_len=128, dtype='float16')"

# expected to fail:
VLLM_WORKER_MULTIPROC_METHOD=fork python -c "from vllm import LLM; LLM(model='HuggingFaceTB/SmolLM2-135M', max_model_len=128, dtype='float16')"
```

## Links

- PR: <https://github.com/vllm-project/vllm-metal/pull/62>
- Issue: (none)
- vLLM multiprocessing docs: <https://docs.vllm.ai/en/latest/usage/troubleshooting.html#python-multiprocessing>
- ObjC + fork reference: <https://www.sealiesoftware.com/blog/archive/2017/6/5/Objective-C_and_fork_in_macOS_1013.html>
