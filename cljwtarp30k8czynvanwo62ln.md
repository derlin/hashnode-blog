---
title: "When plans go astray: my unsuccessful journey of migrating a large Django project to Mypy"
seoDescription: "Learn about my failed attempt at migrating a large Django project to Mypy. We'll talk typed dicts, generics, peps, and more."
datePublished: Mon Jul 10 2023 12:00:39 GMT+0000 (Coordinated Universal Time)
cuid: cljwtarp30k8czynvanwo62ln
slug: my-unsuccessful-journey-of-migrating-a-large-django-project-to-mypy
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1688715670244/cc3f5dfd-152c-450c-b49b-ef6dc6ce5597.jpeg
tags: python, django, type-checking, mypy

---

**TL;DR** - Mypy is amazing, but your code needs to be ready for it. The untyped nature of Python allows magic to happen and shortcuts to be made (at the cost of more runtime errors). Adding a type checker such as Mypy to a codebase developed under this freedom is impossible without massive rewrites. This is at least the lesson I learned the hard way when trying to migrate a Django application (with years in production) to Mypy.

---

* [A brief history of type hints/checks in Python](#heading-a-brief-history-of-type-hintschecks-in-python)
    
* [The context](#heading-the-context)
    
* [The plan](#heading-the-plan)
    
    * [Setting up Mypy](#heading-setting-up-mypy)
        
    * [Ignoring old code: mypy-clean-slate](#heading-ignoring-old-code-mypy-clean-slate)
        
    * [Adding types](#heading-adding-types)
        
* [Some of the problems with Mypy](#heading-some-of-the-problems-with-mypy)
    
    * [Libraries without exposed pyi types (and other perks)](#heading-libraries-without-exposed-pyi-types-and-other-perks)
        
    * [TypedDict limitations](#heading-typeddict-limitations)
        
    * [Optionals and assert](#heading-optionals-and-assert)
        
    * [Mixin classes: protocols needed](#heading-mixin-classes-protocols-needed)
        
    * [Django Models - generics not supported](#heading-django-models-generics-not-supported)
        
* [Conclusion](#heading-conclusion)
    

*(TOC generated with* [*bitdowntoc*](https://derlin.github.io/bitdowntoc/) *using the Gitlab preset and the anchor prefix option set to* `heading-`*)*

---

## A brief history of type hints/checks in Python

Python type hints were introduced with [**PEP 484**](https://www.python.org/dev/peps/pep-0484/) in Python 3.5, which proposed a syntax for adding type annotations to function signatures and variable declarations. This marked the beginning of static typing support in Python. Those annotations, however, were never checked: neither at build nor at runtime.

To address this, Jukka Lehtosalo released **Mypy**, an external static type checker providing static analysis and type inference based on PEP 484 annotations.

Subsequent PEPs such as [**PEP 526**](https://www.python.org/dev/peps/pep-0526/) (variable annotations), [**PEP 563**](https://www.python.org/dev/peps/pep-0563/) (postponed evaluation of type annotations), and [**PEP 585**](https://www.python.org/dev/peps/pep-0585/) (behavior of built-in types) further refined the type hinting capabilities. Collectively, these PEPs contributed to the growth and adoption of type hints in Python, with Mypy serving as a widely used tool for static type checking.

Mypy is not the only one in the game though. To cite a few:

1. [**Pyright**](https://github.com/microsoft/pyright) (Microsoft): A fast and type-aware static type checker designed for efficient type checking and editor support.
    
2. [**Pyre**](https://github.com/facebook/pyre-check) (Facebook): A static type checker that focuses on precise type inference and analysis.
    
3. [**PyType**](https://github.com/google/pytype) (Google): A static type analyzer for large codebases, supposed to be faster than Mypy.
    

## The context

The combination of type hints and type checkers is a revolution in the Python world, offering advantages such as early error detection, improved code quality, enhanced IDE support, better collaboration, stronger codebase resilience, and more. Who wouldn't want that?

I have been using type hints since their introduction back in 2015 when I code for fun, but never transitioned to type checking (I tried Mypy once on one of my personal projects, and discovered many of my hints were factually wrong 🤪).

Last month, my company decided to give Mypy a shot. The goal was to start with one of our smallest Django applications (in production for years) to validate the process before moving the rest of the codebase.

This article outlines the steps we took and explains why we ultimately deemed the experiment a failure, at least for the time being.

## The plan

### Setting up Mypy

To start with Mypy, we have to install it. And since Django uses some Python "magic" that makes having precise types for some code patterns problematic (e.g. ORM classes), we also need [django-stubs](https://github.com/typeddjango/django-stubs), a third-party package that "*provides* [*type stubs*](https://www.python.org/dev/peps/pep-0561/) *and a custom Mypy plugin to provide more precise static types and type inference for the Django framework*":

```bash
pip install mypy django-stubs[compatible-mypy]
```

MyPy can be configured in many ways, one of which being `setup.cfg`. Let's add the relevant section:

```ini
[mypy]
# ↓ For mypy
python_version = 3.9
mypy_path = .

# ↓ For Django-stubs
plugins =
    mypy_django_plugin.main

[mypy.plugins.django-stubs]
django_settings_module = settings
```

With this, we can now run:

```bash
# at the root of the project
mypy . --strict
```

As expected, a zillion errors pop up, because none of the code contains type hints.

### Ignoring old code: mypy-clean-slate

From [mypy-clean-slate](https://pypi.org/project/mypy-clean-slate/)'s README:

> It can be difficult to get a large project to the point where `mypy --strict` can be run on it. Rather than incrementally increasing the severity of mypy, either overall or per module, `mypy_clean_slate` enables one to ignore all previous errors so that `mypy --strict` (or similar) can be used almost immediately. This enables all code written from that point on to be checked with `mypy --strict` (or whichever flags are preferred), gradually removing the `type: ignore` comments from that point onwards.

Running mypy-clean-slate adds `type: ignore[<rule>]` comments on every line throwing an error so that Mypy doesn't complain. This lets us add types incrementally to the existing code and still enforce types on the new code. Here is an excerpt of the result:

```python
from rest_framework import serializers  # type: ignore[import]

class FooSerializer(serializers.Serializer):  # type: ignore[misc]
   def get_queryset(self):  # type: ignore[no-untyped-def]
        return construct_qs(self)  # type: ignore[no-untyped-call]
```

### Adding types

Now that the project is set up, we can start removing type ignore comments and work with types on new features (note that Mypy is smart enough to tell us when an ignore comment is obsolete 😃). And this is where things get complicated.

## Some of the problems with Mypy

After the initial setup, I tried to follow strict type-checking for a new feature. Here are some of the difficulties I faced.

### Libraries without exposed pyi types (and other perks)

***TL;DR*** *Some libraries use non-exposed pyi types, making it difficult (sometimes impossible?) to type their usage properly. The extensive use of* `*args, **kwargs` *is also problematic.*

Take this piece of code:

```python
import tempfile

def temporary_file(*args, named=False, **kwargs):
    cls = tempfile.NamedTemporaryFile if named else tempfile.TemporaryFile
    return cls(*args, **kwargs)
```

First, this is impossible to type without rewriting it so that the returned class is clear:

```python
import tempfile

def temporary_file(*args, named=False, **kwargs):
    if named:
       return tempfile.NamedTemporaryFile(*args, **kwargs)
    return tempfile.TemporaryFile(*args, **kwargs)
```

The `tempfile` library is written in C and has a `pyi` file (defining types) with the following signatures:

```python
def TemporaryFile(...) -> _TemporaryFileWrapper[str]:
   ...
def NamedTemporaryFile(...) -> _TemporaryFileWrapper[str]:
   ...
```

`_TemporaryFileWrapper[str]` is however not exposed!

I had to use the `typing.IO` type instead, which is a generic. Given the signature, I naturally went for `IO[str]`, but discovered this doesn't work on code using bytes:

```python
from typing import IO

def temporary_file(...) -> typing.IO[str]: # ...

with temporary_file("foo") as f:
   f.write("string") # <- OK
   f.write(b"bytes") # <- error!
```

I finally settled on an union type: `IO[Union[str,bytes]]`.

Another problem arises with the `*args, *kwargs` in the signature. But I don't want to have to type them, as I don't know (and don't care) what type they are! I can either leave the *ignore* comment or use `Any`. (Note: see [PEP 692](https://peps.python.org/pep-0692/) for typed kwargs).

### TypedDict limitations

***TL;DR*** *TypedDict values can only be accessed with string literals.*

When using or returning a `dict`, a good practice is to type it properly using a `TypedDict`:

```python
FooDict = TypedDict(
    "FooDict",
    {
        "ok": bool,
        "started_at": str,
        "ended_at": str,
        "size": int,
    },
)
```

This `FooType` can then be used like this:

```python
def some_func() -> FooDict:
   return {
       "ok": True,
       "started_at": "2023-06-04T10:00",
       "ended_at": "2023-06-04T10:01",
       "size": 123,
   }
```

All good. But did you know that one can only access a `TypedDict` value using *string literals* (that is, plain strings)? This means it becomes impossible to have something like:

```python
import pytest

def test_some_funct():
    result = some_func()
    assert result["size"] == 123 # ok
    for key in ["started_at", "ended_at"]:
       assert result[key] # <- FAILS!
```

The last line yields:

```python
foo_test.py:7: error: TypedDict key must be a string literal;
    expected one of ("ok", "started_at", "taken_at", "ended_at", "size")
    [literal-required]
```

This was a bummer. What if in some cases the key is received as an argument? An *ignore* comment will be required.

### Optionals and assert

***TL;DR*** *Optional types require explicit None checks, adding many lines of (unnecessary?) assert.*

As explained in the docs (see [Optional types and the None type](https://mypy.readthedocs.io/en/stable/kinds_of_types.html#optional-types-and-the-none-type)), an `Optional` variable requires an explicit `None` check. That is:

```python
def wrong(x: Optional[int]):
    return x + 1 # <- this is NOT allowed

def ok_if(x: Optional[int]):
    if x is not None:
        return x + 1 # <- ok

def ok_assert(x: Optional[int]):
    assert x is not None
    return x + 1 # <- ok
```

When using Django, you may have model fields that are defined with `null=True` and `blank=True`. This maps to an `Optional` field, thus every time you need to access their value, even if you know from the logic that it *can't* be `None`, you need an `assert` or an `if` statement.

```python
from django.db import models

class Student(models.Model):
    created_at = models.DateTimeField(null=True, blank=True)
    updated_at = models.DateTimeField(null=True, blank=True)
    nickname = models.CharField(null=True, blank=True)
    # ...

   def do_something_when_nicknamed(self) -> None:
       assert self.created_at
       assert self.updated_at
       assert self.nickname

       # ... finally do something ...
```

On the code under migration, this restriction adds many, many lines of assert everywhere!

### Mixin classes: protocols needed

***TL;DR*** *The only way to properly use mixin classes is to define protocols for each, multiplicating the classes used. Better not to use mixins at all if used only once.*

Our legacy code relies heavily on mixin classes.

> mixin classes are small classes that provide attributes but are not included in the standard inheritance tree, working more as "additions" to the current class than as proper ancestors. Mixins originate in the LISP programming language.

In other words, so-called mixin classes are small classes inheriting from `object` that group operations together. They are very useful to split large classes into smaller, more manageable chunks - or to add features to classes (composition).

Python mixins are not a dedicated language feature, they simply hijack the multiple inheritance mechanism. Hence, while mixins should, in theory, be orthogonal (independent of each other), they usually rely on other pre-existing features and attributes of the augmented class.

*(If you are not familiar with mixin classes, I recommend the article* [*Multiple inheritance and mixin classes in Python*](https://www.thedigitalcatonline.com/blog/2020/03/27/mixin-classes-in-python/) *from The Digital Cat* 😉).

Here is a simple example of the mixin pattern:

```python
class Graphic:
    def __init__(self, pos_x, pos_y, size_x, size_y):
        self.pos_x = pos_x
        self.pos_y = pos_y
        self.size_x = size_x
        self.size_y = size_y

class ResizableMixin:
    def resize(self, sx, sy):
        self.size_x = sx # <- UNKNOWN !
        self.size_y = sy # <- UNKNOWN !

class ResizableGraphic(Graphic, ResizableMixin):
    pass
```

The problem is, Mypy doesn't know how each mixin is supposed to be used, and with which other classes it will always be composed. Every line marked `UNKNOWN` above will thus throw an error.

There is a solution though: implementing [protocols](https://mypy.readthedocs.io/en/stable/more_types.html#mixin-classes). A protocol is yet another class that defines the properties a mixin recipient should have:

```python
from typing import Protocol 

class Sizeable(Protocol):
    @property
    def size_x(self) -> int: ... # the dots are part of the syntax
    @property
    def size_y(self) -> int: ...

class ResizableMixin:
    def resize(self: Sizeable, sx: int, sy: int) -> None:
        self.size_x = sx
        self.size_y = sy

class ResizableGraphic(Graphic, ResizableMixin):
    pass
```

It works but at the cost of adding dozens of classes to a code that already got too much! In other words, given the initial codebase, adding protocols would be too verbose and inefficient: better to get rid of the mixins pattern altogether.

### Django Models - generics not supported

***TL;DR*** *Django doesn't allow model classes to extend* `Generic`*. The only way to properly type generic model classes is to use a very verbose and ugly "hack".*

Our codebase relies heavily on generics. For example, let's say I store files in different backends. I could have something like this:

```python
# -- In base.models
from django.db import models

class BaseStorage(models.Model):
    class Meta:
        abstract = True

    name = models.CharField()
    backend = None # Foreign key to a BackendBase model
    file_model = None # File class, subclassing BaseFile
    # ...

# -- In postgres.models
class PostgresBackend(BaseBackend):
    pass
class PostgresFile(BaseFile):
    pass

class PostgresStorage(BaseStorage):
    backend = models.ForeignKey(PostgresBackend)
    file_model = PostgresFile
```

To add types to this pattern, I can use `typing.Generic` and `typing.TypeVar`. This is very similar to Java:

```python
# -- In base.models
from django.db import models
from typing import Type, TypeVar, Generic

# bound reads "B must be a subclass of BaseBackend"
B = TypeVar("B", bound=BaseBackend)
F = TypeVar("F", bound=BaseFile)

class StorageBase(models.Model, Generic[B, F]):
    # ...
    backend: "models.ForeignKey[Union[B, Combinable],B]"
    file_model: Type[F]

# -- In postgres.models
class PostgresStorage("BaseStorage[PostgresBackend, PostgresFile]"):
    pass
```

Notice the quotes around some of the types. When unquoted, python tries to call the method associated with the operator "`[]`", which on class objects is `__class_getitem__`. Neither `models.ForeignKey` nor `BaseStorage` implements it, thus throwing an error. Quotes make it clear it is only a type hint.

Notice also the strange type of the `backend` attribute. It took me a while to figure it out! If the foreign key field were nullable, it would be:

```python
models.ForeignKey[Union[B, Combinable, None],Optional[B]]
```

This code is already quite complex compared to the original, but it replaces the comments explaining each type. So not too bad. There is a problem though. If you try to run Django migrations (`manage.py migrate`), a lengthy stacktrace awaits you:

```python
Traceback (most recent call last):
  File "...django/core/management/__init__.py", line 419, in execute_from_command_line
    utility.execute()
  [...]
  File "/usr/lib/python3.9/typing.py", line 1010, in __init_subclass__
    raise TypeError("Cannot inherit from plain Generic")
TypeError: Cannot inherit from plain Generic
```

You read properly, **Django DOES NOT SUPPORT GENERIC on model classes**! There is an open issue from October 2021, but it is marked as [wontfix](https://code.djangoproject.com/ticket/33174).

Mypy provides a solution though: using `typing.TYPE_CHECKING` - a special constant assumed to be `True` by static type checkers, but always `False` at runtime - to conditionally inherit from `Generic` in model classes.

This is how it looks for the base class:

```python
from django.db import models
from typing import TYPE_CHECKING, Type, TypeVar, Generic

B = TypeVar("B", bound=BaseBackend)
F = TypeVar("F", bound=BaseFile)

if TYPE_CHECKING:
    class _Parent(Generic[B, F]):
        pass
else:
    class _Parent:
        # Implementing this method is necessary to be able to use
        # the _Parent[...] (brackets) syntax at runtime
        def __class_getitem__(cls, _):
            return cls

class StorageBase(models.Model, _Parent[B, F]):
    # ...
    backend: "models.ForeignKey[Union[B, Combinable],B]"
    file_model: Type[F]
```

Note the `__class_getitem__` . It is necessary because when types are on, the `_Parent` becomes a *generic* class as well, thus needing to be typed with `[]`. To use the brackets notation, the parent class needs to implement it.

The subclass is easier:

```python
# In postgres.models
if TYPE_CHECKING:
    class _Parent("BaseStorage[PostgresBackend, PostgresFile]"):
        pass
else:
    class _Parent():
        pass

class PostgresStorage(_Parent):
    # ...
```

This version will pass the Mypy checks and won't make Django complain during migrations. But gosh, it is verbose! Our initial codebase has more than thirty classes based on genericity, which makes for a lot of `_Parent` classes. And PEP 8 defines two blank lines around every class declaration, making the if/else above span 12 lines!

## Conclusion

Due to the limitations outlined above, we decided to revert the migration to Mypy, at least for the time being. While types can enhance code understanding and error detection, the additional complexity introduced to our already large codebase outweighed these benefits. We could have stayed with a partial migration (i.e. kept numerous ignore comments), but it would have defeated the purpose.

The key lesson I learned is that **a Python project should be developed with type checks in mind from the start**. Migrating a codebase written in untyped Python introduces numerous edge cases and complexities that type checkers struggle to handle without substantial refactoring. I can't help but feel the types revolution will drastically change how we code in Python.

It is important to stress that part of the challenges I faced may be attributed to the relatively recent adoption of type-checking in Python. I remain hopeful that future developments will include support for Django generics and offer improved alternatives to typed dicts and mixins. In the meantime, I will continue to utilize Mypy for my personal projects!

Thank you for reading, and let me know if you had another (more successful?) experience!

With love, derlin.