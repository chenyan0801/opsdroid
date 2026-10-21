# Personal contribution statement — Chenyan Zhou (u8071817)

**Deliverable 6.** First person, written to be pasted into the report as-is. Adjust the
delivery sentences if the hand-over happened differently from what is described here.

---

## My role

My work package was **reproducible reproduction and root-cause analysis** for opsdroid
issue [#2073](https://github.com/opsdroid/opsdroid/issues/2073). My brief was explicitly
*not* to write the fix: my conclusions were meant to become the factual baseline that
the implementation, test and verification work is built on, and my job was to prove the
conditions under which the bug occurs, where exactly it fails, and what a correct fix
must preserve.

I worked inside a dedicated workspace at `<workspace>` so that nothing I did
could disturb anyone else's checkout, and I kept my clone of `opsdroid` strictly
read-only — no commits, no branches, no pushes, and no contact with the upstream
maintainers, as required by our task brief.

## What I produced

**1. A reproduction that anyone can run.** I wrote
[`01-reproduction-guide.md`](01-reproduction-guide.md), which starts from nothing but
Python and pip, includes the exact commands and the expected output at each step, and
documents the two traps that make this bug easy to misreport: that the *install*
succeeds, and that the failure is decided by the resolved setuptools version rather than
the Python minor version.

**2. A scripted harness and a version matrix.** `a5/experiments/` contains a driver
(`run_case.py`), a matrix runner (`run_matrix.sh`) and a summariser (`summarise.py`).
Sixteen cases run in isolated virtual environments, and each one records every command,
exit code, stdout and stderr into `logs/<case>.log` and `results/<case>.json`. The
resulting matrix is [`02-version-matrix.md`](02-version-matrix.md). Building this was
the bulk of my time: the first version of the harness produced two *wrong* answers that
I had to chase down — a `PASS` verdict that came from matching the string
`Started opsdroid`, a line opsdroid prints *before* it loads any module, and a
`TypeError: unhashable type: 'dict'` that turned out to be two parallel test cases
fighting over TCP port 8080. Both are now fixed and documented in the guide, because a
reproduction harness that lies is worse than no harness.

**3. A root-cause analysis backed by source and stack traces.** I traced the import
chain from `setup.cfg`'s console-script entry point through `opsdroid/cli/`, `core.py`
and into `loader.py:18`, and confirmed it byte-for-byte against the reporter's
traceback. I found *why* `pkg_resources` is absent — setuptools 82.0.0 removed it — and
*why* pip installs that version: `setup.cfg:69` declares `setuptools>=76.0.0` as a
runtime dependency with no upper bound, and a Python 3.12+ venv starts with no
setuptools at all, so the resolver takes the newest release. `git log -S` then showed
that this dependency was added in commit `2266cf1` (March 2025) in the same change that
raised `python_requires` to `>=3.11` — that is, it exists *only* to keep
`loader.py:18` importable. That turned a vague "setuptools is too new" story into a
specific, checkable claim. Written up in [`03-root-cause.md`](03-root-cause.md).

**4. Evidence that it is causation and not correlation.** Two independent proofs: the
same venv works when setuptools is constrained to `<82`, and a throwaway copy of the
repository with only the import and its call site changed to `importlib.metadata`
starts successfully with **setuptools 84.0.0 and no pin** — the exact environment that
fails on `main`.

## Who I worked with, and what I gave them

Everything below was produced by me and handed over; none of it was written by another
member.

**To Linrui (implementation).** I gave the precise failure point — `opsdroid/loader.py:18`,
reached from every CLI sub-command, so there is no command that can run — together with
[`04-fix-interface-contract.md`](04-fix-interface-contract.md), which lists the five
behaviours the replacement must preserve: filter by the `opsdroid_{type}s` group, build
the `name -> entry point` mapping, call `.load()` and use its return value as the
module, work on 3.11/3.12/3.13 through one code path, and work with no pin at all. I
also gave Linrui the pitfall I found by experiment rather than by reading: the obvious
one-line rename

```python
from importlib.metadata import entry_points
...
entry_points = {ep.name: ep for ep in entry_points(group=epname)}   # loader.py:456
```

**does not work**, because the assignment two lines later makes `entry_points` a local
variable of `_load_modules` and the call raises
`UnboundLocalError: cannot access local variable 'entry_points'`. I reproduced that end
to end so Linrui would not have to discover it in review, and I offered two working
shapes (rename the local, or keep a module-level `iter_entry_points` seam) without
prescribing which to use. I also flagged that the parser group name is misspelled
(`opsdroid_parserss`, from `loader.py:455` plus `loader.py:366`) and asked that it be
left alone in this PR so the migration stays behaviour-preserving.

**To Weixin (tests).** I provided concrete pre-fix-failing / post-fix-passing test
conditions in §4 of the interface contract. The key one is a subprocess test that forces
`import pkg_resources` to fail (`sys.modules['pkg_resources'] = None`) and then imports
`opsdroid.loader`; it fails on `main` and passes after the fix, and it has to run in a
subprocess or it passes vacuously once another test has already imported the module. I
also worked out how the existing test has to change: `pkg_resources.EntryPoint("myep",
"os.path", dist=distro)` has no equivalent, because `importlib.metadata.EntryPoint` takes
`(name, value, group)` and rejects `dist=` — I verified that
`EntryPoint("myep", "os.path", "opsdroid_databases").load()` returns `posixpath` with no
distribution installed, so the test can be ported cleanly. Finally I built an installable
fixture package (`a5/experiments/fixture/`) that registers a connector and a skill in the
`opsdroid_connectors` and `opsdroid_skills` groups, so the group-selection behaviour can
be tested end to end rather than only through a mock.

**To Yifan (verification).** I provided the end-to-end reproduction commands, the
preconditions each one needs, and the matrix template (`run_case.py` /
`run_matrix.sh` / `summarise.py`), so the same evidence can be regenerated on another
machine without re-deriving it. I documented the harness caveats honestly — including
that `HOME` is redirected and that the fixture must be installed from a pre-built wheel
because parallel installs from a source directory corrupt each other's build tree — so
that a re-run on different hardware produces comparable numbers rather than surprises.

**To the whole group.** I recorded the project's conventions for Activity 5: the workflow
in `docs/contributing/workflow.md` (fork, feature branch, PR, review), the test
instructions in `docs/contributing/testing.md` and `tox.ini`, and the lint gate in
`.pre-commit-config.yaml` (black, flake8 with `setup.cfg`, pydocstyle). My last committed
task on this work package is to re-run my reproduction against Linrui's final
implementation and confirm that the exact failure I recorded no longer occurs — I do not
consider my part finished until that check passes.

## What I would do differently

Two things. First, I initially trusted log text as a success signal; it cost me a
misleading `PASS` and taught me that a harness must decide from exit codes and
tracebacks, not from strings a program prints early. Second, I should have made the
fixture package installable from a wheel before running cases in parallel — the race
condition produced a case that failed for a reason that had nothing to do with opsdroid,
and I nearly recorded it as a finding.
