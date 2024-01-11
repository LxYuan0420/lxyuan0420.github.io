---
layout: post
title: "Introducing PEP 736: Shorthand syntax for keyword arguments at invocation"
description: "This article describes PEP 736, which proposes the introductions of syntactic sugar `f(x=)` as a replacement for `f(x=x)`"
---

Have you ever felt that calling functions in Python could be a bit more streamlined? If so, you're in for a treat with the introduction of PEP 736. This new Python proposal is all about making your code cleaner and more efficient.

### What's PEP 736?

PEP 736 proposes a shorthand syntax for keyword arguments in function calls. It's a fresh approach to Python coding, aiming to reduce repetition and enhance readability.

### The Core Idea

The proposal suggests a shortcut for function calls with keyword arguments. Typically, you'd write something like function(argument=variable). But, if your variable name matches the argument name, this feels redundant, right? PEP 736's solution: Just write function(argument=). Python will understand that you're passing the variable with the same name as the argument.

Here's a practical example:

{% highlight python %}
def greet(name, age):
    print(f"Hello, {name}! You are {age} years old.")

name = "Alice"
age = 30

# Traditional way:
greet(name=name, age=age)

# With PEP 736:
greet(name=, age=)
{% endhighlight %}

### Why It's a Game-Changer

- Efficiency: Less typing, more doing. Especially handy for variables with lengthy names.
- Readability: Your code becomes cleaner and easier to scan.
- Best Practices: It promotes the use of consistent naming conventions.

### Compatibility and Security

- No Breaks: It's fully backwards compatible. Your existing code won't be affected.
- Secure: Introduces no new security concerns.


### A Personal Perspective
This concept isn't new in programming languages. It's already seen in languages
like Ruby, JavaScript, and Rust, where similar shorthand coding practices
exist.

At first glance, PEP 736's syntax may appear somewhat unusual and unnecessary.
It's understandable to think that way; after all, change in coding practices
often feels jarring. However, the more you see and use this new syntax, the
neater it begins to feel. It's a bit like getting used to f-strings with = in
Python, such as print(f"{name = }"). What initially seems odd can quickly
become a neat and integral part of your coding style.

PEP 736 is poised to simplify your Python coding experience, making it a
notable enhancement for code readability and efficiency. It may take a moment
to get used to, but its elegance and practicality shine through with use.

Related:
- [PEP 736 – Shorthand syntax for keyword arguments at invocation](https://peps.python.org/pep-0736/)
