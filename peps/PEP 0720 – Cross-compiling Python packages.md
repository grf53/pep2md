---
pep: 720
title: Cross-compiling Python packages
author:
- Filipe Laíns <lains@python.org>
pep_delegate: ''
status: Draft
type: Informational
created: 01-Jul-2023
python_version: '3.12'
python_status: Draft
url: https://peps.python.org/pep-0720/
source_path: https://github.com/python/peps/blob/main/peps/pep-0720.rst
source_commit: 8e4d0cb96028e4ee25a0d1a2574f9d1cb9e6ed60
---

# Abstract

This PEP attempts to document the status of cross-compilation of
downstream projects.

It should give an overview of the approaches currently used by
distributors (Linux distros, WASM environment providers, etc.) to
cross-compile downstream projects (3rd party extensions, etc.).

# Motivation

We write this PEP to express the challenges in cross-compilation and act
as a supporting document in future improvement proposals.

# Analysis

## Introduction

There are a couple different approaches being used to tackle this, with
different levels of interaction required from the user, but they all
require a significant amount of effort. This is due to the lack of
standardized cross-compilation infrastructure on the Python packaging
ecosystem, which itself stems from the complexity of cross-builds,
making it a huge undertaking.

## Upstream support

Some major projects like CPython, setuptools, etc. provide some support
to help with cross-compilation, but it\'s unofficial and at a
best-effort basis. For example, the `sysconfig` module allows
overwriting the data module name via the `_PYTHON_SYSCONFIGDATA_NAME`
environment variable, something that is required for cross-builds, and
setuptools [accepts
patches](https://github.com/pypa/distutils/pulls?q=cross)[^1] to
tweak/fix its logic to be compatible with popular [\"environment
faking\"](#faking-the-target-environment) workflows[^2].

The lack of first-party support in upstream projects leads to
cross-compilation being fragile and requiring a significant effort from
users, but at the same time, the lack of standardization makes it harder
for upstreams to improve support as there\'s no clarity on how this
feature should be provided.

### Projects with decent cross-build support

It seems relevant to point out that there are a few modern Python
package build-backends with, at least, decent cross-compilation support,
those being [scikit-build](https://github.com/pypa/distutils) and
[meson-python](https://github.com/scikit-build/scikit-build). Both these
projects integrate external mature build-systems into Python packaging
--- [CMake](https://github.com/mesonbuild/meson-python) and
[Meson](https://cmake.org/), respectively --- so cross-build support is
inherited from them.

## Downstream approaches

Cross-compilation approaches fall in a spectrum that goes from, by
design, requiring extensive user interaction to (ideally) almost none.
Usually, they\'ll be based on one of two main strategies, using a
[cross-build environment](#cross-build-environment), or [faking the
target environment](#faking-the-target-environment).

### Cross-build environment {#approach-cross-environment}

This consists of running the Python interpreter normally and utilizing
the cross-build provided by the projects\' build-system. However, as we
saw above, upstream support is lacking, so this approach only works for
a small-ish set of projects. When this fails, the usual strategy is to
patch the build-system code to build use the correct toolchain, system
details, etc.[^3].

Since this approach often requires package-specific patching, it
requires a lot of user interaction.

::: {.admonition .note}
Examples

[python-for-android](#python-for-android), [kivy-ios](#kivy-ios), etc.
:::

### Faking the target environment

Aiming to drop the requirement for user input, a popular approach is
trying to fake the target environment. It generally consists of
monkeypatching the Python interpreter to get it to mimic the interpreter
on the target system, which constitutes of changing many of the `sys`
module attributes, the `sysconfig` data, etc. Using this strategy,
build-backends do not need to have any cross-build support, and should
just work without any code changes.

Unfortunately, though, it isn\'t possible to truly fake the target
environment. There are many reasons for this, one of the main ones being
that it breaks code that actually needs to introspect the running
interpreter. As a result, monkeypatching Python to look like target is
very tricky --- to achieve the less amount of breakage, we can only
patch certain aspects of the interpreter. Consequently, build-backends
may need some code changes, but these are generally much smaller than
the previous approach. This is an inherent limitation of the technique,
meaning this strategy still requires some user interaction.

Nonetheless, this strategy still works out-of-the-box with significantly
more projects than the approach above, and requires much less effort in
these cases. It is successful in decreasing the amount of user
interaction needed, even though it doesn\'t succeed in being generic.

::: {.admonition .note}
Examples

[crossenv](#crossenv), [conda-forge](#conda-forge), etc.
:::

## Environment introspection

As explained above, most build system code is written with the
assumption that the target system is the same as where the build is
occurring, so introspection is usually used to guide the build.

In this section, we try to document most of the ways this is
accomplished. It should give a decent overview of environment details
that are required by build systems.

+----------------------------------------------+---------------------------------------+-------------------------+
| Snippet                                      | Description                           | Variance                |
+==============================================+=======================================+=========================+
| ``` python                                   | Extension (native module) suffixes    | This is                 |
| >>> importlib.machinery.EXTENSION_SUFFIXES   | supported by this interpreter.        | implementation-defined, |
| [                                            |                                       | but it **usually**      |
|    '.cpython-311-x86_64-linux-gnu.so',       |                                       | differs based on the    |
|    '.abi3.so',                               |                                       | implementation, system  |
|    '.so',                                    |                                       | architecture, build     |
| ]                                            |                                       | configuration, Python   |
| ```                                          |                                       | language version, and   |
|                                              |                                       | implementation version  |
|                                              |                                       | --- if one exists.      |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Source (pure-Python) suffixes         | This is                 |
| >>> importlib.machinery.SOURCE_SUFFIXES      | supported by this interpreter.        | implementation-defined, |
| ['.py']                                      |                                       | but it **usually**      |
| ```                                          |                                       | doesn\'t differ         |
|                                              |                                       | (outside exotic         |
|                                              |                                       | implementations or      |
|                                              |                                       | systems).               |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | All module file suffixes supported by | This is                 |
| >>> importlib.machinery.all_suffixes()       | this interpreter. It *should* be the  | implementation-defined, |
| [                                            | union of all                          | but it **usually**      |
|    '.py',                                    | `importlib.machinery.*_SUFFIXES`      | differs based on the    |
|    '.pyc',                                   | attributes.                           | implementation, system  |
|    '.cpython-311-x86_64-linux-gnu.so',       |                                       | architecture, build     |
|    '.abi3.so',                               |                                       | configuration, Python   |
|    '.so',                                    |                                       | language version, and   |
| ]                                            |                                       | implementation version  |
| ```                                          |                                       | --- if one exists. See  |
|                                              |                                       | the entries above for   |
|                                              |                                       | more information.       |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | ABI flags, as specified in            | Differs based on the    |
| >>> sys.abiflags                             | `3149`{.interpreted-text role="pep"}. | build configuration.    |
| ''                                           |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | C API version.                        | Differs based on the    |
| >>> sys.api_version                          |                                       | Python installation.    |
| 1013                                         |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Prefix of the installation-wide       | Differs based on the    |
| >>> sys.base_prefix                          | directories where platform            | platform, and           |
| /usr                                         | independent files are installed.      | installation.           |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Prefix of the installation-wide       | Differs based on the    |
| >>> sys.base_exec_prefix                     | directories where platform dependent  | platform, and           |
| /usr                                         | files are installed.                  | installation.           |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Native byte order.                    | Differs based on the    |
| >>> sys.byteorder                            |                                       | platform.               |
| 'little'                                     |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Names of all modules that are         | Differs based on the    |
| >>> sys.builtin_module_names                 | compiled into the Python interpreter. | platform, system        |
| ('_abc', '_ast', '_codecs', ...)             |                                       | architecture, and build |
| ```                                          |                                       | configuration.          |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Prefix of the site-specific           | Differs based on the    |
| >>> sys.exec_prefix                          | directories where platform            | platform, installation, |
| /usr                                         | independent files are installed.      | and environment.        |
| ```                                          | Because it concerns the site-specific |                         |
|                                              | directories, in standard virtual      |                         |
|                                              | environment implementation, it will   |                         |
|                                              | be a virtual-environment-specific     |                         |
|                                              | path.                                 |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Path of the Python interpreter being  | Differs based on the    |
| >>> sys.executable                           | used.                                 | installation.           |
| '/usr/bin/python'                            |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Whether the Python interpreter is an  | Differs based on the    |
| >>> with open(sys.executable, 'rb') as f:    | ELF file, and the ELF header. This    | installation.           |
| ...   header = f.read(4)                     | approach is something used to         |                         |
| ...   if is_elf := (header == b'\x7fELF'):   | identify the target architecture of   |                         |
| ...     elf_class = int(f.read(1))           | the installation                      |                         |
| ...     size = {1: 52, 2: 64}.get(elf_class) | ([example](https://mesonbuild.com/)). |                         |
| ...     elf_header = f.read(size - 5)        |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Low level information about the float | Differs based on the    |
| >>> sys.float_info                           | type, as defined by `float.h`.        | architecture, and       |
| sys.float_info(                              |                                       | platform.               |
|    max=1.7976931348623157e+308,              |                                       |                         |
|    max_exp=1024,                             |                                       |                         |
|    max_10_exp=308,                           |                                       |                         |
|    min=2.2250738585072014e-308,              |                                       |                         |
|    min_exp=-1021,                            |                                       |                         |
|    min_10_exp=-307,                          |                                       |                         |
|    dig=15,                                   |                                       |                         |
|    mant_dig=53,                              |                                       |                         |
|    epsilon=2.220446049250313e-16,            |                                       |                         |
|    radix=2,                                  |                                       |                         |
|    rounds=1,                                 |                                       |                         |
| )                                            |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Integer representing the Android API  | Differs based on the    |
| >>> sys.getandroidapilevel()                 | level.                                | platform.               |
| 21                                           |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Windows version of the system.        | Differs based on the    |
| >>> sys.getwindowsversion()                  |                                       | platform.               |
| sys.getwindowsversion(                       |                                       |                         |
|    major=10,                                 |                                       |                         |
|    minor=0,                                  |                                       |                         |
|    build=19045,                              |                                       |                         |
|    platform=2,                               |                                       |                         |
|    service_pack='',                          |                                       |                         |
| )                                            |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Python version encoded as an integer. | Differs based on the    |
| >>> sys.hexversion                           |                                       | Python language         |
| 0x30b03f0                                    |                                       | version.                |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Interpreter implementation details.   | Differs based on the    |
| >>> sys.implementation                       |                                       | interpreter             |
| namespace(                                   |                                       | implementation, Python  |
|    name='cpython',                           |                                       | language version, and   |
|    cache_tag='cpython-311',                  |                                       | implementation version  |
|    version=sys.version_info(                 |                                       | --- if one exists. It   |
|       major=3,                               |                                       | may also include        |
|       minor=11,                              |                                       | architecture-dependent  |
|       micro=3,                               |                                       | information, so it may  |
|       releaselevel='final',                  |                                       | also differ based on    |
|       serial=0,                              |                                       | the system              |
|    ),                                        |                                       | architecture.           |
|    hexversion=0x30b03f0,                     |                                       |                         |
|    _multiarch='x86_64-linux-gnu',            |                                       |                         |
| )                                            |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Low level information about Python\'s | Differs based on the    |
| >>> sys.int_info                             | internal integer representation.      | architecture, platform, |
| sys.int_info(                                |                                       | implementation, build,  |
|    bits_per_digit=30,                        |                                       | and runtime flags.      |
|    sizeof_digit=4,                           |                                       |                         |
|    default_max_str_digits=4300,              |                                       |                         |
|    str_digits_check_threshold=640,           |                                       |                         |
| )                                            |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Maximum value a variable of type      | Differs based on the    |
| >>> sys.maxsize                              | `Py_ssize_t` can take.                | architecture, platform, |
| 0x7fffffffffffffff                           |                                       | and implementation.     |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Value of the largest Unicode code     | Differs based on the    |
| >>> sys.maxunicode                           | point.                                | implementation, and on  |
| 0x10ffff                                     |                                       | Python versions older   |
| ```                                          |                                       | than 3.3, the build.    |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Platform identifier.                  | Differs based on the    |
| >>> sys.platform                             |                                       | platform.               |
| linux                                        |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Prefix of the site-specific           | Differs based on the    |
| >>> sys.prefix                               | directories where platform dependent  | platform, installation, |
| /usr                                         | files are installed. Because it       | and environment.        |
| ```                                          | concerns the site-specific            |                         |
|                                              | directories, in standard virtual      |                         |
|                                              | environment implementation, it will   |                         |
|                                              | be a virtual-environment-specific     |                         |
|                                              | path.                                 |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Platform-specific library directory.  | Differs based on the    |
| >>> sys.platlibdir                           |                                       | platform, and vendor.   |
| lib                                          |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Python language version implemented   | Differs if the target   |
| >>> sys.version_info                         | by the interpreter.                   | Python version is not   |
| sys.version_info(                            |                                       | the same[^4].           |
|    major=3,                                  |                                       |                         |
|    minor=11,                                 |                                       |                         |
|    micro=3,                                  |                                       |                         |
|    releaselevel='final',                     |                                       |                         |
|    serial=0,                                 |                                       |                         |
| )                                            |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Information about the thread          | Differs based on the    |
| >>> sys.thread_info                          | implementation.                       | platform, and           |
| sys.thread_info(                             |                                       | implementation.         |
|    name='pthread',                           |                                       |                         |
|    lock='semaphore',                         |                                       |                         |
|    version='NPTL 2.37',                      |                                       |                         |
| )                                            |                                       |                         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Version number used to form Windows   | Differs based on the    |
| >>> sys.winver                               | registry keys.                        | platform, and           |
| 3.8-32                                       |                                       | implementation.         |
| ```                                          |                                       |                         |
+----------------------------------------------+---------------------------------------+-------------------------+
| ``` python                                   | Python distribution configuration     | This is                 |
| >>> sysconfig.get_config_vars()              | variables. It includes a set of       | implementation-defined, |
| { ... }                                      | variables[^5] --- like `prefix`,      | but it **usually**      |
| >>> sysconfig.get_config_var(...)            | `exec_prefix`, etc. --- based on the  | differs between         |
| ...                                          | running context[^6], and may include  | non-identical builds.   |
| ```                                          | some extra variables based on the     | Please refer to the     |
|                                              | Python implementation and system.     | [sysconfig              |
|                                              |                                       | configuration           |
|                                              | In CPython and most other             | variables]() table for  |
|                                              | implementations that use the same     | a overview of the       |
|                                              | build-system, the \"extra\" variables | different configuration |
|                                              | mention above are: on POSIX, all      | variable that are       |
|                                              | variables from the `Makefile` used to | usually present.        |
|                                              | build the interpreter, and on         |                         |
|                                              | Windows, it usually only includes a   |                         |
|                                              | small subset of the those[^7] ---     |                         |
|                                              | like `EXT_SUFFIX`, `BINDIR`, etc.     |                         |
+----------------------------------------------+---------------------------------------+-------------------------+

### CPython (and similar)

  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Name                   Example Value                        Description                                                                                                                    Variance
  ---------------------- ------------------------------------ ------------------------------------------------------------------------------------------------------------------------------ ---------------------
  `SOABI`                `cpython-311-x86_64-linux-gnu`       ABI string --- defined by `3149`{.interpreted-text role="pep"}.                                                                Differs based on the
                                                                                                                                                                                             implementation,
                                                                                                                                                                                             system architecture,
                                                                                                                                                                                             Python language
                                                                                                                                                                                             version, and
                                                                                                                                                                                             implementation
                                                                                                                                                                                             version --- if one
                                                                                                                                                                                             exists.

  `SHLIB_SUFFIX`         `.so`                                Shared library suffix.                                                                                                         Differs based on the
                                                                                                                                                                                             platform.

  `EXT_SUFFIX`           `.cpython-311-x86_64-linux-gnu.so`   Interpreter-specific Python extension (native module) suffix --- generally defined as `.{SOABI}.{SHLIB_SUFFIX}`.               Differs based on the
                                                                                                                                                                                             implementation,
                                                                                                                                                                                             system architecture,
                                                                                                                                                                                             Python language
                                                                                                                                                                                             version, and
                                                                                                                                                                                             implementation
                                                                                                                                                                                             version --- if one
                                                                                                                                                                                             exists.

  `LDLIBRARY`            `libpython3.11.so`                   Shared `libpython` library name --- if available. If unavailable[^8], the variable will be empty, if available, the library    Differs based on the
                                                              should be located in `LIBDIR`.                                                                                                 implementation,
                                                                                                                                                                                             system architecture,
                                                                                                                                                                                             build configuration,
                                                                                                                                                                                             Python language
                                                                                                                                                                                             version, and
                                                                                                                                                                                             implementation
                                                                                                                                                                                             version --- if one
                                                                                                                                                                                             exists.

  `PY3LIBRARY`           `libpython3.so`                      Shared Python 3 only (major version bound only)[^9] `libpython` library name --- if available. If unavailable[^10], the        Differs based on the
                                                              variable will be empty, if available, the library should be located in `LIBDIR`.                                               implementation,
                                                                                                                                                                                             system architecture,
                                                                                                                                                                                             build configuration,
                                                                                                                                                                                             Python language
                                                                                                                                                                                             version, and
                                                                                                                                                                                             implementation
                                                                                                                                                                                             version --- if one
                                                                                                                                                                                             exists.

  `LIBRARY`              `libpython3.11.a`                    Static `libpython` library name --- if available. If unavailable[^11], the variable will be empty, if available, the library   Differs based on the
                                                              should be located in `LIBDIR`.                                                                                                 implementation,
                                                                                                                                                                                             system architecture,
                                                                                                                                                                                             build configuration,
                                                                                                                                                                                             Python language
                                                                                                                                                                                             version, and
                                                                                                                                                                                             implementation
                                                                                                                                                                                             version --- if one
                                                                                                                                                                                             exists.

  `Py_DEBUG`             `0`                                  Whether this is a [debug                                                                                                       Differs based on the
                                                              build](https://github.com/pypa/packaging/blob/2f80de7fd2a8bc199dadf5cf3f5f302a17084792/src/packaging/_manylinux.py#L43-L50).   build configuration.

  `WITH_PYMALLOC`        `1`                                  Whether this build has [pymalloc](https://docs.python.org/3/using/configure.html#debug-build) support.                         Differs based on the
                                                                                                                                                                                             build configuration.

  `Py_TRACE_REFS`        `0`                                  Whether reference tracing (debug build only) is enabled.                                                                       Differs based on the
                                                                                                                                                                                             build configuration.

  `Py_UNICODE_SIZE`                                           Size of the `Py_UNICODE` object, in bytes. This variable is only present in CPython versions older than 3.3, and was commonly  Differs based on the
                                                              used to detect if the build uses UCS2 or UCS4 for unicode objects --- before `393`{.interpreted-text role="pep"}.              build configuration.

  `Py_ENABLE_SHARED`     `1`                                  Whether a shared `libpython` is available.                                                                                     Differs based on the
                                                                                                                                                                                             build configuration.

  `PY_ENABLE_SHARED`     `1`                                  Whether a shared `libpython` is available.                                                                                     Differs based on the
                                                                                                                                                                                             build configuration.

  `CC`                   `gcc`                                The C compiler used to build the Python distribution.                                                                          Differs based on the
                                                                                                                                                                                             build configuration.

  `CXX`                  `g++`                                The C compiler used to build the Python distribution.                                                                          Differs based on the
                                                                                                                                                                                             build configuration.

  `CFLAGS`               `-DNDEBUG -g -fwrapv ...`            The C compiler flags used to build the Python distribution.                                                                    Differs based on the
                                                                                                                                                                                             build configuration.

  `py_version`           `3.11.3`                             Full form of the Python version.                                                                                               Differs based on the
                                                                                                                                                                                             Python language
                                                                                                                                                                                             version.

  `py_version_short`     `3.11`                               Custom form of the Python version, containing only the major and minor numbers.                                                Differs based on the
                                                                                                                                                                                             Python language
                                                                                                                                                                                             version.

  `py_version_nodot`     `311`                                Custom form of the Python version, containing only the major and minor numbers, and no dots.                                   Differs based on the
                                                                                                                                                                                             Python language
                                                                                                                                                                                             version.

  `prefix`               `/usr`                               Same as `sys.prefix`, please refer to the entry in table above.                                                                Differs based on the
                                                                                                                                                                                             platform,
                                                                                                                                                                                             installation, and
                                                                                                                                                                                             environment.

  `base`                 `/usr`                               Same as `sys.prefix`, please refer to the entry in table above.                                                                Differs based on the
                                                                                                                                                                                             platform,
                                                                                                                                                                                             installation, and
                                                                                                                                                                                             environment.

  `exec_prefix`          `/usr`                               Same as `sys.exec_prefix`, please refer to the entry in table above.                                                           Differs based on the
                                                                                                                                                                                             platform,
                                                                                                                                                                                             installation, and
                                                                                                                                                                                             environment.

  `platbase`             `/usr`                               Same as `sys.exec_prefix`, please refer to the entry in table above.                                                           Differs based on the
                                                                                                                                                                                             platform,
                                                                                                                                                                                             installation, and
                                                                                                                                                                                             environment.

  `installed_base`       `/usr`                               Same as `sys.base_prefix`, please refer to the entry in table above.                                                           Differs based on the
                                                                                                                                                                                             platform, and
                                                                                                                                                                                             installation.

  `installed_platbase`   `/usr`                               Same as `sys.base_exec_prefix`, please refer to the entry in table above.                                                      Differs based on the
                                                                                                                                                                                             platform, and
                                                                                                                                                                                             installation.

  `platlibdir`           `lib`                                Same as `sys.platlibdir`, please refer to the entry in table above.                                                            Differs based on the
                                                                                                                                                                                             platform, and vendor.

  `SIZEOF_*`             `4`                                  Size of a certain C type (`double`, `float`, etc.).                                                                            Differs based on the
                                                                                                                                                                                             system architecture,
                                                                                                                                                                                             and build details.
  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

  : `sysconfig` configuration variables

## Relevant Information

There are some bits of information required by build systems --- eg.
platform particularities --- scattered across many places, and it often
is difficult to identify code with assumptions based on them. In this
section, we try to document the most relevant cases.

### When should extensions be linked against `libpython`?

Short answer

:   Yes, on Windows. No on POSIX platforms, except Android, Cygwin, and
    other Windows-based POSIX-like platforms.

When building extensions for dynamic loading, depending on the target
platform, they may need to be linked against `libpython`.

On Windows, extensions need to link against `libpython`, because all
symbols must be resolvable at link time. POSIX-like platforms based on
Windows --- like Cygwin, MinGW, or MSYS --- will also require linking
against `libpython`.

On most POSIX platforms, it is not necessary to link against
`libpython`, as the symbols will already be available due to the
interpreter --- or, when embedding, the executable/library in question
--- already linking to `libpython`. Not linking an extension module
against `libpython` will allow it to be loaded by static Python builds,
so when possible, it is desirable to do so (see
[GH-65735](https://docs.python.org/3/c-api/intro.html#debugging-builds)).

This might not be the case on all POSIX platforms, so make sure you
check. One example is Android, where only the main executable and
`LD_PRELOAD` entries are considered to be `RTLD_GLOBAL` (meaning
dependencies are `RTLD_LOCAL`) [^12], which causes the `libpython`
symbols be unavailable when loading the extension.

### What are `prefix`, `exec_prefix`, `base_prefix`, and `base_exec_prefix`?

These are `sys` attributes [set in the Python
initialization](https://docs.python.org/3/c-api/memory.html#pymalloc)
that describe the running environment. They refer to the prefix of
directories where installation/environment files are installed,
according to the table below.

  Name                 Target files                             Environment Scope
  -------------------- ---------------------------------------- -------------------
  `prefix`             platform independent (eg. pure Python)   site-specific
  `exec_prefix`        platform dependent (eg. native code)     site-specific
  `base_prefix`        platform independent (eg. pure Python)   installation-wide
  `base_exec_prefix`   platform dependent (eg. native code)     installation-wide

Because the site-specific prefixes will be different inside virtual
environments, checking `sys.prexix != sys.base_prefix` is commonly used
to check if we are in a virtual environment.

# Case studies

## crossenv

Description
:   Virtual Environments for Cross-Compiling Python Extension Modules.

URL
:   <https://github.com/benfogle/crossenv>

`crossenv` is a tool to create a virtual environment with a
monkeypatched Python installation that tries to emulate the target
machine in certain scenarios. More about this approach can be found in
the [Faking the target environment](#faking-the-target-environment)
section.

## conda-forge

Description
:   A community-led collection of recipes, build infrastructure and
    distributions for the conda package manager.

URL
:   <https://conda-forge.org/>

XXX: Jaime will write a quick summary once the PEP draft is public.

XXX Uses a modified crossenv.

## Yocto Project

Description
:   The Yocto Project is an open source collaboration project that helps
    developers create custom Linux-based systems regardless of the
    hardware architecture.

URL
:   <https://www.yoctoproject.org/>

XXX: Sent email to the mailing list.

TODO

## Buildroot

Description
:   Buildroot is a simple, efficient and easy-to-use tool to generate
    embedded Linux systems through cross-compilation.

URL
:   <https://buildroot.org/>

TODO

## Pyodide

Description
:   Pyodide is a Python distribution for the browser and Node.js based
    on WebAssembly.

URL
:   <https://pyodide.org/en/stable/>

XXX: Hood should review/expand this section.

`Pyodide` is a provides a Python distribution compiled to
[WebAssembly](https://docs.python.org/3/c-api/stable.html#stable-application-binary-interface)
using the [Emscripten](https://github.com/python/cpython/issues/65735)
toolchain.

It patches several aspects of the CPython installation and some external
components. A custom package manager ---
[micropip](https://man.archlinux.org/man/dlopen.3) --- supporting both
Pure and wasm32/Emscripten wheels, is also provided as a part of the
distribution. On top of this, a repo with a [selected set of 3rd party
packages](https://github.com/python/cpython/blob/6a70edf24ca217c5ed4a556d0df5748fc775c762/Modules/getpath.py)
is also provided and enabled by default.

## Beeware

Description
:   BeeWare allows you to write your app in Python and release it on
    multiple platforms.

URL
:   <https://beeware.org/>

TODO

## python-for-android

Description
:   Turn your Python application into an Android APK.

URL
:   <https://github.com/kivy/python-for-android>

resource
<https://github.com/Android-for-Python/Android-for-Python-Users>

`python-for-android` is a tool to package Python apps on Android. It
creates a Python distribution with your app and its dependencies.

Pure-Python dependencies are handled automatically and in a generic way,
but native dependencies need [recipes](https://webassembly.org/). A set
of recipes for [popular dependencies](https://emscripten.org/) is
provided, but users need to provide their own recipes for any other
native dependencies.

## kivy-ios

Description
:   Toolchain for compiling Python / Kivy / other libraries for iOS.

URL
:   <https://github.com/kivy/kivy-ios>

`kivy-ios` is a tool to package Python apps on iOS. It provides a
toolchain to build a Python distribution with your app and its
dependencies, as well as a CLI to create and manage Xcode projects that
integrate with the toolchain.

It uses the same approach as [python-for-android](#python-for-android)
(also maintained by the [Kivy project](https://micropip.pyodide.org/))
for app dependencies --- pure-Python dependencies are handled
automatically, but native dependencies need
[recipes](https://pyodide.org/en/stable/usage/packages-in-pyodide.html),
and the project provides recipes for [popular
dependencies](https://python-for-android.readthedocs.io/en/latest/recipes/).

## AidLearning

Description
:   AI, Android, Linux, ARM: AI application development platform based
    on Android+Linux integrated ecology.

URL
:   <https://github.com/aidlearning/AidLearning-FrameWork>

TODO

## QPython

Description
:   QPython is the Python engine for android.

URL
:   <https://github.com/qpython-android/qpython>

TODO

## pyqtdeploy

Description
:   pyqtdeploy is a tool for deploying PyQt applications.

URL
:   <https://www.riverbankcomputing.com/software/pyqtdeploy/>

contact
<https://www.riverbankcomputing.com/pipermail/pyqt/2023-May/thread.html>
contacted Phil, the maintainer

TODO

## Chaquopy

Description
:   Chaquopy provides everything you need to include Python components
    in an Android app.

URL
:   <https://chaquo.com/chaquopy/>

TODO

## EDK II

Description
:   EDK II is a modern, feature-rich, cross-platform firmware
    development environment for the UEFI and PI specifications.

URL
:   <https://github.com/tianocore/edk2-libc/tree/master/AppPkg/Applications/Python>

TODO

## ActivePython

Description
:   Commercial-grade, quality-assured Python distribution focusing on
    easy installation and cross-platform compatibility on Windows,
    Linux, Mac OS X, Solaris, HP-UX and AIX.

URL
:   <https://www.activestate.com/products/python/>

TODO

## Termux

Description
:   Termux is an Android terminal emulator and Linux environment app
    that works directly with no rooting or setup required.

URL
:   <https://termux.dev/en/>

TODO

[^1]: At the time of writing (Jun 2023), setuptools\' compiler interface
    code, the component that most of affects cross-compilation, is
    developed on the [pypa/distutils](##REF##_) repository, which gets
    periodically synced to the setuptools repository.

[^2]: We specifically mention *popular* workflows, because this is not
    standardized. Though, many of the most popular implementations
    ([crossenv](##REF##crossenv), [conda-forge](##REF##conda-forge)\'s
    build system, etc.) work similarly, and this is what we are
    referring to here. For clarity, the implementations we are referring
    to here could be described as *crossenv-style*.

[^3]: The scope of the build-system patching varies between users and
    usually depends on the their goal --- some (eg. Linux distributions)
    may patch the build-system to support cross-builds, while others
    might hardcode compiler paths and system information in the
    build-system, to simply make the build work.

[^4]: Ideally, you want to perform cross-builds with the same Python
    version and implementation, however, this is often not the case. It
    should not be very problematic as long as the major and minor
    versions don\'t change.

[^5]: The set of config variables that will always be present mostly
    consists of variables needed to calculate the installation scheme
    paths.

[^6]: The context we refer here consists of the \"path initialization\",
    which is a process that happens in the interpreter startup and is
    responsible for figuring out which environment it is being run ---
    eg. global environment, virtual environment, etc. --- and setting
    `sys.prefix` and other attributes accordingly.

[^7]: This is because Windows builds may not use the `Makefile`, and
    instead [use the Visual Studio build system](##REF##_). A subset of
    the most relevant `Makefile` variables is provided to make user code
    that uses them simpler.

[^8]: Due to Python bring compiled without shared or static `libpython`
    support, respectively.

[^9]: This is the `libpython` library that users of the [stable
    ABI](##REF##_) should link against, if they need to link against
    `libpython`.

[^10]: Due to Python bring compiled without shared or static `libpython`
    support, respectively.

[^11]: Due to Python bring compiled without shared or static `libpython`
    support, respectively.

[^12]: Refer to [dlopen\'s man page](##REF##_) for more information.
