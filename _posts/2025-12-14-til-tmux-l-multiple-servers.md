---
layout: post
title: "TIL: Using multiple tmux servers with -L"
description: "Run isolated tmux session lists by choosing a different server socket name"
---

`tmux` is a client/server app. Your `tmux` command (client) talks to a long-running tmux server over a UNIX socket.

By default, tmux uses the `default` socket (usually something like `/tmp/tmux-<uid>/default`). `tmux -L <name>` lets you pick a different socket name, which effectively gives you a separate tmux server with its own independent sessions/windows/panes.

# Using Multiple tmux Servers with -L and Your Commands

Think of `-L` as “which tmux server do I mean?”. Here are two servers: `work` and `personal`.

Create a few sessions (`-d` keeps them detached, so you can run these from a normal shell):

```bash
$ tmux -L work new -d -s workA
$ tmux -L work new -d -s workB
$ tmux -L personal new -d -s personalA
```

List sessions on each server (they’re isolated):

```bash
$ tmux -L work ls
$ tmux -L personal ls
```

Attach to one session (pick one; `attach` takes over your terminal):

```bash
$ tmux -L work attach -t workA
# or
$ tmux -L personal attach -t personalA
```

Any tmux subcommand works the same — just keep the same `-L`:

```bash
$ tmux -L work rename-session -t workA workMain
$ tmux -L work kill-session -t workB
$ tmux -L personal kill-session -t personalA
```

Shortcut: `new -A` means “attach if it exists, otherwise create it”. It’s handy when you just want to jump into a known session:

```bash
$ tmux -L work new -A -s workA
```

## Why this is useful
- Keep “work” vs “personal” tmux state separated.
- Run a throwaway tmux inside an existing tmux without mixing session lists.
- Debug tmux config changes in a fresh server without killing your main sessions.

## Notes
- `-L` is “socket name”. If you want a specific socket path, use `tmux -S /path/to/socket ...` instead.
- If you forget `-L work` / `-L personal`, you’ll be talking to the default server and won’t see those sessions.
- If you’re already inside tmux, `tmux` will normally connect to your current server (because the `TMUX` env var is set). Use `-L work` / `-L personal` to explicitly talk to the other server.
