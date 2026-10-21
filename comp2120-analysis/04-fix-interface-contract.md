# Fix interface contract — for Linrui (implementation) and Weixin (tests)

**From:** Chenyan Zhou (u8071817), reproduction & root-cause work package
**Subject:** what the fix must satisfy, and what it must not change
**Root cause:** [`03-root-cause.md`](03-root-cause.md) · **Issue:** <https://github.com/opsdroid/opsdroid/issues/2073>

This is deliberately a *contract*, not an implementation. It states the observable
behaviour the fix has to preserve and the acceptance tests that decide whether it
works. How you get there is your call.

---

## 1. Hard requirements

| # | Requirement | Why |
|---|---|---|
| R1 | After the fix, importing `opsdroid.loader` must not import `pkg_resources`. | The module does not exist on setuptools ≥ 82; this is the exact failure in #2073. |
| R2 | `_load_modules` must still enumerate entry points **filtered by the group** `opsdroid_{modules_type}s`. | Passing the wrong group silently loads nothing, or loads an unrelated extension type. |
| R3 | It must still build the `name -> entry point` mapping keyed on `ep.name`. | `loader.py:425` looks modules up by their configured name. |
| R4 | `ep.load()` must still be called and its return value used as the module. | `loader.py:86` returns it; `core.py:298`/`:384` then scan its `__dict__` for `Skill`/`Connector` subclasses. |
| R5 | One code path must serve Python 3.11, 3.12 and 3.13. | All three are supported (`docs/maintaining/supported-python-versions.md`) and all three are in CI. Do not add version branching. |
| R6 | `pip install opsdroid` in a *fresh* venv on Python 3.13 must produce a working `opsdroid` command with no pin and no extra manual step. | This is the user-visible bug report. A fix that only works with `setuptools<82` is not a fix. |
| R7 | `tests/test_loader.py` must import and run without `pkg_resources` installed. | Otherweise the test suite itself breaks under setuptools ≥ 82. |

R1–R5 are satisfied by `importlib.metadata`; it is in the standard library from
Python 3.8 and its selectable-group API (`entry_points(group=...)`) exists from 3.10,
so nothing is needed for 3.11+. `pkg_resources.iter_entry_points()` is the API the
setuptools project itself maps to `importlib.metadata.entry_points()`
([migration guide](https://importlib-metadata.readthedocs.io/en/latest/migration.html)).

---

## 2. Concrete pitfall to avoid — do not rename blindly

The obvious one-line diff **does not work**. This is the single most useful thing in
this document, because it fails only at run time and only when a module is actually
loaded:

```python
from importlib.metadata import entry_points      # added
...
entry_points = {ep.name: ep for ep in entry_points(group=epname)}   # loader.py:456
```

The assignment two lines later makes `entry_points` a **local variable of
`_load_modules` for the whole function body**, so the right-hand side raises:

```
File "opsdroid/loader.py", line 456, in _load_modules
    entry_points = {ep.name: ep for ep in entry_points(group=epname)}
                                          ^^^^^^^^^^^^
UnboundLocalError: cannot access local variable 'entry_points' where it is not
associated with a value
```

Reproduced end to end: `../experiments/logs/J-313-poc-naive.log`
(`opsdroid start` exits 1; the `opsdroid version` and import probes all pass, so it is
only visible once a connector or skill is configured).

Two ways out, both verified working in this environment:

* **Rename the local variable** (smallest diff): keep
  `from importlib.metadata import entry_points` and use a different name for the dict,
  e.g. `entry_points_by_name = {ep.name: ep for ep in entry_points(group=epname)}`,
  then update `loader.py:425` accordingly.
* **Keep a module-level seam** named `iter_entry_points`, so the production diff below
  `loader.py:18` is untouched:

  ```python
  from importlib.metadata import entry_points as _entry_points

  def iter_entry_points(group):
      """Yield the installed entry points belonging to ``group``."""
      return _entry_points(group=group)
  ```

Either is fine. Pick one and be consistent; do not do both.

---

## 3. Behavioural equivalence — what must be preserved

I compared the two APIs directly with a real third-party fixture distribution
installed (see `../experiments/compare_entrypoint_api.py` and
`../experiments/ENTRYPOINT-API-COMPARISON.md`). Summary of what opsdroid depends on:

Measured, not assumed: the full side-by-side run is in
[`../experiments/ENTRYPOINT-API-COMPARISON.md`](../experiments/ENTRYPOINT-API-COMPARISON.md),
produced by `../experiments/compare_entrypoint_api.py` with three real fixture
distributions installed (including one that deliberately registers a duplicate
entry-point name).

| Behaviour opsdroid relies on | `pkg_resources.iter_entry_points` | `importlib.metadata.entry_points` | Status |
|---|---|---|---|
| Group filtering (`group=` keyword) | yes | yes | equivalent — identical name sets for `opsdroid_connectors` |
| `.name` attribute | yes | yes | equivalent |
| `.load()` returns the target object | yes | yes | equivalent — the *same* object for the same distribution |
| `{ep.name: ep}` mapping incl. duplicate names | yes | yes | equivalent — same names, same order, same winner |
| Iteration order in this environment | distributions in `sys.path` order | same | identical in the measured case; still not a documented guarantee |
| `.load()` on a hand-constructed object (used by the unit test) | needs a `dist=` argument | `EntryPoint(name, value, group)`; **`dist=` raises `TypeError`** | **the test must be rewritten** |
| `.value` / `.group` attributes | absent | present | bonus, not required |
| `.module_name` / `.attrs` attributes | present | **absent** | not used by production code; the old test used them |
| Validates the entry point eagerly | yes | no — validate by touching a property (e.g. `.name`) | harmless here: `ep.name` is read immediately at `loader.py:456` |
| Checks that declared extras are installed on `.load()` | yes | no | opsdroid never uses extras in entry-point values, so no impact |
| Emits warnings of its own | yes (deprecation) | no | improvement |

Two differences that are worth a comment in the PR but do **not** block the fix:

* **Late additions to `sys.path` — measured, and this one is a real behaviour change.**
  `loader.py:451` appends `DEFAULT_MODULE_DEPS_PATH` to `sys.path` and `loader.py:456`
  then enumerates entry points immediately afterwards. `pkg_resources` reads a
  `working_set` snapshot built when it was first imported, so it never sees that
  directory; `importlib.metadata` re-scans `sys.path` on each call and **does** see it.
  Verified with a uniquely-named fixture distribution:

  ```
  importlib     before append:  ['fixtureconn', 'fixtureconn', 'zzz_dup_only']
  importlib     after  append:  ['fixtureconn', 'fixtureconn', 'lateconn', 'zzz_dup_only']
  pkg_resources before append:  ['fixtureconn', 'fixtureconn', 'zzz_dup_only']
  pkg_resources after  append:  ['fixtureconn', 'fixtureconn', 'zzz_dup_only']
  ```

  In practice this makes opsdroid *better* at finding extensions that were installed
  into its own module directory during the same process, and it is very unlikely that
  anything depends on the old blindness. Still, it belongs in the PR description so
  that a maintainer can agree rather than discover it.
* **Duplicate entry-point names.** Where two distributions register the same name, both
  APIs return both and the last one wins in the dict comprehension. I verified the
  winner is the same object under both APIs, and that a same-name *shadow copy* of a
  distribution on `sys.path` adds nothing under either API (both de-duplicate by project
  name and version). No action needed.

---

## 4. Acceptance tests — for Weixin

### T1 — the module must import with `pkg_resources` unavailable (regression test for #2073)

This is the one test that is guaranteed to fail on `main` and pass after the fix.

```python
def test_loader_import_does_not_need_pkg_resources():
    """opsdroid.loader must import in an environment without setuptools."""
    code = (
        "import sys\n"
        "sys.modules['pkg_resources'] = None\n"     # make `import pkg_resources` fail
        "import opsdroid.loader\n"
        "print('ok')\n"
    )
    result = subprocess.run([sys.executable, "-c", code],
                            capture_output=True, text=True)
    assert result.returncode == 0, result.stderr
    assert "ok" in result.stdout
```

Notes for whoever writes it:
* `sys.modules['pkg_resources'] = None` makes `import pkg_resources` raise
  `ImportError`, which is exactly what setuptools ≥ 82 does for real.
* Run it in a subprocess: `opsdroid.loader` may already be in `sys.modules` from other
  tests, which would make an in-process version pass vacuously.
* Before the fix this prints `ModuleNotFoundError: No module named 'pkg_resources'` and
  exits 1 — verified by hand on `main`.

### T2 — group selection, name mapping and `.load()` must still work

Port of the existing `test_import_module_from_entrypoint`
(`tests/test_loader.py:237-256`), which cannot be ported mechanically because
`pkg_resources.EntryPoint("myep", "os.path", dist=distro)` has no equivalent.

```python
from importlib.metadata import EntryPoint

def test_import_module_from_entrypoint(self):
    ep = EntryPoint("myep", "os.path", "opsdroid_databases")   # name, value, group
    ...
    with mock.patch("opsdroid.loader.entry_points") as mock_entry_points:   # or
    # with mock.patch("opsdroid.loader.iter_entry_points") as ...           # seam name
        mock_entry_points.return_value = (ep,)
        loaded = loader._load_modules("database", modules)

        mock_entry_points.assert_called_once_with(group="opsdroid_databases")  # R2
        self.assertEqual(loaded[0]["config"]["name"], "myep")                  # R3
        self.assertIs(loaded[0]["module"], os.path)                            # R4
```

* The mock target must match the name the fix actually defines — agree it with Linrui
  before writing the test, or the test will fail for the wrong reason.
* `assert_called_once_with(group=...)` is the part that catches a wrong or misspelled
  group, which is the realistic failure mode here (note the pre-existing
  `opsdroid_parserss` group bug at `loader.py:455`, which this must **not** change).
* `importlib.metadata.EntryPoint("myep", "os.path", "opsdroid_databases").load()`
  genuinely returns `posixpath`, so this test needs no distribution installed —
  verified.

### T3 — the group must follow the module type (cheap, catches copy/paste errors)

```python
@pytest.mark.parametrize("modules_type,expected", [
    ("database", "opsdroid_databases"),
    ("skill",    "opsdroid_skills"),
    ("connector","opsdroid_connectors"),
])
def test_group_name_per_module_type(modules_type, expected):
    ...
```

Deliberately omits `"parsers"`: the current code produces `opsdroid_parserss` there.
Assert the *current* behaviour (or leave it untested) so this PR stays a migration.

### T4 — end-to-end, with a real third-party package (strongest; optional but recommended)

I have already built the fixture. `../experiments/fixture/` is a tiny installable
distribution that registers

```
opsdroid_connectors  fixtureconn   = opsdroid_fixture_connector.connector
opsdroid_skills      fixtureskill  = opsdroid_fixture_skill.skill
```

After `pip install <fixture>`, this `configuration.yaml`

```yaml
connectors:
  fixtureconn:
    greeting: hello-from-entry-point
skills:
  fixtureskill: {}
```

must start opsdroid without `All connectors failed to load.` and without
`No skills in configuration`. This is the test that proves R2–R4 together, through the
real code path rather than a mock. It is also the test that would have caught the
naive-rename `UnboundLocalError` from §2.

### T5 — (only if the team also removes the dependency)

If `setuptools>=76.0.0` is dropped from `install_requires`, add a check that nothing in
the shipped package imports setuptools:

```python
def test_shipped_package_does_not_import_setuptools():
    ...  # grep-style walk over opsdroid/**.py, or import every module in a venv
         # where setuptools has been uninstalled
```

### Why the test change is not optional — measured

I ran opsdroid's own `tests/test_loader.py` three times in the same environment,
changing only the setuptools version and the loader patch
(`../experiments/logs/TESTSUITE-evidence.log`):

| Configuration | Result |
|---|---|
| A — `main`, setuptools 84.0.0 | **collection error**: `tests/test_loader.py:10: import pkg_resources` → `ModuleNotFoundError` |
| B — `main`, setuptools 81.0.0 | **48 passed** in 7.59 s |
| C — loader patched to `importlib.metadata`, setuptools 84.0.0, test file **unchanged** | **collection error**, identical to A |

Row C is the point: patching `opsdroid/loader.py` alone does **not** make the test suite
green under setuptools ≥ 82, because `tests/test_loader.py` imports `pkg_resources`
itself. A PR that changes only `loader.py` will fail CI on a modern environment, so the
test file must be updated in the same change.

---

## 5. Things that must NOT change in this PR

* **Do not add an upper bound to `setuptools`** as the fix. `setuptools<82` works today
  (verified, see `02-version-matrix.md`) but it is a countdown, not a fix: it pins the
  project to a setuptools that has already announced the removal, and any later
  `pip install --upgrade setuptools` or dependency that needs a newer setuptools
  re-breaks the exact same line. Evidence: `../experiments/logs/H-313-prepin-then-upgrade.log`.
* **Do not fix the `opsdroid_parserss` group typo** in the same change. Separate issue,
  separate PR.
* **Do not touch `opsdroid/core.py`.** The `module.__dict__` scan at `core.py:298`/`:384`
  is the contract that entry points must keep satisfying, not something to work around.
* **Do not change the four-step import order** in `Loader.import_module`
  (`loader.py:74-127`) or the `entrypoint`-implies-installed check at `loader.py:469`.

---

## 6. One decision for the group: keep or drop `setuptools>=76.0.0`?

`setup.cfg:69` lists `setuptools>=76.0.0` in `install_requires`. `git log -S` shows it
was added in commit `2266cf1` (March 2025, PR #2054) in the same change that raised
`python_requires` to `>=3.11` — i.e. it exists **only** so that `pkg_resources` is
importable at runtime. Nothing else in `opsdroid/` imports setuptools.

* **Drop it:** the runtime dependency disappears, and the requirement can never again
  pull in a setuptools that breaks the code. Build-time setuptools is unaffected —
  pip provisions it in the isolated build environment from `setup.py`'s
  PEP 517 fallback (there is no `pyproject.toml` in this repo).
* **Keep it:** harmless once nothing imports `pkg_resources`, and it keeps the PR
  smaller for reviewers.

My recommendation is to **drop it in a separate commit** (or at least mention it in the
PR description) so the maintainers can see the intent, and so the diff that fixes the
bug stays minimal and easy to review. It is not required for the fix to work.
