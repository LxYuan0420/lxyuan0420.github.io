---
layout: post
title: "Accelerating LLM Inference on Mac: vllm-metal and Docker Model Runner"
description: "FAQ + quickstart for running MLX models on Apple Silicon with an OpenAI-compatible API"
---

Apple Silicon Macs are now powerful LLM inference servers, thanks to vllm-metal and Docker Model Runner. Traditionally, vLLM’s high-throughput model serving has been most common on NVIDIA GPUs (CUDA), but vllm-metal brings a Metal GPU backend to Apple Silicon Macs. Docker Model Runner wraps this in a single workflow, so “serve a local OpenAI-compatible API” becomes two commands: serve + `curl`.

## Quickstart (2 commands)

1) Serve the model (enables host TCP on port 12434, then loads the model):

```bash
docker desktop enable model-runner --tcp=12434 && \
  docker model run -d hf.co/mlx-community/Mistral-7B-Instruct-v0.3-4bit
```

Why the `--tcp=12434` part? Docker Model Runner is reachable from *containers* by default, but host processes (like `curl` or a local app) need a TCP port exposed to talk to it.

2) `curl` the OpenAI-compatible API from your host:

```bash
curl -sS http://localhost:12434/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hf.co/mlx-community/Mistral-7B-Instruct-v0.3-4bit",
    "prompt": "Explain quantum computing in simple terms:",
    "max_tokens": 128
  }'
```

## Real-world use case: local LLM API for development

Suppose you’re building a chat assistant or code completion tool. With vllm-metal, you can run a performant LLM API locally, test prompts, and iterate quickly—no cloud GPU needed.

Some practical ways this shows up:
- **Local CLI tools**: point any OpenAI-compatible CLI/client at `http://localhost:12434/v1` for fast prompt iteration.
- **IDE/editor integrations**: many plugins let you set a custom “OpenAI base URL”, so you can develop with a local model and keep the same API shape.
- **Local apps**: wire up a chat/completions endpoint while you prototype product flows (latency feels “instant” compared to cloud).
- **Integration tests**: exercise your OpenAI-client codepath against a local endpoint (no credentials, no flaky network).
- **Container workflows**: dev containers can call `http://model-runner.docker.internal/v1` without exposing anything publicly.

## Why it matters

- Performance: Metal backend leverages Apple Silicon GPU + unified memory.
- Simplicity: Docker Model Runner abstracts most of the setup—one command to load/run a model, standard API to integrate.
- Cost & Privacy: Run models locally, no cloud costs, and you keep prompts/data on your machine.

## Conclusion

vllm-metal and Docker Model Runner make LLM serving on Apple Silicon much easier. Try it and turn your Mac into a local LLM server for development and experimentation.

---

## FAQ

### Q1: What is Docker Model Runner?

Docker Model Runner is a feature in Docker Desktop that lets you run AI models—including LLMs—using a single command. It selects an inference engine based on your platform and the model you’re running: `llama.cpp` is the default engine, `vllm` is used for GPU serving on NVIDIA/AMD (CUDA/ROCm), and on Apple Silicon Macs you can use `vllm-metal` (Metal GPU) for MLX models.

### Q2: How do I know if vllm-metal or vLLM is being used?

After running `docker model run ...`, check which backend is serving your model:

```bash
docker model ps
```

If `BACKEND` is `vllm`, Docker Model Runner is routing through vLLM (on Apple Silicon this is the vllm-metal runner). If it’s `llama.cpp`, you’re on the llama.cpp engine.

If you want the exact runner build/version, use:

```bash
docker model status --json
```

### Q3: What Docker version do I need?

Docker Model Runner requires Docker Desktop 4.40+ on macOS (4.41+ on Windows). For vllm-metal on macOS, Docker’s vllm-metal announcement recommends Docker Desktop 4.62+.

On Docker Desktop, you typically don’t “install the runner” manually. To verify the vllm-metal runner is present, check:

```bash
docker model status --json
```

You should see `vllm` reporting something like `Running: vllm-metal ...`.

### Q4: How do I check which backend and hardware are being used?

Use:

```bash
docker model status
docker model logs
```

The `status` output shows which inference engine is running, and the logs typically indicate the device/backend (e.g., Metal vs CUDA).

### Q5: Can I use this on Linux or Windows?

Yes, but vllm-metal is only for macOS on Apple Silicon. On Linux and Windows (via WSL2), Docker Model Runner can use `vllm` on NVIDIA GPUs; otherwise it falls back to `llama.cpp`.

### Q6: How do I stop the model server?

Docker Model Runner will unload models after inactivity (you can see the TTL in the `UNTIL` column of `docker model ps`). If you want to free memory immediately, unload manually:

```bash
docker model unload --all
```

### Q7: How do I send requests to the model?

Use `curl` or any OpenAI-compatible client.

From your host (with TCP enabled), use:

```text
http://localhost:12434/v1
```

If `localhost:12434` doesn’t respond, you probably haven’t enabled TCP. Run `docker desktop enable model-runner --tcp=12434` again and retry.

From another container, use:

```text
http://model-runner.docker.internal/v1
```

Note: you may also see examples using the `/engines/v1` prefix (for example `http://localhost:12434/engines/v1/completions`). If your `/v1/...` URL 404s, try the `/engines/v1/...` path.

Example (OpenAI-style completions):

```bash
curl http://localhost:12434/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hf.co/mlx-community/Mistral-7B-Instruct-v0.3-4bit",
    "prompt": "Explain quantum computing in simple terms:"
  }'
```

### Q8: Can I make this reachable from another machine (a public Mac VM)?

Docker Desktop’s Model Runner TCP endpoint is meant for local development and binds to loopback (`127.0.0.1`), not `0.0.0.0` (there isn’t currently a documented “bind all interfaces” flag for Docker Desktop).

For remote access, use an SSH tunnel (recommended):

```bash
ssh -L 12434:127.0.0.1:12434 user@YOUR_MAC_VM
```

Or put an authenticated reverse proxy in front of it. Don’t expose an unauthenticated inference endpoint to the public internet.

### Q9: Why did `docker model configure --context-size ...` crash my vllm-metal runner?

Some `docker model configure` options are passed through to the underlying engine. If a runner build doesn’t support a particular flag (for example a `--max-model-len` mapping), model load can fail.

Reset the model config back to empty and retry:

```bash
docker model configure hf.co/mlx-community/Mistral-7B-Instruct-v0.3-4bit
```

You can confirm what’s currently set via:

```bash
docker model configure show hf.co/mlx-community/Mistral-7B-Instruct-v0.3-4bit
```

Also note: `docker model configure` only updates per-model runtime config. It doesn’t “keep the model running”. If your model disappears from `docker model ps`, reload it with:

```bash
docker model run -d hf.co/mlx-community/Mistral-7B-Instruct-v0.3-4bit
```

### Q10: What’s the difference between Docker Model Runner and Ollama?

They overlap (both can run LLMs locally), but they optimize for different workflows:

- **Docker Model Runner**: integrated into Docker Desktop, routes requests to different engines depending on model/hardware (`llama.cpp`, `vllm`, and on Apple Silicon the vLLM path uses `vllm-metal` for MLX models). When your model is on the `vllm` backend, you also get vLLM’s serving optimizations like a request scheduler with continuous batching, efficient KV cache management (PagedAttention), and prefix caching support. It’s a good fit when you want an OpenAI-compatible `/v1` endpoint that works cleanly with containers and local dev tools.
- **Ollama**: a standalone local model runtime/daemon with its own model packaging workflow (Modelfiles). It’s a good fit when you want the simplest “pull + run” local experience and you’re building around the Ollama ecosystem and APIs.

## Sources

- <https://www.docker.com/blog/docker-model-runner-vllm-metal-macos/>
- <https://docs.docker.com/ai/model-runner/>
- <https://docs.docker.com/ai/model-runner/api-reference/>
