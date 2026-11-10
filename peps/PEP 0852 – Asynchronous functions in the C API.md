---
pep: 852
title: Asynchronous functions in the C API
author:
- Peter Bierma <peter@python.org>
discussions_to: Pending
status: Draft
type: Standards Track
created: 02-Nov-2026
python_version: '3.16'
post_history:
- '`14-Jan-2023 <https://discuss.python.org/t/22786>`__'
- '`09-Jan-2024 <https://discuss.python.org/t/42842>`__'
- '`12-Apr-2024 <https://discuss.python.org/t/50792>`__'
python_status: Draft
url: https://peps.python.org/pep-0852/
source_path: https://github.com/python/peps/blob/main/peps/pep-0852.rst
source_commit: bf430a89798e61f642a58ab2748a0668471f996b
---

# Abstract

This PEP describes a new suite of functions in the C API to interact
with `coroutines <coroutine>`{.interpreted-text role="term"}. For
example:

``` c
#include <Python.h>

static int
return_result(PyObject *asyncobj, PyObject *result)
{
    PyAsync_SetResult(asyncobj, result);
    return 1;
}

/* Equivalent to: async def await_from_c(coro): return await coro */
static PyObject *
await_from_c(PyObject *self, PyObject *coro) /* METH_O */
{
    PyObject *asyncobj = PyAsync_New();
    if (asyncobj == NULL) {
        return NULL;
    }
    if (PyAsync_AddAwait(asyncobj, coro, return_result, NULL) < 0) {
        PyAsync_Cancel(asyncobj);
        Py_DECREF(asyncobj);
        return NULL;
    }
    return asyncobj;
}
```

# Motivation

## The C API has no easy way to invoke asynchronous code

Currently, Python\'s C API provides no simple interface for defining
`coroutine
functions <coroutine function>`{.interpreted-text role="term"}. This
lack of API support has not gone unnoticed by users. For example,
`mypyc` still [does not support all asynchronous
features](https://github.com/mypyc/mypyc/issues/868), because they are
forced to build coroutine types from scratch, which is difficult to do
correctly.

This problem is not unique to `mypyc`; it shows up across the extension
ecosystem:

- PyO3 has had an [open tracking
  issue](https://github.com/PyO3/pyo3/issues/1632) for `async fn`
  support since

  2021\. Its experimental support relies on a coroutine type implemented
  by PyO3 itself, which still has gaps; for example, it [does not
  support](https://github.com/PyO3/pyo3/issues/5855) the asynchronous
  context manager protocol.

- A contributor to `pydantic-core`
  [asked](https://github.com/pydantic/pydantic-core/issues/1468) how to
  await a Python coroutine from Rust, and was told by a maintainer that
  moving the asynchronous logic out of Python \"is *really* hard\".

- pybind11 has had an [open feature
  request](https://github.com/pybind/pybind11/issues/2658) for defining
  asynchronous functions since 2020, in which the author resorted to
  hand-writing their own awaitable class in C++.

For extension developers, the current solution to this problem is to do
one of three things:

1.  Don\'t write the asynchronous code in C, and instead define a
    pure-Python function that handles the `await`{.interpreted-text
    role="keyword"}s, and then call an event-loop-specific API from C.
    Given that a common use case for building extension modules is to
    improve performance, this can be a step backwards for developers.
2.  Implement a type that implements the `coroutine`{.interpreted-text
    role="term"} protocol on its own, complete with all the necessary
    logic for a state machine. This is possible, but non-trivial and bad
    for extension maintenance; C API users should not have to learn
    about the inner workings of generators in order to use a built-in
    syntax construct. This is the root of `mypyc`\'s problem, for
    example.
3.  Use a binding generator that has done one of the above. To
    [quote](https://github.com/python/cpython/issues/133678#issuecomment-2865557709)
    Petr Viktorin (on a separate proposal): \"CPython should have a
    usable and stable C API for all stakeholders, and a list of a few
    blessed users goes against that goal.\" In other words, CPython
    should aim to make its C API usable (and ergonomic) where possible
    \-- not encourage users to switch to binding generators.

All of these options are less than ideal, and as a result, there is
support from both Python users and core developers for adding such a C
API to Python. To [quote](https://discuss.python.org/t/42842/3)
Marc-André Lemburg:

> I very much do think that we need a core maintained async C API in
> CPython.
>
> We currently don't have a way to write Python C extensions which work
> in async mode without using a shim written in Python which takes care
> of the interactions with the asyncio (and other) event loops.

## A third-party library is bad for both maintainers and users {#pep-852-no-third-party}

The API described in this PEP is heavily influenced by the third-party
[PyAwaitable](https://pyawaitable.zintensity.dev/) library \-- but why
does that need to come into CPython at all?

First and foremost, adding a dependency to a C extension typically has a
very high bar. In most cases, it\'s deemed safer to pick one of the
solutions described above, which each come with their own set of
problems.

Second, distributing native dependencies is extremely difficult in
Python\'s extension ecosystem. As of recently, extension modules are
increasingly being written using C API binding libraries and not the C
API itself. Unfortunately, these binding libraries cannot wrap APIs that
don\'t come directly from CPython, so native libraries are effectively
unusable to projects using other C API bindings. In fact, PyAwaitable
uses a vendoring technique that would make it functionally impossible to
use from something such as PyO3.

There have been [some](https://discuss.python.org/t/54860)
[discussions](https://discuss.python.org/t/51710/) about making this
easier (both of which arose *because* PyAwaitable was so difficult to
distribute), but in general, fixing this problem seems like a much more
massive proposal than what is currently being proposed. An asynchronous
C API in Python\'s core would be useful, and adding one to CPython is
much simpler and easier for both maintainers and users than trying to
solve a more generic problem in the packaging ecosystem; in other words,
to quote `20`{.interpreted-text role="pep"}, \"practicality beats
purity\".

### Why was a third-party library created in the first place?

When PyAwaitable was first created in early 2024, a relatively simple
implementation was possible without changes to the CPython core, and as
such, PyAwaitable was better fit as a library for the time being \--
though always with the intent of eventually becoming part of the public
C API. The idea was that a third-party library would iron out potential
design flaws in the API as a whole before they were immortalized into
CPython, and this was largely successful (see
`the deviations from PyAwaitable
<pep-852-deviations>`{.interpreted-text role="ref"}); as such, the
third-party library has served its purpose, and it is now time to add it
to the C API.

It is also worth noting that the reference implementation of this
proposal does require the use of some internal helper functions (that
PyAwaitable, as a third-party library, had to copy and work around),
which is more evidence that PyAwaitable is a good fit for CPython, as
one of the original arguments against adding such an API was that it was
implementable using only the public C API.

### The C API is allowed to be ergonomic where possible

When discussed in early 2024, the main argument against this proposal
was that it was implementable using only Python\'s public C API. To
[quote](https://discuss.python.org/t/42842/2) Petr Viktorin:

> To set expectations: I don't think this needs to be in CPython itself.
> I know that sounds disappointing, but after all, Python is made to be
> extendable; we want to have interesting projects outside the core.

Since then, there is a much larger precedent for adding convenience
functions to Python\'s C API (as in, functions that can be implemented
in terms of the existing C API or in standard C). For example:

- There have been numerous attribute access helpers added in recent
  years, such as `PyObject_GetOptionalAttr`{.interpreted-text
  role="c:func"}, `PySys_GetAttr`{.interpreted-text role="c:func"},
  `PyImport_ImportModuleAttr`{.interpreted-text role="c:func"}, and
  related functions.
- `PyMutex`{.interpreted-text role="c:type"}, while useful for the
  `free threading`{.interpreted-text role="term"} effort, is a generic
  locking interface that could have been implemented outside CPython\'s
  core.
- Several private APIs that were promoted to public, such as the
  `c-api-time`{.interpreted-text role="ref"}.
- The APIs described by `788`{.interpreted-text role="pep"} can largely
  be implemented through `atexit`{.interpreted-text role="mod"}
  callbacks.

APIs like these were added because Python\'s C API should be easy to use
where possible \-- merely because something *can* be implemented using
the public C API does not mean that it\'s the best option for users,
especially given the difficulty of distributing native code that wraps
Python\'s C API. When something is tightly coupled to a core language
feature (in this case, `async`/`await`), it is typically a better fit
for the C API and not for a third-party library.

# Specification

The following APIs are available through `Python.h` and require an
`attached thread state`{.interpreted-text role="term"}. Unless otherwise
specified, an *asyncobj* argument must be a `PyAsync`{.interpreted-text
role="c:type"} object; passing any other object is undefined behavior.
Callbacks receive `borrowed references <borrowed
reference>`{.interpreted-text role="term"} to their arguments.

## Async objects

> An opaque Python object that implements the
> `~collections.abc.Coroutine`{.interpreted-text role="class"} protocol.
> A `PyAsync`{.interpreted-text role="c:type"} object stores a queue of
> asynchronous operations and callbacks to execute when it is awaited.
>
> Awaiting a `PyAsync`{.interpreted-text role="c:type"} object runs its
> queued operations in order. The object suspends whenever an operation
> suspends, and completes once the queue is empty, returning the result
> set with `PyAsync_SetResult`{.interpreted-text role="c:func"} (`None`
> by default). A `PyAsync`{.interpreted-text role="c:type"} object is
> recognized by `asyncio.iscoroutine`{.interpreted-text role="func"} and
> `inspect.isawaitable`{.interpreted-text role="func"}, and can be
> passed to `asyncio.create_task`{.interpreted-text role="func"}.
>
> The type cannot be subclassed or instantiated from Python, and is not
> exposed in any Python module.

> The type object for `PyAsync`{.interpreted-text role="c:type"}
> objects.

> Return nonzero if *obj* is a `PyAsync`{.interpreted-text
> role="c:type"} object, and `0` otherwise. This function cannot fail.

> Create a new, empty `PyAsync`{.interpreted-text role="c:type"} object.
> Return a new `strong reference`{.interpreted-text role="term"} on
> success, or `NULL` with an exception set on failure.
>
> By default, awaiting the object returns `None`. The result can be
> changed with `PyAsync_SetResult`{.interpreted-text role="c:func"}. An
> object can only be awaited once; attempting to reuse a completed
> object raises `RuntimeError`{.interpreted-text role="exc"}.
>
> Destroying an object that was never awaited, cancelled with
> `PyAsync_Cancel`{.interpreted-text role="c:func"}, or closed emits a
> `RuntimeWarning`{.interpreted-text role="exc"}, as destroying a native
> coroutine that was never awaited does.

> Set the object returned by awaiting *asyncobj* to *result*. *result*
> must not be `NULL`.
>
> This function stores a strong reference to *result*, releasing any
> previously stored result. It cannot fail and returns `0`.

## Coroutine protocol

A `PyAsync`{.interpreted-text role="c:type"} object behaves as a native
coroutine object does:

- `send(value)` resumes the object. *value* must be `None` if the object
  has not started. The object raises `StopIteration`{.interpreted-text
  role="exc"} carrying its result when it completes.
- `throw(exc)` raises *exc* inside the currently awaited operation, as
  `await`{.interpreted-text role="keyword"} would. If the operation does
  not handle it, the operation\'s error callback is invoked with the
  exception, and so on as described under
  `PyAsync_AddAwait`{.interpreted-text role="c:func"}. If the object has
  not started, or is between operations, the exception propagates and
  the object completes.
- `close()` throws `GeneratorExit`{.interpreted-text role="exc"} into
  the object. The currently awaited operation is closed, pending error
  callbacks and `async with`{.interpreted-text role="keyword"} exits
  observe `GeneratorExit`{.interpreted-text role="exc"}, and the object
  completes. Closing an object that has not started or has completed
  does nothing.

A `PyAsync`{.interpreted-text role="c:type"} object may not be resumed
while it is already executing; doing so raises
`ValueError`{.interpreted-text role="exc"}. Queue and storage mutations
are serialized per object, so on the
`free-threaded build`{.interpreted-text role="term"} it is safe to add
operations or set values from another thread, but an object can only be
driven by one thread at a time.

## Awaiting and callbacks

Callbacks report their outcome through their return value. Every
callback must set an exception if and only if it reports failure; a
callback that returns success with an exception set, or failure without
one, causes the `PyAsync`{.interpreted-text role="c:type"} object to
raise `SystemError`{.interpreted-text role="exc"}.

> The type of a result callback. *asyncobj* is the
> `PyAsync`{.interpreted-text role="c:type"} object executing the
> callback, and *result* is the value returned by the awaited operation.
>
> The callback must return one of the following:
>
> - `1` on success.
> - `0` with an exception set to invoke the associated error callback,
>   or propagate the exception if no error callback was provided.
> - `-1` with an exception set to propagate the exception directly,
>   skipping the associated error callback.

> The type of an error callback. *exception* is the exception raised by
> an awaited operation or its result callback. The error callback is
> invoked with no exception set.
>
> The callback must return one of the following:
>
> - `1` to handle the exception and continue executing queued
>   operations.
> - `0` to restore and propagate the original exception.
> - `-1` with a new exception set to propagate that exception instead.

> The type of a deferred callback, registered with
> `PyAsync_DeferAwait`{.interpreted-text role="c:func"}. The callback
> must return `1` on success, or `0` with an exception set to stop
> execution and propagate the exception to the caller awaiting
> *asyncobj*.

> Append *awaitable* to the operations to await when *asyncobj*
> executes. Operations execute in the order they were added. This
> function stores a strong reference to *awaitable*, which must be a
> non-`NULL` `awaitable`{.interpreted-text role="term"} object distinct
> from *asyncobj*; anything else is rejected with
> `TypeError`{.interpreted-text role="exc"} when this function is
> called, not when *asyncobj* is awaited.
>
> On successful completion, *result_callback* is invoked with *asyncobj*
> and the operation\'s return value. If *result_callback* is `NULL`, the
> return value is discarded.
>
> If the operation raises an exception, or *result_callback* returns
> `0`, *error_callback* is invoked. If *error_callback* is `NULL`, the
> exception propagates to the caller awaiting *asyncobj*.
>
> Callbacks may append further operations to *asyncobj*; they are added
> to the end of the queue and run after all previously queued
> operations. Adding operations to a completed object raises
> `RuntimeError`{.interpreted-text role="exc"}.
>
> Return `0` on success, or `-1` with an exception set on failure.

> Like `PyAsync_AddAwait`{.interpreted-text role="c:func"}, but steals a
> reference to *expr* on both success and failure. If *expr* is `NULL`,
> return `-1` without changing the current exception, which is assumed
> to have been set by whatever produced *expr*.
>
> Otherwise, return `0` on success, or `-1` with an exception set on
> failure.

> Append a synchronous callback to the operations executed when
> *asyncobj* is awaited. *callback* must not be `NULL`. It is invoked
> with *asyncobj* after previously queued operations complete, and may
> queue further operations.
>
> Return `0` on success, or `-1` with an exception set on failure.

> Clear the queued operations and callbacks, releasing their references
> and discarding the current await iterator. This also suppresses the
> `RuntimeWarning`{.interpreted-text role="exc"} for an object that was
> never awaited.
>
> This function may be called from a callback, in which case the object
> completes once the callback returns, unless the callback adds new
> operations. Further operations may be added to *asyncobj* until it
> completes. A pending `async with`{.interpreted-text role="keyword"}
> exit (see `PyAsync_AsyncWith`{.interpreted-text role="c:func"}) is not
> cleared and still runs.
>
> The currently awaited operation, if any, is dropped without being
> closed. This function is unrelated to `asyncio`{.interpreted-text
> role="mod"} task cancellation, which reaches the object through
> `throw()`. It cannot fail.

> Append an `async with`{.interpreted-text role="keyword"} block using
> *context_manager*. When the operation runs, the `__aenter__` method of
> *context_manager* is awaited, *result_callback* is invoked, any
> operations it queued are awaited, and finally the `__aexit__` method
> is awaited.
>
> *result_callback*, which may be `NULL`, is the body of the block. It
> receives a `PyAsync`{.interpreted-text role="c:type"} object
> representing the block and the value returned by `__aenter__`. The
> block object shares its result and value storage with *asyncobj*
> (`PyAsync_SetResult`{.interpreted-text role="c:func"},
> `PyAsync_SetValue`{.interpreted-text role="c:func"} and the other
> storage functions act on *asyncobj*), but has its own operation queue:
> operations added to the block object run inside the block, before
> `__aexit__`. Operations added to *asyncobj* run after the block, as
> they would in Python.
>
> Once `__aenter__` has completed, `__aexit__` is always awaited,
> whether the block completes normally, *result_callback* or an
> operation inside the block raises, an exception is thrown into the
> object (including `asyncio.CancelledError`{.interpreted-text
> role="exc"}), or the object is closed. On an exception, `__aexit__`
> receives the exception and may suppress it by returning a true value.
> An exception from `__aenter__` propagates without invoking
> `__aexit__`. If `__aexit__` itself raises, the block\'s exception
> becomes the `__context__` of the new one.
>
> *error_callback* handles exceptions from the operation as a whole,
> including any exception left unsuppressed by `__aexit__`. It receives
> *asyncobj* and the exception. If it is `NULL`, the exception
> propagates to the caller awaiting *asyncobj*.
>
> Blocks may be nested by calling this function on a block object.
>
> Return `0` on success, or `-1` with an exception set on failure.

## Value storage

Each `PyAsync`{.interpreted-text role="c:type"} object has one object
slot and one arbitrary pointer slot for storing state between callbacks.
Both slots are initially `NULL`. Code that needs more than one object
stores a tuple or other container.

> Store a strong reference to *value* in the object slot, releasing the
> previously stored reference. If *value* is `NULL`, clear the slot.
>
> This function cannot fail and returns `0`.

> Return a new `strong reference`{.interpreted-text role="term"} to the
> object stored in the object slot, or `NULL` if the slot is empty. This
> function cannot fail and does not set an exception when the slot is
> empty.

> Store *value* in the arbitrary pointer slot, replacing the previous
> pointer. *value* may be `NULL`. The caller is responsible for the
> lifetime of any memory referenced by the pointer.
>
> This function cannot fail and returns `0`.

> Return the pointer stored in the arbitrary pointer slot, or `NULL` if
> the slot is empty. This function cannot fail and does not set an
> exception when the slot is empty.

## Limited API

All APIs in this PEP are part of the
`Limited API <limited-c-api>`{.interpreted-text role="ref"} and the
`Stable ABI <stable-abi>`{.interpreted-text role="ref"} as of Python
3.16.

# Rationale

## An API for coroutine functions, not futures

This proposal aims to add a C API for generic `async`/`await`
constructs, *not* an API for describing all capabilities of
`asyncio`{.interpreted-text role="mod"}. While this PEP may be reason
for adding an `asyncio` C API in the future, its only goal is to make
`async def`{.interpreted-text role="keyword"} functions possible to
implement in the C API.

## Deviations from PyAwaitable {#pep-852-deviations}

The primary purpose of making PyAwaitable a third-party library and not
a formal C API proposal (yet) was to identify the best design choices
based on user experience and feedback. As such, this proposal does not
copy PyAwaitable\'s API exactly, and instead describes a new API without
the original design flaws of PyAwaitable.

### `PyAsync` instead of `PyAwaitable`

The most obvious deviation from the original library is the use of the
`PyAsync` prefix over `PyAwaitable`. This is because the original
\"PyAwaitable\" name is a historical artifact; it was chosen as a
parallel to the `PyCoro` prefix, with the idea that a `PyCoro` was
analogous to a native `coroutine` object, while a `PyAwaitable` would be
analogous to a native `awaitable` object. Given that this API (and new
versions of PyAwaitable) support all of the
`~collections.abc.Coroutine`{.interpreted-text role="class"} protocol
(in contrast to the `~collections.abc.Awaitable`{.interpreted-text
role="class"} protocol), this name is now misleading, and thus the
rename to `PyAsync` felt necessary.

### Simpler storage for callback values

This API uses a single `PyObject *` reference and `void *` pointer for
value storage between callbacks instead of variadic argument lists that
were stored as arrays. PyAwaitable used the latter because it was
optimized for ergonomics when writing pure C code, but since this API is
also designed to be accessible from binding generators (and FFIs) as
well, a single pointer for each felt like the better call.
PyAwaitable\'s array system also turned out to be a source of complexity
and bugs, and avoiding extra complexity for CPython where possible is
ideal.

This API also returns a `strong reference`{.interpreted-text
role="term"} from `GetValue` instead of a
`borrowed reference`{.interpreted-text role="term"}. PyAwaitable used
the latter because it predated the acceptance of `703`{.interpreted-text
role="pep"}, and thus any concurrency implications were not considered.

### Different callback return values

In PyAwaitable, callbacks may return `0`, `-1`, and `-2`, which
correspond to `1`, `0`, and `-1` in this proposal, respectively. This is
also a historical artifact; PyAwaitable did not originally have the `-2`
return value, and changing the meaning of `0` and `-1` could not be done
for the sake of backwards compatibility.

# Backwards Compatibility

This PEP has no known backwards compatibility concerns.

# Security Implications

This PEP has no known security concerns.

# How to Teach This

The new APIs will be documented in the C API documentation. A tutorial
can also be added based on the tutorial present in PyAwaitable\'s
documentation.

## Examples

### Example: Forwarding an asynchronous result

An extension method can call an asynchronous function and forward the
result of awaiting it. `PyAsync`{.interpreted-text role="c:type"}
handles suspension and resumption for the extension.

``` c
static int
return_result(PyObject *asyncobj, PyObject *result)
{
    PyAsync_SetResult(asyncobj, result);
    return 1;
}

/* Equivalent to: async def call_async(func): return await func() */
static PyObject *
call_async(PyObject *self, PyObject *func) /* METH_O */
{
    PyObject *asyncobj = PyAsync_New();
    if (asyncobj == NULL) {
        return NULL;
    }
    if (PyAsync_AddExpr(asyncobj, PyObject_CallNoArgs(func),
                        return_result, NULL) < 0) {
        PyAsync_Cancel(asyncobj);
        Py_DECREF(asyncobj);
        return NULL;
    }
    return asyncobj;
}
```

### Example: Keeping state across an await

To implement `return (await get_number()) * factor`, store *factor* with
`PyAsync_SetValue`{.interpreted-text role="c:func"} before queuing
`get_number()` with this callback. The `PyAsync`{.interpreted-text
role="c:type"} object keeps the factor alive while execution is
suspended. The callback releases the strong reference returned by
`PyAsync_GetValue`{.interpreted-text role="c:func"} after using it.

``` c
static int
multiply_result(PyObject *asyncobj, PyObject *value)
{
    PyObject *factor = PyAsync_GetValue(asyncobj);
    PyObject *result = PyNumber_Multiply(value, factor);
    Py_DECREF(factor);
    if (result == NULL) {
        return 0;
    }
    PyAsync_SetResult(asyncobj, result);
    Py_DECREF(result);
    return 1;
}
```

### Example: Handling a timeout

In `call_async` above, replace the final `NULL` argument to
`PyAsync_AddExpr`{.interpreted-text role="c:func"} with
`timeout_to_none`. Awaiting the result then returns `None` on
`TimeoutError`{.interpreted-text role="exc"}. Returning `0` from the
callback lets other exceptions propagate.

``` c
static int
timeout_to_none(PyObject *asyncobj, PyObject *exception)
{
    if (!PyErr_GivenExceptionMatches(exception, PyExc_TimeoutError)) {
        return 0;
    }
    PyAsync_SetResult(asyncobj, Py_None);
    return 1;
}
```

For example, `asyncio.timeout`{.interpreted-text role="func"} can impose
a deadline on an operation:

``` python
import asyncio

async def slow_operation():
    async with asyncio.timeout(0.01):
        return await asyncio.sleep(1, result=42)

assert asyncio.run(call_async(slow_operation)) is None
```

### Example: Scheduling a dependent read

Suppose a protocol reader exposes asynchronous `read_size()` and
`readexactly(size)` methods. Store the reader with
`PyAsync_SetValue`{.interpreted-text role="c:func"}, then queue
`reader.read_size()` with this callback. The first read\'s result
determines the next asynchronous call, which reuses `return_result` to
return the completed packet.

``` c
static int
read_body(PyObject *asyncobj, PyObject *size)
{
    PyObject *reader = PyAsync_GetValue(asyncobj);
    PyObject *coro = PyObject_CallMethod(reader, "readexactly", "O", size);
    Py_DECREF(reader);
    return PyAsync_AddExpr(asyncobj, coro, return_result, NULL) == 0;
}
```

### Example: Acquiring an async lock for native work

An asynchronous context manager can acquire a lock and return the
protected dictionary to the result callback:

``` python
import asyncio
from contextlib import asynccontextmanager

@asynccontextmanager
async def locked_cache(lock, cache):
    async with lock:
        yield cache
```

The C callback updates that dictionary while the lock is held:

``` c
static int
mark_ready(PyObject *asyncobj, PyObject *cache)
{
    return PyDict_SetItemString(cache, "ready", Py_True) == 0;
}

static PyObject *
update_locked(PyObject *self, PyObject *lock) /* METH_O */
{
    PyObject *asyncobj = PyAsync_New();
    if (asyncobj == NULL) {
        return NULL;
    }
    if (PyAsync_AsyncWith(asyncobj, lock, mark_ready, NULL) < 0) {
        PyAsync_Cancel(asyncobj);
        Py_DECREF(asyncobj);
        return NULL;
    }
    return asyncobj;
}
```

Python code passes the lock and dictionary through the context manager:

``` python
async def main():
    lock, cache = asyncio.Lock(), {}
    await update_locked(locked_cache(lock, cache))
    assert cache["ready"] is True
    assert not lock.locked()

asyncio.run(main())
```

`PyAsync_AsyncWith`{.interpreted-text role="c:func"} invokes the result
callback after acquiring the lock, and releases the lock even if the
dictionary update raises an exception.

# Reference Implementation

A reference implementation of this PEP can be found
[here](https://github.com/python/cpython/compare/main...ZeroIntensity:async-c-api).

# Rejected Ideas

## A single callback with a state number

Instead of one callback per `await`{.interpreted-text role="keyword"},
Serhiy Storchaka suggested in the [2023
discussion](https://discuss.python.org/t/22786) a single callback that
is re-entered with a state number and dispatches with a `switch`. This
is closer to a generator API than to `await`{.interpreted-text
role="keyword"}, and pushes the state machine onto the user, which is
the burden this PEP aims to remove. Users who prefer that style can
implement it on top of the proposed API.

In addition, a single function with a `switch` would likely encourage
users to use `static` variables, which have thread-safety implications
that would likely go against the efforts of the
`free threading`{.interpreted-text role="term"} project.

## C++20 coroutines

Steve Dower suggested in the same discussion that native coroutine
support is better left to languages that have it, such as C++20 via
pybind11. The C API is a C API, and most of the extension ecosystem
(including binding generators) cannot consume C++ interfaces. Nothing in
this PEP prevents a C++ wrapper from offering `co_await` on top of
`PyAsync`{.interpreted-text role="c:type"}.

## Reusing native coroutine objects

Native coroutine objects (instances of
`types.CoroutineType`{.interpreted-text role="class"}) are bound to a
Python frame with bytecode, and thus there is no way to construct one
around C code without emulating a frame. A new type is simpler and
avoids exposing frame internals.

## Keeping the API in a third-party library

See `pep-852-no-third-party`{.interpreted-text role="ref"}.

# Open Issues

TBD.

# Acknowledgements

Thanks to Petr Viktorin for his help with the original PyAwaitable
design discussion, and to all users of PyAwaitable over the past few
years.

# Change History

TBD.

# Copyright

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
