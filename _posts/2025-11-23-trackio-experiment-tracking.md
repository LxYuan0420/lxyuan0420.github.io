---
layout: post
title: "Trackio Experiment Tracking"
date: 2025-11-23
---
# Trackio Experiment Tracking

<!-- Filename: 2025-11-23_til_trackio-experiment-tracking.md -->

# Track experiments locally with Trackio

## What
Use Trackio when you want simple experiment tracking that feels like wandb but stays local by default and can sync to a Hugging Face Space when you need a shared dashboard.

## Context
- Stack: Python 3.10+, common ML tools (PyTorch, transformers, accelerate or plain training loops).
- You want to log metrics and configs without running a separate tracking server.
- You already use Hugging Face Hub or Spaces and prefer something open source and free.

## Steps / Snippet
```bash
# 1) Install Trackio in your environment
pip install trackio

# 2) Save this as trackio_demo.py (see below)

# 3) Run the script
python trackio_demo.py

# 4) Open the local dashboard for that project
trackio show --project til-trackio-demo
```

```python
# trackio_demo.py
import math

import trackio

def run(epochs: int = 5) -> None:
    trackio.init(
        project="til-trackio-demo",
        config={"epochs": epochs, "optimizer": "adamw"},
    )

    for step in range(1, epochs + 1):
        loss = math.exp(-step / 4)
        lr = 3e-4 * (1 - step / epochs)

        trackio.log(
            {
                "train/loss": loss,
                "train/lr": lr,
            },
            step=step,
        )

    trackio.finish()


if __name__ == "__main__":
    run()
```

To also send metrics to a Hugging Face Space dashboard, add a `space_id`:

```python
trackio.init(
    project="til-trackio-demo",
    space_id="YOUR_USERNAME/trackio-dashboard",
)
```

## Pitfalls
- Trackio focuses on metrics and configs; it does not yet replace wandb’s more advanced features (sweeps, rich custom panels, or a full artifact manager).
- On Spaces, the live dashboard uses an in-memory database with periodic backups to a dataset. If the Space restarts between syncs, you can lose a small window of recent points.
- Trackio is young and the API can still change; pin the version you use and skim the release notes when upgrading.

## Links
- Trackio GitHub repository: search for `gradio-app trackio github`.
- Trackio documentation and examples: search for `hugging face trackio docs`.
- Launching Trackio dashboards on Spaces: search for `trackio hugging face space example`.
