---
layout: post
title: "TIL: Setting Up Snippets in Vim with UltiSnips and vim-snippets"
description: "Exploring UltiSnips and vim-snippets"
---

Today, I learned setting up snippets for Vim using [UltiSnips](https://github.com/SirVer/ultisnips) and [vim-snippets](https://github.com/honza/vim-snippets). These plugins have drastically improved my coding efficiency, especially for C++ and Python. Here’s a quick guide on how I did it.

### Why Use UltiSnips and vim-snippets?

- **UltiSnips** is a powerful snippet engine for Vim, enabling you to create and expand custom code snippets quickly.
- **vim-snippets** provides a collection of pre-made snippets for various programming languages, working seamlessly with UltiSnips to boost productivity.

### Step 1: Installing UltiSnips and vim-snippets Using Vim-Plug

First, I installed both plugins using **Vim-Plug**. If you don’t have Vim-Plug set up, add the following to your `.vimrc`:


{% highlight bash %}
call plug#begin('~/.vim/plugged')
Plug 'SirVer/ultisnips'
Plug 'honza/vim-snippets'
call plug#end()
{% endhighlight %}

After adding the plugins, run `:PlugInstall` in Vim to install them.

### Step 2: Configure UltiSnips

I configured UltiSnips to use convenient keybindings for expanding and navigating snippets. Here’s what I added to my `.vimrc`:


{% highlight bash %}
let g:UltiSnipsExpandTrigger="<tab>"
let g:UltiSnipsJumpForwardTrigger="<c-b>"
let g:UltiSnipsJumpBackwardTrigger="<c-z>"
- `<tab>`: Expands the snippet.
- `<c-b>`: Jumps forward to the next placeholder in the snippet.
- `<c-z>`: Jumps backward to the previous placeholder.
{% endhighlight %}

### Step 3: Creating a Custom C++ Snippet

Next, I created a custom snippet for my C++ workflow. I added it in the file `~/.vim/UltiSnips/cpp.snippets`:

{% highlight bash %}
snippet main
#include <iostream>

int main()
{
    ${1:return 0;}
}
endsnippet
{% endhighlight %}


- The **`${1:return 0;}`** part sets a **placeholder** at the cursor’s initial position when the snippet expands.
- You can overwrite the default content (`return 0;`) by typing and pressing `<c-b>` to jump to the next tab stop.

### Step 4: Testing the Snippet

When I type `main` and press `<tab>` in a C++ file, it expands to:

{% highlight c++ %}
#include <iostream>

int main()
{
    return 0;
}
{% endhighlight %}

The cursor lands on the placeholder (`return 0;`), letting me modify the code quickly.

### 5. (Optional) Install Additional Snippets via vim-snippets
If you want a rich collection of community-maintained snippets for multiple languages, the vim-snippets repository (already installed in Step 2) offers a variety of pre-made snippets.

Note: You can find these snippets in ~/.vim/plugged/vim-snippets/snippets/. You can customize them further or add your own!

### 6. Reload Vim and Test Your Snippet
Reload your Vim configuration or restart Vim to ensure everything is set up:

{% highlight bash %} source ~/.vimrc {% endhighlight %}

Now, open a new .cpp file, type main, and hit <tab> to expand your snippet!


### Summary

Setting up UltiSnips and vim-snippets was straightforward and significantly boosted my coding speed in Vim. By creating custom snippets, I tailored my workflow even further. I recommend these plugins to anyone looking to streamline repetitive code patterns.

**References:**
- [UltiSnips on GitHub](https://github.com/SirVer/ultisnips)
- [vim-snippets on GitHub](https://github.com/honza/vim-snippets)

