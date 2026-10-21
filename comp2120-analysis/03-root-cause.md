# Root-cause analysis — opsdroid issue #2073

**Author:** Chenyan Zhou (u8071817) — COMP6120 Assignment 5
**Issue:** <https://github.com/opsdroid/opsdroid/issues/2073>
**Repository:** <https://github.com/opsdroid/opsdroid> (upstream), analysed at commit
`4f1ef7c45a2ed26a95fb88b8fdb7e4deb33bde4a` (`main`, 2025-11-24)

---

## 1. One-line summary

`opsdroid/loader.py` imports `pkg_resources` at **module import time**. `pkg_resources`
is not part of the standard library — it is shipped by `setuptools`, and setuptools
82.0.0 (2026-02-08) removed it. Because opsdroid declares `setuptools>=76.0.0` as a
*runtime* dependency with no upper bound, a clean install pulls the newest setuptools,
so the very first `import opsdroid.loader` raises `ModuleNotFoundError`. Installation
itself succeeds; the program cannot start.

---

## 2. The failure is at run time, not at install time

This is the first question the task brief asks, and the two are easy to confuse
because pip prints a wall of output before anything else.

| Stage | Command | Result on Python 3.13 + Debian-like venv |
|---|---|---|
| install | `pip install opsdroid` | **exit 0** — sdist downloaded, wheel built, `Successfully installed ... opsdroid-0.31.0` |
| run | `opsdroid version` | **exit 1** — `ModuleNotFoundError: No module named 'pkg_resources'` |
| run | `python -m opsdroid` | **exit 1** — same traceback as the issue report |
| run | `opsdroid start` | **exit 1** — same traceback |

Evidence: `../experiments/results/A-313-pypi-latest.json`
(`install_exit_code: 0`), raw transcript `../experiments/logs/A-313-pypi-latest.log`.

The reason installation succeeds is that setuptools is only imported by
`setup.py`/`versioneer` **inside the isolated PEP 517 build environment**, where pip
installs a *separate* copy of setuptools and never puts `pkg_resources` on the target
environment's `sys.path`. Build-time setuptools and run-time setuptools are two
different installations, and only the run-time one matters here.

---

## 3. The exact import chain

The console script and the module entry point converge on the same chain:

```
setup.cfg:82                  console_scripts: opsdroid = opsdroid.cli:cli
                                                        │
opsdroid/__main__.py:3        import opsdroid.cli        │   (`python -m opsdroid`)
        │                                               │
        └──────────────────┬────────────────────────────┘
                           ▼
opsdroid/cli/__init__.py:5    from opsdroid.cli.config import config
opsdroid/cli/config.py:5      from opsdroid.cli.utils import (...)
opsdroid/cli/utils.py:19      from opsdroid.core import OpsDroid
opsdroid/core.py:23           from opsdroid.loader import Loader
opsdroid/loader.py:18         from pkg_resources import iter_entry_points
                              ✗ ModuleNotFoundError: No module named 'pkg_resources'
```

Every CLI sub-command is imported eagerly in `opsdroid/cli/__init__.py`, so even
harmless commands such as `opsdroid version` cannot run. There is no lazy-import path
that avoids the failure.

`opsdroid/cli/utils.py` is on the chain because it imports `OpsDroid`, and
`opsdroid/core.py:78` instantiates `Loader(self)` in `OpsDroid.__init__`.

---

## 4. Why `pkg_resources` is missing

Three independent facts combine:

1. **`pkg_resources` is not in the standard library.** It has always been part of
   setuptools. The standard-library replacements are `importlib.metadata` (Python 3.8+)
   and `importlib.resources`.
2. **setuptools 82.0.0 removed it.** From the official changelog, v82.0.0
   (2026-02-08): *"`pkg_resources` has been removed from Setuptools. Most common uses
   of `pkg_resources` have been superseded by the `importlib.resources` and
   `importlib.metadata` projects."* setuptools 80.9.0 had already announced the
   removal deadline, and setuptools 81.0.0 emits at import time:
   `UserWarning: pkg_resources is deprecated as an API. ... The pkg_resources package is
   slated for removal as early as 2025-11-30. Refrain from using this package or pin to
   Setuptools<81.`
3. **A clean Python 3.12+ virtual environment contains no setuptools at all.**
   CPython 3.12 stopped installing setuptools (and wheel) from `ensurepip`
   (python/cpython#101039), and Debian unbundles it further. So the venv starts with
   `pip` only, and pip is then free to choose any version the project allows.

opsdroid's `setup.cfg:69` declares:

```
install_requires =
  ...
  setuptools>=76.0.0
```

With no upper bound, pip resolves that floor to the newest release — setuptools
**84.0.0** during this investigation. There is no `pkg_resources` distribution on PyPI
to install separately, and pip has no way to know that the code needs the old module.
So the resolver does exactly what it was asked to do and installs a setuptools that
cannot satisfy `loader.py:18`.

### Where the requirement came from

`git log -S "setuptools>=76.0.0" -- setup.cfg` finds a single commit:

```
2266cf1  Fixing the CI build by adjusting the supported Python versions
         and switching no longer compatible dependencies (#2054)   (2025-03-11)
```

That commit also raised `python_requires` from `>=3.8` to `>=3.11`. The requirement
was therefore added as a *workaround*: once Python 3.12+ venvs stopped shipping
setuptools, `pkg_resources` disappeared and someone had to put setuptools back. In
other words the dependency exists **only** to keep `loader.py:18` importable. It is
not used by any other part of the shipped package:

```
$ grep -rn "import setuptools\|from setuptools\|import pkg_resources\|from pkg_resources" opsdroid/
opsdroid/loader.py:18:from pkg_resources import iter_entry_points
```

`opsdroid/_version.py` (vendored versioneer) imports only `errno, os, re, subprocess,
sys, typing, functools`. Nothing else in `opsdroid/` needs setuptools.

---

## 5. Blast radius

| Location | Kind | Needs changing? |
|---|---|---|
| `opsdroid/loader.py:18` | production import | **yes** — this is the bug |
| `opsdroid/loader.py:456` | production call `iter_entry_points(group=epname)` | **yes** — the call site |
| `tests/test_loader.py:10` | test import `import pkg_resources` | **yes** — tests cannot even be collected without setuptools |
| `tests/test_loader.py:245-246` | test constructs `pkg_resources.Distribution()` / `EntryPoint(..., dist=...)` | **yes** — `pkg_resources`-only constructor API |
| `tests/test_loader.py:252` | `mock.patch("opsdroid.loader.iter_entry_points")` | **yes** — patches a name that will no longer exist |
| `versioneer.py:244` | docstring only | no |
| `setup.cfg:69` | `setuptools>=76.0.0` runtime requirement | recommended: remove once nothing imports `pkg_resources` |

The single production line is the whole bug; the test file is collateral damage and is
what makes a "just add a `# noqa`" non-fix impossible.

---

## 6. What the entry points are used for

`opsdroid/loader.py:434-465` (`_load_modules`) is how opsdroid discovers third-party
extensions that are installed as ordinary Python packages — the documented mechanism
in `docs/packaging.md` ("Packaged Python extensions"). A user writes:

```yaml
connectors:
  fixtureconn: {}
skills:
  fixtureskill: {}
```

and opsdroid has to find the *installed* package that provides `fixtureconn` without
the user naming a module or a path.

```python
453:        # entry point group naming scheme: opsdroid_ + module type plural,
454:        # eg. "opsdroid_databases"
455:        epname = "opsdroid_{}s".format(modules_type)
456:        entry_points = {ep.name: ep for ep in iter_entry_points(group=epname)}
457:        for epname in entry_points:
458:            _LOGGER.debug(
459:                _("Found installed package for %s '%s' support."), modules_type, epname
460:            )
```

and then, per configured module:

```python
425:        config["entrypoint"] = entry_points.get(config["name"], None)
...
469:            if not (config["is_builtin"] or config["module"] or config["entrypoint"]):
470:                # module is not installed anywhere -> clone it from GitHub
...
82:        if config.get("entrypoint"):
86:            return config["entrypoint"].load()
```

So the entry-point lookup is the *first* of four import strategies and the one that
makes packaged extensions work at all. It is also the marker that stops opsdroid from
trying to `git clone` an already-installed package (line 469).

The groups in use are `opsdroid_connectors`, `opsdroid_databases`, `opsdroid_skills`
and, nominally, `opsdroid_parsers`.

### Matching third-party convention

`docs/packaging.md` documents the value as a **module**, not a class:

```python
entry_points = {
    'opsdroid_databases': [
        'zodb = opsdroid_zodb.database'
    ]
},
```

This matters, because `opsdroid/core.py:384` and `:298` scan the loaded object's
`__dict__` for `Connector`/`Skill` subclasses:

```python
384:            for _, cls in connector_module["module"].__dict__.items():
385:                if (isinstance(cls, type) and issubclass(cls, Connector)
388:                     and cls is not Connector):
```

I confirmed experimentally that an entry point resolving to a *class* instead of a
*module* produces `CRITICAL opsdroid.core All connectors failed to load.` — the class's
own `__dict__` does not contain itself. This is a property of the existing contract
that a replacement must not change: `.load()` must return the same kind of object
`pkg_resources` returned.

---

## 7. Evidence that this is the cause and not a guess

Three independent confirmations, in increasing strength:

1. **The traceback names the line.** Every failing probe ends at
   `opsdroid/loader.py, line 18, in <module>` — the same frame as the issue report
   (the reporter's `/home/chatbot/opsdroid/lib/python3.13/site-packages/...` path is
   the only difference).
2. **The version boundary is sharp and reproducible.** setuptools 81.0.0 → works
   (with a deprecation warning). setuptools 84.0.0 → `ModuleNotFoundError`. The only
   change between the two runs is the setuptools version; see
   `02-version-matrix.md`.
3. **Removing the import removes the failure.** A read-only proof-of-concept copy with
   `loader.py:18` replaced by `importlib.metadata` starts successfully under setuptools
   84.0.0 with no pin — the exact environment that fails on `main`
   (`../experiments/results/G-313-poc-a.json`).

---

## 8. Two related observations (out of scope, but recorded)

* **The parser group name is misspelled in the code.**
  `loader.py:455` builds `"opsdroid_{}s".format(modules_type)`, and `loader.py:366`
  passes `modules_type="parsers"`, producing the group `opsdroid_parserss`. A parser
  package registering in the documented `opsdroid_parsers` group would never be found.
  This is a pre-existing bug unrelated to #2073. It is listed here so that the
  migration does **not** silently change or "accidentally fix" group strings: the
  contract for this task is behavioural equivalence, and a drive-by fix would widen the
  PR's blast radius. Verified in `../experiments/logs/B-313-constraint-lt82.log`
  ("group isolation" step) that `opsdroid_parserss` and `opsdroid_parsers` are both
  empty when no parser packages are installed.
* **`python -m opsdroid` exits 2 even on success**, because `opsdroid/__main__.py`
  calls `cli()` with no sub-command and click prints help. That is why the harness
  classifies that probe by looking for the help text rather than by exit code.
