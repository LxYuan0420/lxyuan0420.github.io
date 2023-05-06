---
layout: post
title: "TIL: vim-pydocstring"
description: "This article introduce the vim plugin for auto generate python docstring."
---

If you're a Python programmer who values thorough documentation practices, you
might find the vim-pydocstring plugin to be a useful addition to your workflow.
This plugin for Vim enables the automatic generation of Python (template)
docstrings, which can save significant time and effort compared to manually
creating them.

To begin using vim-pydocstring, you need to install it as a Vim plugin. It is
recommended that you use a plugin manager like vimplug and add this line to
your `.vimrc` file:

{% highlight bash %}
Plug 'heavenshell/vim-pydocstring', { 'do': 'make install', 'for': 'python' } 
{% endhighlight %}


One of the standout features of vim-pydocstring is its ability to insert both
one-line and multi-line docstrings quickly and easily. By moving the cursor to
a def or class keyword line and typing ":Pydocstring," a docstring template
appears below the line, reducing the time needed to create documentation
significantly. Another way that I prefer to run this plugin is `Ctrl _` (ctrl
button and underscore). I achieved that by adding the following command to my
`.vimrc` file:

{% highlight bash %}
nmap <silent> <C-_> <Plug>(pydocstring)
autocmd FileType python setlocal tabstop=4 shiftwidth=4 smarttab expandtab
{% endhighlight %}

In addition to the core functionality of the plugin, vim-pydocstring allows
customization of various settings to fit your preferences. For instance, you
can adjust the tab and shift widths and choose between different built-in
docstring formats like Sphinx, Numpy, and Google. Personally, I prefer the
numpy style and so I also added the following command to my `.vimrc` file:

{% highlight bash %}
let g:pydocstring_formatter = 'numpy'
{% endhighlight %}


Once it all set, you can start using vim-pydocstring to generate Python
docstrings and enhance the quality of your code documentation.  Try it out to
see if it can make your workflow more efficient!

Below is the `before` and `after` demo:

{% highlight python %}
# before
class Person():
    def __init__(self, name: str):
        self.name = name

    def hello(self) -> str:
        return f"My name is {self.name}"

def group_intro(p1: Person, p2: Person) -> str:
    output = p1.hello() + p2.hello()
    return output
{% endhighlight %}


{% highlight python %}
# after
class Person():
    """Person.
    """

    def __init__(self, name: str):
        """__init__.

        Parameters
        ----------
        name : str
            name
        """
        self.name = name

    def hello(self) -> str:
        """hello.

        Parameters
        ----------

        Returns
        -------
        str

        """
        return f"My name is {self.name}"

def group_intro(p1: Person, p2: Person) -> str:
    """group_intro.

    Parameters
    ----------
    p1 : Person
        p1
    p2 : Person
        p2

    Returns
    -------
    str

    """
    output = p1.hello() + p2.hello()
    return output
{% endhighlight %}
