# Version matrix — what fails and what works

**Deliverable 2** · Author: Chenyan Zhou (u8071817)
Generated from `../experiments/results/*.json` by `../experiments/summarise.py`
(`../experiments/RESULTS-MATRIX.md` is the machine-generated copy; this page adds the
interpretation).

---

## Environment in which these numbers were produced

| | |
|---|---|
| Host | macOS 15.3 (build 24D2059), arm64 |
| Interpreters | CPython 3.11.17, 3.12.15, 3.13.16 (uv-managed standalone builds, installed under `a5/pythons/`) |
| pip | 26.2.1 |
| opsdroid (PyPI cases) | 0.31.0 — the only file on PyPI for 0.31.0 is the sdist `opsdroid-0.31.0.tar.gz` |
| opsdroid (local cases) | `0.31.0+3.g4f1ef7c`, i.e. `main` at `4f1ef7c` |
| setuptools versions observed | 79.0.1 (3.11 bootstrap), 81.0.0, 84.0.0 |
| Isolation | one fresh `python -m venv` per case; `HOME` redirected inside the workspace so appdirs paths are writable |

**No Debian machine was available.** Debian Trixie's relevant property is that
`python3 -m venv` produces an environment with no setuptools at all. That is modelled
with `--drop-setuptools`, which uninstalls the bootstrap copy. It is a faithful model of
the *mechanism* and is labelled as a model everywhere, not presented as a Debian run.

---

## The matrix

| case | Python | opsdroid source | setuptools | pin / constraint | install | `opsdroid version` | `opsdroid start` | later operation |
|---|---|---|---|---|---|---|---|---|
| `A-313-pypi-latest` | 3.13.16 | PyPI 0.31.0 | 84.0.0 | none | yes | FAIL (ModuleNotFoundError) | FAIL (ModuleNotFoundError) | - |
| `A2-312-pypi-latest` | 3.12.15 | PyPI 0.31.0 | 84.0.0 | none | yes | FAIL (ModuleNotFoundError) | FAIL (ModuleNotFoundError) | - |
| `A3-311-pypi-latest` | 3.11.17 | PyPI 0.31.0 | 79.0.1 | none | yes | PASS | PASS | - |
| `A4-311-pypi-noboot` | 3.11.17 | PyPI 0.31.0 | 84.0.0 | no bootstrap setuptools | yes | FAIL (ModuleNotFoundError) | FAIL (ModuleNotFoundError) | - |
| `F-313-local-latest` | 3.13.16 | repo `main` | 84.0.0 | none | yes | FAIL (ModuleNotFoundError) | FAIL (ModuleNotFoundError) | - |
| `M-313-pypi-all` | 3.13.16 | PyPI 0.31.0 `[all]` | 84.0.0 | none | yes | FAIL (ModuleNotFoundError) | FAIL (ModuleNotFoundError) | - |
| `B-313-constraint-lt82` | 3.13.16 | PyPI 0.31.0 | 81.0.0 | setuptools<82 | yes | PASS | PASS | - |
| `B2-312-constraint-lt82` | 3.12.15 | PyPI 0.31.0 | 81.0.0 | setuptools<82 | yes | PASS | PASS | - |
| `B3-311-constraint-lt82` | 3.11.17 | PyPI 0.31.0 | 79.0.1 | setuptools<82 | yes | PASS | PASS | - |
| `C-313-downgrade-after` | 3.13.16 | PyPI 0.31.0 | 81.0.0 | `setuptools<82` after install | yes | FAIL (ModuleNotFoundError) | FAIL (ModuleNotFoundError) | then `pip install "setuptools<82"` -> **PASS** |
| `H-313-prepin-then-upgrade` | 3.13.16 | PyPI 0.31.0 | 84.0.0 | pre-pin `setuptools<82` | yes | PASS | PASS | `pip install -U setuptools` -> **FAIL (ModuleNotFoundError)** |
| `G-313-poc-a` | 3.13.16 | PoC: aliased import | 84.0.0 | **none** | yes | PASS | PASS | - |
| `I-313-poc-b` | 3.13.16 | PoC: helper seam | 84.0.0 | **none** | yes | PASS | PASS | - |
| `J-313-poc-naive` | 3.13.16 | PoC: naive rename | 84.0.0 | none | yes | PASS | **FAIL (UnboundLocalError)** | - |
| `G2-312-poc-a` | 3.12.15 | PoC: aliased import | 84.0.0 | none | yes | PASS | PASS | - |
| `G3-311-poc-a` | 3.11.17 | PoC: aliased import | 79.0.1 | none | yes | PASS | PASS | - |
| `G4-311-poc-a-noboot` | 3.11.17 | PoC: aliased import | 84.0.0 | no bootstrap setuptools | yes | PASS | PASS | - |
| `G5-313-poc-a-nosetuptools` | 3.13.16 | PoC: aliased import | 84.0.0 | none; setuptools removed afterwards | yes | PASS | PASS | `pip uninstall setuptools` -> **PASS** |

Figure: `../experiments/figures/fig4-version-matrix.png`

---

## How to read it

### The install never fails

`install` is `yes` in all eighteen rows. That is the first thing worth internalising:
this bug is invisible in the install log, and any bug report that stops at "pip failed"
would be wrong.

### setuptools 82 is the boundary, exactly

* setuptools **84.0.0** → `ModuleNotFoundError` (rows A, A2, A4, F, M).
* setuptools **81.0.0** → works, with a deprecation warning on every start (row B).
* setuptools **79.0.1** → works (row B3, and A3 by accident).

The setuptools changelog puts the removal at v82.0.0 (2026-02-08), and the measured
boundary agrees: `< 82` works, `>= 82` does not.

### `A3` passing is the trap, not a counterexample

On Python 3.11.17 this interpreter's fresh venv contains **setuptools 79.0.1**, because
that CPython build's `ensurepip` still bundles a setuptools. 79.0.1 already satisfies
opsdroid's `setuptools>=76.0.0` floor, so pip leaves it alone, `pkg_resources` survives,
and everything passes.

`A4` is the same interpreter with that bootstrap copy removed first — which is what
Debian's `python3-venv` produces — and it installs setuptools 84.0.0 and **fails**.

> **Therefore: the failure is decided by the resolved setuptools version, not by the
> Python minor version.** Reporting "works on 3.11, fails on 3.13" would be wrong; the
> correct statement is "works when setuptools < 82 ends up installed".

The same effect appears on the fixed code: `G3` (3.11) kept setuptools 79.0.1, so it is
not evidence that the fix was needed there. `G4` is the meaningful 3.11 case.

### `main` fails exactly like the release

`F-313-local-latest` installs from the local `main` checkout (`0.31.0+3.g4f1ef7c`) and
fails identically. This rules out "the reporter used an old release". It is also why the
fix has to go upstream rather than being worked around by pinning the installed wheel.

### The workaround works, and is not durable

* `C` measures the workaround from the issue thread: after `pip install "setuptools<82"`
  the same venv goes from failing to passing.
* `H` measures its durability: pre-pin `<82`, everything passes, then one
  `pip install --upgrade setuptools` (a command any tool or developer may run) installs
  84.0.0 and the identical `ModuleNotFoundError` returns.

Pinning also keeps a deprecation warning on every single start:

```
UserWarning: pkg_resources is deprecated as an API. ... The pkg_resources package is
slated for removal as early as 2025-11-30. Refrain from using this package or pin to
Setuptools<81.
```

Figure: `../experiments/figures/fig5-workaround-not-durable.png`

### The fix works where `main` does not

`G` (aliased import) and `I` (module-level helper seam) both pass on Python 3.13 with
**setuptools 84.0.0 and no pin at all** — the exact environment in rows A and F. `G2`
and `G4` extend that to 3.12 and to 3.11 without a bootstrap setuptools. `G5` goes
further and removes setuptools entirely after installation; opsdroid still starts.

### One wrong way to write the fix

`J` is the naive one-line rename. It imports cleanly and `opsdroid version` works, but
`opsdroid start` fails with

```
UnboundLocalError: cannot access local variable 'entry_points' where it is not
associated with a value
```

because `loader.py:456` assigns to a local variable of the same name. This is why the
hand-over to Linrui is a contract plus a pitfall, not a suggested diff, and why Weixin
needs an `opsdroid start`-level test rather than only an import test.

Figure: `../experiments/figures/fig3-unboundlocalerror.png`

---

## Answers to the five questions in my brief

1. **Which stage fails?** Not installation. The install exits 0 and builds
   `opsdroid-0.31.0-py3-none-any.whl` successfully. The failure is at **run time, on
   `import`**, the first time the CLI is loaded — so every command is affected,
   including `opsdroid version`. Evidence: rows A/A2/F/M, plus
   `../experiments/figures/fig1-install-succeeds.png` and `fig2-modulenotfounderror.png`.
2. **Which combinations fail?** Any combination in which the *resolved* setuptools is
   `>= 82`: 3.12, 3.13, and 3.11 with no bootstrap setuptools. Combinations where
   setuptools `< 82` ends up installed work, which includes this machine's 3.11 by
   accident. Python version alone does not determine the outcome.
3. **Is the failure only the runtime import in `loader.py`?** Yes for production code —
   `loader.py:18` is the only `pkg_resources` usage in the shipped package (verified by
   grep across `opsdroid/`). But it is not the only file that has to change:
   `tests/test_loader.py` imports `pkg_resources` and constructs
   `pkg_resources.EntryPoint(..., dist=...)`, so under setuptools ≥ 82 the test suite
   cannot even be collected.
4. **Why does downgrading work, and why is it not the long-term fix?** It restores a
   module that setuptools 82 deleted, so the existing import succeeds again — see rows C
   and B. It is not a fix because it (a) constrains every user's environment forever,
   (b) emits a deprecation warning on every start, (c) is trivially undone by any later
   setuptools upgrade (row H), and (d) pins the project to a setuptools the maintainers
   have already announced they will keep shrinking.
5. **What entry-point behaviour must be preserved by `importlib.metadata`?** Group
   filtering, `.name`, `.load()` returning the same object, and the
   `{ep.name: ep}` mapping — all measured as equivalent in
   `../experiments/ENTRYPOINT-API-COMPARISON.md`. Two things are *not* equivalent and
   need conscious handling: the `dist=` constructor used by the unit test (must be
   rewritten) and the visibility of directories appended to `sys.path` after
   `pkg_resources` was first imported (`importlib.metadata` sees them,
   `pkg_resources` does not). Details in
   [`04-fix-interface-contract.md`](04-fix-interface-contract.md).
