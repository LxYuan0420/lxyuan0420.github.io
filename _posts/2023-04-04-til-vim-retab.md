---
layout: post
title: "TIL: retab command in vim"
description: "This article describe a new vim command I just discovered."
---

My typical workflow would be to first write my Python functions in JupyterLab,
and finally write them into a Python script using the Jupyter magic command:
`%%writefile -a my_func.py.`

It works pretty well most of the time; however, sometimes I would encounter the error
`TabError: inconsistent use of tabs and spaces in indentation`. If it's just a
small function, I would fix it by rewriting the function. But if today this
error came up again and it occured in multiple functions, I decided to Google
it (couldn't find a good answer from ChatGPT :sleepy:) and finally found this nice
command in Vim: `:retab` to fix the indentation error in my code.

Check this [stackoverflow post](https://vi.stackexchange.com/questions/27219/fixing-tab-and-space-inconsistency) to learn more.

