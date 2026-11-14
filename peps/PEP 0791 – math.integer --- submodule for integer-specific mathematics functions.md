---
pep: 791
title: math.integer --- submodule for integer-specific mathematics functions
author:
- Neil Girdhar <misterhsheik@gmail.com>
- Sergey B Kirpichev <skirpichev@gmail.com>
- Tim Peters <tim.peters@gmail.com>
- Serhiy Storchaka <storchaka@gmail.com>
sponsor: Victor Stinner <vstinner@python.org>
discussions_to: https://discuss.python.org/t/92548
status: Final
type: Standards Track
created: 12-May-2025
python_version: '3.15'
post_history:
- '`12-Jul-2018 <https://mail.python.org/archives/list/python-ideas@python.org/thread/YYJ5YJBJNCVXQWK5K3WSVNMPUSV56LOR/>`__'
- '`09-May-2025 <https://discuss.python.org/t/91337>`__'
- '`19-May-2025 <https://discuss.python.org/t/92548>`__'
resolution: '`23-Oct-2025 <https://discuss.python.org/t/92548/154>`__'
python_status: Final
url: https://peps.python.org/pep-0791/
source_path: https://github.com/python/peps/blob/main/peps/pep-0791.rst
source_commit: 629b3d43c2d223ba135b38daa850211a51b30890
---

::: canonical-doc
[math.integer --- integer-specific mathematics
functions](https://docs.python.org/3.15/library/math.integer.html)
:::

# Abstract

This PEP proposes a new submodule for number-theoretical, combinatorial
and other functions defined for integer arguments, like
`math.gcd`{.interpreted-text role="external+py3.14:func"} or
`math.isqrt`{.interpreted-text role="external+py3.14:func"}.

# Motivation {#pep791-motivation}

The `math`{.interpreted-text role="external+py3.14:mod"} documentation
says: \"This module provides access to the mathematical functions
defined by the C standard.\" But, over time the module was populated
with functions that aren\'t related to the C standard or floating-point
arithmetics. Now it\'s much harder to describe module scope, content and
interfaces (returned values or accepted arguments).

For example, the following statement from the documentation: \"Except
when explicitly noted otherwise, all return values are floats.\" This is
no longer true: *None* of the functions listed in the [Number-theoretic
functions](https://docs.python.org/3.14/library/math.html#number-theoretic-functions)
subsection of the documentation return a float, but the documentation
doesn\'t say so. In the documentation for the proposed `math.integer`
submodule the sentence \"All return values are integers\" would be
accurate. In a similar way we can simplify the description of the
accepted arguments for functions in both the new submodule and in
`math`{.interpreted-text role="external+py3.14:mod"}.

Now it\'s a lot harder to satisfy people\'s expectations about the
module content. For example, should they expect that
`math.factorial(100)` will return an exact answer? Many languages,
Python packages (like `scipy`{.interpreted-text role="pypi"}) or pocket
calculators have functions with same or similar name, that return a
floating-point value, which is only an approximation in this example.

Apparently, the `math`{.interpreted-text role="external+py3.14:mod"}
module can\'t serve as a catch-all place for mathematical functions
since we also have the `cmath`{.interpreted-text
role="external+py3.14:mod"} and `statistics`{.interpreted-text
role="external+py3.14:mod"} modules. Let\'s do the same for
integer-related functions. It provides shared context, which reduces
verbosity in the documentation and conceptual load. It also aids
discoverability through grouping related functions and makes IDE (e.g.
new CPython\'s REPL) suggestions more helpful.

Currently the `math`{.interpreted-text role="external+py3.14:mod"}
module code in the CPython is around 4200LOC, from which the new module
code is roughly 1/3 (1300LOC). This is comparable with the
`cmath`{.interpreted-text role="external+py3.14:mod"} (1340LOC), which
is *not* a simple wrapper to the `libm`, as most functions in the
`math`{.interpreted-text role="external+py3.14:mod"} module.

And this situation tends to get worse. When the module split [was first
proposed](https://mail.python.org/archives/list/python-ideas@python.org/thread/YYJ5YJBJNCVXQWK5K3WSVNMPUSV56LOR/),
there were only two integer-related functions:
`~math.factorial`{.interpreted-text role="external+py3.14:func"}
(accepting also `float`{.interpreted-text role="class"}\'s, like other
functions in the module) and `~math.gcd`{.interpreted-text
role="external+py3.14:func"} (moved from the
`fractions`{.interpreted-text role="external+py3.14:mod"} module). Then
`~math.isqrt`{.interpreted-text role="external+py3.14:func"},
`~math.comb`{.interpreted-text role="external+py3.14:func"} and
`~math.perm`{.interpreted-text role="external+py3.14:func"} were added,
and addition of the new module was [proposed second
time](https://github.com/python/cpython/issues/81313), so all new
functions would go directly to it, without littering the
`math`{.interpreted-text role="external+py3.14:mod"} namespace. Now
there are six functions and `~math.factorial`{.interpreted-text
role="external+py3.14:func"} doesn\'t accept `float`{.interpreted-text
role="class"}s anymore.

Some possible additions, among those proposed in the initial discussion
thread and issue
[python/cpython#81313](https://github.com/python/cpython/issues/81313)
are:

- `c_div()` and `n_div()` \-\-- for integer division with rounding
  towards positive infinity (ceiling divide) and to the nearest integer,
  see [relevant discussion thread](https://discuss.python.org/t/91269).
  This is reinvented several times in the stdlib, e.g. in
  `datetime`{.interpreted-text role="mod"} and
  `fractions`{.interpreted-text role="mod"}. And it\'s easy to do this
  wrongly, as demonstrated by the thread.
- `gcdext()` \-\-- to solve linear [Diophantine
  equation](https://en.wikipedia.org/wiki/Diophantine_equation) in two
  variables (the `int`{.interpreted-text role="external+py3.14:class"}
  implementation actually includes an extended Euclidean algorithm)
- `isqrt_rem()` \-\-- to return both an integer square root and a
  remainder (which is non-zero only if the integer isn\'t a perfect
  square)
- `ilog()` \-\-- integer logarithm, `math.log`{.interpreted-text
  role="external+py3.14:func"} has special handling for integer
  arguments. It\'s unique (with respect to other module functions) and
  not documented so far, see issue
  [python/cpython#120950](https://github.com/python/cpython/issues/120950).
- `fibonacci()` \-\-- [Fibonacci
  sequence](https://en.wikipedia.org/wiki/Fibonacci_sequence).

Separated namespace eliminates possible name clash with existing
`math`{.interpreted-text role="external+py3.14:mod"}\'s module
functions. For example, possible names `ceil_div()` or `ceildiv()` for
integer ceiling division will interfere with the
`~math.ceil`{.interpreted-text role="external+py3.14:func"} (which is
for `float`{.interpreted-text role="class"}\'s and *sometimes* does
right things for integer division, as an accident \-\-- but [usually
not](https://discuss.python.org/t/91269/6)).

# Rationale

Is this all about documentation, why not fix it instead? No, it isn\'t.
Sure, we can be much more vague in the module preamble (i.e. roughly say
that \"the `math`{.interpreted-text role="external+py3.14:mod"} module
contains some mathematical functions\"), we can accurately describe
input/output for each function and its behavior (e.g. whether the
`~math.factorial`{.interpreted-text role="external+py3.14:func"} output
is exact or not, like the
[scipy.special.factorial](https://docs.scipy.org/doc/scipy/reference/generated/scipy.special.factorial.html#scipy.special.factorial),
per default).

But the major issue is that the current module mixes different, almost
non-interlaced application domains. Adding more documentation will just
highlight this and make the issue worse for end users (more text to
read/skip). And it will not fix the issue with discoverability (to know
in which module to find a function, and that it can be found at all, you
need to look at all the functions in the module), nor with
tab-completion.

# Specification

The PEP proposes moving the following integer-related functions to a new
submodule, called `math.integer`:

- `~math.comb`{.interpreted-text role="external+py3.14:func"}
- `~math.factorial`{.interpreted-text role="external+py3.14:func"}
- `~math.gcd`{.interpreted-text role="external+py3.14:func"}
- `~math.isqrt`{.interpreted-text role="external+py3.14:func"}
- `~math.lcm`{.interpreted-text role="external+py3.14:func"}
- `~math.perm`{.interpreted-text role="external+py3.14:func"}

Their aliases in `math`{.interpreted-text role="external+py3.14:mod"}
will be `soft deprecated`{.interpreted-text role="term"}. This PEP
doesn\'t introduce backward-incompatible changes.

Module functions will accept integers and objects that implement the
`~object.__index__`{.interpreted-text role="external+py3.14:meth"}
method, which is used to convert the object to an integer number.
Suitable functions must be computed exactly, given sufficient time and
memory.

The `intmath`{.interpreted-text role="pypi"} package, available on PyPI,
will provide new submodule content for older Python versions.

# Possible Extensions

New functions (like mentioned in `pep791-motivation`{.interpreted-text
role="ref"} section) are not part of this proposal.

Though, we should mention that, unless we can just provide bindings to
some well supported mathematical library like the GMP, the submodule
scope should be limited. For example, no primality testing and
factorization, as production-quality implementatons will require a
decent mathematical background from contributors and belongs rather to
specialized libraries.

When proposed function already exists in the `gmpy2`{.interpreted-text
role="pypi"}, we should prefer a compatible interface for the stdlib.

# Backwards Compatibility

As aliases in `math`{.interpreted-text role="external+py3.14:mod"} will
be kept indefinitely (their use would be discouraged), there are no
anticipated code breaks.

# How to Teach This

The new submodule will be a place for functions, that 1) accept
`int`{.interpreted-text role="external+py3.14:class"}-like arguments and
also return integers, and 2) are also in the field of
arbitrary-precision integer arithmetic, i.e. have no dependency on the
platform floating-point format or behaviour and/or on the platform math
library (`libm`).

For users it would be natural first to look on the
`int`{.interpreted-text role="external+py3.14:class"}\'s methods, which
cover most basic use-cases (e.g. `int.bit_length`{.interpreted-text
role="external+py3.14:meth"} method), than to some dedicated place in
the stdlib.

# Reference Implementation

[python/cpython#133909](https://github.com/python/cpython/pull/133909)

# Rejected ideas

## isqrt() renaming

There was a brief discussion about exposing
`math.isqrt`{.interpreted-text role="external+py3.14:func"} as `sqrt` in
the new namespace in the same way that `cmath.sqrt`{.interpreted-text
role="external+py3.14:func"} is the complex version of
`math.sqrt`{.interpreted-text role="external+py3.14:func"}. However,
`isqrt` is ultimately a different function: it is the floor of the
square root. It would be confusing to give it the same name (under a
different submodule).

## Module name

[Polling showed](https://discuss.python.org/t/92548/67) `intmath` as
most popular candidate with `imath` as a second winner.

Other proposed names include `ntheory` (like SymPy\'s submodule),
`integermath`, `zmath`, `dmath` and `imaths`.

But the SC prefers a submodule rather than a new top-level module. Most
popular variants of the `math`{.interpreted-text
role="external+py3.14:mod"}\'s submodule are: `integer`, `discrete` or
`ntheory`.

# Acknowledgements

Thanks to Victor Stinner for sponsoring this PEP. Thanks to everyone who
participated in the discussions on discuss.python.org, providing
feedback, especially to Oscar Benjamin, Steve Dower and Paul Moore.

# Copyright

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
