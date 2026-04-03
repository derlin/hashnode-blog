---
title: "TODO title"
slug: todo-title

---

Not long ago, I came across a bug in one of our projects that highlighted very interesting changes in how Python handles type hints at runtime.

## The bug

We have an API written with django-rest-framework (drf) which use [drf-yasg](https://github.com/axnsan12/drf-yasg) to generate its OpenAPI schema at runtime. You don't need to know the intricacies of those libraries, just that API endpoints return serializers, and drf-yasg inspects them at runtime to generate a schema with the proper fields and types.

Now, the problem arose with a snippet similar to this:

```python
from rest_framework import serializers

class MySerializer(serializers.Serializer):
   # This allows to return something that is not a direct
   # attribute of a model, but a "computed" value (function)
   is_foo = serializers.SerializerMethodField()

   # The type hint (-> bool) is used by drf-yasg to
   # guess the proper type is "is_foo"
   def get_is_foo(self, obj) -> bool:
      return obj.is_foo
```

Here, drf-yasg determines the type of `is_foo` based on the type hints of `get_is_foo`. The annotation says `bool`, so this converts correctly to the OpenAPI schema `{"is_foo": "BOOLEAN"}`.

However, upon refactoring, we added the following at the top of the file:

```python
from __future__ import annotations
```

Suddenly, the schema became `{"is_foo": "STRING"}`. What?? (Note that it took us a while to pinpoint that the problem came from this import, but let's keep the story short).

If you understand why, you can stop here. If, like me, you find this utterly confusing and want to understand what is going on, keep reading!

## A brief history of type hints

In 2006, Python 3 introduced a syntax for adding *arbitrary* metadata annotations to Python functions ([PEP 3107: Function Annotations](https://peps.python.org/pep-3107)). (Ab)used by many 3rd parties in different ways, until Python 3.5 finally formalized the semantics in [PEP 484: Type Hints](https://peps.python.org/pep-0484/) (Python 3.5, October 2014 - see also [PEP 483 – The Theory of Type Hints](https://peps.python.org/pep-0483/)) and added the `typing` to the standard library.

Over the years, the type hinting system evolved, offering new syntaxes and meta-classes. This is not only drastically changing the way we write Python code, but also the way we ship it to the masses. Since Python 3.7, packages can incorporate type information and stubs files, making the life of IDEs and type checkers such as MyPy, pytype, etc. easier (if you want to know more, read [PEP 561: Distributing and Packaging Type Information](https://peps.python.org/pep-0561/) !)

I wasn't aware of most of the intricacies of type hints until I tried [migrating a large project to mypy](https://blog.derlin.ch/my-unsuccessful-journey-of-migrating-a-large-django-project-to-mypy) (a good read if you are in a similar process), but I assume most of you are now familiar with the following:

```python
from typing import List, Dict, Optional

def foo(a: List[int], b: Dict[str, object], c: int) -> Optiona[None]:
  ...
```

This can now be rewritten without imports, thanks to [PEP 585 – Type Hinting Generics In Standard Collections](https://peps.python.org/pep-0585/) (Python 3.9, March 2019) and [PEP 604: Allow writing union types as X | Y](https://peps.python.org/pep-0604/) (Python 3.10, May 2020):

```python
def foo(a: list[int], b: dict[str, object], c: int) -> str | None:
  ...
```

## Introducing PEP 563, or what lies behind future annotations

If you ever tried using new type hinting features in old Python versions, you may have stumbled upon (and used) the famous *future import*:

```python
from __future__ import annotations

def foo(a: list[int], b: dict[str, object], c: int) -> str | None:
  ...
```

Adding this line at the top of the file makes the snippet work for versions even in Python 3.7. Ever wondered why?

The only thing this import does is turn on [PEP 563: Postponed Evaluation of Type Annotations](https://peps.python.org/pep-0563/):

> This PEP proposes changing function annotations and variable annotations so that they are **no longer evaluated at function definition time**. Instead, they are preserved in `__annotations__` in **string form**.

Treating all annotations as string *at runtime* have clear advantages, including:

* It removes issues with forward references: annotations can now refer to a name that has not yet been defined
    
* It gives a simple way to deal with cyclic references: we can only import the names when running type checks (using if `typing.TYPE_CHECKING: <import>`)
    
* It improves runtime performances, as type hints are no longer executed at module import time, which is computationally expensive
    

This little trick, turning type hints into strings, also explains why we can use new type hints features in old Python versions: even if `str | None` is invalid in Python 3.9 (`TypeError: unsupported operand type(s) for |: 'str' and 'NoneType'`), the string `'str | None'` is perfectly fine.

All those advantages explain why PEP 563 is planned to become the default behaviour soon:

> `from __future__ import annotations` was previously scheduled to become mandatory in Python 3.10, but the Python Steering Council twice decided to delay the change ([announcement for Python 3.10](https://mail.python.org/archives/list/python-dev@python.org/message/CLVXXPQ2T2LQ5MP2Y53VVQFCXYWQJHKZ/); [announcement for Python 3.11](https://mail.python.org/archives/list/python-dev@python.org/message/VIZEBX5EYMSYIJNDBF6DMUMZOCWHARSO/)). No final decision has been made yet.

## The explanation

For a long time, the `inspect` module was the best way to determine a method's return type using annotations. It is what the drf-yasg library uses in [https://github.com/axnsan12/drf-yasg/blob/1.21.7/src/drf\_yasg/inspectors/field.py#L616](https://github.com/axnsan12/drf-yasg/blob/1.21.7/src/drf_yasg/inspectors/field.py#L616):

```python
import inspect

def foo() -> bool:
  return True

sig = inspect.signature(foo)
assert sig.return_annotation == bool
```

This worked perfectly until [PEP 563: Postponed Evaluation of Type Annotations](https://peps.python.org/pep-0563/) came along. With it, all annotations are now `str`, so that was previously the *type* `bool` becomes the *string* `'bool'`! Worse, libraries such as drf-yasg have to support both typed and stringized annotations - at least during the transition period.

## How to get the return type of a function after PEP 563?

Well, the answer varies based on the Python versions you plan to support.

### In Python 3.10 and newer

From the documentation:

> Python 3.10 adds a new function to the standard library: [`inspect.get_annotations()`](https://docs.python.org/3/library/inspect.html#inspect.get_annotations). In Python versions 3.10 and newer, calling this function **is the best practice** for accessing the annotations dict of any object that supports annotations. This function **can** also “un-stringize” stringized annotations for you.

To turn on the "un-stringize" feature, simply pass the parameter `eval_str=True` :

```python
from __future__ import annotations

def foo() -> bool:
  return True

import inspect
# eval_str does the magic
ann = inspect.get_annotations(foo, eval_str=True)
assert ann["return"] == bool
```

Under the hood, this new function looks up the `foo.__annotations__` dictionary, and calls `eval` on values of type `str`.

If you never heard of `__annotations__`, it is a mutable dictionary *"mapping parameter names to the evaluated annotation expression*". It was introduced in Python 3.0 in 2006 (see [PEP 3107 – Function Annotations](https://peps.python.org/pep-3107/)). As its name suggests, `return` is a special entry key reserved for return annotations.

Note that `inspect.signature` is also updated to support `eval_str`, but it just defers the work to `get_annotations()`, so better to call the latter directly.

### In Python 3.9 and older

In older versions, getting the actual type is tricky, even before PEP 563.

First, as explained in [Accessing The Annotations Dict Of An Object In Python 3.9 And Older](https://docs.python.org/3/howto/annotations.html#accessing-the-annotations-dict-of-an-object-in-python-3-9-and-older), just getting the annotation has some flaws. Without going into the details, the safest is to use this code snippet:

```python
def get_return_annotation(func):
  if isinstance(o, type):
    ann = o.__dict__.get('__annotations__', None)
  else:
    ann = getattr(o, '__annotations__', None)
  return (ann or {})["return"]
```

This can return a `type`, a `str`, or `None`.

In the case of a `str`, we may need to "un-stringize" it. Again, the doc gives some pointers in [Manually Un-Stringizing Stringized Annotations](https://docs.python.org/3/howto/annotations.html#manually-un-stringizing-stringized-annotations), but the gist is to use `eval` and see if this works. But to be honest, there are many catches... So what do we do?

From my own experience - that isn't mentioned in the docs - I suggest one of two approaches:

1. Either give a shot at [get-annotations](https://github.com/shawwn/get-annotations). It doesn't have many stars, but seems to backport[`inspect.get_annotations()`](https://docs.python.org/3/library/inspect.html#inspect.get_annotations) to older Python versions quite well.
    

* Or try `typing.get_type_hints`. Python 3.7's doc says "*In addition, forward references encoded as string literals are handled by evaluating them in* `globals` *and* `locals` *namespaces*", which roughly translates to "*if* `str`*, I will try my best to still return a* `type`".
    

## A word of caution

Using `future.annotations` let you use anything as annotations, including features that are not yet available. This is very dangerous if you plan on reading types at runtime.

For example, if you are using `list[str]` or `str | None` in a version that does not yet incorporate them, any attempt to *unstringize* the annotation will raise an exception (e.g. `unsupported operands type(s) for |`).

Thus, the safest (especially for libraries) is to **stick with the features present in the lowest version of Python you support**. In the example above, this means using `typing.List` and `typing.Union` instead before Python 3.9.

## Conclusion

The advent of [PEP 563: Postponed Evaluation of Type Annotations](https://peps.python.org/pep-0563/) will greatly change the way Python code can access types through annotations, as it turns all annotations from `type` to `str` representing types.

Before this PEP becomes the default (Python 3.12?), it can already be activated by importing `__future__.annotations`. Any annotation in the same file is automatically *stringized* before being stored in `__annotations__` dict. If a piece of code needs to get the return type of a function at runtime, it needs to take this new reality into account.

In short, here are the different ways of getting annotations at runtime:

* `inspect.get_annotations(f)` → only available for Python 3.10+, it is the new best practice. It deals properly with *stringized* annotations when `eval_str=True`.
    
* `inspect.signature(f)` → properly get the types, but only supports `eval_str` since Python 3.10.
    
* `f.__annotations__` → dangerous before Python 3.10, due to some quirks. Never access it directly unless you know what you are doing!
    
* `get_annotations.get_annotations(f)` → very small library that offers `eval_str` starting from Python 3.6.
    
* `typing.get_type_hints(f)` → works for nearly al
    

Keep in mind that stringized annotations let you write anything, but as soon as you try to unstringize them, exceptions will be thrown if the annotation doesn't make sense to the Python version you are using.