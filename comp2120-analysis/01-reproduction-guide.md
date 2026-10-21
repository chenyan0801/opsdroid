# Minimal reproduction guide — opsdroid issue #2073

**Deliverable 1** · Author: Chenyan Zhou (u8071817) · Issue: <https://github.com/opsdroid/opsdroid/issues/2073>

Anyone in the group should be able to follow this and see the failure. Part A needs
nothing but Python, pip and network access. Part B is the scripted harness used to
produce the version matrix, and needs the helper interpreters described in §B1.

---

## Part A — the minimal reproduction (no tooling required)

### Preconditions

* A POSIX machine (Linux or macOS); the issue reporter used Debian Trixie.
* **Python 3.12 or newer** on `PATH` as `python3`. Python 3.11 reproduces too, but only
  if the venv has no setuptools to begin with — see §A3.
* Network access to PyPI.
* Do **not** run this in an existing environment; `python -m venv` guarantees
  isolation. Nothing here touches the system Python.

### Steps

```bash
# 1. a brand-new virtual environment, exactly as in the issue report
python3 -m venv /tmp/venv2073
. /tmp/venv2073/bin/activate
python -m pip install --upgrade pip

# 2. record the starting state - there is no setuptools and no pkg_resources yet
python -m pip list --format=freeze
python -c "import pkg_resources"
```

Expected at this point:

```
pip==26.2.1                        <- pip is the only package in the venv
Traceback (most recent call last):
  ...
ModuleNotFoundError: No module named 'pkg_resources'
```

```bash
# 3. install opsdroid, exactly as in the issue report
pip install "opsdroid[all]"        # `pip install opsdroid` reproduces just as well
```

Expected: **it succeeds.** The last line is
`Successfully installed ... opsdroid-0.31.0 ...` and the exit status is `0`. Nothing
about this step looks wrong.

```bash
# 4. find out which setuptools the resolver chose
python -m pip show setuptools | head -2
```

Expected: `Version: 84.0.0` (or whatever the newest release is at the time — anything
`>= 82` fails, see §A2).

```bash
# 5. start opsdroid
opsdroid version
```

### Actual result

```
Traceback (most recent call last):
  File "/tmp/venv2073/bin/opsdroid", line 3, in <module>
    from opsdroid.cli import cli
  File "/tmp/venv2073/lib/python3.13/site-packages/opsdroid/cli/__init__.py", line 5, in <module>
    from opsdroid.cli.config import config
  File "/tmp/venv2073/lib/python3.13/site-packages/opsdroid/cli/config.py", line 5, in <module>
    from opsdroid.cli.utils import (
  File "/tmp/venv2073/lib/python3.13/site-packages/opsdroid/cli/utils.py", line 19, in <module>
    from opsdroid.core import OpsDroid
  File "/tmp/venv2073/lib/python3.13/site-packages/opsdroid/core.py", line 23, in <module>
    from opsdroid.loader import Loader
  File "/tmp/venv2073/lib/python3.13/site-packages/opsdroid/loader.py", line 18, in <module>
    from pkg_resources import iter_entry_points
ModuleNotFoundError: No module named 'pkg_resources'
```

Exit status `1`. `python -m opsdroid` produces the same traceback with
`File "<frozen runpy>", line 88, in _run_code` and
`opsdroid/__main__.py, line 3` at the top — that is the exact traceback pasted into the
issue.

`opsdroid --version` does **not** work either, but for an unrelated reason: the option
is the sub-command `opsdroid version`, not a flag. It fails at the same import anyway.

### A3 — the same reproduction on Python 3.11

On Python 3.11 the outcome depends on the *bootstrap* setuptools in the venv:

```bash
python3.11 -m venv /tmp/venv2073-311
. /tmp/venv2073-311/bin/activate
python -m pip list --format=freeze     # some builds ship setuptools 79.x here
python -m pip install opsdroid
```

If that fresh venv already contains a setuptools **≥ 76 and < 82** (measured: 79.0.1 in
the uv-managed CPython 3.11.17 build), it satisfies opsdroid's `setuptools>=76.0.0`
floor, pip leaves it alone, `pkg_resources` still exists, and **opsdroid starts
normally** — a false negative. Debian's `python3-venv` ships no setuptools at all, so
the same command fails there.

To get the true 3.11 behaviour deterministically, remove the bootstrap copy first:

```bash
python -m pip uninstall -y setuptools     # models Debian's python3-venv
python -m pip install opsdroid            # now resolves to setuptools 84.x
opsdroid version                          # -> ModuleNotFoundError
```

This is the reason the report must state the *setuptools* version and not just the
Python version: **the bug is decided by the resolved setuptools, not by the Python
minor version.**

### A4 — the workaround, and how to see that it is a workaround

```bash
. /tmp/venv2073/bin/activate
pip install "setuptools<82"
python -m pip show setuptools | head -2   # Version: 81.0.0
opsdroid version                          # opsdroid 0.31.0      <- works
```

and it emits a warning on every single run, straight from setuptools:

```
.../site-packages/opsdroid/loader.py:18: UserWarning: pkg_resources is deprecated as an
API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources
package is slated for removal as early as 2025-11-30. Refrain from using this package or
pin to Setuptools<81.
```

Then the part that matters:

```bash
pip install --upgrade setuptools   # a routine, innocent command any tool may run
opsdroid version                   # broken again: ModuleNotFoundError
```

Clean up with `deactivate && rm -rf /tmp/venv2073 /tmp/venv2073-311`.

---

## Part B — the scripted harness (version matrix)

### B1. Prepare the interpreters (once)

The harness needs Python 3.11, 3.12 and 3.13 side by side. Any interpreters work; this
is how the recorded results were produced (uv-managed CPython builds, kept inside the
workspace so no system Python is touched):

```bash
cd <workspace>
python3 -m venv tools/uvvenv
tools/uvvenv/bin/python -m pip install --quiet --upgrade pip uv
export UV_CACHE_DIR=<workspace>/.uv-cache
export UV_PYTHON_INSTALL_DIR=<workspace>/pythons
tools/uvvenv/bin/uv python install 3.11 3.12 3.13
```

That yields `pythons/cpython-3.11-macos-aarch64-none/bin/python3.11` and so on. On Linux
the directory names differ; adjust `PY311/PY312/PY313` at the top of `run_matrix.sh`.

### B2. Run everything

```bash
cd <workspace>/experiments
bash run_matrix.sh          # add `fast` to skip the slow `opsdroid[all]` case
python3 summarise.py        # regenerates RESULTS-MATRIX.md
```

Each case gets its own venv, its own log and its own JSON record:

```
experiments/venvs/<case>/          the virtual environment
experiments/startdir-<case>/       cwd for `opsdroid start` (configuration.yaml + logs)
experiments/logs/<case>.log        full transcript: every command, exit code, stdout, stderr
experiments/results/<case>.json    the same, machine-readable, plus a verdict per probe
```

### B3. Run a single case

```bash
python3 run_case.py \
    --id my-case \
    --python <workspace>/pythons/cpython-3.13-macos-aarch64-none/bin/python3.13 \
    --source pypi --start --fixture
```

Useful flags:

| flag | effect |
|---|---|
| `--source pypi` / `--source local` | install from PyPI 0.31.0, or from the `main` checkout at `a5/upstream-opsdroid` |
| `--source poc-a` / `poc-b` / `poc-naive` | install one of the read-only proof-of-concept copies (`experiments/make_poc.sh` builds them) |
| `--extras "[all]"` | use the literal command from the issue report |
| `--constraint <file>` | apply a pip constraints file, e.g. `constraints-setuptools-lt82.txt` |
| `--preset-setuptools "<82"` | install a setuptools version *before* opsdroid |
| `--post-setuptools "<82"` | install a setuptools version *after* opsdroid (the commenter's workaround) |
| `--phase2-pip "--upgrade setuptools"` | then run a later routine pip operation and re-probe |
| `--drop-setuptools` | delete the bootstrap setuptools before installing (models Debian) |
| `--fixture` | install the third-party entry-point fixture and configure opsdroid to use it |
| `--start` | also run `opsdroid start` with a minimal configuration |

Probes recorded for every case: `opsdroid version`; `python -m opsdroid`;
`import opsdroid.loader`; `import opsdroid.cli.utils`; `opsdroid start`; the entry
points visible in each `opsdroid_*` group via both APIs; and `import pkg_resources`.

### B4. Harness caveats (read before quoting a number)

* `HOME` is redirected to `a5/home` for every command, because opsdroid writes to
  `~/Library/Application Support/opsdroid` and `~/Library/Logs/opsdroid` (appdirs) and
  the sandbox only allows writes inside `<home>`. This changes where
  opsdroid's *data* directories live and nothing else. The minimal `configuration.yaml`
  also sets `logging.path` and `module-path` explicitly for the same reason.
* The harness never modifies `upstream-opsdroid`. The proof-of-concept variants live in
  `a5/poc/`, generated by `experiments/make_poc.sh`, and are throwaway verification
  artefacts — not the group's implementation.
* `pip install` is not pinned to a date. Re-running later may pick a newer setuptools;
  record the version from each `results/*.json` rather than trusting this document's
  numbers.
* A `FAIL` verdict is generated from the exit code and the traceback, never from log
  text, because opsdroid prints `Started opsdroid ...` *before* it loads any module.
* `opsdroid start` is expected to be killed by the harness timeout when it works (the
  bot runs until interrupted). That is recorded as `PASS: still running when killed
  after Ns`, not as a timeout failure.

---

## Part C — what "reproduced" means here

A reproduction counts only if all four hold, and all four are visible in the recorded
logs for this investigation:

1. the install exits `0` and prints `Successfully installed ... opsdroid-0.31.0`;
2. `python -m pip show setuptools` reports a version `>= 82`;
3. `python -c "import pkg_resources"` fails with `ModuleNotFoundError`;
4. `opsdroid version` fails at `opsdroid/loader.py, line 18` with the same traceback as
   the issue, **and** the identical failure is absent from a run whose only difference
   is `setuptools<82`.
