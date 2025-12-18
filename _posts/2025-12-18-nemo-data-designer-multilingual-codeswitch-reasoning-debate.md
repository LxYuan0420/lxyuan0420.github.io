---
layout: post
title: "Multilingual code-switch dataset generation with NeMo Data Designer"
description: "Generate code-switched query/reasoning/debate/final-answer rows with one structured LLM call per record"
---

I wanted a repeatable way to generate “hard-mode” synthetic data: multilingual **code-switched** prompts with **long-form reasoning**, a **counter-argument debate**, and a **concise final answer** — all aligned to the same scenario.

So I built a single-file generator on top of **NVIDIA NeMo Data Designer**.

Script (gist): <https://gist.github.com/LxYuan0420/93db7cc99421aacacd397f203c9780c1>

# What it generates

Each row contains **4 core columns**:

- `query`: a realistic multi-sentence scenario + constraints
- `reasoning`: long-form multi-paragraph reasoning (no final answer)
- `debate`: critique / counterargument + alternative POV (no final answer)
- `final_answer`: the conclusion (concise, no step-by-step)

It can also keep “context columns” you can filter on later:

- `language_mix` (weighted toward Singlish/Manglish)
- `mix_level` (`light|medium|heavy`)
- `domain`, `topic`, `task_kind`, `difficulty`

# Why “one LLM call per row” matters

Generating `query`, `reasoning`, `debate`, `final_answer` as separate calls often gives you:

- drift (debate argues against a different query)
- mismatched tone/language mix across fields
- slower iterations and more cost

This script generates **one structured JSON object per row**, then “fans out” into the final columns. You get better alignment with fewer moving parts.

# How it works (pipeline)

High level flow:

1) **Samplers** pick context for the row (`language_mix`, `mix_level`, `domain`, `topic`, `task_kind`, `difficulty`)
2) A single structured generation produces a JSON object with `query`, `reasoning`, `debate`, `final_answer`
3) The final dataset columns are extracted from that object

The result is consistent rows that still vary a lot across language mixes and problem types.

# Run it (step-by-step)

## 0) Set your OpenAI key

```bash
$ export OPENAI_API_KEY="..."
```

## 1) Run directly from the gist (no download)

```bash
$ uv run \
  https://gist.github.com/LxYuan0420/93db7cc99421aacacd397f203c9780c1/raw/nemo_data_designer_multilingual_codeswitch_reasoning_debate.py \
  --model-alias openai-text \
  --num-records 20 \
  --artifact-path artifacts/multilingual_codeswitch_reasoning_debate \
  --max-parallel-requests 8 \
  --max-tokens 1500 \
  --print-records 1
```

## 2) Or download it and run locally

```bash
$ curl -L \
  https://gist.github.com/LxYuan0420/93db7cc99421aacacd397f203c9780c1/raw/nemo_data_designer_multilingual_codeswitch_reasoning_debate.py \
  -o nemo_data_designer_multilingual_codeswitch_reasoning_debate.py

$ uv run nemo_data_designer_multilingual_codeswitch_reasoning_debate.py \
  --model-alias openai-text \
  --num-records 20 \
  --artifact-path artifacts/multilingual_codeswitch_reasoning_debate \
  --max-parallel-requests 8 \
  --max-tokens 1500 \
  --print-records 1
```

## Script arguments

- `--model-alias`: model alias from NeMo Data Designer config (must use `provider=openai`)
- `--num-records`: number of rows to generate
- `--artifact-path`: output folder for run artifacts
- `--max-parallel-requests`: override chat-completion concurrency (higher = faster; watch rate limits)
- `--max-tokens`: override `max_tokens` per row (higher = longer outputs; more latency/cost)
- `--print-records`: print up to N records to stdout as JSON (0 disables)
- `--drop-context-columns` / `--no-drop-context-columns`: drop sampler context columns from the final dataset
- `--keep-raw-structured` / `--no-keep-raw-structured`: keep the intermediate structured JSON column (`example`)
- `--dotenv` / `--no-dotenv`: load environment variables from a local `.env` file

## 3) Where the output goes

By default, artifacts land under:

```bash
$ ls artifacts/multilingual_codeswitch_reasoning_debate/
```

Each run creates a timestamped folder like `dataset_YYYY-MM-DD_HHMMSS/` containing:

- `dataset.jsonl` (easy to inspect / stream / diff)
- `parquet-files/` (for training pipelines)
- `column_configs.json`, `model_configs.json` (reproducibility)

# Inspect the output quickly

```bash
$ head -n 1 artifacts/multilingual_codeswitch_reasoning_debate/**/dataset.jsonl
```

If you have `jq`:

```bash
$ head -n 1 artifacts/multilingual_codeswitch_reasoning_debate/**/dataset.jsonl | jq .
```

# Example row (trimmed)

A “heavy code-switch” sample looks like this (trimmed for readability):

```json
{
  "language_mix": "Singlish/Manglish (English + 中文(简体) + Bahasa Melayu)",
  "mix_level": "heavy",
  "domain": "Environment & sustainability",
  "topic": "comparing two eco choices with constraints",
  "task_kind": "optimization (best choice under constraints)",
  "difficulty": "hard",
  "query": "Eh bro, my office kena new green initiative, need choose between two types of aircon systems: one is the biasa inverter type, another is a solar-assisted aircon setup.\n\nBoth can fit budget, but boss say must jaga electricity bill and carbon footprint, plus maintenance cannot too mafan. Office got ~30 staff, use aircon 9am–6pm weekdays, and the roof sometimes shaded by a nearby building. Which option should we pick, and how to justify it properly?",
  "reasoning": "Ok lah, let’s break it down step by step.\n\nFirst, define what we care about: (1) monthly kWh cost, (2) carbon footprint, (3) operational risk and maintenance load over ~10 years.\n\nInverter system: common parts, known reliability, and efficiency gains come from variable-speed operation. If we estimate baseline usage at X kWh/day, and inverter improves efficiency by ~Y%, we can approximate annual savings = X * Y% * working_days.\n\nSolar-assisted system: savings depend heavily on actual sun-hours and roof shading. If roof shading reduces output during peak hours, the expected offset might drop a lot (and it’s not just 'average sunshine'). Also add more components (panels/inverter/wiring), so failure points + maintenance complexity increase.\n\nSo the decision is basically expected savings vs added risk: compare expected annual kWh offset (with conservative shading assumptions) against added maintenance + downtime risk.",
  "debate": "But you know hor, the reasoning above may still be too optimistic or too simplified.\n\n1) The inverter savings estimate assumes steady usage and stable HVAC load, but real offices have meeting rooms, peak occupancy, and hot afternoons that change the load curve.\n\n2) For solar, the 'roof sometimes shaded' detail is huge. During monsoon season or haze, output can jatuh below your assumptions. If the solar-assisted system’s payback relies on high utilization, the ROI might collapse.\n\n3) There’s also a third option: pick inverter aircon now (lower risk), and invest the remaining budget into better insulation/film/shades + a smaller rooftop solar array just to offset general office electricity (not tightly coupled to the HVAC system).\n\nSo unless you can measure real roof exposure and maintenance capacity, the safer engineering decision is still the simplest system with predictable performance.",
  "final_answer": "Pick the inverter aircon as the default: it’s lower-risk, easier to maintain, and its savings are more predictable. Only choose the solar-assisted setup if you can validate roof exposure (including shading + monsoon/haze impact) and you’re confident you can maintain the extra components over time."
}
```

# Tweaks I actually use

- Make it cheaper/faster: reduce `--max-tokens` and/or `--num-records`, increase later.
- Control “spiciness”: adjust `mix_level` weights (more `heavy` if you want aggressive mixing).
- Training format: keep context columns during iteration, drop them for the final release.
