---
layout: post
title: "TIL: Better F-string expression with python 3.12"
description: "This article describe new updates of f-string on python3.12"
---

TIL that python3.12 (PEP701) will remove some restrictions on the usage of
f-strings, such as, reusing the same quotes as the containing f-string,
allowing usage of backslashes and unicode escaped sequences inside f-strings.

Personally I prefer using double quotes `" "` for all my python strings.
However, before python 3.12, we are not allowed to use the same quotes inside a
f-string:

{% highlight python %}

# before: python 3.10 
# bad: can't use the same double quote inside the curly bracket
>>> names = ["Alex", "Bob", "Cat"]
>>> f"This is my name list: {" - ".join(names)}"
  File "<stdin>", line 1
    f"This is my name list: {" - ".join(names)}"
                               ^
SyntaxError: f-string: expecting '}'

# ok: replace with single quote
>>> f"This is my name list: {' - '.join(names)}"
'This is my name list: Alex - Bob - Cat'


# after: python 3.12 
# ok: to use same quote
>>> f"This is my name list: {" - ".join(names)}"
'This is my name list: Alex - Bob - Cat'

{% endhighlight %}

This also means that, before this changes, f-strings can be nested only if we
are using different string quotes, like:

{% highlight python %}
# before: python 3.10
# bad: can't use the same quote
>>> f"{f"{f"{f"{f"{f"{1+1}"}"}"}"}"}"
  File "<stdin>", line 1
    f"{f"{f"{f"{f"{f"{1+1}"}"}"}"}"}"
         ^
SyntaxError: f-string: expecting '}'

# ok: must use different quotes
>>> f"""{f'''{f'{f"{1+1}"}'}'''}"""
'2'

{% endhighlight %}

Now it is possible to write nested f-strings with the same quotes:
{% highlight python %}

# after: python 3.12
# ok: to use same quotes 
>>> f"{f"{f"{f"{f"{f"{1+1}"}"}"}"}"}"
'2'

{% endhighlight %}

Another thing that I like about python 3.12 f-string is that we can use
`backlashes` and `unicode escaped sequences`. The following example is taken
from the official PEP changelog:

{% highlight python %}
>>> print(f"This is the playlist: {"\n".join(songs)}")
This is the playlist: Take me back to Eden
Alkaline
Ascensionism

>>> print(f"This is the playlist: {"\N{BLACK HEART SUIT}".join(songs)}")
This is the playlist: Take me back to Eden♥Alkaline♥Ascensionism

{% endhighlight %}

Related:
- [What's New In Python 3.12](https://docs.python.org/3.12/whatsnew/3.12.html)
