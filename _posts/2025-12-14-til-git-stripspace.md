---
layout: post
title: "TIL: Clean up text with git stripspace"
description: "Normalize whitespace (and optionally strip comments) the same way Git does"
---

`git stripspace` is a small Git command that cleans text the same way Git cleans commit messages, tag messages, and notes.

With no flags it:
- removes trailing whitespace on every line
- collapses multiple blank lines into a single blank line
- trims blank lines at the start/end
- ensures the file ends with a newline

<br>
Important: it reads from stdin and writes to stdout — it does not take filenames (so `git stripspace a.py` won’t work).

Usage:

```bash
git stripspace [-s | --strip-comments]
git stripspace [-c | --comment-lines]
```

# Example (with `$` indicating end of line):

Note: the leading `|` and trailing `$` are just visual markers (they are not part of the file). To render any file that way:

```bash
$ sed 's/$/$/' noisy.txt | sed 's/^/|/'
```

### Noisy input:

```text
|A brief introduction   $
|   $
|$
|A new paragraph$
|# with a commented-out line    $
|explaining lots of stuff.$
|$
|# An old paragraph, also commented-out. $
|      $
|The end.$
|  $
```


# Run it:

```bash
$ git stripspace < noisy.txt
```

### Output:

```text
|A brief introduction$
|$
|A new paragraph$
|# with a commented-out line$
|explaining lots of stuff.$
|$
|# An old paragraph, also commented-out.$
|$
|The end.$
```


# Strip comment lines:

```bash
$ git stripspace --strip-comments < noisy.txt
```

### Output:

```text
|A brief introduction$
|$
|A new paragraph$
|explaining lots of stuff.$
|$
|The end.$
```

# Print the cleaned version to stdout:

```bash
$ git stripspace < a.md
```

# Overwrite a file safely:

```bash
$ tmp="$(mktemp)"
$ git stripspace < a.md > "$tmp" && mv "$tmp" a.md
```

# If you’re cleaning something like `.git/COMMIT_EDITMSG`, use `--strip-comments` to drop lines that start with `#`:

```bash
$ git stripspace --strip-comments < .git/COMMIT_EDITMSG
```
<br>
Note: this is for cleaning Git “metadata” text. If you’re fixing whitespace in patches/files, prefer `git apply --whitespace=fix`.
