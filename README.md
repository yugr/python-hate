A growing list of things I dislike about Python.

There are workarounds for some of them (often half-broken and usually unintuitive).
And other may even be treated as virtues by some people.

# Generic

The most critical problem of Python is that it does not perform any static checks
(not even missing variable definitions).
This looks particularly funny when you run your app
on a huge amount of data overnight, just to detect missing initialization in some
rarely called function the next day.
This increases debugging time and makes refactoring more time-consuming
than needed. Linter does not help much because it fails miserably
to catch even the most basic errors (invalid string formatting,
iterating over unsorted dicts, dead list computations).

GIL precludes CPU parallelism which is suprising at the age of multicores.

Lack of type annotations forces people to use hungarian notation
in complex programs (hello 90-s!).

# Language

This is not a valid syntax:
```
x == not y
```

Syntax checking is quite primitive. In most cases you simply get `SyntaxError: invalid syntax`.

It's too easy to inadertently share references:
```
a = b = []
```
or
```
def foo(x=[]):  # foo() will return [1], [1, 1], [1, 1, 1], etc.
  x.append(1)
  return x
```

Default return value from function when return is omitted is `None`
(which makes it impossible to declare subroutines which are not supposed
to return anything).

# Standard Libraries

Convention of having a pair of modifying and non-modifying accessors
(e.g. `reverse`/`reversed` or `sort`/`sorted`) is not followed for all standard APIs. E.g. not for `str.replace()`.

Thanks to active use of list comprehensions in Python 3 it's so easy
to misuse standard APIs:
```
if filter(lambda x: x == 0, [1,2]):
  print("Yes")  # Prints "Yes"!
```
Surpisingly enough this does not apply to `range` (i.e. `bool(range(0))` returns `True`).

`argparse` does not provide automatic support for `--no-XXX` flags.

`split` and `join` use different order of arguments:
```
sep,join(lst)
lst.split(sep)
```

# Name Resolution

There is no way to localize variable withing a function.
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

Similarly to above, there is no control over visibility (can't hide class methods,
can't hide module functions). You are left with a _convention_ to precede
private functions with `_` and hope for the best.

Assigning to variable automatically declares it local, even if there a global definition:
```
xxx = 0

def foo():
  xxx = 1

foo()
print(xxx)  # Prints 0!
```
This isn't consistent with reading a variable (which will read global definition if it's present,
rather than throwing `NameError`). A "workaround" is to convert global variable to
array as this will force interpreter to prefer global definition (what?!):
```
xxx = [0]

def foo():
  xxx[0] = 1

foo()
print(xxx[0])  # Prints 1
```

# Performance

It's very hard to optimize Python code because there are far too many ways
in which program may change execution environment e.g.
```
  for re in regexes:
    ...
```
Existing optimizers (e.g. pypy) have to rely on idioms and heuristics.

# Infrastructure

Windows and Linux use different naming convention for Python executables
(`python` on Windows, `python2`/`python3` on Linux).
