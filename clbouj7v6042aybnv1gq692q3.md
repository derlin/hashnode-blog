# 7 Python 3.11 new features 🤩

[Python 3.11](https://docs.python.org/3.11/whatsnew/3.11.html) is out since October, 25 and comes with great new features! Here are my top picks.

**Covered in this article**:

*   Adding notes to exceptions
    
*   Better tracebacks
    
*   `Self` type
    
*   `StrEnum`, `ReprEnum` and other enum improvements
    
*   New `logging.getLevelNamesMapping()` method
    
*   [TOML](https://github.com/toml-lang/toml) built-in support
    
*   (🤔 `LiteralString` ??)
    

See all other features on [What’s New In Python 3.11](https://docs.python.org/3.11/whatsnew/3.11.html) !

* * *

**🚀🚀🚀🚀 speed improvements**: Python 3.11 is supposed to be way faster, thanks to improvements from [Faster CPython](https://docs.python.org/3.11/whatsnew/3.11.html#whatsnew311-faster-cpython) project:

> *Python 3.11 is between 10-60% faster than Python 3.10. On average, we measured a 1.25x speedup on the standard benchmark suite.*

It won't be the focus of this article, but if you are interested you should be able to find many benchmarks and details online 🤓

* * *

## Adding notes to exceptions

From the release notes:

> *The* `add_note()` method is added to `BaseException`. It can be used to enrich exceptions with context information that is not available at the time when the exception is raised. The added notes appear in the default traceback.

For example:

```python
if __name__ == "__main__":
    try:
        try:
            raise TypeError("bad type")
        except TypeError as type_error:
            type_error.add_note("Some information")
            raise
    except TypeError as type_error:
        type_error.add_note("And some more information")
        raise
```

This will output:

```python
python Traceback (most recent call last): File "/app/notes.py", line 4, in <module> raise TypeError("bad type") TypeError: bad type Some information And some more information
```

## Better tracebacks

Staying on the exception topic, tracebacks are enriched to show the exact expression that caused the error. This is especially useful when a lot is going on on a single line.

```python
Traceback (most recent call last):
  File "distance.py", line 11, in <module>
    print(manhattan_distance(p1, p2))
          ^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "distance.py", line 6, in manhattan_distance
    return abs(point_1.x - point_2.x) + abs(point_1.y - point_2.y)
                           ^^^^^^^^^
AttributeError: 'NoneType' object has no attribute 'x'
```

## `Self` type

When using type hints, it has always bothered me not to be able to refer to the current class without importing some `__future__`. There is now, *finally*, a `Self` type that can be used !

This is Python 3.10:

```python
from __future__ import annotations # this is necessary ...

class Point:
    def __init__(self, x: int, y: int):
        self.x = x
        self.y = y

    @classmethod
    def origin(cls) -> Point: # .. for this to compile
        return cls(0, 0)
```

With Python 3.11:

```python
from typing import Self # now, import Self

class Point:
    # ...

    @classmethod
    def origin(cls) -> Self: # and use it instead of Point
        return cls(0, 0)
```

This makes it easy to rename the `Point` class to anything, and makes the code more readable.

## `StrEnum`, `ReprEnum` and other enum improvements

The enum class has a new member, `StrEnum` especially for enum with string values. It basically adds the `auto()` feature, that avoids painful repetitions.

The new `verify()` decorator allows to ensure various constraints such as `UNIQUE`, and - for integer values only - `CONTINUOUS` and `NAMED_FLAGS`. See `EnumChecks` for details. I hope they will add more in a future version.

Finally, `IntEnum`, `IntFlag` and `StrEnum` now inherit from `ReprEnum`, which makes their `str()` output match the value of the enum instead of its class.

```python
from enum import StrEnum, verify, UNIQUE, auto

@verify(UNIQUE)
class Color(StrEnum):
    RED = auto()
    GREEN = auto()
    BLUE = auto()

if __name__ == "__main__":
    print(Color.RED) # prints "red" instead of "Color.RED"
    print(f"my color is {Color.BLUE}") # prints "my color is blue"
    print("green" == Color.GREEN) # print "True"
```

## New `logging.getLevelNamesMapping()` method

This is a detail, but I can't count how often I had to manually list the logging levels available in my `argparse` choices for command line tools... Python 3.11 finally provides this mapping for us, using `getLevelNamesMapping()`. Here is how I would typically use it:

```python
import argparse, logging

if __name__ == "__main__":
    # Get the logging levels available ...
    levels = logging.getLevelNamesMapping()

    parser = argparse.ArgumentParser()
    parser.add_argument("-l", "--level", 
      choices=levels.keys(), # ... list them as arguments ...
      default="CRITICAL", 
      type=str.upper) # (make it case insensitive)
    
    args = parser.parse_args()
    # ... and apply the chosen one
    logging.basicConfig(level=levels[args.level])
```

## [TOML](https://github.com/toml-lang/toml) built-in support

Python 3.11 is adding the [tomllib](https://docs.python.org/3.11/library/tomllib.html#module-tomllib) to the standard library.

[TOML - Tom's Obvious, Minimal Language](https://github.com/toml-lang/toml), is a minimal configuration file format that's easy to read due to obvious semantics. I often find it better than YAML for simple configurations.

Take the following TOML file:

```ini
title = "TOML Example"

[owner]
name = "Tom Preston-Werner"
dob = 1979-05-27T07:32:00-08:00 # First class dates

[database]
server = "192.168.1.1"
ports = [ 8000, 8001, 8002 ]
connection_max = 5000
enabled = true

[servers]

  # Indentation (tabs and/or spaces) is allowed but not required
  [servers.alpha]
  ip = "10.0.0.1"
  dc = "eqdc10"

  [servers.beta]
  ip = "10.0.0.2"
  dc = "eqdc10"
# ... more config
```

The result, `data`, holds a dictionary with all the config and proper types (see the `datetime` here ?):

```python
{'title': 'TOML Example', 'owner': {'name': 'Tom Preston-Werner', 'dob': datetime.datetime(1979, 5, 27, 7, 32, tzinfo=datetime.timezone(datetime.timedelta(days=-1, seconds=57600)))}, 'database': {'server': '192.168.1.1', 'ports': [8000, 8001, 8002], 'connection_max': 5000, 'enabled': True}, 'servers': {'alpha': {'ip': '10.0.0.1', 'dc': 'eqdc10'}, 'beta': {'ip': '10.0.0.2', 'dc': 'eqdc10'}}}
```

## 🤔 `LiteralString` ??

Ok, I must admit I didn't even know about those `LiteralString` before reading the release notes... But they are quite nice !

**The theory**

When a function receives a `LiteralString` instead of a `str`, it allows the type checks to fail in case an argument is passed that contains some dynamic, user-provided value. It is mostly used with databases, to avoid SQL injections.

**In practice**

I couldn't make it work. Here is my code, which runs perfectly whell, without any of the errors defined in [PEP 675](https://peps.python.org/pep-0675/):

```python
from typing import LiteralString
import argparse


def run_query(sql: LiteralString) -> None:
    print(f"Executing: {sql}")

if __name__ == "__main__":
    static_table: str = "bar"

    parser = argparse.ArgumentParser()
    parser.add_argument("-t", "--table", required=True)
    args = parser.parse_args()
    # ok
    run_query("SELECT foo FROM bar")
    run_query("SELECT " + 'foo' + f" FROM {static_table}")
    # should fail (dynamic argument) 
    run_query(f"SELECT foo from {args.table}")
```

All of this compiles and runs fine from my `python:3.11.0` docker image... If you understand this feature, please let me know in the comments !

## And much more !

Have a look at the release notes for more awesome new features: [What’s New In Python 3.11](https://docs.python.org/3.11/whatsnew/3.11.html)

Let me know if the comments what *you* found interesting in this release !