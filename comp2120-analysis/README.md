# Issue #2073 — reproduction baseline and root-cause analysis

Analysis package for **opsdroid** issue
[#2073 "Installing on Debian Trixie with python virtualenv doesn't work"](https://github.com/opsdroid/opsdroid/issues/2073).

**Author:** Chenyan Zhou (u8071817) · **Role:** reproducible reproduction and root-cause
analysis · **Part of:** COMP6120 Assignment 5 group work

> **This folder is not part of the proposed fix.** It is the analysis record that the
> implementation, tests and verification were built on. The production change is a separate
> branch and pull request owned by other group members; these documents are deliberately kept
> out of it so the fix stays small and reviewable.

---

## The finding in five lines

1. **Stage:** the *install* succeeds (exit code 0). It fails at **run time**, on the first
   import, so no opsdroid command works at all — not even `opsdroid version`.
2. **Cause:** `opsdroid/loader.py:18` does `from pkg_resources import iter_entry_points` at
   module import time; `pkg_resources` comes from setuptools, and setuptools 82.0.0 removed it.
3. **Why pip installs that setuptools:** `setup.cfg:69` declares `setuptools>=76.0.0` as a
   runtime dependency with no upper bound, and a Python 3.12+ venv starts with no setuptools
   at all, so the resolver picks the newest release.
4. **Not a Python-version bug:** this machine's Python 3.11 passes only because its venv ships
   setuptools 79.0.1, which already satisfies the floor. Remove that bootstrap copy the way
   Debian's packaging does and 3.11 fails identically.
5. **Fix boundary:** migrate that one import to `importlib.metadata`; preserve group filtering,
   the `name -> entry point` mapping and `.load()`; one path for 3.11–3.13; no pin. Capping
   `setuptools<82` works today, but one `pip install --upgrade setuptools` brings the failure
   straight back.

## Contents

| File | What it is |
|---|---|
| `01-reproduction-guide.md` | Preconditions, exact commands, expected vs actual at each step |
| `02-version-matrix.md` | 18 recorded cases across Python 3.11/3.12/3.13, PyPI and source, four setuptools versions |
| `03-root-cause.md` | The call chain with file and line numbers, and why the dependency exists |
| `04-fix-interface-contract.md` | The seven behaviours a correct fix must preserve, plus two traps |
| `05-evidence-index.md` | Provenance for every claim: logs, transcripts, sources |
| `06-contribution-statement.md` | First-person account of this work package and its handovers |

## Reproducing the failure

```bash
python3 -m venv /tmp/venv2073          # Python 3.12 or newer
. /tmp/venv2073/bin/activate
python -m pip install --upgrade pip
pip install opsdroid                   # exits 0 — the install succeeds
python -m pip show setuptools          # Version: 84.0.0
python -c "import pkg_resources"       # ModuleNotFoundError
opsdroid version                       # ModuleNotFoundError at loader.py:18
```

Two traps make this easy to misreport, and both are covered in `01-reproduction-guide.md`:
the install log looks clean, and a venv that already ships setuptools hides the bug entirely.

## Scope and limits

* No production code is modified here; the fix belongs to the implementation branch.
* The Debian Trixie condition is **modelled** by removing the bootstrap setuptools from a venv,
  which is the mechanism Debian's packaging produces. This is a faithful model of the
  mechanism, not a Debian run, and it is labelled as a model wherever it appears.
* Every claim about the project is linked to a file, line number, log or transcript in
  `05-evidence-index.md`.
