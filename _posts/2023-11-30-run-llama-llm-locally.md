---
layout: post
title: "llamafile: Run Llama LLM Locally"
description: "This article describe how to run a LLM on your own computing using llamafile"
---

Ever wanted to run a big language model like ChatGPT on your computer? Thanks
to Mozilla and Justine Tunney, now you can with something called "llamafile".

llamafile is a multi-gigabyte file containing the model weights and code for an
LLM, and in some cases, includes a full local server with a web UI. The
executable is made using Cosmopolitan Libc, enabling it to work across various
operating systems and architectures.

To use llamafile with the LLaVA 1.5 model, one can download a specific file
from Justine Tunney's repository, make the binary executable, and run it to
start a web server for interaction.

### How to Use Llamafile

{% highlight bash %}
$ curl -LO https://huggingface.co/jartine/llava-v1.5-7B-GGUF/resolve/main/llamafile-server-0.1-llava-v1.5-7b-q4
$ chmod 755 llamafile-server-0.1-llava-v1.5-7b-q4
$ ./llamafile-server-0.1-llava-v1.5-7b-q4
{% endhighlight %}

Finally, navigate to http://127.0.0.1:8080/ in your browser to start interacting with the model.

It's that simple! You don't even need the internet. Special thanks to Simon Willison for this easy guide.


### Screenshots 

{% include image.html path="documentation/llamafile_image.png" path-detail="documentation/llamafile_image.png" alt="Example output vision" %}
{% include image.html path="documentation/llamafile_text.png" path-detail="documentation/llamafile_text.png" alt="Example output text" %}

Related:
- [llamafile is the new best way to run a LLM on your own computer](https://simonwillison.net/2023/Nov/29/llamafile/)
- [Introducing llamafile](https://hacks.mozilla.org/2023/11/introducing-llamafile/)
