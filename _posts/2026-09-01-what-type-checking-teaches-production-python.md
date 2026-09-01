---
layout: post
title: "What if TYPE_CHECKING Actually Teaches You About Production Python"
description: "TYPE_CHECKING is more than a circular-import workaround. It makes Python's runtime and static dependency graphs visible."
---

I recently came across Vicki Boykis’s article [Why if TYPE_CHECKING?](https://vickiboykis.com/2023/12/11/why-if-type_checking/). What caught my attention was not the syntax itself. I had seen this pattern many times in production Python code:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from myapp.models import User
```

The usual explanation is that this helps avoid circular imports. That is true, but it feels incomplete. Why does putting an import behind an `if` statement help? If `User` is needed by the code, why is it safe not to import it? And why can the type checker still understand the code when Python never runs that branch?

The answer reveals something important about Python: the same source code is read by two different systems.

```text
                         Python source
                             |
                +------------+------------+
                |                         |
                v                         v
          Type checker                CPython runtime
          mypy / pyright
                |                         |
        understands types            executes objects
        and relationships            and imports
```

The dependency graph needed to understand a Python program does not always need to be the same as the graph needed to run it. `TYPE_CHECKING` lets us express that difference.

## Imports are executable code

An import is not just a declaration. It executes code.

When Python sees:

```python
from users import User
```

it finds the `users` module, creates or reuses a module object, executes the module’s top-level code, and then binds `User` in the current namespace.

This is why circular imports are usually execution-order problems.

Imagine two modules.

`user.py`:

```python
from __future__ import annotations

from order import Order


class User:
    def latest_order(self) -> Order:
        ...
```

`order.py`:

```python
from __future__ import annotations

from user import User


class Order:
    def owner(self) -> User:
        ...
```

The import sequence looks like this:

```text
import user
    |
    v
execute user.py
    |
    v
from order import Order
    |
    v
execute order.py
    |
    v
from user import User
    |
    v
user.py is only partially initialized
```

Python has created the `user` module object, but it has not reached the `User` class yet. `order.py` asks for an object that does not exist in that module yet.

A type checker has a different job. It does not need to execute the modules, construct the classes, connect to a database, or run import-time setup. It only needs to understand that `User` and `Order` exist and that the methods refer to them.

## What `TYPE_CHECKING` does

`TYPE_CHECKING` is a constant from the `typing` module. It is `False` at runtime, but static type checkers treat it as `True`.

```python
from typing import TYPE_CHECKING

print(TYPE_CHECKING)
# False
```

So this code has two interpretations:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from order import Order
```

```text
CPython                         type checker
TYPE_CHECKING = False           TYPE_CHECKING = True
        |                               |
        v                               v
skip the import                 analyze Order
```

That is why “it avoids circular imports” is too narrow. `TYPE_CHECKING` removes a dependency from the runtime graph while keeping it in the static type graph.

```text
Runtime graph:                  Type graph:
Service ----> Database          Service ----> Database
                                      |
                                      +----> User
```

Some dependencies are needed because the running program uses the object. Others exist only so that type checkers and IDEs can describe the program more precisely.

## Who actually needs the dependency?

Suppose we have:

```python
class UserService:
    def save(self, user: User) -> None:
        self.repository.save(user)
```

Why is `User` mentioned here?

- The function might construct a `User`.
- The function might call methods on the `User` class.
- A framework might inspect the annotation at runtime.
- The name might exist only for the type checker and IDE.

These cases have different dependency requirements.

A useful test is to erase the annotations mentally:

```python
def save(self, user):
    self.repository.save(user)
```

Does the module still need to import `User` to execute correctly?

- If yes, `User` is a runtime dependency and should usually be imported normally.
- If no, and it is needed only for typing, the import may belong behind `TYPE_CHECKING`.

This small distinction helps keep larger systems easier to maintain. A module should not import every object it knows about if it does not need those objects to run.

## Why this matters beyond circular imports

Imports can be expensive. A large ML library may load native extensions, inspect hardware, register plugins, or import hundreds of other modules. If a module only mentions `giant_ml_library.Model` in an annotation, importing the whole library at application startup may be unnecessary.

The same applies to optional dependencies. If a package has optional Pandas support, a top-level `import pandas` makes Pandas part of the package’s import-time requirements. Users who do not use that feature may now get an import error simply by importing the package.

A type-only import can keep Pandas out of the base runtime dependency set. The type checker still needs Pandas or its stubs available in the analysis environment, but normal users do not need it just to import the package.

This is why `TYPE_CHECKING` is relevant to production engineering. It can reduce import-time coupling, protect optional dependency boundaries, and make runtime dependencies easier to see.

## Annotations can still matter at runtime

There is an important exception. An annotation may not matter to the function body, but another library may inspect it.

```python
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from models import User


def get_name(user: User) -> str:
    return user.name
```

When `get_name` runs, Python only needs an object with a `.name` attribute. It does not need the `User` class just to execute the function.

But frameworks may inspect annotations with tools such as `typing.get_type_hints()`. FastAPI, Pydantic, dependency-injection frameworks, serializers, ORMs, and custom decorators may use annotations at runtime.

That gives us three useful dependency graphs:

1. Runtime execution: what must exist for the code to run?
2. Runtime reflection: what must exist because a framework inspects annotations?
3. Static analysis: what must the type checker understand?

So blindly moving every annotation-related import behind `TYPE_CHECKING` can break a program even when mypy or pyright is happy. Always ask who consumes the dependency.

| Consumer | Dependency requirement |
| --- | --- |
| Function body | Runtime |
| Object construction | Runtime |
| Framework reflection | Runtime |
| mypy / pyright | Static analysis only |

## Forward references solve a different problem

`TYPE_CHECKING` is often paired with a quoted annotation:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from order import Order


def process(order: "Order") -> None:
    ...
```

These two features solve different problems:

- `TYPE_CHECKING` controls whether the import runs.
- The quoted `"Order"` prevents Python from immediately looking up `Order` while defining the function.

Before Python 3.14, `from __future__ import annotations` could also postpone annotation evaluation by storing annotations as strings. Starting with Python 3.14, annotations are lazily evaluated by default. These mechanisms reduce the need to quote forward references, but they do not change the main design question: does this name need to exist at runtime, or only during static analysis?

## `TYPE_CHECKING` can reveal architecture problems

Sometimes two modules refer to each other only in their type annotations:

```text
user.py  - - -type- - -> order.py
order.py - - -type- - -> user.py
```

That is a good use for `TYPE_CHECKING`, because the runtime dependency may not really exist.

But sometimes the cycle is real:

```text
user.py  ---> creates Order
   ^                  |
   |                  v
   +------ creates User <--- order.py
```

If both modules genuinely need each other during execution, hiding one import does not fix the architecture. The dependency graph itself needs to change.

Possible refactors include extracting shared concepts into a lower-level module:

```text
 user.py          order.py
    \               /
     \             /
      v           v
       domain_types.py
```

Another option is to depend on a `Protocol` or interface instead of a concrete implementation:

```text
service.py -----> protocol.py <----- repository.py
```

If `TYPE_CHECKING` fixes the problem, great. But if two modules really need each other at runtime, hiding one import probably is not the right fix. I’d rather rethink the module boundary than hide the dependency.

## The mental model I would keep

The simplest explanation is still:

> `TYPE_CHECKING` helps avoid circular imports caused by type annotations.

But the more useful explanation is:

> `TYPE_CHECKING` lets Python’s static dependency graph differ from its runtime dependency graph.

When I see this:

```python
if TYPE_CHECKING:
    from users import User
```

I no longer read it only as a trick to suppress a circular import. I read it as a design statement:

> `User` is part of the conceptual model of this module, but it is not part of this module’s runtime execution path.

That is a small piece of syntax carrying an architectural idea.

Once a Python codebase becomes large enough, production engineering is largely about deciding which dependencies need to exist, where they should exist, and when they should become real. `TYPE_CHECKING` is one place where Python makes that distinction visible.
