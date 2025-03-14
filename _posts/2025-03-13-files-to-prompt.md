---
layout: post  
title: "TIL: Using `files-to-prompt` to Concatenate Files for LLMs"  
description: "Concatenate a directory full of files into a single prompt for use with LLMs"  
---

Today, I learned about the `files-to-prompt` Python tool, which helps you
concatenate files from a directory into a single prompt for LLMs like GPT or
Claude. I particularly found it useful for processing Python files (`.py`) and
formatting them in Markdown for further analysis.

##### Key Features:

- **Concatenate Python Files**: Easily concatenate all Python files in a directory.
- **Markdown Output**: Output the concatenated files in Markdown format.
- **Custom Output File**: You can specify the output file where the concatenated content is saved.

### Python-Specific Command Examples

{% highlight bash %}
# 1. Concatenate all Python files in a directory and output in Markdown format
# This will only include `.py` files and format them as Markdown.
files-to-prompt path/to/directory -e py -m

# 2. Save the concatenated Python files into a specific output file
# This command will save the output into `python_files.md`.
files-to-prompt path/to/directory -e py -m -o python_files.md

# 3. Use pbcopy to copy the concatenated Python files to clipboard
# This will copy the concatenated Python files into the clipboard for easy pasting.
# for example, paste the content to chatgpt
files-to-prompt path/to/directory -e py | pbcopy
{% endhighlight %}


Hope this helps make your experience with LLM and terminal in a large project more efficient and productive.

Reference: [files-to-prompt](https://github.com/simonw/files-to-prompt/tree/main)
