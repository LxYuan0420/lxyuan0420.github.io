---
layout: post
title: "Codex skills are basically prompt aliases"
description: "Skills = reusable prompts (plus optional scripts) for Codex CLI"
---

Codex CLI is smart, but it can still “wander”:
- you ask for something
- it tries a few approaches
- you keep nudging it toward the tool/workflow you actually wanted

A **skill** is the blunt fix: it’s basically a **predefined prompt** (in markdown) that tells Codex “when the task looks like this, do it *this* way”. Sometimes it also ships a small script so Codex can run the boring parts reliably.

It feels like a **slash command**, or a **command alias**, but for prompts.

# What’s in a skill?

On disk it’s just a folder under `$CODEX_HOME/skills` (usually `~/.codex/skills`) that looks like:

```text
skill-name/
  SKILL.md
  LICENSE.txt     (usually)
  scripts/        (optional)
  examples/       (optional)
  reference/      (optional; sometimes `references/`)
  assets/         (optional)
```

`SKILL.md` is the main thing. It’s basically “the prompt you wish you didn’t have to keep rewriting”.

# How skills get invoked

You can invoke a skill explicitly, like:

```text
$some-skill do the thing
$some-skill run the standard workflow for this task
```

Skills can also trigger implicitly if your request matches the `description` in the skill’s YAML header, but explicit is more reliable if you know what you want.

After installing a new skill, you usually need to restart Codex so it can pick it up.

# Why skills exist (the practical version)

- **Less repetition**: you don’t rewrite the same “do X, then Y, then Z” prompt every time.
- **Less guessing**: you stop the agent from trying random tools before it lands on the right one.
- **More reliable steps**: if the workflow is fragile, the skill can enforce the exact sequence (and sometimes ship a script to make it deterministic).
- **Shareable**: a skill is just a folder; people can pass it around like a dotfile or a small repo (always check `LICENSE.txt`).

# What a skill is *not*

It’s not magic. A skill doesn’t “install new intelligence”. It mainly:
- adds a stable, reusable set of instructions
- optionally adds helper files/scripts so the agent can execute steps consistently

Net: skills are just **reusable prompt packs**. You trade a bit of setup for less back-and-forth later.
