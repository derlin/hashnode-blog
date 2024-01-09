---
title: "Diving Deeper into Python Exceptions"
seoDescription: "Discuss some (advanced) concepts and learn tricks about Python exceptions, such as exception chains, __context__, warnings, and more."
datePublished: Tue Jan 09 2024 13:30:42 GMT+0000 (Coordinated Universal Time)
cuid: clr6e3gkp000108jq7bsd3mu6
slug: diving-deeper-into-python-exceptions
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1704646681410/c84bd44f-71f3-4ceb-abf0-bc7f630f2f45.png
tags: programming-blogs, python, python3, programming-tips

---

I have been coding in Python for a long time, yet I am puzzled by how little I knew about `Exception`s. This post is about some of my recent findings on this topic.

---

**Content**

* [A little story](#heading-a-little-story)
    
* [Exception chaining (and the magic of `__context__` )](#heading-exception-chaining-and-the-magic-of-context)
    
* [Bare except vs except Exception](#heading-bare-except-vs-except-exception)
    
* [Raising shorthands](#heading-raising-shorthands)
    
* [Annotating exceptions (3.11+)](#heading-annotating-exceptions-311)
    
* [What about `UserWarning`?](#heading-what-about-userwarning)
    
* [Bonus](#heading-bonus)
    

---

## A little story

I had an interesting use case at work lately: some external library code was "swallowing" another exception, and I needed to get back the message of the initial one somehow, without touching the library itself.

The library code looked something like this:

```python
class TokenBackend:
   def decode(self, token: str) -> dict:
      # Decode a JWT token,
      # It may fail for many reasons, detailed in the
      # TokenError message
      if not token:
         raise TokenError("Empty token")
      if len(token.split(".")):
         raise TokenError("A JWT must have 3 parts")
      if ...:
         raise TokenError("Invalid signature")
      # ...
      
class Token:
 
   def __init__(self, raw_token):
      # ...
      try:
         self.backend.decode(token) # <- calls TokenBackend.decode
      except TokenError:
         # Here, the library swallows the exception,
         # we LOSE the detailed error message!
         raise DecodeError("Token could not be decoded")
```

I had no clue how to do this except to override the `Token` class in my codebase. When I asked my boss about this, he looked at me and said: "*just use* `__context__`". Huh? Never heard of it. I started digging, and long story short: he was right. This was the perfect solution.

Those small discoveries happened a lot lately, and I wanted to share them. If this intrigues you, keep reading!

## Exception chaining (and the magic of `__context__` )

So, what is this `__context__`??

Formalised in [PEP 3134](https://peps.python.org/pep-3134/) (I love PEPs), exceptions in Python 3 have three *dunder* attributes that provide [information about the context](https://docs.python.org/3/library/exceptions.html#exception-context) in which they were raised: `__cause__`, `__context__` and `__suppress_context__`. To understand, let's first make sense of implicit and explicit exception chaining.

An *exception chain* starts when a new exception is raised during the handling of another, for example from an `except` clause:

```python
try:
    open("foo.bar")
except OSError:
    raise RuntimeError("oops")

Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
FileNotFoundError: [Errno 2] No such file or directory: 'foo.bar'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "<stdin>", line 4, in <module>
RuntimeError: oops
```

This is called an *implicit* chain (hence the "*During handling ...*"). To make it explicit and clearly state an exception is the *cause* of another, one can use the special `raise ... from` :

```python
try:
    open("foo.bar")
except OSError as e:
    raise RuntimeError("oops") from e

Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
FileNotFoundError: [Errno 2] No such file or directory: 'foo.bar'

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "<stdin>", line 4, in <module>
RuntimeError: oops
```

As you can see, we now have another log message: "*was the direct cause of*". Of course, chains can be longer than two.

To go back to the context attributes of an exception:

* `__suppress_context__` is false by default.
    
* When raising a new exception while another exception is already being handled (in `except`, `finally` or `with`), the new exception’s `__context__` attribute is automatically set to the handled exception.
    
* When using `raise ... from` , the supplied exception will additionally be saved in the `__cause__` attribute of the raised exception, and `__suppress_context__` will be set to true.
    

The default traceback uses those attributes to display stacktraces in the following way:

* if `__cause__` is present, always show it
    
* if `__cause__` is `None`, show the `__context__` only if `__suppress_context__` is false.
    

To ensure you followed, what does this valid Python code prints?

```python
try:
   1/0
except ZeroDivisionError:
   raise RuntimeError("zero!") from None
```

It only shows the `RuntimeError`, because the `from` will set the `__cause__` (to `None`) and the `__suppress_context__` (to true).

Now, this doesn't completely swallow the original exception. Even when using a `from`, the initial `ZeroDivisionError` is still in the `__context__`, just ignored when printing the stacktrace.

Back to the problem in the introduction, I simply catch the exception `e` raised by the library (in `Token.__init__`), and then use `e.__context__.args[0]` to get the initial exception message.

## Bare except vs except Exception

I learned this one from the ruff rule [bare-except (E722)](https://docs.astral.sh/ruff/rules/bare-except/). When you don't care about which exception is raised, you may be tempted to use a *bare except* (but should NOT):

```python
try:
   do_something()
except: # <- no exception class is called bare except
   print("oops")
```

A bare `except` catches `BaseException`

> [`BaseException`](https://docs.python.org/3/library/exceptions.html#BaseException) is the common base class of all exceptions. One of its subclasses, [`Exception`](https://docs.python.org/3/library/exceptions.html#Exception), is the base class of all the non-fatal exceptions. Exceptions which are not subclasses of [`Exception`](https://docs.python.org/3/library/exceptions.html#Exception) are not typically handled, because they are used to indicate that the program should terminate.

This `except` thus catches `Exception`, but also `KeyboardInterrupt`, `SystemExit`, and other fatal errors, making it hard to interrupt the program (e.g., with Ctrl-C) and potentially disguising other problems or leaving the program in an unexpected state.

So instead, always specify an exception type, or simply `Exception` if you are in doubt:

```python
try:
   do_something()
except Exception: # now we are good
   print("oops")
```

## Raising shorthands

When re-raising inside an `except` clause, you don't need to pass an argument to `raise`, as it re-raises the caught exception by default.

```python
try:
   x / y
except ZeroDivisionError: # no need to add "as e"
   log.error("we got a zero here")
   raise
```

Similarly, when raising an exception with no argument, no need for parentheses. If an exception class is passed to `raise`, it will be implicitly instantiated by calling its constructor with no arguments.

Hence the following is perfectly valid and more concise (see ruff rule [unnecessary-paren-on-raise-exception (RSE102)](https://docs.astral.sh/ruff/rules/unnecessary-paren-on-raise-exception/)):

```python
raise ValueError
```

## Annotating exceptions (3.11+)

Since Python 3.11, it is possible to attach notes to exceptions, effectively enriching their context. This is a very interesting feature, that could replace re-raising an exception with a different message.

```python
try:
  try:
    raise ValueError
  except Exception as e:
    e.add_note("This was raised as an example")
    raise
except Exception as e:
  e.add_note("Great article btw!")
  raise

Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
ValueError
This was raised as an example
Great article btw!
```

Those notes are saved in the `__notes__` attribute (list of strings).

## What about `UserWarning`?

If you look at the Python documentation, you will see some strange built-in exceptions such as `UserWarning`, `DeprecationWarning`, etc. They all inherit from `Warning` (which itself inherits from `Exception`) but they **are NOT meant to be raised**. Instead, they are used as [warning categories](https://docs.python.org/3/library/warnings.html#warning-categories).

In short, `Warning` exceptions are to be used with the [`warnings`](https://docs.python.org/3/library/warnings.html#module-warnings) module. There is much more to it, but let's look at a simple example:

```python
import warnings

def foo():
    # Explicit category
    warnings.warn("Don't use me anymore!", DeprecationWarning)
    # Implicit category
    warnings.warn("bar") # <- default to UserWarning
```

What is nice about `warning` is that users have complete control over what is reported, thanks to the warning filter:

```python
foo()
# <stdin>:3: DeprecationWarning: Don't use me anymore!
# <stdin>:5: UserWarning: bar
foo() # second time, no more warnings printed
---
warnings.simplefilter("ignore")
foo() # -> nothing printed
---
warnings.simplefilter("error")
foo() # -> raises!
# Traceback (most recent call last):
#   File "<stdin>", line 1, in <module>
#   File "<stdin>", line 3, in foo
# DeprecationWarning: Don't use me anymore!
```

For all available filters, see [The Warnings Filter](https://docs.python.org/3/library/warnings.html#the-warnings-filter).

So, why use exceptions for that? You guessed it, it simplifies turning warnings into exceptions (`error` filter): one just has to raise it.

## Bonus

This article is already way too long, so here is a bullet list of other interesting subjects and picks:

* Python 3.11 introduced `ExceptionGroup`, a nice way to pack multiple exceptions into one. The new syntax `except*` allows filtering groups efficiently. Find out more [in the documentation](https://docs.python.org/3/library/exceptions.html#exception-groups).
    
* Python supports try-except-**else**\-finally, although I never found a good use case for the `else` (also supported in `for` loops). The `else` block is executed after the `try` block, but before the `finally` block (if the `except` block doesn't run). Exceptions raised inside the `else` block are *not* caught by the `except`. See [Handling exceptions](https://docs.python.org/3/tutorial/errors.html#handling-exceptions) for more info.
    
* The `NotImplementedError` (not to confuse with the constant `NotImplemented`) signals a missing implementation *that should come one day*. If the feature will never be implemented, raise a `TypeError` instead.
    
* Exceptions store their `__init__` arguments in the `args` attribute.
    
* I am always tempted to name my custom exception classes with the `Exception` suffix (a remnant of Java perhaps?). However, [PEP 8](https://peps.python.org/pep-0008/#exception-names) clearly states we should use the suffix `Error` for exception class names.
    
* It is possible to catch multiple exceptions using parentheses: `except (FooException, BarException)`
    
* Never `return` in a `finally`: this will override whatever `return` you may have inside the `try` or the `except`.
    

---

I haven't written in a while, so I hope you enjoyed this article.

\- With ❤️, [@derlin](https://derlin.ch)