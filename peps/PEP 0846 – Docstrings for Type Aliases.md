---
pep: 846
title: Docstrings for Type Aliases
author:
- Bartosz Sławecki <bartosz@ilikepython.com>
sponsor: Jelle Zijlstra <jelle.zijlstra@gmail.com>
discussions_to: https://discuss.python.org/t/pep-846-docstrings-for-type-aliases/109116
status: Draft
type: Standards Track
topic: Typing
created: 06-Sep-2026
python_version: '3.16'
post_history:
- '`06-Sep-2026 <https://discuss.python.org/t/docstrings-for-type-aliases/108901>`__'
- '`18-Sep-2026 <https://discuss.python.org/t/pep-846-docstrings-for-type-aliases/109116>`__'
python_status: Draft
url: https://peps.python.org/pep-0846/
source_path: https://github.com/python/peps/blob/main/peps/pep-0846.rst
source_commit: ac723cfc3c0c44a070c426db86e66934f9276510
---

# Abstract

This PEP proposes preserving a string literal immediately following a
`type`{.interpreted-text role="py:keyword"} statement as the resulting
type alias object\'s `__doc__` attribute, exposing that documentation
through `ast`{.interpreted-text role="py:mod"}, and displaying it
through `pydoc`{.interpreted-text role="py:mod"} and
`help`{.interpreted-text role="py:func"}. It follows the placement
already supported by source-based documentation tools. The parser stores
the docstring in a new optional `doc` field on
`ast.TypeAlias`{.interpreted-text role="py:class"} instead of creating a
separate `ast.Expr`{.interpreted-text role="py:class"} node for it.
`ast.get_docstring`{.interpreted-text role="py:func"} retrieves alias
docstrings from this field.

# Motivation

Several widely used development tools already recognize docstrings
following type alias declarations. [Pyright supports docstrings
following type
statements](https://discuss.python.org/t/docstrings-for-new-type-aliases-as-defined-in-pep-695/39816/5)
(since 2023), [Sphinx\'s autotype
directive](https://www.sphinx-doc.org/en/master/usage/extensions/autodoc.html#automatically-document-type-aliases)
documents aliases and their docstrings (since 2025), and [Pylint
recognizes these strings as
documentation](https://pylint.readthedocs.io/en/latest/whatsnew/3/3.0/index.html#what-s-new-in-pylint-3-0-3)
(since 2023).

For example, an alias can explain how callers should interpret its
values:

``` python
type Timeout = float | None
"""
Maximum wait in seconds.

Use None to wait indefinitely, or zero to return immediately.
"""
```

Calling `help(Timeout) <help>`{.interpreted-text role="py:func"}
displays generic information about
`~typing.TypeAliasType`{.interpreted-text role="py:class"} [rather than
the alias\'s
docstring](https://github.com/python/cpython/issues/156925). A tool that
needs the alias\'s documentation must find and parse its source, which
may be unavailable after installation or when the alias is passed in
from another component.

The `type`{.interpreted-text role="py:keyword"} statement introduced by
`695`{.interpreted-text role="pep"} creates a dedicated runtime object.
That object can carry its own documentation, as functions and classes
do. Preserving the docstring would make it available from the imported
alias alone, including when the alias is re-exported.

Runtime documentation could also be consumed by third-party frameworks.
Frameworks that already recognize
`~typing.TypeAliasType`{.interpreted-text role="py:class"} (such as
Pydantic) could choose to use `__doc__` as descriptive metadata. Such
integrations would be up to those projects.

# Specification

## Docstring Placement

`257`{.interpreted-text role="pep"} defines the convention of placing
attribute docstrings immediately after assignments and calls strings
following another docstring \"additional docstrings\". This PEP uses the
same placement convention for `type`{.interpreted-text
role="py:keyword"} statements: a string literal immediately following
the statement becomes the alias\'s docstring.

If the next logical line after a `type`{.interpreted-text
role="py:keyword"} statement in the same block is an expression
statement consisting of a string literal, that string is the alias\'s
docstring and is part of the `type`{.interpreted-text role="py:keyword"}
statement. Comments and blank lines may appear between the
`type`{.interpreted-text role="py:keyword"} statement and its docstring.
A string literal on the same line as the alias, separated by a
semicolon, does not qualify.

``` python
type Timeout = float | None
"""Maximum wait in seconds."""

type OtherTimeout = float | None
default_timeout = 30
"""This is not OtherTimeout's docstring."""
```

The rule applies wherever a `type`{.interpreted-text role="py:keyword"}
statement is allowed, including inside functions, classes, and
control-flow blocks. The string must be in the same block as the alias.
A string in a nested or enclosing block does not qualify. Generic
aliases follow the same rule:

``` python
type ListOrSet[T] = list[T] | set[T]
"""A collection whose order and duplicate handling depend on its type."""
```

As with modules, functions, and classes, a docstring must be an
expression statement whose value is a string constant. Adjacent string
literals combined by the parser qualify, as does a parenthesized string
literal. Bytes literals, f-strings, t-strings, and expressions such as
`"first" + "second"` do not qualify, even if compilation could reduce an
expression to a constant string.

Only the first following string statement supplies `__doc__`. Additional
docstrings, in the `257`{.interpreted-text role="pep"} sense, remain
ordinary expression statements. They are not concatenated or assigned to
the alias.

## Runtime Behavior

The alias stores its docstring in `__doc__`. An undocumented alias has
`__doc__` equal to `None`. Accessing this attribute does not evaluate
the alias\'s value.

Compilation applies the same docstring whitespace processing as it does
for function and class docstrings. This expands tabs and cleans
indentation while retaining surrounding blank lines.
`inspect.cleandoc`{.interpreted-text role="py:func"} also removes
surrounding blank lines.

After an alias is created, its `__doc__` attribute can be reassigned.
Deleting it resets it to `None`. The
`~typing.TypeAliasType`{.interpreted-text role="py:class"} constructor
gains a keyword-only `doc` parameter, defaulting to `None`, that
initializes `__doc__` without whitespace processing:

``` python
from typing import TypeAliasType

Timeout = TypeAliasType(
    "Timeout", float | None, doc="Maximum wait in seconds."
)
```

This proposal requires no changes to type checker behavior. An alias\'s
docstring does not affect its meaning to a type checker or how its value
is evaluated at runtime.

## Optimization

Optimization level 2, selected by `-OO`{.interpreted-text role="option"}
or `compile(..., optimize=2) <compile>`{.interpreted-text
role="py:func"}, strips alias docstrings as it strips function and class
docstrings. The resulting alias has `__doc__` equal to `None`. In the
AST, preprocessing clears the `doc` field of
`ast.TypeAlias`{.interpreted-text role="py:class"}. Optimization levels
0 and 1 retain the docstring.

Assignments to `__doc__` remain ordinary runtime assignments and are not
stripped by `-OO`{.interpreted-text role="option"}.

## AST Support

The grammar of the `type`{.interpreted-text role="py:keyword"} statement
gains an optional trailing docstring, shown schematically:

``` peg
type_alias:
   | "type" NAME [type_params] '=' expression [NEWLINE type_alias_docstring]
```

Here, `type_alias_docstring` denotes an expression statement whose value
is a string constant, as defined under [Docstring
Placement](#docstring-placement).

This PEP adds an optional string field, `doc`, at the end of
`ast.TypeAlias`{.interpreted-text role="py:class"} (`name`,
`type_params`, `value`, `doc`). An omitted `doc` defaults to `None`.

When docstrings are retained, `ast.parse`{.interpreted-text
role="py:func"} populates this field with the original string, before
compilation\'s whitespace processing. The docstring does not appear as a
following `Expr(Constant(...))` statement. The node\'s end position
covers the docstring:

``` pycon
>>> import ast
>>> tree = ast.parse(
...     'type Timeout = float | None\n"Maximum wait in seconds."'
... )
>>> len(tree.body)
1
>>> tree.body[0].doc
'Maximum wait in seconds.'
>>> tree.body[0].end_lineno
2
```

`ast.get_docstring`{.interpreted-text role="py:func"} accepts
`~ast.TypeAlias`{.interpreted-text role="py:class"} nodes. As with the
node kinds it already supports, its default behavior cleans the
docstring using `inspect.cleandoc`{.interpreted-text role="py:func"}.
With `clean=False`, the function returns the original string. It returns
`None` for an undocumented alias.

Both `ast.dump`{.interpreted-text role="py:func"} and AST
`repr`{.interpreted-text role="py:func"} display the docstring in the
`doc` field. By default, `ast.dump`{.interpreted-text role="py:func"}
omits the field when its value is `None`, as it does for other optional
fields.

Code generation reads the `doc` field, and it does not inspect
neighboring statements. For programmatically constructed
`ast.TypeAlias`{.interpreted-text role="py:class"} nodes, the `doc`
field supplies the alias\'s docstring. `ast.unparse`{.interpreted-text
role="py:func"} emits the docstring as a string statement on the line
after the alias, so that parsing the result populates the field again.

## Standard Library Support

`pydoc`{.interpreted-text role="py:mod"}, including
`help`{.interpreted-text role="py:func"}, will recognize type aliases
and display their own documentation. Aliases will also be distinguished
from other data members in module documentation. This applies to both
text and HTML output.

For the opening example, the reference implementation displays:

``` text
Help on type alias Timeout in module mymodule:

type Timeout = float | None
    Maximum wait in seconds.

    Use None to wait indefinitely, or zero to return immediately.

    Lazy value access:

    __value__
        Lazily evaluated value of the type alias.

    evaluate_value
        Evaluation function for __value__.

    See help(typing.TypeAliasType) for the full type alias interface.
```

This PEP does not add automatic discovery of type alias docstrings to
`doctest`{.interpreted-text role="py:mod"}. Supporting this would
require a separate change to its discovery rules.

# Rationale

Placing the string after the declaration follows the convention already
used by tools for type aliases and described for attribute docstrings in
`257`{.interpreted-text role="pep"}. Existing documented aliases would
gain runtime documentation without requiring their authors to rewrite
them.

Storing the docstring in the alias node\'s `doc` field lets
`ast.get_docstring`{.interpreted-text role="py:func"} retrieve it
without searching the surrounding statements. Tools that inspect or
modify alias docstrings can read or update the field directly. The cost
is the AST change described under [Backwards
Compatibility](#backwards-compatibility).

# Backwards Compatibility

Previously, a string literal immediately following a
`type`{.interpreted-text role="py:keyword"} statement had no effect at
runtime. Under this proposal, a qualifying string becomes the alias\'s
`__doc__` and is removed from the AST as a separate
`ast.Expr`{.interpreted-text role="py:class"} statement. The string is
stored in the alias node\'s `doc` field, and the node\'s end position
extends to cover it.

Tools that find alias docstrings by looking at the following statement,
or that rely on the alias node\'s end position, need adjusting when
parsing with Python 3.16. `ast.TypeAlias`{.interpreted-text
role="py:class"} gains a fourth, optional field. Constructing the node
with three positional arguments continues to work.

# Security Implications

This PEP has no known security implications.

# How to Teach This

The reference documentation for the `type`{.interpreted-text
role="py:keyword"} statement should show a docstring immediately after
the declaration, note that the accepted forms match function and class
docstrings, then demonstrate `Alias.__doc__` and
`help(Alias) <help>`{.interpreted-text role="py:func"}. The
`typing.TypeAliasType`{.interpreted-text role="py:class"} documentation
should describe the new attribute and how to assign it for aliases
created with the constructor.

Users already familiar with source-based alias documentation can keep
writing the same strings. Documentation should emphasize that only
`type`{.interpreted-text role="py:keyword"} statements gain runtime
docstrings. Ordinary assignments, including those annotated with
`typing.TypeAlias`{.interpreted-text role="py:data"}, do not.

Documentation for `ast.TypeAlias`{.interpreted-text role="py:class"} and
`ast.get_docstring`{.interpreted-text role="py:func"} should explain how
to read and modify the `doc` field and how whitespace is processed. It
should also note that the docstring no longer has a separate
`ast.Expr`{.interpreted-text role="py:class"} node.

# Reference Implementation

A CPython prototype is available at these revisions:

- [Compiler, runtime, and AST
  support](https://github.com/johnslavik/cpython/commit/a807d9bd68eaf2243b4869a4c2213c705ec18206).
- [pydoc
  support](https://github.com/johnslavik/cpython/commit/67f572e4cadc0eeef9f1d8cbd22404622ccc2c92).

The grammar rule for the `type`{.interpreted-text role="py:keyword"}
statement gains an optional group that parses the expression statement
on the following logical line. The `type_alias_docstring[expr_ty]` rule
uses an action helper that returns the expression node if it is a string
constant. Otherwise, the helper returns `NULL` without setting an error.
This causes the optional group to fail, so the parser backtracks to
before the newline. Blank lines and comment-only lines do not prevent
the parser from recognizing the docstring. The string must be in the
same block as the `type`{.interpreted-text role="py:keyword"} statement.
The type alias action extracts the constant\'s string value and stores
it in the `doc` field.

The `pydoc`{.interpreted-text role="py:mod"} implementation requests the
alias expression in string format. This evaluation can trigger lazy
imports. If evaluation raises an `Exception`{.interpreted-text
role="py:exc"}, `pydoc`{.interpreted-text role="py:mod"} tries to
recover the original alias expression from the source without evaluating
it. If source recovery also fails, the declaration contains a
placeholder with `repr`{.interpreted-text role="py:func"} of the
original exception, and rendering continues with the docstring. A full
traceback is not included. Failures to render type parameter bounds,
constraints, or defaults cause that part of the declaration to be
omitted.

# Rejected Ideas

## Retaining the String Statement

An alternative design would have AST preprocessing populate `doc` while
retaining the string as a separate `ast.Expr`{.interpreted-text
role="py:class"} statement following the alias. Existing tools could
continue to inspect that statement, but the AST would contain the same
documentation in both the field and the statement. After an AST
transformation, the two could contain different strings. The compiler
would then need a rule for choosing which string to use.
`ast.unparse`{.interpreted-text role="py:func"} and
`compile`{.interpreted-text role="py:func"} could also produce different
docstrings from the same tree if they used different copies.

When stripping the docstring under `-OO`{.interpreted-text
role="option"}, AST preprocessing would also need to prevent an
additional docstring from taking its place if the tree were compiled
again. With parser-level recognition, the AST contains the docstring
only in the `doc` field, so these rules are unnecessary.

In one variant, `doc` would refer to the original
`~ast.Constant`{.interpreted-text role="py:class"} node. In-place edits
would be visible through both references, but visitors would reach the
same node twice. Replacing the node through one reference would leave
the other reference pointing to the old node.

In another variant, a private attribute would hold the docstring,
accessible only through `ast.get_docstring`{.interpreted-text
role="py:func"}. This design would avoid a public field, but AST
preprocessing would still need rules for invalidating the stored
documentation when surrounding statements change.

# Acknowledgements

Thanks to Jelle Zijlstra for reviewing the proposal and agreeing to
sponsor the PEP, to Guido van Rossum for suggesting that the parser
recognize the docstring, and to the participants in [the initial
discussion on
Discourse](https://discuss.python.org/t/runtime-docstrings-for-type-aliases/108901).

Thanks to Peter Bierma and Jakub Romańczuk for convincing me to pursue
the idea.

# Change History

- [06-Sep-2026](https://discuss.python.org/t/docstrings-for-type-aliases/108901):
  Initial proposal and first PEP draft.
- 15-Sep-2026: The parser recognizes the docstring as part of the
  `type`{.interpreted-text role="py:keyword"} statement instead of AST
  preprocessing associating a following statement with the alias. The
  AST no longer retains the string as a separate statement.

# Copyright

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
