---
pep: 851
title: Add list.discard()
author:
- Stan Ulbrych <stan@python.org>
discussions_to: https://discuss.python.org/t/pep-851-add-list-discard/109701
status: Draft
type: Standards Track
created: 27-Oct-2026
python_version: '3.16'
post_history:
- '`12-Oct-2025 <https://discuss.python.org/t/add-list-discard-and-deque-discard/104330>`__'
- '`27-Oct-2026 <https://discuss.python.org/t/pep-851-add-list-discard/109701>`__'
python_status: Draft
url: https://peps.python.org/pep-0851/
source_path: https://github.com/python/peps/blob/main/peps/pep-0851.rst
source_commit: e2b0c2bd7f1d05dbf3381f44d27f5cfe5162685d
---

# Abstract

Add `list.discard(x)`, which removes the first item equal to *x* if one
is present and does nothing otherwise, returning whether an item was
removed. It is the non-raising counterpart to
`list.remove`{.interpreted-text role="meth"}, which currently has none.
The name is taken from `set.discard`{.interpreted-text role="meth"},
which already means \"remove if present\" in Python.

# Motivation

Removing a value that may not be present is [usually written
as](https://stackoverflow.com/questions/4915920/how-to-delete-an-item-in-a-list-if-it-exists):

    if x in lst:
        lst.remove(x)

or:

    try:
        lst.remove(x)
    except ValueError:
        pass

The first form is clear but scans the list twice on a hit, and if the
list is shared between threads another thread can remove the item
between the two calls, turning a no-op into a
`ValueError`{.interpreted-text role="exc"}. The second avoids both
problems but obscures the intent. It is four lines for one operation,
pays for raising and catching an exception on every miss, and
`except ValueError` is broad enough to hide unrelated errors from a
user-defined `~object.__eq__`{.interpreted-text role="meth"}.

Both are common. A search of the 5,000 most downloaded PyPI projects[^1]
finds 1,062 matches for the first form across 411 projects and 331 for
the second across 164 projects. Some of these are on other container
types; in the standard library, excluding tests, two of the 29 matches
are on deques and the remaining seven and 20 are on lists, for example
in
[logging](https://github.com/python/cpython/blob/3acc941f2b7288b212a4163ab21aa79faff9cd13/Lib/logging/__init__.py#L840-L841),
[warnings](https://github.com/python/cpython/blob/3acc941f2b7288b212a4163ab21aa79faff9cd13/Lib/_py_warnings.py#L326-L329),
[tkinter](https://github.com/python/cpython/blob/3acc941f2b7288b212a4163ab21aa79faff9cd13/Lib/tkinter/__init__.py#L490-L493),
[runpy](https://github.com/python/cpython/blob/3acc941f2b7288b212a4163ab21aa79faff9cd13/Lib/runpy.py#L312-L315)
and
[asyncio](https://github.com/python/cpython/blob/3acc941f2b7288b212a4163ab21aa79faff9cd13/Lib/asyncio/subprocess.py#L100-L101).
Each becomes a single call that states the intent directly:

    lst.discard(x)

The cost of providing this is small, as `discard()` reuses the existing
search and removal implementation of
`remove() <list.remove>`{.interpreted-text role="meth"}.

# Specification

`list` gains a positional-only method `discard(value, /)`. It removes
the first item equal to *value* and returns `True`, or returns `False`
if there is no such item. Only the absence of a matching item is
reported by the return value; any other error, including a
`ValueError`{.interpreted-text role="exc"} raised during equality
comparison, propagates as an exception. The implementation shares the
search loop of `remove() <list.remove>`{.interpreted-text role="meth"},
so which item is removed, how values are compared and the thread-safety
properties are identical to `remove()`.
`collections.UserList`{.interpreted-text role="class"} gains the same
method for consistency.

# Rationale

Symmetry with `set`{.interpreted-text role="class"} is not the goal. The
motivation is the missing non-raising form of
`list.remove`{.interpreted-text role="meth"}. `set`{.interpreted-text
role="class"} only supplies the name, since
`set.discard`{.interpreted-text role="meth"} already means \"remove if
present\" in Python.

It has been raised that after `s.discard(x)` the test `x not in s`
holds, while after `lst.discard(x)` it need not if *x* appears more than
once. That is a property of sets, not of `discard()`, and
`list.remove`{.interpreted-text role="meth"} has the same behaviour
today. `discard()` follows `remove()` here, and its will documentation
clearly say so.

Unlike `set.discard`{.interpreted-text role="meth"}, which returns
`None`, a bool is returned. This costs nothing, since the implementation
already knows the outcome, and lets `if lst.discard(x):` replace the
double scan for callers who care.

# Backwards Compatibility

`list`{.interpreted-text role="class"} subclasses defining their own
`discard` keep it. Code using `hasattr(obj, "discard")` to tell sets
from lists would be affected. None was found in the standard library or
the top 5,000 PyPI packages.

# Security Implications

None.

# How to Teach This

Introduce it in the
[tutorial](https://docs.python.org/3/tutorial/datastructures.html#more-on-lists)
directly after `list.remove`{.interpreted-text role="meth"}, with
`remove()` for when the item must be present and `discard()` for when it
may not be. The same pairing already exists for `set`{.interpreted-text
role="class"}.

# Reference Implementation

A reference implementation can be found in
[python/cpython#160726](https://github.com/python/cpython/pull/160726).

# Rejected Ideas

## A parameter on `remove()`

A separate method is preferred over a parameter such as
`remove(x, missing_ok=True)`. The standard library has precedent for
both shapes,
`Path.unlink(missing_ok=True) <pathlib.Path.unlink>`{.interpreted-text
role="meth"} and
`os.makedirs(exist_ok=True) <os.makedirs>`{.interpreted-text
role="func"} on one side and `set.remove`{.interpreted-text role="meth"}
/ `set.discard`{.interpreted-text role="meth"} on the other, but the
closest precedent is `set`, and following it means the two container
types share one vocabulary. A method also reads more clearly at the call
site, since `lst.discard(x)` says what happens while a flag describes
what does not.

## Adding `discard()` to `collections.abc.MutableSequence`

`collections.abc.MutableSequence`{.interpreted-text role="class"} is
left unchanged, as is the case for `list.copy`{.interpreted-text
role="meth"}. Adding `discard()` to the ABC would change the interface
of every third-party subclass, while virtual subclasses registered with
`register()` would not gain it at all.

## `deque.discard()`

The [original
discussion](https://discuss.python.org/t/add-list-discard-and-deque-discard/104330)
also proposed `collections.deque`{.interpreted-text role="class"}
gaining the same method. It is left out of this PEP to keep the proposal
to the builtin `list`{.interpreted-text role="class"}. It can follow as
a routine change if this PEP is accepted.

# Footnotes

# Acknowledgements

Thanks to Guido van Rossum for suggesting that `discard()` return
whether an item was removed.

# Copyright

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.

[^1]: Using [Victor Stinner\'s
    scripts](https://hugovk.dev/blog/2022/how-to-search-5000-python-projects/),
    modified to match across lines
    ([vstinner/misc#24](https://github.com/vstinner/misc/pull/24)), with
    the regular expressions:

        if ([\w.\[\]]+) in ([\w.\[\]]+):\n\s+\2\.remove\(\1\)

        try:\n\s+([\w.\[\]]+)\.remove\(([\w.\[\]]+)\)\n\s*except ValueError:

    The first pattern\'s backreferences require the same names on both
    lines. The counts are approximate, as the second pattern also
    matches handlers that do something other than ignore the absence,
    while neither matches variants such as compound conditions,
    `except (ValueError, ...)` or a comment between the lines.
