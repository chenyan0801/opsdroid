# Evidence index

**Deliverable 5.** Every claim in [`03-root-cause.md`](03-root-cause.md),
[`07-report-draft.md`](07-report-draft.md) and
[`04-fix-interface-contract.md`](04-fix-interface-contract.md) traces back to an entry
here. Nothing in this work package is asserted without a recorded transcript.

All paths are relative to `<workspace>/` unless stated otherwise.

---

## 1. Primary sources

### Starting state of the working tree (recorded before any experiment)

My brief asked me to record the existing tree state and not to disturb anyone else's
work. Everything below was captured before the first experiment and re-checked at the
end; it is unchanged.

```
$ mkdir -p <workspace> && cd <workspace>
$ git clone https://github.com/opsdroid/opsdroid.git upstream-opsdroid
$ cd upstream-opsdroid && git status --short --branch
## main...origin/main
$ git remote -v
origin  https://github.com/opsdroid/opsdroid.git (fetch)
origin  https://github.com/opsdroid/opsdroid.git (push)
$ git log --oneline origin/main..HEAD      # commits I made: none
$ git reflog show --date=iso | head -1
4f1ef7c HEAD@{2026-10-14 17:58:37 +1100}: clone: from https://github.com/opsdroid/opsdroid.git
```

There was no pre-existing opsdroid checkout on this machine (`<home>/...`
from the brief is another member's machine), so nothing of anyone else's was touched:
I cloned my own reference copy and never wrote to it. **No commit, branch, push, issue,
comment or PR was created, and no maintainer was contacted.**

| What | Where |
|---|---|
| The issue | <https://github.com/opsdroid/opsdroid/issues/2073> — opened 2026-04-22 by `szimszon`, **open**, unassigned, 0 linked PRs |
| Confirming comment | `aquatix`, 2026-08-31: *"I encountered the same issue. I was able to get OpsDroid running by installing setuptools v81 with a `pip install "setuptools<82"`"* |
| Repository analysed | <https://github.com/opsdroid/opsdroid> at `4f1ef7c45a2ed26a95fb88b8fdb7e4deb33bde4a` (`main`, 2025-11-24) — local clone at `a5/upstream-opsdroid` |
| Release used for "PyPI" cases | opsdroid 0.31.0, sdist only (`opsdroid-0.31.0.tar.gz`, uploaded 2025-04-30) |
| setuptools removal | setuptools v82.0.0 changelog (2026-02-08): *"`pkg_resources` has been removed from Setuptools."* — <https://setuptools.pypa.io/en/latest/history.html#v82-0-0> |
| setuptools removal ticket | [pypa/setuptools#3085](https://github.com/pypa/setuptools/issues/3085), cited in the issue report as #5174 |
| Migration guidance | <https://importlib-metadata.readthedocs.io/en/latest/migration.html> — maps `pkg_resources.iter_entry_points()` to `importlib.metadata.entry_points()` |
| Why the venv has no setuptools | CPython [gh-95299 / PR #101039](https://github.com/python/cpython/pull/101039) — ensurepip stopped installing setuptools in Python 3.12 |

### Source lines cited in the analysis

| File | Line(s) | Relevance |
|---|---|---|
| `upstream-opsdroid/opsdroid/loader.py` | 18 | `from pkg_resources import iter_entry_points` — **the failing import** |
| | 74–127 | `Loader.import_module`, the four-step import strategy |
| | 82–86 | `config["entrypoint"].load()` — the `.load()` requirement |
| | 425 | `entry_points.get(config["name"], None)` — the name-mapping requirement |
| | 451 | `sys.path.append(DEFAULT_MODULE_DEPS_PATH)` — before the lookup |
| | 453–456 | group name construction + `iter_entry_points(group=epname)` |
| | 466–469 | `entrypoint` truthiness prevents a git clone of an installed package |
| | 366 vs 455 | `"parsers"` + `"s"` produces the group `opsdroid_parserss` (pre-existing bug) |
| `upstream-opsdroid/setup.cfg` | 69 | `setuptools>=76.0.0` in `install_requires` |
| | 82 | `console_scripts: opsdroid = opsdroid.cli:cli` |
| `upstream-opsdroid/opsdroid/cli/__init__.py` | 5–8 | every sub-command imported eagerly |
| `upstream-opsdroid/opsdroid/core.py` | 23, 78 | imports `Loader` and instantiates it |
| | 298, 384 | `module.__dict__` scan for `Skill`/`Connector` subclasses |
| `upstream-opsdroid/tests/test_loader.py` | 10, 245–246, 253–254 | `pkg_resources` import, `dist=` constructor, `mock.patch` target |
| `upstream-opsdroid/docs/packaging.md` | — | entry points must resolve to a **module**, e.g. `zodb = opsdroid_zodb.database` |
| `upstream-opsdroid/docs/maintaining/supported-python-versions.md` | — | 3.11 / 3.12 / 3.13 are all supported |
| `upstream-opsdroid/.github/workflows/ci.yml` | 16 | CI matrix is `["3.11", "3.12", "3.13"]` |

### Git history evidence

```
$ git log --oneline -S "setuptools>=76.0.0" -- setup.cfg
2266cf1 Fixing the CI build by adjusting the supported Python versions and
        switching no longer compatible dependencies (#2054)     (2025-03-11)
```

The same commit raised `python_requires` from `>=3.8` to `>=3.11`. Recorded from
`a5/upstream-opsdroid`.

---

## 2. Recorded experiments

Each case has a **full transcript** (`experiments/logs/<case>.log` — every command,
exit code, stdout and stderr) and a **machine-readable record**
(`experiments/results/<case>.json` — the same data plus a verdict per probe).
Figures rendered from these files are in `experiments/figures/`.

Reproduce everything with `cd experiments && bash run_matrix.sh`.

### Baseline — latest setuptools, no pin

| Case | Python | Source | setuptools | Outcome |
|---|---|---|---|---|
| `A-313-pypi-latest` | 3.13.16 | PyPI 0.31.0 | 84.0.0 | install exit 0; all five probes fail with `ModuleNotFoundError` |
| `A2-312-pypi-latest` | 3.12.15 | PyPI 0.31.0 | 84.0.0 | same |
| `A3-311-pypi-latest` | 3.11.17 | PyPI 0.31.0 | **79.0.1** | **passes** — the venv's bootstrap setuptools already satisfies `>=76` |
| `A4-311-pypi-noboot` | 3.11.17 | PyPI 0.31.0 | 84.0.0 | fails — same interpreter, bootstrap setuptools removed first |
| `F-313-local-latest` | 3.13.16 | repo `main` | 84.0.0 | fails — so this is not a stale-release problem |

`A3` vs `A4` is the key pair: identical Python, opposite results, and setuptools is the
only variable. Files: `logs/A-313-pypi-latest.log` (contains the byte-identical
traceback from the issue report, probe D2), `logs/A4-311-pypi-noboot.log`.

### The workaround, and why it is not a fix

| Case | Procedure | Outcome |
|---|---|---|
| `B-313-constraint-lt82` | `pip install -c setuptools<82` | all probes pass; setuptools 81.0.0; deprecation warning on every run |
| `B2-312-constraint-lt82` | same, Python 3.12 | passes |
| `B3-311-constraint-lt82` | same, Python 3.11 | passes |
| `C-313-downgrade-after` | default install, then `pip install "setuptools<82"` | fails before the downgrade, passes after |
| `H-313-prepin-then-upgrade` | pre-pin `<82`, then `pip install --upgrade setuptools` | passes, then **fails again** with the identical `ModuleNotFoundError` |

`H` is the durability argument: a single routine command undoes the workaround. Files:
`logs/B-313-constraint-lt82.log`, `logs/H-313-prepin-then-upgrade.log`.

### Proof-of-concept `importlib.metadata` variants (read-only copies)

Built by `experiments/make_poc.sh` into `a5/poc/`. **These are throwaway verification
artefacts, not the group's implementation** — the real fix belongs to Linrui in the
group fork.

| Case | Variant | Python | setuptools | Outcome |
|---|---|---|---|---|
| `G-313-poc-a` | alias the import | 3.13.16 | **84.0.0, no pin** | all probes pass, `opsdroid start` runs |
| `I-313-poc-b` | module-level `iter_entry_points` seam | 3.13.16 | 84.0.0, no pin | all probes pass |
| `J-313-poc-naive` | naive one-line rename | 3.13.16 | 84.0.0 | imports fine, but `opsdroid start` → **`UnboundLocalError`** |
| `G2-312-poc-a` | alias the import | 3.12.15 | 84.0.0 | passes |
| `G3-311-poc-a` | alias the import | 3.11.17 | 84.0.0 | passes |
| `G4-311-poc-a-noboot` | alias, no bootstrap setuptools | 3.11.17 | 84.0.0 | passes |
| `G5-313-poc-a-nosetuptools` | alias, then uninstall setuptools | 3.13.16 | **absent** | passes — setuptools is not needed at run time |

`G`/`I` are the causation proof; `J` is the pitfall handed to Linrui; `G5` shows the
runtime dependency can be dropped.

### The literal command from the issue report

| Case | Command | Outcome |
|---|---|---|
| `M-313-pypi-all` | `pip install "opsdroid[all]"` | fails identically — the `[all]` extra exists (it is built dynamically in `setup.py`) and makes no difference |

### opsdroid's own test suite vs the setuptools version

`experiments/logs/TESTSUITE-evidence.log`. Same environment throughout, Python 3.13.16,
opsdroid installed editable from `main`:

| Configuration | Result |
|---|---|
| `main`, setuptools 84.0.0 | collection error at `tests/test_loader.py:10` (`import pkg_resources`) |
| `main`, setuptools 81.0.0 | **48 passed** in 7.59 s |
| loader patched to `importlib.metadata`, setuptools 84.0.0, test file unchanged | collection error, identical to the first row |

The third row is the one that matters for planning: the loader fix alone leaves CI red,
so the test file must change in the same PR.

### Independent verification of a candidate fix

`experiments/verify_fix.sh` re-runs *my* reproduction against an arbitrary checkout,
without reusing the harness's classifier, and refuses to report a pass unless the
environment genuinely has no `pkg_resources` (a negative control). It was validated in
both directions:

| Candidate | Transcript | Result |
|---|---|---|
| `a5/upstream-opsdroid` (`main`, unfixed) | `experiments/logs/VERIFY-main-FAIL.log` | **2 passed, 4 failed**, exit 1 |
| `a5/poc/opsdroid-variant-a` (fixed) | `experiments/logs/VERIFY-poc-a-PASS.log` | **9 passed, 0 failed**, exit 0 |

The passing run also captures the direct proof that entry points were used:

```
DEBUG opsdroid.loader Found installed package for skill 'fixtureskill' support.
DEBUG opsdroid.loader Found installed package for connector 'fixtureconn' support.
```

Run it against Linrui's implementation once it exists:

```bash
cd <workspace>/experiments
bash verify_fix.sh /path/to/linrui/opsdroid
```

Exit code 0 means the failure I reproduced no longer occurs; 1 means it does; 2 means the
check could not be run meaningfully (for example the environment still had
`pkg_resources`, which would make the result vacuous).

### Regression detail: two harness bugs I had to find and fix

Recorded here because a reproduction harness that reports a wrong verdict is worse than
none, and both mistakes are easy to repeat.

1. **A false `PASS`.** The first classifier treated the string `Started opsdroid` as
   success, but `opsdroid/cli/utils.py` prints that *before* loading any module. The
   naive-rename case (`J`) was therefore reported as passing while actually raising
   `UnboundLocalError`. Fixed by deciding from exit codes and tracebacks only; see the
   `classify()` docstring in `experiments/run_case.py`.
2. **A collision on TCP port 8080.** `opsdroid start` binds `0.0.0.0:8080`, and cases run
   in parallel, so two working cases fought over the port. The visible symptom was not
   the real error but opsdroid's own exception handler raising
   `TypeError: unhashable type: 'dict'` (`opsdroid/core.py:164`,
   `warnings.warn(context)` where `context` is a dict). Each case now gets its own port.
   *(That secondary bug — an error handler masking the real error — is a separate
   opsdroid issue and is out of scope here.)*

---

## 3. Entry-point API comparison

`experiments/ENTRYPOINT-API-COMPARISON.md` (regenerate with
`experiments/compare_entrypoint_api.py` in a venv that has setuptools < 82 plus the
fixtures). Headline results:

* group filtering, `.name`, `.load()` and the `{ep.name: ep}` mapping are equivalent;
  `.load()` returns the *same object* for the same distribution;
* `importlib.metadata.EntryPoint(name, value, group)` rejects the `dist=` keyword that
  `tests/test_loader.py:246` relies on → the test must be rewritten;
* **one real behaviour change**: a directory appended to `sys.path` *after*
  `pkg_resources` was imported is invisible to `pkg_resources` but visible to
  `importlib.metadata`. Measured with the `opsdroid-fixture-late` distribution. This
  matters because `loader.py:451` appends `DEFAULT_MODULE_DEPS_PATH` immediately before
  the lookup at line 456.

---

## 4. Test fixtures I built for the group

| Path | Purpose |
|---|---|
| `experiments/fixture/` | installable distribution registering `fixtureconn` in `opsdroid_connectors` and `fixtureskill` in `opsdroid_skills` — proves group isolation and end-to-end entry-point loading |
| `experiments/fixture-dup/` | registers a *duplicate* name `fixtureconn` plus `zzz_dup_only` — used for the tie-break/ordering comparison |
| `experiments/fixture-late/` | uniquely named `lateconn` — used for the late-`sys.path` comparison |
| `experiments/dist/*.whl` | pre-built wheels; `build_fixtures.sh` exists because parallel `pip install <dir>` calls corrupt each other's `build/` tree |
| `experiments/run_case.py`, `run_matrix.sh`, `summarise.py`, `make_poc.sh`, `build_fixtures.sh`, `make_figures.py`, `compare_entrypoint_api.py` | the harness |

---

## 5. Figures for the report and slides

Rendered from the recorded logs by `experiments/make_figures.py` into
`experiments/figures/`:

| Figure | File | Shows |
|---|---|---|
| 1 | `fig1-install-succeeds.png` | `pip install opsdroid` exits 0 and installs setuptools 84.0.0 |
| 2 | `fig2-modulenotfounderror.png` | the failure at `loader.py:18`, matching the issue |
| 3 | `fig3-unboundlocalerror.png` | the naive rename failing differently |
| 4 | `fig4-version-matrix.png` | the full matrix as a table |
| 5 | `fig5-workaround-not-durable.png` | `setuptools<82` works, then `pip -U setuptools` breaks it again |

---

### Upstream implementation PR

`experiments/logs/VERIFY-pr2079.log` and `experiments/logs/VERIFY-pr2079-tests.log`,
produced by `verify_fix.sh` and a pytest run against `a5/review/pr2079` (a `git archive`
of PR `d4ef713`):

| Check | Result |
|---|---|
| Verifier on PR #2079, Python 3.13.16, setuptools 84.0.0, no pin | **9 passed, 0 failed**, exit 0 |
| Entry-point loading proven from opsdroid's own debug log | yes (`fixtureconn`, `fixtureskill`) |
| Works with setuptools uninstalled entirely | yes |
| `pytest tests/test_loader.py`, setuptools 84.0.0 | **48 passed** on 3.11.17, 3.12.15 and 3.13.16 |
| Verifier on unpatched `main`, same environment | 2 passed, 4 failed, exit 1 |

Full analysis and the small gaps against my contract:
[`10-implementation-review.md`](10-implementation-review.md).

---

## 6. Self-audit

`experiments/SELF-AUDIT.txt` is the output of `experiments/audit_claims.py`, which
re-derives every load-bearing claim in these deliverables from the raw records and from
the source files, and fails loudly if any of them stops holding. It is the answer to
"how do you know the report is not overstated?".

Re-run it at any time:

```bash
cd <workspace>/experiments && python3 audit_claims.py
```

Current result: **all 68 checks hold**. It covers, among others:

* install exits 0 while every probe fails, for all four baseline configurations;
* the 3.11 false negative and its control (`A3` vs `A4`, same interpreter);
* the workaround working (`B`, `C`) and being undone by one command (`H`);
* the fix working with setuptools 84.0.0 and no pin (`G`, `I`, `G2`, `G4`) and with
  setuptools absent entirely (`G5`);
* the naive rename failing only at `opsdroid start` (`J`);
* the exact source lines: `loader.py:18`, `setup.cfg`'s `setuptools>=76.0.0`,
  `tests/test_loader.py`'s `pkg_resources` import and `dist=` usage, and the
  `"opsdroid_{}s"` group construction;
* the recorded API differences and the verifier's before/after results.

---

## 7. What is **not** evidence

To keep the hand-over honest:

* The `A3-311-pypi-latest` pass is **not** a working configuration; it is an artefact of
  that interpreter build shipping setuptools 79.0.1. See `A4` for the same interpreter
  without it.
* The PoC copies in `a5/poc/` are **not** a proposed implementation and must not be
  submitted. They exist only to prove the direction of causation. Linrui's implementation
  is the deliverable.
* `opsdroid start` being "killed by the harness timeout" is the **expected** result of a
  successful start (the bot runs until interrupted); it is recorded as
  `PASS: still running when killed after Ns`, not as a failure.
* No experiment was run against a real Debian Trixie machine. The Debian condition is
  modelled by removing the bootstrap setuptools from the venv (`--drop-setuptools`),
  which is the mechanism Debian's packaging produces. This is stated plainly rather than
  presented as a Debian run.
* Nothing was pushed to any remote, and no issue, comment or PR was created.
  `a5/upstream-opsdroid` is at `## main...origin/main` with an empty
  `origin/main..HEAD` log.
