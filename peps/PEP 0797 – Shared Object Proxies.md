---
pep: 797
title: Shared Object Proxies
author:
- Peter Bierma <peter@python.org>
discussions_to: https://discuss.python.org/t/105709
status: Rejected
type: Standards Track
created: 08-Aug-2025
python_version: '3.16'
post_history:
- '`01-Jul-2025 <https://discuss.python.org/t/97306>`__'
- '`13-Jan-2026 <https://discuss.python.org/t/105709>`__'
resolution: '`08-Jul-2026 <https://discuss.python.org/t/105709/35>`__'
python_status: Rejected
url: https://peps.python.org/pep-0797/
source_path: https://github.com/python/peps/blob/main/peps/pep-0797.rst
source_commit: a10f24821d975725d962e73ef83e6eb515219d71
---

# Abstract

This PEP introduces a new
`~concurrent.interpreters.SharedObjectProxy`{.interpreted-text
role="func"} type to the `concurrent.interpreters`{.interpreted-text
role="mod"} module, which allows any arbitrary object to be shared
across interpreters using an object proxy, at the cost of being less
efficient to concurrently access across multiple interpreters.

For example:

``` python
from concurrent import interpreters

with open("spanish_inquisition.txt") as unshareable:
    interp = interpreters.create()
    proxy = interpreters.SharedObjectProxy(unshareable)
    interp.prepare_main(file=proxy)
    interp.exec("file.write('I didn't expect the Spanish Inquisition')")
```

# Terminology

This PEP uses the term \"share\", \"sharing\", and \"shareable\" to
refer to objects that are *natively* shareable between interpreters.
This differs from `734`{.interpreted-text role="pep"}, which uses these
terms to also describe an object that supports the
`pickle`{.interpreted-text role="mod"} module.

In addition to the new
`~concurrent.interpreters.SharedObjectProxy`{.interpreted-text
role="class"} type, the list of natively shareable objects can be found
in `the documentation
<interp-object-sharing>`{.interpreted-text role="ref"}.

# Motivation

## Many objects cannot be shared between subinterpreters

In Python 3.14, the new `concurrent.interpreters`{.interpreted-text
role="mod"} module can be used to create multiple interpreters in a
single Python process. This works well for code without shared state,
but since one of the primary applications of subinterpreters is to
bypass the `global interpreter lock`{.interpreted-text role="term"}, it
is fairly common for programs to require highly-complex data structures
that are not easily shareable. In turn, this damages the practicality of
subinterpreters for concurrency.

As of writing, subinterpreters can only share `a handful of types
<interp-object-sharing>`{.interpreted-text role="ref"} natively, relying
on the `pickle`{.interpreted-text role="mod"} module for other types.
This can be very limited, as many types of objects cannot be serialized
with `pickle` (such as file objects returned by `open`{.interpreted-text
role="func"}). Additionally, serialization can be a very expensive
operation, which is not ideal for multithreaded applications.

# Rationale

## A fallback for object sharing

A shared object proxy is designed to be a fallback for sharing an object
between interpreters. A shared object proxy should only be used as a
last-resort for highly complex objects that cannot be serialized or
shared in any other way.

This means that even if this PEP is accepted, there is still benefit in
implementing other methods to share objects between interpreters.

# Specification

::: concurrent.interpreters.SharedObjectProxy(obj)
A proxy type that allows access to an object across multiple
interpreters. Instances of this object are natively shareable between
subinterpreters.
:::

## Interpreter switching

When interacting with the wrapped object, the proxy will switch to the
interpreter in which the object was created. This must happen for any
access to the object, such as accessing attributes. To visualize, `foo`
in the following code is only ever called in the main interpreter,
despite being accessed in subinterpreters through a proxy:

``` python
from concurrent import interpreters

def foo():
    assert interpreters.get_current() == interpreters.get_main()

interp = interpreters.create()
proxy = interpreters.share(foo)
interp.prepare_main(foo=proxy)
interp.exec("foo()")
```

## Method proxying

Methods on a shared object proxy will switch to their owning interpreter
when accessed. In addition, any arguments passed to the method are
implicitly ensured to be shareable. If they aren\'t natively shareable,
they are wrapped in an instance of `SharedObjectProxy`. The same happens
to the return value of the method.

For example, the `__add__` method on an object proxy is roughly
equivalent to the following code:

``` python
def __add__(self, other):
    with self.switch_interpreter():
        result = self.value.__add__(share(other))
        return share(result)
```

## Multithreaded scaling

To switch to a wrapped object\'s interpreter, an object proxy must swap
the `attached thread state`{.interpreted-text role="term"} of the
current thread, which will in turn wait on the `GIL`{.interpreted-text
role="term"} of the target interpreter, if it is enabled. This means
that a shared object proxy will experience contention when accessed
concurrently, but is still useful for multicore threading, since other
threads in the interpreter are free to execute while waiting on the GIL
of the target interpreter.

As an example, imagine that multiple interpreters want to write to a log
through a proxy for the main interpreter, but don\'t want to constantly
wait on the log. By accessing the proxy in a separate thread for each
interpreter, the thread performing the computation can still execute
while accessing the proxy.

``` python
from concurrent import interpreters

def write_log(message):
    print(message)

def execute(n, write_log):
    from threading import Thread
    from queue import Queue

    log = Queue()

    # By performing this in a separate thread, 'execute' can still run
    # while the log is being accessed by the main interpreter.
    def log_queue_loop():
        while True:
            write_log(log.get())

    thread = Thread(target=log_queue_loop)
    thread.start()

    for i in range(100000):
        n ** i
        log.put(f"Completed an iteration: {i}")

    thread.join()

proxy = interpreters.SharedObjectProxy(write_log)
for n in range(4):
    interp = interpreters.create()
    interp.call_in_thread(execute, n, proxy)
```

## Proxy copying

Contrary to what one might think, a shared object proxy itself can only
be used in one interpreter, because the proxy\'s reference count is not
thread-safe (and thus cannot be accessed from multiple interpreters).
Instead, when crossing an interpreter boundary, a new proxy is created
for the target interpreter that wraps the same object as the original
proxy.

For example, in the following code, there are two proxies created, not
just one.

``` python
from concurrent import interpreters

interp = interpreters.create()
foo = object()
proxy = interpreters.SharedObjectProxy(foo)

# The proxy crosses an interpreter boundary here. 'proxy' is *not* directly
# send to 'interp'. Instead, a new proxy is created for 'interp', and the
# reference to 'foo' is merely copied. Thus, both interpreters have their
# own proxy that are wrapping the same object.
interp.prepare_main(proxy=proxy)
```

## Thread-local state

Accessing an object proxy will retain information stored on the current
`thread state`{.interpreted-text role="term"}, such as thread-local
variables stored by `threading.local`{.interpreted-text role="class"}
and context variables stored by `contextvars`{.interpreted-text
role="mod"}. This allows the following case to work correctly:

``` python
from concurrent import interpreters
from threading import local

thread_local = local()
thread_local.value = 1

def foo():
    assert thread_local.value == 1

interp = interpreters.create()
proxy = interpreters.SharedObjectProxy(foo)
interp.prepare_main(foo=proxy)
interp.exec("foo()")
```

In order to retain thread-local data when accessing an object proxy,
each thread will have to keep track of the last used thread state for
each interpreter. In C, this behavior looks like this:

``` c
// Error checking has been omitted for brevity
PyThreadState *tstate = PyThreadState_New(interp);

// By swapping the current thread state to 'interp', 'tstate' will be
// associated with 'interp' for the current thread. That means that accessing
// a shared object proxy will use 'tstate' instead of creating its own
// thread state.
PyThreadState *save = PyThreadState_Swap(tstate);

// 'save' is now the most recently used thread state, so shared object
// proxies in this thread will use it instead of 'tstate' when accessing
// 'interp'.
PyThreadState_Swap(save);
```

In the event that no thread state exists for an interpreter in a given
thread, a shared object proxy will create its own thread state that will
be owned by the interpreter (meaning it will not be destroyed until
interpreter finalization), which will persist across all shared object
proxy accesses in the thread. In other words, a shared object proxy
ensures that thread local variables and similar state will not
disappear.

## Memory management

All proxy objects hold a `strong reference`{.interpreted-text
role="term"} to the object that they wrap. As such, destruction of a
shared object proxy may trigger destruction of the wrapped object if the
proxy holds the last reference to it, even if the proxy belongs to a
different interpreter. For example:

``` python
from concurrent import interpreters

interp = interpreters.create()
foo = object()
proxy = interpreters.share(foo)
interp.prepare_main(proxy=proxy)
del proxy, foo

# 'foo' is still alive at this point, because the proxy in 'interp' still
# holds a reference to it. Destruction of 'interp' will then trigger the
# destruction of 'proxy', and subsequently the destruction of 'foo'.
interp.close()
```

Shared object proxies support the garbage collector protocol, but will
only traverse the object that they wrap if the garbage collection is
occurring in the wrapped object\'s interpreter. To visualize:

``` python
from concurrent import interpreters
import gc

proxy = interpreters.share(object())

# This prints out [<object object at 0x...>], because the object is owned
# by this interpreter.
print(gc.get_referents(proxy))

interp = interpreters.create()
interp.prepare_main(proxy=proxy)

# This prints out [], because the wrapped object must be invisible to this
# interpreter.
interp.exec("import gc; print(gc.get_referents(proxy))")
```

## Interpreter lifetime management

When an interpreter is destroyed, shared object proxies wrapping objects
owned by that interpreter may still exist elsewhere. To prevent this
from causing crashes, an interpreter will invalidate all proxies
pointing to any object it owns, so any subsequent access to a proxy will
raise an exception.

To demonstrate, the following snippet first prints out `Alive`, and then
raises a `RuntimeError` after deleting the interpreter:

``` python
from concurrent import interpreters

def test():
    from concurrent import interpreters

    class Test:
        def __str__(self):
            return "Alive"

    return interpreters.share(Test())

interp = interpreters.create()
wrapped = interp.call(test)
print(wrapped)  # Alive
interp.close()
print(wrapped)  # RuntimeError
```

# Backwards Compatibility

This PEP has no known backwards compatibility issues.

# Security Implications

This PEP has no known security implications.

# How to Teach This

New APIs and important information about how to use them will be added
to the `concurrent.interpreters`{.interpreted-text role="mod"}
documentation.

# Reference Implementation

A reference implementation of this PEP can be found at
[python/cpython#145150](https://github.com/python/cpython/pull/145150).

# Rejected Ideas

## Introducing a generic sharing protocol

This PEP used to specify a `share()` function that would call a
`__share__()` method on an object, or otherwise implicitly wrap the
object in a `SharedObjectProxy`.

It was deemed that this wasn\'t necessary for this proposal to work, so
this protocol is left to be done by a future PEP.

## Directly sharing proxy objects

The initial revision of this proposal took an approach where an instance
of `~concurrent.interpreters.SharedObjectProxy`{.interpreted-text
role="class"} was `immortal`{.interpreted-text role="term"}. This
allowed proxy objects to be directly shared across interpreters, because
their reference count was thread-safe (since it never changed due to
immortality).

This proved to make the implementation significantly more complicated,
and also ended up with a lot of edge cases that would have been a burden
on CPython maintainers.

# Acknowledgements

This PEP would not have been possible without discussion and feedback
from Eric Snow, Petr Viktorin, Kirill Podoprigora, Adam Turner, Yury
Selivanov, and Steve Dower.

# Copyright

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
