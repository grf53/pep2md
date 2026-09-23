---
pep: 844
title: '``public`` and ``private`` builtins'
author:
- Barry Warsaw <barry@python.org>
discussions_to: https://discuss.python.org/t/108515
status: Draft
type: Standards Track
created: 05-Aug-2026
python_version: '3.16'
post_history:
- '`11-Aug-2026 <https://discuss.python.org/t/108515>`__'
- '`22-Sep-2026 <https://discuss.python.org/t/108515/152>`__'
python_status: Draft
url: https://peps.python.org/pep-0844/
source_path: https://github.com/python/peps/blob/main/peps/pep-0844.rst
source_commit: 1ff8e09615d1bf6755c36a81ae2f7820355bb122
---

# Abstract

This PEP proposes adding two new builtin functions, `public()` and
`private()`, which document the public interface of a module by keeping
its `__all__` synchronized with the names actually defined to be public
in that module. Both are used as decorators (`@public` and `@private`)
on class and function definitions, so that a name\'s visibility is
declared exactly once, at the point where the name is defined. Both
additionally have a single argument call form for names that are bound
by other means (such as `from ... import` statements), and `public()`
has a keyword argument form for names that cannot be decorated, such as
constants.

For example:

``` python
# spam.py
from strings import Bass

@public
class Public:
    ...

@private
class Private:
    ...

public(Bass)
public(TEMPO=120)
```

``` pycon
>>> import spam
>>> spam.__all__
['Public', 'Bass', 'TEMPO']
```

The proposed semantics are those of the third-party
[atpublic](https://public.readthedocs.io) package, which has provided
this functionality since 2016.

This PEP is an adjunct to `843`{.interpreted-text role="pep"}, and to
the withdrawn `842`{.interpreted-text role="pep"}; see [Relationship to
PEP 842 and PEP 843](#relationship-to-pep-842-and-pep-843).

# Motivation

The module global variable `~module.__all__`{.interpreted-text
role="attr"} is the mechanism Python currently defines for declaring a
module\'s public names. However, `__all__` suffers from a well-known
problem: it is typically defined as a separate list often far from the
objects whose names are contained in it. An object defined at one point
in the file is repeated as a string literal in an `__all__` list
somewhere else, usually at the top of the file.

Nothing keeps the two in sync, leading to these problems:

- Names get added to the module but never added to `__all__`.
- Names get removed from or renamed in the module but not in `__all__`,
  so `from spam import *` raises `AttributeError`{.interpreted-text
  role="exc"}.
- It\'s easy to typo a name (or leave out a list item delimiting comma)
  in `__all__`.
- Drift is only detectable in one direction. A linter can flag a name in
  `__all__` that doesn\'t exist as an object in the module, but no tool
  can flag a public name missing from `__all__`, because nothing in the
  source says that name was meant to be public.
- Readers of the code must scroll to a different part of the file (or a
  different screen) to answer \"is this name public?\"

The convention of prefixing private names with an underscore addresses a
related but different problem, and `842`{.interpreted-text role="pep"}
describes at length why `prefixing is not by itself a sufficient
answer <pep-842-why-not-prefixed-names>`{.interpreted-text role="ref"}.

The pattern proposed here \-- declaring visibility at the definition
site with a decorator \-- is not new or speculative. The `atpublic`
package on PyPI has implemented it for a decade, and it is already
depended on by a number of projects. What this PEP proposes is that the
pattern is common enough, and useful enough, to be spelled without a
third-party dependency. Thus it proposes to add `atpublic`\'s `public()`
and `private()` functions to the builtins.

## `__all__` already defines the public API {#pep-844-all-is-normative}

It is sometimes said that Python has no way to express which names in a
module are public and which are private, and that `__all__` is merely a
convention governing `from spam import *`. However, the
`language reference <import>`{.interpreted-text role="ref"} explicitly
says:

> The *public names* defined by a module are determined by checking the
> module\'s namespace for a variable named `__all__`; if defined, it
> must be a sequence of strings which are names defined or imported by
> that module. \[\...\] The names given in `__all__` are all considered
> public and are required to exist. If `__all__` is not defined, the set
> of public names includes all names found in the module\'s namespace
> which do not begin with an underscore character (`'_'`). `__all__`
> should contain the entire public API. It is intended to avoid
> accidentally exporting items that are not part of the API (such as
> library modules which were imported and used within the module).

This PEP explicitly adopts the definition of \"public\" in the Python
Language Reference. Following from this:

**The concept already exists and is normative.** \"Public name\" is a
term the language reference defines, and it defines it in terms of
`__all__`. This is specification text, not folklore. Python is not
missing a way to say what is public; it has one, and it is documented.

**Exhaustiveness is already the contract.** \"`__all__` should contain
the entire public API\" is unambiguous. A module whose `__all__` lists
only part of its public surface is not exercising some alternative
reading \-- it is out of conformance with what the reference says
`__all__` means.

**The imported-module problem is already in scope.** The reference names
it outright: `__all__` exists in part to avoid \"accidentally exporting
items that are not part of the API (such as library modules which were
imported and used within the module).\" A module that declares `__all__`
accurately and imports `argparse` does not leak the name `argparse` as
part of *its* public API, with or without renaming the import to
`_argparse`.

The gap, then, is not semantic but ergonomic. Python already specifies
what it means for a name to be public, and already recommends that
`__all__` say so exhaustively, while providing no convenient way to keep
that promise as a module evolves. It asks authors to maintain a list of
string literals by hand, in a different part of the file from the
definitions, which is subject to being quite error prone.

This PEP supplies the missing ergonomics. It does not redefine what it
means to be \"public\", or introduce a second notion of visibility, or
change what `__all__` already means. It doesn\'t try to redefine what it
means for a name to be exported. It does however make the documented
contract easy enough to actually honor.

# Specification

Two new builtins are added: `public()` and `private()`.

This PEP concerns module-level visibility only. `public()` and
`private()` declare which of a module\'s global names make up its public
interface, and they do so by maintaining `__all__`, which is defined for
modules and nothing else. Visibility in any other scope is explicitly
excluded: class attributes and methods, names local to a function, and
names bound in nested scopes are all untouched by this proposal. Python
has no `__all__` equivalent for those scopes, and this PEP does not
propose one. Whether a method is part of a class\'s public interface
remains, as today, a matter of naming convention and documentation.

## `public()`

`public()` has three call forms. The decorator form (`@public`) is the
most common use.

**Decorator form.** When called with a single positional argument that
has a `__name__` attribute (such as a function or a class), `public()`
appends that object\'s `__name__` to the `__all__` of the module in
which `public()` is called, and returns the object unchanged:

``` python
@public
def tune():
    ...

@public
class Cello:
    ...

# __all__ == ['tune', 'Cello']
```

Note that the bare decorator is used; Python\'s semantics are to
implicitly pass the object it decorates as the first argument to the
decorator function.

**Single argument form.** The same call written without the `@` appends
to `__all__` a name that is already bound in the module\'s globals, such
as a name bound by import from another module:

``` python
from strings import Bass
from reeds import Harmonica as Harp
from woodwinds import piccolo

public(Bass)
public(Harp)
public(piccolo)

# __all__ == ['Bass', 'Harp', 'piccolo']
```

The name appended is the one the object is bound to *in the calling
module\'s globals*, not necessarily the one it was defined with. So
`public(Harp)` appends `'Harp'`. Modules and submodules resolve the same
way, which is how a package exports a submodule. The argument is
returned unchanged.

When an object is bound to more than one name in the calling module, the
name it was defined with wins; by the time `public()` runs, nothing
distinguishes the two bindings:

``` python
class Fiddle:
    ...

Violin = Fiddle

public(Violin)

# __all__ == ['Fiddle']
```

To export an alias specifically, use the keyword argument form below.

The single argument can also be a string, which is appended as given:

``` python
public('Tuba')
```

The string must be a valid Python identifier and must not be a reserved
word, or `ValueError`{.interpreted-text role="exc"} is raised; nothing
can ever be bound to a reserved word, so such an entry in `__all__`
would be guaranteed to name something that can never exist. Soft
keywords such as `match` and `type` are ordinary names and are accepted.
Nothing else checks the string against the module\'s contents, so the
name need not be bound, or even exist. This is the escape hatch for
names that do not appear in the source, such as bindings made
dynamically or re-exports guarded by `try`/`except ImportError`. It
should be a last resort since string literals are precisely the kind of
thing that goes stale.

If no name can be inferred from the argument \-- such as for a constant
or an instance, neither of which has a `__name__` \--
`TypeError`{.interpreted-text role="exc"} is raised, with an error
message referring to the keyword argument form.

**Keyword argument form.** Names which can be neither decorated nor
inferred, such as constants, instances, and aliases, are declared by
calling `public()` with keyword arguments. Each keyword binds its value
in the calling module\'s globals *and* appends the name to `__all__`:

``` python
public(TEMPO=120)
public(a_cello=Cello())
public(Violin=Fiddle)
public(ROOT=1, FIFTH=5)
```

When used with a single keyword argument, the value is returned. For
multiple keyword arguments a tuple of the values is returned in order:

``` python
second, third, seventh = public(second=2, third=3, seventh=7)
ninth = public(ninth=9)
```

In all cases, `public()` modifies only the `__all__` of the module in
which it is called. No other module\'s `__all__` is ever affected.

If the module does not already define `__all__`, `public()` creates it
as an empty `list`{.interpreted-text role="class"} before appending. If
`__all__` exists but is not a list, `TypeError`{.interpreted-text
role="exc"} is raised; see `pep-844-all-must-be-list`{.interpreted-text
role="ref"}. Any strings already present in an existing `__all__` are
left in the list. Appending is idempotent, so a name that already
appears in `__all__` is not added a second time.

## `private()`

`private()` is the dual of `public()`\'s decorator and single argument
forms. It documents that a name is *not* part of the module\'s public
interface, and guarantees that the name does not appear in `__all__`,
removing it if it is already present. The argument is returned
unchanged:

``` python
import argparse

@private
def helper():
    ...

private(argparse)
```

The single argument form is how a module keeps an imported name it uses
internally, such as `argparse` above, out of an `__all__` that something
else in the module has created.

Names are resolved exactly as they are for `public()`: an object with a
`__name__`, a module, or a submodule resolves to the name it is bound to
in the calling module\'s globals, while a string must be a valid Python
identifier that isn\'t a reserved word and is otherwise taken as given.
The same `TypeError`{.interpreted-text role="exc"} and
`ValueError`{.interpreted-text role="exc"} conditions apply.

`private()` has no keyword argument form. Binding a name and declaring
it private in a single call would be a contradiction: the keyword form
of `public()` exists to introduce a name into the module globals, and
for `private()` there is nothing to remove from `__all__` that the call
itself just created.

Unlike `public()`, `private()` never creates `__all__`. If the module
does not define `__all__`, `@private` has no effect on the module
namespace at all; it serves purely to document the author\'s intent at
the point of definition. The argument is still resolved in that case,
and the result discarded, so that an argument no name can be inferred
from is rejected whether or not the module has an `__all__` yet;
otherwise a bad argument would sit unnoticed until something else in the
module created one. If `__all__` does exist it must be a list, or
`TypeError`{.interpreted-text role="exc"} is raised, and the name is
removed from it if present.

Declarations take effect in the order they execute, and the last one
wins. `public(X)` followed later by `private(X)` leaves `X` out of
`__all__`, while `private(X)` followed by `public(X)` puts it in.
Neither combination is an error. See
`pep-844-why-private`{.interpreted-text role="ref"}.

`@private` deliberately does not create an empty `__all__`, because
doing so would silently change the meaning of `from spam import *`. With
no `__all__`, a wildcard import binds every name not beginning with an
underscore; with `__all__ = []` it binds nothing. A decorator whose
purpose is documentation should not have that effect.

It follows that `@private` alone does not exclude a name from
`from spam import *`. Excluding names is the job of `@public`: as soon
as any name in the module is marked public, `__all__` exists, and
everything not marked public is excluded automatically. `@private`
records the author\'s intent; `@public` is what makes that intent
observable.

## Restrictions

Because `@public` and `@private` exist to keep the `__all__` module
global in sync, only module-level objects may be declared. Decorating a
method inside a class body is not supported, since `__all__` documents
module contents, not class contents.

Neither function inspects the scope it is called from, so this misuse is
not currently diagnosed. A decorator applied to a method appends the
method\'s name to the enclosing *module\'s* `__all__`, a single argument
call in a class body does the same, and the keyword argument form binds
its keywords in the module globals rather than in the class body. None
of these outcomes is likely to be what the author intended. Whether
these cases should raise an exception instead is an [Open
Issues](#open-issues) question.

Because `__all__` must be mutable for these functions to append to it, a
module that assigns `__all__` itself must assign a list. A module that
wants an immutable `__all__` can freeze it after the last declaration
with `__all__ = tuple(__all__)`. This constrains only those modules that
call `public()` or `private()`; see
`pep-844-all-must-be-list`{.interpreted-text role="ref"}.

# Rationale

## Why builtins?

The declaration of a module\'s public interface is a fundamental and
often requested property, especially as code bases grow. Many use cases
have been identified in discussions, and different approaches have been
developed in different libraries and applications. Enough experience has
been gained over the decade of `atpublic`\'s existence that requiring a
third-party dependency (or an import at the top of every module) to
spell something this fundamental is friction that discourages its use.
It is also awkward in exactly the places where it matters most: the
standard library itself, and small single-file modules.

To address this, `atpublic` offers an optional install extra
(`pip install atpublic[install]`) that injects `public` and `private`
into `builtins`{.interpreted-text role="mod"} at interpreter startup, so
that no import is needed. This extra pulls in a second distribution,
`atpublic-install`, which exists only to provide a legacy `.pth` file
and a `829`{.interpreted-text role="pep"} `.start` file that add the two
names to `builtins`{.interpreted-text role="mod"} as the interpreter
comes up. Because that extra is now a separate distribution, PyPI\'s
[download counts](https://pypistats.org/packages/atpublic-install) for
`atpublic-install`, relative to those for `atpublic` itself, give an
imperfect [^1] but ongoing measure of how much this convenience is
useful.

## Why decorators?

A decorator puts the declaration exactly where the definition is, which
is the entire point.

The mechanical benefit is that the name appears only once. It cannot
drift out of sync, refactoring tools rename it correctly for free, there
is no second list to maintain, and the need to repeat yourself largely
disappears.

The value of these decorators as source code documentation matters just
as much. `@public` and `@private` record the author\'s intent on the
line a reader is already looking at. Answering \"is this part of the
API?\" takes no scrolling to a list elsewhere in the file, no
cross-checking that list against the definitions, and no guessing about
whether a leading underscore was deliberate. The declaration stops being
bookkeeping attached to the definition and becomes part of it.

`@private` demonstrates this most clearly. In a module with no `__all__`
it does nothing mechanically at all: it adds no name, removes no name,
and changes no behavior. Its entire value is to say, at the point of
definition, that the name is deliberately not public.

## Why `private()`? {#pep-844-why-private}

Two objections to `private()` were raised repeatedly during discussion:
that it is redundant once `@public` exists, and that declaring the same
name both public and private should be an error rather than a silent
removal from `__all__`. This PEP keeps `private()` and its removal
behavior.

**Doing nothing mechanically is the point.** In a module that has an
`__all__`, everything not included explicitly is already excluded, so
`@private` changes no behavior. That is precisely the case in which its
value as source code documentation is highest. A reader looking at an
undecorated definition cannot tell whether the author considered its
visibility and decided against it, or never considered the question at
all. `@private` makes that decision explicit. The alternatives are a
`# private` comment, which no tool can reliably act on and which sits
asymmetrically beside the `@public` decorators elsewhere in the file, or
a leading underscore, which changes the name.

**It preserves the option to promote a name later.** Marking
`function()` private with a decorator rather than by naming it
`_function()` keeps the name stable. If it later joins the public API,
the change is to delete one decorator, rather than to rename the
function and leave an alias behind for the users who found the
underscore version anyway.

**It removes non-public names that come from elsewhere.** A module whose
`__all__` comes from somewhere else, whether written by hand or derived
from the `__all__` of other modules, may include names this module does
not want to export. `private()` removes them at the point where the
reason for removing them is obvious. `@public` cannot express this,
because there is nothing to add.

**Ordering follows ordinary Python semantics.** Declaring a name public
and later private is not indeterminate behavior. The calls run in the
order they are written, like every other statement in a module body, and
the last one wins. Rejecting the combination would mean carrying a
record of every prior declaration in order to diagnose a construct whose
meaning is already well defined. Whether it is ever good style is a
separate question, and one this PEP leaves to the author.

## Static analysis of the function call forms {#pep-844-static-analysis}

The strongest objection to this proposal concerns the keyword argument
form, and it is worth stating explicitly. Given:

``` python
public(TEMPO=120)
```

`TEMPO` is bound in the module\'s globals by a function that reaches
into its caller\'s frame. Nothing about that binding is visible to
static analyzers. A type checker, linter, or language server reading the
source sees a bare function call and no assignment, and will therefore
report `TEMPO` as undefined at every use site. A soft keyword like
`export TEMPO = 120`, as `842`{.interpreted-text role="pep"} proposed,
has no such problem, because syntax is by construction visible to
anything that parses the file. This, and not the DRY objection raised in
`842`{.interpreted-text role="pep"}, is the real cost of choosing a
builtin over a keyword.

The objection is specific to the keyword argument form. The single
argument form binds nothing: `public(Bass)` is an ordinary reference to
a name the module has already bound, so no tool can be misled into
thinking `Bass` is undefined. All a checker has to learn there is that
the call contributes a name to `__all__` \-- exactly what it already
learns from `__all__.append('Bass')`, and with the same information
available in the source.

This could easily be rectified by future modifications to linting tools,
so that they explicitly recognize the keyword argument form of
`public()`. This would be a one-time, bounded cost paid by a handful of
tools, not an ongoing cost paid by every Python programmer.

`public()` is not an arbitrary function performing mysterious magic. It
is a builtin with a small, fixed, specified signature, and its effect on
the module namespace is fully determined by the keyword names at the
call site, which are *literally present in the source*. Teaching a
checker that `public(TEMPO=120)` binds `TEMPO` and appends `"TEMPO"` to
`__all__` is a simple analysis that these tools could easily perform.

There is direct precedent. Static analyzers already model `__all__`
mutation beyond simple assignment, including `__all__ += [...]` and
`__all__.append(...)`, precisely because real code does this. They
already special-case namespace-creating callables whose behavior is not
evident from the grammar, such as
`~collections.namedtuple`{.interpreted-text role="func"},
`~typing.TypedDict`{.interpreted-text role="class"}, and
`~dataclasses.dataclass`{.interpreted-text role="func"}. Adding
`public()` to that list is an increment on work these tools have already
done, not a new category of problem.

If this PEP is accepted, that support is expected to follow quickly, for
the ordinary reason that tools support what the language provides. In
the interim (and for older tool versions) the return value of `public()`
gives an entirely explicit spelling that requires no special support at
all:

``` python
TEMPO = public(TEMPO=120)
```

Here the binding is a plain assignment, visible to every tool that
parses Python. This form is a transition aid rather than the recommended
spelling, and it should not be needed for long.

The conclusion is that the data and type alias use cases, which are the
places a decorator genuinely cannot be utilized, do not require new
syntax at all. A function call that tools can recognize serves just as
well, without the need for a new, dedicated `export` keyword.

## Is this urgent? {#pep-844-urgency}

[Guido van Rossum](https://discuss.python.org/t/108353/46) raised this
question about `842`{.interpreted-text role="pep"}, and it applies with
equal force here:

> But Python has existed without this feature for over 35 years \-- is
> it really urgent? Remember the Zen of Python, which says \"Now is
> better than never. Although never is often better than *right* now.\"

No. This PEP is not urgent, and it does not claim to be. Nothing about
module name visibility, or about a module\'s exported public API, is
urgent. But urgency is the wrong test to apply to this particular
proposal, for three reasons.

**The feature is not new.** This PEP does not ask Python to adopt an
untried idea; `atpublic` has implemented these exact semantics since
2016. The question is not \"should Python have this?\" since users who
want it already have it, but \"should having it cost a third-party
dependency?\" A decade of production use is the opposite of rushing. It
has already surfaced and settled the corner cases, syntax, and semantics
a fresh design would have to guess at: that only module-level objects
can be decorated, what to do about a non-list `__all__`, which module\'s
`__all__` a re-exported name belongs in and which of its names to use,
and what each call form should return.

**The cost of being wrong is low.** The urgency argument has the most
weight against changes that cannot be walked back. Syntax is permanent:
a soft keyword constrains the grammar forever, must be taught to every
future Python programmer, and is unavailable to any module supporting an
older interpreter. A new module-level variable with runtime consequences
changes the observable behavior of code without warning. A builtin
function is the cheapest thing in this design space on both counts: it
is inert until called, it changes nothing about modules that ignore it,
and if it proves to be a mistake it can be deprecated in the ordinary
way without touching the grammar.

**The sequencing matters more than the timing.** Three proposals in this
cycle addressed the same problem space, two of them asking for new
syntax. `842`{.interpreted-text role="pep"} has since been withdrawn,
but `843`{.interpreted-text role="pep"} remains, and the point is
unchanged: if Python is going to change its grammar to address this
need, that decision should be made *after* weighing the option that
requires no grammar change, not before. Once an `export` keyword exists,
builtins covering the same ground are redundant and may never be added,
regardless of whether they were the better answer. That asymmetry is the
reason to consider this PEP now rather than later.

## Import-time performance {#pep-844-performance}

When this idea was informally floated with core developers some years
ago, before either `842`{.interpreted-text role="pep"} or
`843`{.interpreted-text role="pep"} existed, the objection raised was
not the design but the cost weighed against its utility: a decorator
runs at import time, once per decorated name, and CPython\'s startup
time is a closely watched number. The concern is legitimate and deserves
a direct answer.

**The work per call is small and bounded.** `public()` in decorator form
reads the decorated object\'s `__name__`, obtains the calling module\'s
globals, creates `__all__` as an empty list if needed, and appends one
string. There is no complicated introspection, no allocation or work
proportional to module size, and no I/O. Whatever the constant factor
turns out to be, it does not grow with the size of the module.

**The cost is opt-in and proportional to the public API.** A module that
does not call `public()` pays nothing at all, unlike a change to module
attribute access, which affects every module whether or not it
participates. A module that does call it pays once per *public* name,
and a module\'s public surface is typically a small fraction of the
names it defines.

**Syntax is not free either.** It is worth being precise about what the
alternative saves. `842`{.interpreted-text role="pep"}\'s `export`
statement was specified to check that the name exists in globals, create
`__export__` if absent, and call `list.append` \-- the same operations,
expressed in bytecode rather than a call. The saving is the function
call dispatch, not the underlying work. That is a real difference, but
it is a constant factor on an already small constant, not a difference
in kind.

**A C implementation is feasible and fast.** This is the point on which
a builtin is strictly better positioned than the third-party package.
`atpublic` shipped a C implementation of `public()` for a time, and it
was substantially faster than the pure Python version. It was ultimately
dropped, not because it did not work, but because requiring a compiled
extension module in a third-party package is a significant packaging and
installation burden for a library this small \-- a burden borne entirely
so that the pure Python fallback could be avoided.

That trade-off does not exist in CPython. A builtin is compiled as part
of the interpreter, so the fast implementation is simply *the*
implementation, with no wheel platform support matrix, no fallback path,
and no optional extra. Moreover, a C implementation inside the
interpreter can do less work than any third-party one: it has the
calling frame in hand and can read its globals directly, where a pure
Python implementation must call `sys._getframe`{.interpreted-text
role="func"} on every call, as `atpublic` (as of version 8.0.1) does in
all three forms. The name resolution that the single argument form
performs is the same work either way. What the builtin saves is the
frame lookup and the Python-level call itself.

The argument is therefore somewhat the reverse of the original
objection. The performance concern is a reason to put `public()` in
builtins where it can be made fast, rather than a reason to leave it on
PyPI, where it cannot.

:::: note
::: title
Note
:::

This section argues that the cost is acceptable; it does not yet
demonstrate it. Measurements against CPython\'s startup benchmarks, for
both a decorated standard library and a synthetic worst case, should
accompany the reference implementation. See [Open Issues](#open-issues).
::::

# Relationship to PEP 842 and PEP 843

In brief: `842`{.interpreted-text role="pep"} proposed adding an
`export` keyword and a new module global `__export__` variable, with
runtime enforcement of the resulting declaration.
`843`{.interpreted-text role="pep"} proposes adding a
`from ... export ...` form for re-exports, which populates `__all__`.

`842`{.interpreted-text role="pep"} has since been withdrawn. Its
author\'s stated reason is that the proposal grew out of a need to
improve standard library maintenance, and the solution it described
\"did not align with the needs of third-party packages.\" Its material
is retained in the comparisons below because the questions it raised
about `__all__` outlive it, and because this PEP\'s design is in part a
response to them.

## Two problems, not one

Discussion of module visibility addresses two separable problems:

1.  **Bookkeeping.** A name\'s visibility as public or private is
    declared in a different place from where the object so named is
    defined, so the declaration drifts out of sync with the
    implementation. This is a problem about *where you add the
    declaration*.
2.  **Runtime consequences.** `__all__` declares the public API, but the
    only place that declaration is enforced is `from spam import *`. It
    has no effect on attribute access, `dir`{.interpreted-text
    role="func"}, `help`{.interpreted-text role="func"}, or
    autocompletion, so a name that is intended to be kept private is
    indistinguishable from public names to these patterns of module
    introspection. This is a problem about *what the declaration does*.

This PEP addresses only the first. It takes the position that the first
problem is the more pressing and the more broadly applicable of the two,
that it can be solved without new syntax and without a new variable, and
that solving it does not commit Python to any particular answer to the
second.

## Why `__all__` and not `__export__` {#pep-844-why-all}

`842`{.interpreted-text role="pep"} proposed a new `__export__`
variable. This PEP proposes to keep using `__all__`. The choice between
reusing `__all__` and introducing a second variable is relevant
regardless of that PEP\'s withdrawal, so the reasoning is set out here
in full.

`842`{.interpreted-text role="pep"} gave two reasons why `__all__` is
inadequate. The first is that `__all__` drifts out of sync with the
module. That is true, and it is precisely the problem `atpublic` and
this PEP solve. However, a *new list of string literals in the same
distant part of the file* does not directly solve this problem.
`842`{.interpreted-text role="pep"}\'s own revision history concedes the
point, quoting [Guido van
Rossum](https://discuss.python.org/t/108353/46) on the original
`__export__`-only design:

> But the ergonomics are similar to those of `__all__`, and those are
> bad. It\'s too easy to forget to add (or remove!) something to the
> list, and it\'s distracting to have to update the export info in a
> totally different part of a file than the definition of the exported
> thing.
>
> If we just cared about classes and functions, a more ergonomic
> approach would be an `@export` decorator. If we also care about
> exporting data or type aliases, I\'d much rather look for a solution
> that adds a soft keyword named `export` (or `private`, for a better
> default).

Drift is a property of *declaring at a distance*, not a property of
`__all__`. Any variable maintained by hand has it, and no variable
maintained at the definition site does.

The first half of that quote is the argument this PEP is built on, and
the second half names the decorator as the ergonomic answer for classes
and functions. The remaining question \-- what to do about data and type
aliases, where there is nothing to decorate \-- is addressed in
`pep-844-static-analysis`{.interpreted-text role="ref"}.

The second reason is that `__all__` is not always exhaustive in
practice. A module may deliberately keep a public type alias out of
`__all__` to avoid polluting wildcard-importing namespaces, so its
public API can end up a *superset* of what `__all__` lists.

That is an accurate observation about existing code, but it is a weaker
argument than it first appears, because it describes a *deviation from
the specification* rather than an alternative reading of it. As
`pep-844-all-is-normative`{.interpreted-text role="ref"} sets out, the
language reference already states that `__all__` \"should contain the
entire public API.\" A module that withholds public names from `__all__`
is not asserting that `__all__` means something narrower than the public
API; it is trading conformance away for control over `import *`.

What that trade exposes is a real flaw, but a different one from the one
`842`{.interpreted-text role="pep"} diagnosed: `__all__` does double
duty. It is at once the declaration of what is public and the control
surface for wildcard imports, and when those two purposes conflict,
authors sacrifice the declaration because only the wildcard behavior has
any teeth.

Introducing `__export__` does not repair that conflation. It leaves
`__all__` doing both jobs, adds a second declaration to keep
synchronized with the first, and transfers the word \"public\" to the
new module variable, while the language reference continues to define it
in terms of `__all__`. A module conscientious enough to maintain
`__export__` accurately would have been conscientious enough to maintain
`__all__` accurately; the ones that drift will drift in both.

This PEP takes no position on whether unexported-name warnings are
desirable. It observes only that the bookkeeping question is separable
from the runtime-semantics question, and it answers the former.
`public()` populates a list; if Python later decides that some list
should carry runtime consequences, `public()` can populate that one
instead, or both. Nothing here closes the door on a future proposal
along `842`{.interpreted-text role="pep"}\'s lines.

## Why PEP 843 is a good companion

This PEP narrows the DRY problem for re-exports, but it does not close
it and cannot close it gracefully. A \"hub module\" that pulls names out
of private submodules writes each name twice:

``` python
from ._core import Widget
public(Widget)
```

Manually maintaining `__all__` also names `Widget` twice, once in the
import and once as a string literal, so the single argument form is an
improvement in kind rather than in count: the second mention comes from
the object\'s `__name__` itself, so typos are impossible. The name
survives refactoring, and a checker can see the binding. It is still a
second mention on a second line, one per exported name, and a hub that
exports three hundred names carries three hundred of them.

`from ._core export Widget` names it once, in the statement that had to
be there anyway. That is exactly the gap `843`{.interpreted-text
role="pep"} identifies, and no call form can ergonomically close it
[^2]. In an import statement, the decorator form has nothing to
decorate, the single argument form needs the name as an argument, and
the keyword form needs it on both sides. Aliases show the same shape \--
`843`{.interpreted-text role="pep"} writes
`from ._core export Widget as PublicWidget`, where this PEP needs the
import followed by `public(PublicWidget)`.

The two proposals therefore partition the problem cleanly, and provide
excellent synergy:

- `public()` and `private()` handle the names a module **defines**.
- `from <module> export <name>` handles the names a module **passes
  through**.

Both populate `__all__`. Neither requires the other, and neither
requires new runtime semantics for the result.

:::: note
::: title
Note
:::

`843`{.interpreted-text role="pep"} was published as this PEP was being
drafted, and is now in its second round of discussion.
`842`{.interpreted-text role="pep"}, which had grown an `export`
statement of its own overlapping both this PEP and
`843`{.interpreted-text role="pep"}, has since been withdrawn. What
remains to be settled is therefore the relationship between this PEP and
`843`{.interpreted-text role="pep"}; see [Open Issues](#open-issues).
::::

# Backwards Compatibility

Adding names to `builtins`{.interpreted-text role="mod"} shadows
nothing, but it does mean that modules which define their own
module-level `public` or `private` names will shadow the builtins
instead. This is the same situation as any other builtin (`id`, `type`,
`list`), and is well understood.

Code that imports `public` and `private` from the `atpublic` package
will continue to work unchanged (as long as the semantics continue to
match), since an explicit import shadows the builtin.

Code that already uses `public` or `private` as a variable or parameter
name will begin to trip linters that flag shadowed builtins, such as
`flake8-builtins` and the equivalent `ruff` rule. This is a diagnostic
change rather than a behavioral one, and the same has been true of every
builtin added to Python. How much existing code this affects has not
been measured.

Modules using these builtins will not run on Python 3.15 and earlier
without either a dependency on `atpublic` or a compatibility shim.

# Security Implications

This PEP has no known security implications. Like `__all__` itself,
`public()` and `private()` are documentation, not access control.

# How to Teach This

`public()` and `private()` would be documented alongside the other
builtins, and referenced from the tutorial section on modules where
`__all__` is introduced.

The rule to teach is a single sentence: decorate a class or function
definition with `@public` if users of your module are meant to use it,
and don\'t decorate it (or decorate it with `@private`, to say so
explicitly) if they aren\'t.

Constants and other names that cannot be decorated use the keyword
argument form, which both binds the name and marks it public:

``` python
public(TEMPO=120)
```

This replaces the assignment rather than accompanying it. Writing
`TEMPO = 120` as well would define the name twice, which is the
repetition these builtins exist to remove.

Names that arrive by import are declared by passing the object itself,
after the import that bound it:

``` python
from ._core import Widget

public(Widget)
```

The rule to teach for the two positional spellings is that they are the
same call: `@public` is what you write when you are defining the name
here, and `public(name)` is what you write when it is already bound.

Adoption can be incremental. A module with a hand-written `__all__` can
start decorating definitions without removing it, because names already
listed are not added twice, and the two styles can coexist indefinitely.

# Reference Implementation

The [atpublic](https://public.readthedocs.io) package, available on PyPI
and maintained since 2016, implements the proposed semantics in pure
Python. Its [source repository](https://gitlab.com/flufl/public) is
hosted on GitLab. The specification above describes `atpublic` 8.0.1,
released on 21-Sep-2026.

A CPython PR has not yet been written.

For a time, `atpublic` also included a C implementation of `public()`,
which was considerably faster than the pure Python one. It was dropped
for packaging reasons that do not apply to a builtin. See
`pep-844-performance`{.interpreted-text role="ref"}.

Three changes in 8.0.0 are worth calling out, because 7.0.0 and earlier
diverge from the specification above:

- `@private` no longer creates `__all__` when the module does not
  already have one. Leaving an empty `__all__` behind is not the same
  thing as not adding one, and the difference is observable in
  `from spam import *`. This was identified as a bug while drafting this
  PEP.
- `public(thing)` and `private(thing)` used to resolve against
  `sys.modules[thing.__module__]`, the module where `thing` was
  *defined*. For a decorator those are the same module, but passing an
  imported object added the name to the wrong module\'s `__all__`. Both
  functions now always use the globals of the module where the call
  appears, as specified above.
- The single argument form is consequently new as a supported spelling
  in 8.0.0, along with the string form and the resolution rules given
  above. Before that, a re-export had to be written
  `public(Widget=Widget)`, which is the spelling earlier drafts of this
  PEP specified.

# Rejected Ideas

## New `export` syntax instead of decorators

Before its withdrawal, `842`{.interpreted-text role="pep"} proposed an
`export` soft keyword covering the same ground as this PEP, for example
`export def`, `export class`, `export NAME = value`. Its `Rejected
Ideas <842#rejected-ideas>`{.interpreted-text role="pep"} section
considered builtin `public` and `private` decorators, described them as
the author\'s next preferred alternative to syntax, and rejected them on
the grounds that \"there\'s no easy way to export simple variables
without duplicating the name.\" The objection outlives the PEP that
raised it, and is answered here.

That objection doesn\'t fully apply to the design proposed here. The
keyword argument form exists precisely for the cases which can\'t be
decorated, and writes the name exactly once:

``` python
public(TEMPO=120)
```

is the whole declaration. The name `TEMPO` is bound to `120` in the
module globals, and `"TEMPO"` is appended to `__all__`. There is no
separate assignment to keep in sync. Compare `export TEMPO = 120`: the
two spellings carry the same information, cost roughly the same
keystrokes, and differ only in that one of them requires a grammar
change.

The substantive version of the objection is not about keystrokes but
about tooling: a soft keyword is visible to static analysis, while a
function call that binds through its caller\'s frame is not. That is a
real cost, and it is answered in
`pep-844-static-analysis`{.interpreted-text role="ref"}.

The general argument holds beyond this example. New syntax is the most
expensive thing Python can add: it must be taught, it cannot be
back-ported, it constrains the grammar permanently, and it is
unavailable to every module that must still run on an older interpreter.
A builtin costs none of that, is trivially shimmed on old versions, and
(as is the case here) has a decade of usage experience behind it.

## Add a new `__export__` variable

See `pep-844-why-all`{.interpreted-text role="ref"}.

## Accept an `__all__` that is not a list {#pep-844-all-must-be-list}

The [Specification](#specification) requires `__all__` to be a list for
`public()` and `private()` to operate on it, which is stricter than
either the Python language or the CPython implementation. The language
reference only requires it to be \"a sequence of strings\", and CPython
is even looser: `from spam import *` iterates over `__all__` until it
raises `IndexError`{.interpreted-text role="exc"}, so a tuple, a string,
and a custom object with a suitable
`~object.__getitem__`{.interpreted-text role="meth"} are all legal
today.

The restriction that `__all__` be a list is deliberate and local to this
PEP, and applies only to modules that call `public()` or `private()`.
This PEP does not propose any change to what `__all__` may be in either
the language reference or CPython implementation.

Accepting any sequence is not implementable. There is no general way to
append to an arbitrary object that merely supports `__getitem__`, so
\"whatever the import system accepts\" cannot be the rule. Any rule
broader than *list* is an arbitrary line drawn somewhere short of what
the language permits, and the obvious candidate, *list or tuple*, is
unworkable:

- `tuple`{.interpreted-text role="class"} has no `__iadd__`. While
  `__all__ += (name,)` genuinely extends a list in place, it only
  appears to work for a tuple, because the augmented assignment
  statement falls back to `__add__` and rebinds the name. That is
  behavior of the statement, not a protocol a function can require of
  its argument.
- `public()` could rebind the name in the calling module\'s globals
  itself, but then every declaration copies the whole tuple, which is
  quadratic in the size of the public API and paid at import time. See
  `pep-844-performance`{.interpreted-text role="ref"}.
- `private()` could not be made symmetric. Python has no `-=` on tuples,
  so a tuple `__all__` would accept declarations but refuse removals.

Officially deprecating the non-list case was also considered and
rejected. `public()` would accept a non-list `__all__` indefinitely,
issue a `DeprecationWarning`{.interpreted-text role="exc"}, and decline
to append the name. Declining to append yields a silently incomplete
public API, reported only through a warning category that is suppressed
by default. Deprecating a working construct is out of scope for this
PEP.

The requirement that `__all__` be a list incurs minimal costs, and
affects only modules opting in to this PEP. A module that wants an
immutable `__all__` can still get one by freezing it after the last
declaration with `__all__ = tuple(__all__)`. Two long-time users of a
tuple `__all__` said on the discussion thread that the constraint does
not trouble them: [one](https://discuss.python.org/t/108515/141)
concluded, after some weeks of discussion, that every argument for
supporting tuples they had thought of was \"baseless or petty\", and
[another](https://discuss.python.org/t/108515/142) offered to switch to
lists. The \"list rule\" is also the easiest one for static analysis to
follow, which matters for a proposal whose value depends on tools
recognizing it: `__all__` is a list, built by a literal, by `append`, or
by these declarations.

## Leave it on PyPI

Leaving `atpublic` on PyPI is the status quo option. Users who want to
opt into this functionality can simply add that library as a dependency
and import the functions (or use the `pip install atpublic[install]`
extra, which pulls in the companion `atpublic-install` distribution to
populate builtins at startup).

However, if this *is* a problem worth solving now, then leaving this in
a third-party package on PyPI doesn\'t serve our users adequately. The
need to include a dependency and an explicit import may be just enough
of a hurdle (albeit small) to stop widespread use of it. Adding it to
builtins endorses the pattern in a way that should broaden its adoption.

## A new standard library module instead of builtins

This would eliminate the third-party dependency problem, but still
leaves the explicit import usability cost. In addition, there\'s no
obvious place to add it to the stdlib *other than* in builtins. Two
functions likely aren\'t worth the cost of a new top-level module.
Besides, since `__all__` is in a sense built into Python, these
functions should be built in too.

# Open Issues

- How should this PEP and `843`{.interpreted-text role="pep"} be
  reconciled? With `842`{.interpreted-text role="pep"} withdrawn, the
  two remaining proposals overlap only on re-exports, where this PEP\'s
  single argument form and `843`{.interpreted-text role="pep"}\'s
  `from ... export ...` statement do the same job with different costs.
  Whether both are wanted, and in what order they should be considered,
  needs to be settled on the discussion threads.
- Should `populate_all()`, `atpublic`\'s heuristic to infer `__all__`
  from the module\'s own definitions, also be included? This is deferred
  for now; a heuristic is a harder case to make for a builtin than the
  two explicit declarations are, and is less essential for improving
  module visibility ergonomics.
- Should `public()` and `private()` diagnose being called outside module
  scope? Neither inspects its calling scope today, so `@public` on a
  method silently adds the method\'s name to the module\'s `__all__`.
  Raising an exception would be friendlier, at the cost of a scope check
  on every call, which bears on `pep-844-performance`{.interpreted-text
  role="ref"}.
- Should the standard library itself adopt these decorators, and if so
  on what schedule? This question is entangled with
  `pep-844-performance`{.interpreted-text role="ref"} and should be
  settled with startup measurements in hand. Also, as with all new
  capabilities (such as lazy imports), Python\'s policy is generally not
  to wholesale update the stdlib to embrace the new functionality. These
  new functions can be utilized opportunistically in modules where the
  most benefit can be gained, or when a module undergoes substantial
  rewrite.
- Import-time benchmarks for a C implementation are outstanding.

# Acknowledgements

Thanks to Peter Bierma and Neil Girdhar, whose `842`{.interpreted-text
role="pep"} and `843`{.interpreted-text role="pep"} prompted this
proposal, and to the contributors to and users of `atpublic` over the
past decade.

# Footnotes

# Change History

- [22-Sep-2026](https://discuss.python.org/t/108515/152)
  - Update references to `842`{.interpreted-text role="pep"} (since
    withdrawn) and `843`{.interpreted-text role="pep"} (since
    published).
  - Synchronize this PEP\'s proposed semantics with `atpublic` 8.0.0,
    and its reference implementation versions with the 8.0.1 and
    `atpublic-install` 1.0.0 releases of 21-Sep-2026.
  - Reject non-list `__all__` outright. The list requirement is kept,
    and only affects those modules that opt in to this PEP by calling
    `public()` or `private()`.
  - Say what the `atpublic[install]` extra costs outside the
    interpreter, now that it pulls in a second distribution to populate
    `builtins`{.interpreted-text role="mod"} at startup.

# Copyright

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.

[^1]: The count is a lower bound. It misses anyone who calls `install()`
    directly, and anyone who simply imports the two names, neither of
    which needs the extra at all.

[^2]: Except for a function call that *also* does the import, but
    that\'s even more magical.
