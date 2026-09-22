---
pep: 824
title: None-coalescing operators
author:
- Marc Mueller
sponsor: Guido van Rossum <guido@python.org>
discussions_to: https://discuss.python.org/t/pep-824-none-coalescing-operators/109161
status: Draft
type: Standards Track
created: 20-Sep-2026
python_version: '3.16'
post_history:
- '`22-Sep-2026 <https://discuss.python.org/t/pep-824-none-coalescing-operators/109161>`__'
python_status: Draft
url: https://peps.python.org/pep-0824/
source_path: https://github.com/python/peps/blob/main/peps/pep-0824.rst
source_commit: 45f94657d3d2e609a80993b1ecd78c11551b3110
---

# Abstract

This PEP proposes adding two new operators.

- The \"`None`-coalescing\" operator `??`
- The \"`None`-coalescing assignment\" operator `??=`

The general idea is to provide a conditional operator, similar to `or`,
which instead of truthiness, checks for `None` values.

The \"`None`-coalescing\" operator evaluates the left-hand side, checks
whether it is `not None`, and if not, returns the result. If the value
is `None`, the right-hand side is evaluated and returned.

The \"`None`-coalescing assignment\" operator will only assign the
right-hand side to the left-hand side if the left-hand side evaluates to
`None`.

They are roughly equivalent to:

``` python
# a ?? b
_t if ((_t := a) is not None) else b

# a ??= b
if a is None:
    a = b
```

# Motivation

First officially proposed over ten years ago in (the now deferred)
`505`{.interpreted-text role="pep"}, the idea to add `None`-coalescing
operators has been along for some time now, discussed at length in
numerous threads, most recently in [^1] and[^2]. This PEP aims to
capture the current state of discussion and proposes a specification for
addition to the Python language. In contrast to `505`{.interpreted-text
role="pep"}, it will only focus on the two `None`-coalescing operators.
See the [Deferred Ideas](#deferred-ideas) section for more details.

`None`-coalescing operators are not a new invention. Several other
modern programming languages have so called \"`null` coalescing\"
operators, including TypeScript[^3], ECMAScript (a.k.a. JavaScript)
[^4][^5], C#[^6], Dart[^7][^8], Swift[^9], Kotlin[^10], PHP[^11][^12]
and more.

The general idea is to provide a conditional operator, similar to `or`,
which instead of truthiness, checks for `None` values.

## Explicit checks for `None`

It is common in Python to use `None` as a default value that cannot be
confused with other inherent default values like `[]`, `""` or `0`. Most
code working with such functions either does some kind of validation to
make sure a valid value is actually returned, maybe returning early or
raising an exception if it is not, or provides a default / fallback
value. To do this, it is common to use `is None` / `is not None` checks.

``` python
class User:
    name: str | None
    @property
    def age(self) -> int | None: ...

def show_user_age(user: User):
    age = user.age
    if age is None:
        age = "unknown"
    print(f"The user age is {age}")
```

Though the intent is clear, it is quite verbose. The target name has to
be repeated multiple times just to be able to assign a default value to
the target variable. It is possible to write this in a more concise form
using an if-expression. However, that has its own issues.

``` python
def show_user_age(user: User):
    age = user.age if user.age is not None else "unknown"
    print(f"The user age is {age}")
```

Especially for fairly simple value expressions, it is not uncommon to
just repeat them, like in the example above, instead of using an
assignment expression. This could cause problems if the value expression
has side effects which would be executed twice now. Furthermore, there
is not a single agreed upon way to write these expressions. Instead of
putting the value first, some might prefer to invert the check to use
`is None` instead. Reader need to constantly be aware of these
challenges, increasing the mental load.

Using the \"`None`-coalescing\" operator `??` instead, helps to keep the
expression short and predictable while still clearly communicating the
intent.

    def show_user_age(user: User):
        age = user.age ?? "unknown"
        print(f"The user age is {age}")

## Overwrite `None` values

Sometimes it might be necessary to assign a fallback value inside an
object. To do so, the expression is usually written twice. Once for the
`is None` check, and again for the assignment.

``` python
def fix_user_name(user: User):
    if user.name is None:
        user.name = "unknown"
```

Using the \"`None`-coalesce assignment\" operator `??=` helps to avoid
repeating the expression. Especially for more complex once, this will
make it easier to read and write.

    def fix_user_name(user: User):
        user.name ??= "unknown"

## Defaults for function arguments

Function argument defaults are evaluated in the parent scope. That is a
common issue in cases where the default value is a mutable object or
depends on the function context itself. In these cases, a typical
solution is to allow `None` as argument and assign the fallback value
inside the function itself.

``` python
def show_user_name(user: User | None):
    if user is None:
        user = create_default_user()
    print(f"The user name is {user.name}")
```

This could be rewritten as:

    def show_user_name(user: User | None):
        user ??= create_default_user()
        print(f"The user name is {user.name}")

# Specification

## The `None`-coalescing operator

The `??` operator is added. It first evaluates the left-hand side. The
result is cached, so that the expression is not evaluated again. If the
value is `not None`, the cached result is returned. If it is `None`, the
right-hand side expression is evaluated and returned instead.

``` python
# a ?? b
_t if ((_t := a) is not None) else b
```

### Precedence

The precedence of `??` will be lower than `or` but higher than
conditional expressions. Parentheses can be added as necessary to modify
the precedence of individual expressions. A few examples of how implicit
parentheses would be placed:

    # x or y ?? 2
    (x or y) ?? 2

    # "Hello" if x ?? True else 0
    "Hello" if (x ?? True) else 0

### AST changes

A new `Coalesce` operator is added to `boolop` for use in `BoolOp`
nodes.

    expr = BoolOp(boolop op, expr* values)
        | ...

    boolop = And | Or | Coalesce

### Grammar changes

A new `??` token is added, as well as a new `coalesce` rule. Every rule
which previously referenced the `disjunction` rule is updated to refer
to the `coalesce` rule instead.

``` PEG
coalesce:
    | disjunction ('??' disjunction)+
    | disjunction

disjunction:
    | conjunction ('or' conjunction)+
    | conjunction
```

## The `None`-coalescing assignment operator

The `??=` operator is added. It performs a conditional assignment. As
such it will first evaluate the left-hand side and check that the value
is `None` and only then evaluate and assign the result from the
right-hand side. If the first value is `not None`, the evaluation and
assignment of the right-hand side are skipped.

``` python
# a ??= b
if a is None:
    a = b
```

### Subexpression caching

Subexpressions on the left-hand side are only evaluated once before
being cached. This is similar to augmented assignments.

``` python
# a.b.c ??= d
if (_t1 := a.b).c is None:
    _t1.c = d

# func().var[other()] ??= x
if (_t2 := func().var)[(_t3 := other())] is None:
    _t2[_t3] = x
```

### AST changes

A new `BoolAssign` AST node is added. Similar to `AugAssign`, it stores
a `target` and `value` expression as well as the boolean operator
`Coalesce`.

    stmt = ...
        | AugAssign(expr target, operator op, expr value)
        | BoolAssign(expr target, boolop op, expr value)

### Grammar changes

A new `??=` token is added. Additionally, the `assignment` rule is
extended to include the \"`None`-coalesce assignment\".

``` PEG
assignment:
    | NAME ':' expression ['=' annotated_rhs]
    | ('(' single_target ')'
        | single_subscript_attribute_target) ':' expression ['=' annotated_rhs]
    | (star_targets '=')+ annotated_rhs !'=' [TYPE_COMMENT]
    | single_target augassign ~ annotated_rhs
    | single_target boolassign ~ annotated_rhs

boolassign:
    | '??='
```

# Backwards Compatibility

Existing programs will continue to run as is. So far code which used
either `??` or `??=` raised a `SyntaxError`.

# Security Implications

There are no new security implications from this proposal.

# How to Teach This

In a practical sense it might be helpful to think of the
\"`None`-coalescing\" operator `??` as a special case for the
conditional `or` operator, with the caveat that `??` checks for
`is not None` instead of truthiness. As such it makes sense to include
`??` when teaching the other conditional operators `and` and `or`.

The \"`None`-coalescing assignment\" operator `??=` can be best thought
of as a conditional assignment operator. As it is closely related to
`??`, explaining these together would make sense. Though it looks
related to binary assignment operators like `+=` as well, it is worth
pointing out the distinction between these. Since `??=` is a
**conditional** assignment operator, the right-hand side will be skipped
entirely in some case, while `+=` always evaluates both sides.

## Reading expressions out loud

Reading expressions out loud is always lossy. This PEP does not intent
to define an unambiguous way of speaking these operators. The following
is therefore merely meant as a suggestion.

### `None`-coalescing operator

+---------------------------+---------------------+-----------------------+
| Code                      | Pattern             | Example               |
+===========================+=====================+=======================+
|     user.age ?? "unknown" | \"\... or \... if   | \"user dot age `or`   |
|                           | None\"              | unknown `if None`\"   |
|                           +---------------------+-----------------------+
|                           | \"\... coalesce     | \"user dot age        |
|                           | with \...\"         | `coalesce with`       |
|                           |                     | unknown\"             |
+---------------------------+---------------------+-----------------------+

### `None`-coalescing assignment operator

+-----------------------------+---------------------+-------------------------+
| Code                        | Pattern             | Example                 |
+=============================+=====================+=========================+
|     user.name ??= "unknown" | \"if \... is None,  | \"`if` user dot name    |
|                             | assign \...\"       | `is None`, `assign`     |
|                             |                     | unknown\"               |
|                             +---------------------+-------------------------+
|                             | \"assign \... to    | \"`assign` unknown `to` |
|                             | \... if None\"      | user dot name           |
|                             |                     | `if None`\"             |
+-----------------------------+---------------------+-------------------------+

# Reference Implementation

A reference implementation is available at
<https://github.com/cdce8p/cpython/tree/pep824-none-coalescing-operators>.
An online demo can be tested at
<https://pep823-and-pep824-demo.pages.dev/>.

# Deferred Ideas

## `None`-aware access operators

`505`{.interpreted-text role="pep"} also suggest the addition of the
\"`None`-aware access\" operators `?.` and `?[ ]`. As the
\"`None`-coalescing\" operators have their own use cases, the
\"`None`-aware access\" operators were moved into a separate document,
see `823`{.interpreted-text role="pep"}. Both proposals can be adopted
independently of each other.

## Other conditional assignment operators

While this PEP is focused on the \"`None`-coalescing assignment\"
operator, it is worth pointing out that the logic behind it could easily
be extended to cover `and=` / `or=`. This would match the existing
conditional assignment operators `&&=` and `||=` in other programming
languages such as ECMAScript (a.k.a. JavaScript)[^13] [^14], Ruby[^15]
and Perl[^16].

# Rejected Ideas

## Add new (soft-) keyword

Python does have a history of preferring keywords over symbols. For
example, though a lot of languages use `&&` and `||` as conditional
operators, Python uses `and` and `or` respectively. As such it was
suggested to use a new (soft-) keyword, e.g. `otherwise`, instead of
`??`.

While the keywords `and` and `or` help avoid ambiguity with the binary
operators `&` and `|`, there is not a corresponding binary operator for
`??`. Furthermore, both keywords are well established in spoken and
written language and therefore immediately obvious to the reader. Not to
mention they are also quite short with just two and three characters.

In comparison, a new (soft-) keyword would likely not have the same
benefits. There is no established short name for it, so while ideas like
`otherwise` could be added, they do not convey an inherent meaning and
for that reason do not provide an immediate advantage. In contrast, the
`??` operator is well-known from other major programming languages.

Lastly, using a (soft-) keyword for the \"`None`-coalescing assignment\"
operator poses additional questions and readability concerns.

    a = otherwise b

    a otherwise= b

## Add `??` as a binary operator

`505`{.interpreted-text role="pep"} originally suggested to add `??` as
another binary operator. As such it would have bound more tightly than
the proposed [specification](#precedence).

Though this would have worked fine, it would have suggested that `??` is
similar to other binary operators like `+` or `**`. This is not the
case. While binary operators first evaluate the left- **and** right-hand
side before performing the operation, for `??` only the left-hand side
is evaluated if the values is `not None`. The \"`None`-coalescing\"
operator is much more closely related to the conditional operators `or`
and `and` which also short-circuit the expression for truthy and falsy
values respectively.

Furthermore, setting the precedence between `or` and conditional
expressions matches other languages which have implemented the operator,
like JS[^17] and C#[^18].

## Add `??=` as `AugAssign`

`505`{.interpreted-text role="pep"} also suggested to add `??=` as an
`AugAssign` node.

So far `AugAssign` is only used for binary operators, including `??=`
which is a conditional assignment operator would therefore be confusing.
Furthermore, `AugAssign` statements always evaluate the left- **and**
right-hand side, without any short-circuiting. This is a major
difference compared to \"`None`-coalescing\" assignments.

## Make `??=` atomic

It was proposed to make the \"`None`-coalescing\" assignment operator
atomic to avoid race conditions when different threads try to use `??=`
on the same target. As the operator first checks if the left-hand side
evaluates to `None` before assigning a value, it is possible that two
threads see `None` and try to overwrite one another.

This was rejected since other operators with multiple steps, for example
`+=`, are not atomic either. In cases where race conditions are an
issue, existing synchronization primitives like locks should be used
instead.

# Common objections

## Just use a conditional expression

The \"`None`-coalescing\" operators can be considered syntactic sugar
for existing conditional expressions and statements. As such some
questioned whether they would add anything meaningful to the language as
a whole.

As shown in the [Motivation](#motivation) section, there are clear
benefits to using the \"`None`-coalescing\" operators. To summarize them
again:

- They help avoid repeating the variable expression or having to
  introduce a temporary variable.
- Clear control flow, no more `if ... is not None else ...` and the
  inverse `if ... is None else ...` in the same code blocks, reducing
  the mental load while reading code.
- More concise while also being more explicit.

## Proliferation of `None` in code bases

One of the reasons why `505`{.interpreted-text role="pep"} stalled was
that some expressed their concern how \"`None`-coalescing\" and
\"`None`-aware\" operators will affect the code written by developers.
If it is easier to work with `None` values, this will encourage
developers to use them more. They believe that e.g. returning an
optional `None` value from a function is usually an anti-pattern. In
their ideal world the use of `None` would be limited as much as
possible, for example with early data validation.

It is certainly true that new language features affect how the language
as a whole develops. Therefore any changes should be considered
carefully. However, just because `None` represents an anti-pattern for
some, has not prevented the community as a whole from using it
extensively. Rather the lack of \"`None`-coalescing\" operators has
stopped developers from writing concise expressions and instead often
leads to more complex code which is more difficult to read than
necessary, see the [Motivation](#motivation) section for more details.

## `None` is not special enough

Some mentioned that `None` is not special enough to warrant dedicated
operators.

\"`None`-coalescing\" operators have been added to a number of other
modern programming languages. Furthermore, adding `??` and `??=` is
something which was suggested numerous times since
`505`{.interpreted-text role="pep"} was first proposed over ten years
ago.

In Python `None` is frequently used to indicate the absence of something
better or a missing value. As such it is common to look specifically for
`None` values, for example, to provide a default or fallback value.

## There are better default values than `None`

It was pointed out that there are better domain-specific values to
signal the absence of a value. For example using `""` or an empty
collection as default where appropriate can eliminate unnecessary code.

While this is helpful to keep in mind, it does have its limitations
whenever the inherent default value is also a valid one. As example, for
number values `0` often cannot be used since there is no way to
differentiate it between being used as a default or as an actual value.

It also does not work for custom types since those often do not have
default types at all.

## Use custom sentinels instead of `None`

In Python 3.15, `661`{.interpreted-text role="pep"} added the option to
define custom sentinels using `sentinel(...)`. This addressed an issue
in cases where `None` itself is a valid value and thus could not be used
as sentinel.

In general though, `None` will still be preferred if a sentinel is
needed, simply because it already exists for that exact purpose and is
easier to use.

## Late-bound function argument defaults

Some suggested `671`{.interpreted-text role="pep"}, currently in draft
and last updated 2022, might be a better solution for the problem
describe in [Defaults for function
arguments](#defaults-for-function-arguments). While it arguably could be
helpful in some cases, using the syntax suggested in
`671`{.interpreted-text role="pep"} here just shifts the responsibility
upstream because the function signature itself would need to be changed.
Instead of `user: User | None` as argument, it would be
`user: User => create_default_user()`. It would be up to the caller now
to make sure `None` is never passed to the function and the argument is
omitted instead.

`671`{.interpreted-text role="pep"} cannot help though with other use
cases like overwriting `None` values inside an object, as shown in
[Overwrite None values](#overwrite-none-values).

# Footnotes

# Copyright

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.

[^1]: discuss.python.org: Revisiting PEP 505 - None-aware operators
    (<https://discuss.python.org/t/revisiting-pep-505-none-aware-operators/74568>)

[^2]: discuss.python.org: None-coalescing operator and null-coalescing
    assignment
    (<https://discuss.python.org/t/none-coalescing-operator-and-null-coalescing-assignment/107510>)

[^3]: TypeScript: Nullish Coalescing
    (<https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#nullish-coalescing>)

[^4]: JavaScript: Nullish coalescing operator (??)
    (<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing>)

[^5]: JavaScript: Nullish coalescing assignment (??=)
    (<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing_assignment>)

[^6]: C# Reference: ?? and ??= operators
    (<https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator>)

[^7]: Dart: Conditional expressions
    (<https://dart.dev/language/operators#conditional-expressions>)

[^8]: Dart: Assignment operators
    (<https://dart.dev/language/operators#assignment-operators>)

[^9]: Swift: Nil-Coalescing Operator
    (<https://docs.swift.org/swift-book/documentation/the-swift-programming-language/basicoperators/#Nil-Coalescing-Operator>)

[^10]: Kotlin: Elvis operator
    (<https://kotlinlang.org/docs/null-safety.html#elvis-operator>)

[^11]: PHP: Null Coalesce Operator
    (<https://wiki.php.net/rfc/isset_ternary>)

[^12]: PHP: Null Coalescing Assignment Operator
    (<https://wiki.php.net/rfc/null_coalesce_equal_operator>)

[^13]: JavaScript: Logical AND assignment (&&=)
    (<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_AND_assignment>)

[^14]: JavaScript: Logical OR assignment (\|\|=)
    (<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_OR_assignment>)

[^15]: Ruby: Abbreviated Assignment
    (<https://ruby-doc.org/3.2/syntax/assignment_rdoc.html#label-Abbreviated+Assignment>)

[^16]: Perl: Assignment Operators
    (<https://perldoc.perl.org/perlop#Assignment-Operators>)

[^17]: JavaScript: Operator precedence
    (<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_precedence#table>)

[^18]: C# Reference: Operator precedence
    (<https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/#operator-precedence>)
