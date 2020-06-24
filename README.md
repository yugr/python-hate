A growing list of things I dislike about Python.

There are workarounds for some of them (often half-broken and usually unintuitive)
and others may even be considered to be virtues by some people.

# Generic

## Zero static checking

The most critical problem of Python is that it does not perform any static checks
(not even missing variable definitions).
Lack of static checking increases debugging time and makes refactoring
more time-consuming than needed.
This becomes particularly obvious when you run your app on a huge amount of data overnight,
just to detect missing initialization in some rarely called function in the morning.

There is [Pylint](https://www.pylint.org) but it is a linter (i.e. style checker)
rather than a real static analyzer so it is unable, by design, to detect
many serious errors in programs which require dataflow analysis.
For example if fails on basic stuff like
* invalid string formatting (fixed [here](https://github.com/PyCQA/pylint/pull/2465))
* iterating over unsorted dicts (reported [here](https://github.com/PyCQA/pylint/issues/2467) with draft patch, rejected because maintainers consider it unimportant (no particular reasons provided))
* dead list computations (e.g. using `sorted(lst)` instead of `lst.sort()`)
* modifying list while iterating over it (reported [here](https://github.com/PyCQA/pylint/issues/2471) with draft patch, rejected because maintainers consider it unimportant (no particular reasons provided))
* etc.

## Global interpreter lock

[GIL](https://wiki.python.org/moin/GlobalInterpreterLock) precludes high-performant multithreading
which is suprising at the age of multicores.

## No type annotations

_Type annotations have finally been introduced in Python 3.5. Google developed a type inferencer [pytype](https://github.com/google/pytype) (it seems to have [serious](https://github.com/google/pytype/issues/581) [limitations](https://github.com/google/pytype/issues/580) so it's unclear whether it's production-ready)._

Lack of type annotations forces people to use hungarian notation
in complex programs (hello 90-s!).

# Language

## Negation operator syntax issue

This is not a valid syntax:
```
x == not y
```

## Unable to overload logic operators

It's not possible to overload `and`, `or` or `not` (which might have been handy e.g. in `Interval` class).
There's even a [PEP](https://www.python.org/dev/peps/pep-0335/) which was rejected
because Guido disliked particular implementation.

## Hiding type errors via (un)helpful conversions

It's very easy to write `len(lst1) == lst2` instead of `len(lst1) == len(lst2)`.
Python will (un)helpfully make it harder to find this error
by converting first variant to `[len(lst1)] * len(lst2) == lst2`
(instead of aborting with a type fail).

## Limited lambdas

For unclear reason lambda functions only support expressions
so anything that has control flow requires a local named function.

[PEP 3113](https://www.python.org/dev/peps/pep-3113/)
tries to persuade you that this was a good design decision:
```
While an informal poll of the handful of Python programmers I know personally ...
indicates a huge majority of people do not know of this feature ...
```

## Problematic operator precedence

The `is` and `is not` operators have the same precedence as comparisons
so this code
```
op.post_modification is None != full_op.post_modification is None
```
would rather unexpectedly evalute as
```
((op.post_modification is None) != full_op.post_modification) is None
```

## The useless self

Explicitly writing out `self` in all method declarations and calls
should not be needed. Apart from being an unnecessary boilerplate,
this enables another class of bugs:
```
class A:
  def first(x, *y):
    return x

a = A
print(a.first(1,2,3))  # Will print a, not 1
```

## Optional parent constructors

Python does not require a call to parent class constructor:
```
class A(B):
  def __init__(self, x):
    super().__init__()
    self.x = x
```
so when it's missing you'll have hard time understanding
whether it's been omitted deliberately or accidentally.

## Inconsistent syntax for tuples

Python allows omission of parenthesis around tuples in most cases:
```
for i, x in enumerate(xs):
  pass

x, y = y, x

return x, y
```
but not all cases:
```
foo = [x, y for x in range(5) for y in range(5)]
SyntaxError: invalid syntax
```

## No tuple unpacking in lambdas

It's not possible to do tuple unpacking in lambdas so instead
of concise and readable
```
lst = [(1, 'Helen', None), (3, 'John', '121')]
lst.sort(key=lambda n, name, phone: (name, phone))  # TypeError: <lambda>() missing 2 required positional arguments
```
you should use
```
lst.sort(key=lambda n_name_phone: (n_name_phone[1], n_name_phone[2]))
```

To make things even better, tuple unpacking does work in Python 2.

## Inconsistency of set literals

Sets can be initialized via syntax sugar:
```
>>> x = {1, 2, 3}
>>> type(x)
<class 'set'>
```
but it breaks for empty sets:
```
>>> x = {}
>>> type(x)
<class 'dict'>
```

## Syntax checking

Syntax error reporting in Python is extremely primitive.
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
or even
```
def foo(obj, lst=[]):
  obj.lst = lst
foo(obj)
obj.lst.append(1)  # Hoorah, this modifies default value of foo
```

## Functions always return

Default return value from function (when `return` is omitted) is `None`.
This makes it impossible to declare subroutines which are not supposed
to return anything (and verify this at runtime).

## Automatic field insertion

Assigning a non-existent object field adds it instead of throwing an exception:
```
class A:
  def __init__(self):
    self.x = 0
...
a = A()
a.y = 1  # OK
```

This complicates refactoring because forgetting to update an outdated field name
deep inside your (or your colleague's) program will silently work,
breaking your program much later.

This can be overcome with `__slots__` but when have you seen them
used last time?

## "Is" operator does not work for primitive types

No comments:
```
> 2+2 is 4
True
> 999+1 is 1000
False
```

This happens because [only sufficiently small integer objects are reused](https://stackoverflow.com/a/306353/2170527):
```
# Two different instances of number "1000"
>>> id(999+1)
140068481622512
>>> id(1000)
140068481624112

# Single instance of number "4"
>>> id(2+2)
10968896
>>> id(4)
10968896
```

## Inconsistent index checks

Invalid indexing throws an exception
but invalid slicing does not:
```
>>> a=list(range(4))
>>> a[4]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IndexError: list index out of range
>>> a[4:10]
[]
```

## Spurious `:`s

Python got rid of spurious bracing but introduced a spurious `:` lexeme instead.
The lexeme is not needed for parsing and it's only purpose
[was](http://effbot.org/pyfaq/why-are-colons-required-for-the-if-while-def-class-statements.htm)
to somehow "enhance readability".

# Standard Libraries

## List generators fail check for emptiness

Thanks to active use of generators in Python 3 it became easier
to misuse standard APIs:
```
if filter(lambda x: x == 0, [1,2]):
  print("Yes")  # Prints "Yes"!
```
Surpisingly enough this does not apply to `range` (i.e. `bool(range(0))` returns `False` as expected).

## Argparse does not support negated flags

`argparse` does not provide automatic support for `--no-XXX` flags.

## Argparse has useless default formatter

By default formatter used by argparse
* won't display default option values
* will make code examples provided via `epilog` unreadable
  by stripping leading whitespaces

Enabling both features requires defining a custom formatter:
```
class Formatter(argparse.ArgumentDefaultsHelpFormatter, argparse.RawDescriptionHelpFormatter):
  pass
```

## Getopt does not allow parameters after positional arguments

```
>>> opts, args = getopt.getopt(['A', '-o', 'B'], 'o:')
>>> opts
[]
>>> args
['A', '-o', 'B']
>>> opts, args = getopt.getopt(['-o', 'B', 'A'], 'o:')
>>> opts
[('-o', 'B')]
>>> args
['A']
```

## Split and join disagree on argument order

`split` and `join` accept list and separator in different order:
```
sep.join(lst)
lst.split(sep)
```

# Name Resolution

## No way to localize a name

Python lacks lexical scoping i.e. there is no way to localize variable
in a scope smaller than a function.
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

## Multiple syntaxes for aliasing imported module

Python allows different syntaxes for the aliasing functionality:
```
from mod import submod as X
import mod.submod as X
```

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
This is particularly puzzling in long functions when someone accidentally
adds a local variable which matches name of a global variable used in other part
of the same function:
```
def value():
  ...

def foo():
  ...  # A lot of code
  value(1)  # Surprise! UnboundLocalError
  ...  # Yet more code
  for value in data:
    ...
```

Once you've lost some time debugging the issue, you can be overcome
for global variables by declaring their names as `global`
before first use:
```
def foo():
  global xxx
  a = xxx
  xxx = 2
```
But there are no magic keywords forvariables from non-global outer scopes so they
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

## Unreachable code that gets executed

Normally statements that belongs to false branches are not executed.
E.g. this code works:
```
if True:
  import re
re.search(r'1', '1')
```
and this one raises `NameError`:
```
if False:
  import re
re.search(r'1', '1')
```
This does not apply to `global` declarations though:
```
xxx = 1
def foo():
  if False:
    global xxx
  return xxx
print(foo())  # Prints 1
```

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
Also see [When are circular imports fatal?](https://datagrok.org/python/circularimports/)
for more weird limitations of relative imports with respect to circular dependencies.

# Performance

## Automatic optimization is hard

It's very hard to automatically optimize Python code
because there are far too many ways
in which program may change execution environment e.g.
```
  for re in regexes:
    ...
```
(see e.g. [this quote](https://youtu.be/2wDvzy6Hgxg?t=690) from Guido).
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

## Unable to break on pass statement

Python debugger will [ignore breakpoints set on pass statements](https://stackoverflow.com/a/47626134/2170527).
Thus poor-man's conditional breakpoints like
```
if x > 0:
  pass
```
will silently fail to work, leaving a false impression that condition is always false.

## Semantic changes in Python 3

Most people blame Python 3 for syntax changes which break existing code
(e.g. making `print` a regular function) but the real problem is
_semantic_ changes as they are much harder to detect and debug.
Some examples
* integer division:
```
print(1/2)  # Prints "0" in 2, "0.5" in 3
```
* checking `filter`ed list for emptyness:
```
if filter(lambda x: x, [0]):
  print("X")  # Executes in 3 but not in 2
```
* order of keys in dictionary is random until Python 3.6

## Dependency hell

Python community does not seem to have a strong culture of preserving API backwards compatibility
or following [semver convention](https://semver.org/)
(which is hinted by the fact that there are no widespread tools for checking Python package API
compatibility). This is not surprising given that even minor versions of Python 3 itself
break old and popular APIs (e.g. [time.clock](https://bugs.python.org/issue31803)).
Another likely reason is lack of good mechanisms to control what's exported from module
(prefixing methods and objects with underscore is _not_ a good mechanism).

In practice this means that it's too risky to allow differences in minor (and even patch) versions
of dependencies.
Instead the most robust (and thus most common) solution is to fix _all_ app dependencies
(including the transitive ones) down to patch versions (via blind `pip freeze > requirements.txt`)
and run each app in a dedicated virtualenv or Docker container.

Apart from complicating deployment, fixing versions also
complicates importing module in other applications
(due to increased chance of conflicing dependencies)
and upgrading dependencies later on to get bugfixes and security patches.

For more details see the excellent ["Dependency hell: a library author's guide" talk](https://www.youtube.com/watch?v=OaBhcueqNqw)
