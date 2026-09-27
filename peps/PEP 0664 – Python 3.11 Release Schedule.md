---
pep: 664
title: Python 3.11 Release Schedule
author:
- Pablo Galindo Salgado <pablogsal@python.org>
status: Active
type: Informational
topic: Release
created: 12-Jul-2021
python_version: '3.11'
python_status: Active
url: https://peps.python.org/pep-0664/
source_path: https://github.com/python/peps/blob/main/peps/pep-0664.rst
source_commit: a36ec367781c01fb3d48d60f8ef791186a9729e7
---

# Abstract

This document describes the development and release schedule for Python
3.11. The schedule primarily concerns itself with PEP-sized items.

# Release Manager and Crew

- 3.11 Release Manager: Pablo Galindo Salgado
- Windows installers: Steve Dower
- Mac installers: Ned Deily
- Documentation: Julien Palard

# Release Schedule

## 3.11.0 schedule

Note: the dates below use a 17-month development period that results in
a 12-month release cadence between feature versions, as defined by
`602`{.interpreted-text role="pep"}.

Actual:

- 3.11 development begins: Monday, 2021-05-03
- 3.11.0 alpha 1: Tuesday, 2021-10-05
- 3.11.0 alpha 2: Tuesday, 2021-11-02
- 3.11.0 alpha 3: Wednesday, 2021-12-08
- 3.11.0 alpha 4: Friday, 2022-01-14
- 3.11.0 alpha 5: Thursday, 2022-02-03
- 3.11.0 alpha 6: Monday, 2022-03-07
- 3.11.0 alpha 7: Tuesday, 2022-04-05
- 3.11.0 beta 1: Sunday, 2022-05-08 (No new features beyond this point.)
- 3.11.0 beta 2: Tuesday, 2022-05-31
- 3.11.0 beta 3: Wednesday, 2022-06-01
- 3.11.0 beta 4: Monday, 2022-07-11
- 3.11.0 beta 5: Tuesday, 2022-07-26
- 3.11.0 candidate 1: Monday, 2022-08-08
- 3.11.0 candidate 2: Monday, 2022-09-12
- 3.11.0 final: Monday, 2022-10-24

## Bugfix releases

Actual:

- 3.11.1: Tuesday, 2022-12-06
- 3.11.2: Wednesday, 2023-02-08
- 3.11.3: Wednesday, 2023-04-05
- 3.11.4: Tuesday, 2023-06-06
- 3.11.5: Thursday, 2023-08-24
- 3.11.6: Monday, 2023-10-02
- 3.11.7: Monday, 2023-12-04
- 3.11.8: Tuesday, 2024-02-06
- 3.11.9: Tuesday, 2024-04-02 (Final regular bugfix release with binary
  installers)

## Source-only security fix releases

Provided irregularly on an \"as-needed\" basis until October 2027.

- 3.11.10: Saturday, 2024-09-07
- 3.11.11: Tuesday, 2024-12-03
- 3.11.12: Tuesday, 2025-04-08
- 3.11.13: Tuesday, 2025-06-03
- 3.11.14: Thursday, 2025-10-09
- 3.11.15: Tuesday, 2026-03-03
- 3.11.16: Wednesday, 2026-08-12

## 3.11 Lifespan

3.11 received bugfix updates approximately every 2 months for
approximately 18 months. Some time after the release of 3.12.0 final,
the ninth and final 3.11 bugfix update was released. After that, it is
expected that security updates (source only) will be released until 5
years after the release of 3.11.0 final, so until approximately October
2027.

# Features for 3.11

Some of the notable features of Python 3.11 include:

- `654`{.interpreted-text role="pep"}, Exception Groups and `except*`.
- `657`{.interpreted-text role="pep"}, Enhanced error locations in
  tracebacks.
- `680`{.interpreted-text role="pep"}, Support for parsing TOML in the
  standard library
- Python 3.11 is up to 10-60% faster than Python 3.10. On average, we
  measured a 1.25x speedup on the standard benchmark suite. See [Faster
  CPython](https://docs.python.org/3.11/whatsnew/3.11.html#faster-cpython)
  for details.

Typing features:

- `646`{.interpreted-text role="pep"}, Variadic generics.
- `655`{.interpreted-text role="pep"}, Marking individual TypedDict
  items as required or potentially-missing.
- `673`{.interpreted-text role="pep"}, Self type.
- `675`{.interpreted-text role="pep"}, Arbitrary literal string type.
- `681`{.interpreted-text role="pep"}, Dataclass transforms

# Copyright

This document has been placed in the public domain.
