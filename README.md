A growing list of things I dislike about Python.

There are workarounds for some of them (often half-broken and usually unintuitive).
And other may even be treated as virtues by some people.

# Generic

## Zero static checking

The most critical problem of Python is that it does not perform any static checks
(not even missing variable definitions).
This looks particularly funny when you run your app
on a huge amount of data overnight, just to detect missing initialization in some
rarely called function the next day.

Lack of static checking increases debugging time and makes refactoring
more time-consuming than needed.

There is a Pylint but it is a linter (i.e. style checker) rather than analyzer
so it is unable to detect many serious errors in programs
which require dataflow analysis. In addition if fails on basic stuff like
* invalid string formatting (fixed [here](https://github.com/PyCQA/pylint/pull/2465))
* iterating over unsorted dicts (reported [here](https://github.com/PyCQA/pylint/issues/2467) with draft patch, rejected because maintainers consider it unimportant (no particular reasons provided))
* dead list computations (e.g. using `sorted(lst)` instead of `lst.sort()`)
* modifying list while iterating over it (reported [here](https://github.com/PyCQA/pylint/issues/2471) with draft patch, rejected because maintainers consider it unimportant (no particular reasons provided))
* etc.

## GIL

GIL precludes CPU parallelism which is suprising at the age of multicores.

## No type annotations

Lack of type annotations forces people to use hungarian notation
in complex programs (hello 90-s!).

# Language

## Negation operator syntax issue

This is not a valid syntax:
```
x == not y
```

## Syntax checking

Syntax error reporting is extremely primitive.
In most cases you simply get `SyntaxError: invalid syntax`.

## Inadvertent sharing

It's too easy to inadvertently share references:
```
a = b = []
```
or
```
def foo(x=[]):  # foo() will return [1], [1, 1], [1, 1, 1], etc.
  x.append(1)
  return x
```

## Function always returns

Default return value from function when return is omitted is `None`.
This makes it impossible to declare subroutines which are not supposed
to return anything.

## Automatic field insertion

Assigning a non-existent object field creates it instead of throwing exception:
```
class A:
  def __init__(self):
    self.x = 0
...
a = A()
a.y = 1  # OK
```

This complicates refactoring because forgetting to update old field name
deep inside your program will silently work, breaking your program in
much later.

This can of course be overcome with `__slots__` but when have you seen them
used last time?

## Relative imports are unusable

Relative imports (`from .xxx.yyy import mymod`) have many weird limitations
e.g. they will not allow you to import module from parent folder and
they will seize work in main script
```
ModuleNotFoundError: No module named '__main__.xxx'; '__main__' is not a package
```

A workaround is to use extremely ugly `sys.path` hackery:
```
import sys
import os.path
sys.path.append(os.path.join(os.path.dirname(__file__), 'xxx', 'yyy'))
```

Search for "python relative imports" on stackoverflow to see some really clumsy Python code
(e.g. [here](https://stackoverflow.com/questions/279237/import-a-module-from-a-relative-path)
or [here](https://stackoverflow.com/questions/1918539/can-anyone-explain-pythons-relative-imports)).

# Standard Libraries

## List comprehensions fail check for emptiness

Thanks to active use of list comprehensions in Python 3 it's so easy
to misuse standard APIs:
```
if filter(lambda x: x == 0, [1,2]):
  print("Yes")  # Prints "Yes"!
```
Surpisingly enough this does not apply to `range` (i.e. `bool(range(0))` returns `True`).

## Argparse does not support negated flags

`argparse` does not provide automatic support for `--no-XXX` flags.

## Split and join disagree on argument order

`split` and `join` use different order of arguments:
```
sep.join(lst)
lst.split(sep)
```

# Name Resolution

## No way to localize a name

There is no way to localize variable in a scope smaller than a function.
This often hurts when renaming variables during code refactoring.
Forgetting to rename variable name in a single place, causes interpreter
to pick up an unrelated name from unrelated block 50 lines above or
from previous loop iteration.

This is especially inconvenient for one-off variables (e.g. loop counters):
```
for i in range(100):
  ...

# 100 lines later

for j in range(200):
  a[i] = 0  # Yikes, forgot to rename!
```

One could remove a name from scope via `del i` but this is considered too
verbose so noone uses it.

## Everything is exported

Similarly to above, there is no control over visibility (can't hide class methods,
can't hide module functions). You are left with a _convention_ to precede
private functions with `_` and hope for the best.

## Assignment makes variable local

Python scoping rules require that assigning a variable automatically declares it local.
This causes inconsistencies and weird limitations in practice. E.g. variable would
be considered local even if assignment follows first use:
```
global xxx
xxx = 0

def foo():
  a = xxx  # Throws UnboundLocalError
  xxx = 2
```
and even if assignment is never ever executed:
```
def foo():
  a = xxx  # Still aborts...
  if False:
    xxx = 2
```
For global variables this can be overcome by declaring variable name as `global`
before first use:
```
def foo():
  global xxx
  a = xxx
  xxx = 2
```
But variables from non-global outer scopes there are no magic keywords so they
are essentially unwritable from nested scopes i.e. closures:
```
def foo():
  xxx = 1
  def bar():
    xxx = 2  # No way to modify xxx...
```

The only available "solution" is to wrap the variable into a fake 1-element array
(whaat?!):
```
def foo():
  xxx = [1]
  def bar():
    xxx[0] = 2
```

# Performance

## Automatic optimization is hard

It's very hard to automatically optimize Python code
because there are far too many ways
in which program may change execution environment e.g.
```
  for re in regexes:
    ...
```
Existing optimizers (e.g. pypy) have to rely on idioms and heuristics.

# Infrastructure

## Different conventions on OSes

Windows and Linux use different naming convention for Python executables
(`python` on Windows, `python2`/`python3` on Linux).

## Debugger is slow

Python debugging is super-slow (few orders of magnitude slower than
interpretation).

## No static analyzer

Already mentioned in [Zero static checking](#zero-static-checking).
