# falsify — Find pytest tests that pass for the wrong reason

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/falsify**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

A pytest plugin and command that finds tests which pass for the wrong reason: assertions that are always true, assertions that never run, patches aimed at names that don't exist, and checks that only a comment satisfies.

## Get it

Buy falsify at https://dominares.org/tools/falsify: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the pytest plugin and the falsify command.

Requirements: The static checks need no other package (tomli comes with it on Python 3.10). The [coverage] extra adds coverage.py, which falsify dead-asserts needs.

This page documents version 1.0.2. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

**Your test suite is green. falsify checks whether it can fail.**

A passing test proves only that nothing it checked went wrong. falsify finds the tests that
could never have gone wrong in the first place: assertions that are always true, assertions
that never run, patches aimed at names that do not exist, source checks satisfied by a
comment, imports that break only when a rarely-used function runs, and waits that can hang
CI with no output.

It is a pytest plugin and a command-line tool, for Python 3.10+.

### Who it is for

Teams whose tests are increasingly written by AI coding agents, and anyone reviewing them.
An agent asked to make the suite pass will make it pass, and the shapes below are exactly
what "pass" looks like when nobody checks how:

```python
assert result.saved or True                                    # always true
monkeypatch.setattr(client, "fetch_rates", fake, raising=False)  # client has no fetch_rates
if note["active"]:
    assert "limit" in note["text"]                             # this branch never runs
src = SETTINGS.read_text()
assert "RETRY_LIMIT" in src                                    # only a comment says it
```

Every one of those is green. Every one of them checks nothing. A reviewer reading hundreds
of generated tests will not spot them; falsify does, in seconds, on every run.

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

Install it into the same virtual environment as your project and its tests. The only
runtime dependency is `tomli`, and only on Python 3.10.

### 60-second quickstart

```bash
cd your-project
falsify check                     # static rules over the whole project; no code is run
pytest --falsify                  # your normal test run, failing if falsify finds anything
falsify dead-asserts              # runs the suite under coverage; asserts that never ran
falsify comment-guards            # reruns source-reading tests with comments stripped
```

`falsify check` prints one line per finding and one fix hint per rule:

```
tests/test_orders.py:41:5: [unfalsifiable-assert] `or True` short-circuits the assertion to always-true
    fix: assert on the real outcome. For a deliberate determinism check, add `# falsify: ok <why>`.
tests/test_rates.py:12:5: [phantom-patch] patches `app.client.fetch_rates` with raising=False, and `app.client.fetch_rates` does not exist, so this line patches nothing

falsify: 2 finding(s) (48 files checked, 9 rules)
```

To make it part of every test run, add it to your pytest options (the pytest runs that
`dead-asserts` and `comment-guards` start never repeat it):

```toml
[tool.pytest.ini_options]
addopts = "--falsify"
```

### What it checks

#### Static rules (`falsify check`, `pytest --falsify`)

These read your code; nothing is imported from your project and nothing is run. (To
resolve a patch target in the standard library or an installed package, `phantom-patch`
imports that module, as the test itself would.)

| Rule | Scope | Finds |
|---|---|---|
| `unfalsifiable-assert` | tests | `assert x or True`, `assert True`, `assert "text"`, `assert [1, 2]`, `assert (x, "msg")`, `assert "" in s`, `assert f(x) == f(x)`, and the unittest forms `assertTrue(1)`, `assertEqual(a, a)`. |
| `phantom-patch` | tests | `monkeypatch.setattr(..., raising=False)`, `monkeypatch.delattr(..., raising=False)`, `mock.patch(..., create=True)` and `mock.patch.object(..., create=True)` aimed at a name that does not exist. The patch runs, patches nothing, and the real code stays wired, sometimes making the very network call the test meant to stub. |
| `missing-import` | all | `from yourpkg.mod import name` where `mod` no longer defines `name` (or the module is gone). Most useful for imports inside functions, which fail only when that function runs. Imports wrapped in `try/except ImportError` are left alone. |
| `call-arity` | all | A call to a function imported from your own package with more positional arguments than it takes, or a keyword it does not accept. A certain `TypeError` whenever the line runs. |
| `shadowed-import` | all | A name used in a function above that function's own `import` of it. The local import makes the name local to the whole function, so the earlier line raises `UnboundLocalError`, even when the module imports the same name at the top. Undefined-name linters do not report it. |
| `unwired-guard` | source | A function whose name says it validates or refuses something (`validate_*`, `verify_*`, `sanitize_*`, `guard_*`, ...) that only the tests call. It passes its own tests and protects nothing. |
| `unbounded-wait` | tests | `threading.Barrier(n)` with no timeout, `t.join()`, `event.wait()`, `proc.wait()`, `queue.get()`, `future.result()` with no timeout. A test stuck on one prints nothing until CI kills the job. |
| `await-under-lock` | all | `await` (or `async for` / `async with`) inside a plain `with` block holding a thread or file lock. It stalls the event loop when the lock is busy, and does not keep two coroutines apart. `async with` on an asyncio lock is correct and never flagged. |
| `success-from-except` | source | `return {"ok": True}` (or `success=True`, `{"status": "ok"}`, ...) from inside an `except` block, so no caller or test can see the failure. A response that also names what went wrong (`skipped`, `error`, `warning`, `reason`, ...) is honest and is not flagged. |

"tests" means files matching `test_files` or inside a `test_dirs` directory; "source" means
everything else. Run `falsify check --list-rules` for the list.

#### `falsify dead-asserts`

Runs your suite under coverage.py and reports every assertion in a test file whose line never
executed: a check that cannot fail because it never runs. Assertions are `assert` statements
and unittest-style calls on `self`: `self.assertEqual(...)`, `with self.assertRaises(...)`,
`self.fail(...)`.

```
tests/test_notes.py:12: in test_the_note_names_the_setting(): assert 'retry_limit' in note['text']
    (4/6 lines of test_the_note_names_the_setting ran)
```

It is careful not to cry wolf. It does not report assertions in tests that were skipped (read
from pytest's own report, never guessed), in helpers reached only by skipped tests, in
modules skipped wholesale (`pytestmark` skip marks, `pytest.importorskip` at import), in
tests that did not pass (failed, errored, xfailed), or in `except` / `finally` blocks. It
does report assertions in test classes, in parametrized tests, and inside callbacks a test
defined but never triggered.

Extra pytest arguments go after `--`:

```bash
falsify dead-asserts -- -m "not slow" -p no:xdist
```

#### `falsify comment-guards`

Finds tests that read a source file and assert a string is in it, where the string exists
only in a comment:

```python
# app/settings.py
# RETRY_LIMIT = 5          <- the fix, commented out
RETRY = 3

# tests/test_settings.py
def test_retry_limit_is_set():
    assert "RETRY_LIMIT" in SETTINGS.read_text()      # still green
```

It works by execution: for each suspect it copies your project to a temporary directory,
runs the test, strips the comment lines from the sources the test reads, and runs it again.
A test that now fails was passing because of a comment. **Your working tree is never
modified.** It understands `.py`, `.toml`, `.yml`/`.yaml`, `.cfg`, `.ini`, `.sh`, `.js`/`.ts`
(and friends), `.html` and `.css` sources, read through `read_text`, `read_bytes`,
`open(...).read()` or `inspect.getsource`, whether the text is kept in a variable or read
right in the assertion, and asserted as a `str` or a `bytes` literal. Test methods in
classes are rerun through their class.

A suspect it could not judge (the rerun timed out, or pytest could not select the test) is
listed, and the command exits `2` rather than claim a clean result. A test that was already
failing before any change is listed too; your suite reports that one itself.

To write such tests safely, match against code only:

```python
from falsify.source_scan import code_only

src = code_only(Path("app/settings.py"))    # comments (and Python docstrings) removed
assert "RETRY_LIMIT" in src
```

For Python, everything but the comments and docstrings comes back exactly as written (same
quotes, same spacing, same line numbers), so `'os.environ["API_KEY"]' in src` matches when
the file says so. `code_only` refuses formats where it cannot tell a comment from content
(Markdown, JSON, plain text) rather than guessing.

### Excusing a finding

Some findings are correct code. A determinism test compares a call with itself on purpose;
a branch that is today's normal state has a sibling branch that exists as a tripwire. Say so
next to the code:

```python
assert build(seed=1) == build(seed=1)  # falsify: ok determinism check

# falsify: ok the fixture kills the child after 5 seconds
proc.wait()
```

The marker counts on any line of the flagged statement, or in the block of comment (and
decorator) lines directly above it. **The reason is required**: a bare `# falsify: ok`
excuses nothing, so the escape hatch cannot become free. `falsify dead-asserts` also accepts
`# dead-assert-ok: <reason>`. Excused findings are counted in the summary.

### Configuration

Everything is optional. Put it in your `pyproject.toml`:

```toml
[tool.falsify]
paths = ["src", "tests"]                 # what `falsify check` and `pytest --falsify` read
exclude = ["tests/data", "src/generated/*"]
test_files = ["test_*.py", "*_test.py", "conftest.py"]
test_dirs = ["tests", "test"]            # every .py file under these counts as test code
packages = ["myapp"]                     # your own top-level packages (default: detected)
import_roots = [".", "src"]              # where those packages live
lock_helpers = ["state_lock"]            # your own lock context managers
guard_patterns = ['^_?(?:validate|verify|guard|sanitize|check)_\w+$']
tests = "tests"                          # the test directory for dead-asserts / comment-guards

[tool.falsify.rules]
unwired-guard = false                    # switch a rule off
```

| Key | Default | Meaning |
|---|---|---|
| `paths` | `["."]` | Files and directories to check, relative to the project root. A path that does not exist is an error. |
| `exclude` | `[]` | Paths or glob patterns, relative to the project root, to leave out. Also honored by `dead-asserts` and `comment-guards`. Hidden directories (`.venv`, `.git`, ...), `venv`, `build`, `dist`, `node_modules`, `site-packages` and cache directories are always left out. |
| `test_files` | `["test_*.py", "*_test.py", "conftest.py"]` | File name patterns that mark test code. |
| `test_dirs` | `["tests", "test"]` | Directory names whose files are all test code. |
| `packages` | detected | Top-level packages that are yours, for import checks. Detected as the directories with an `__init__.py` under `import_roots`. |
| `import_roots` | `[".", "src"]` | Where your packages live. |
| `lock_helpers` | `[]` | Extra context-manager names that hold a blocking lock. Names like `lock`, `_lock`, `Lock`, `RLock`, `FileLock`, `locked_*` and `*_mutex` are recognized already. |
| `guard_patterns` | `validate`, `verify`, `guard`, `refuse`, `sanitize`, `forbid`, `lint`, `ratchet`, `enforce` prefixes | Regular expressions for guard function names. Setting it replaces the default. |
| `tests` | `"tests"` | Test directory used by `dead-asserts` and `comment-guards`. |
| `rules` | all on | A table of `rule-id = true/false`. |

An unknown key, a misspelled rule id, or a value of the wrong type is an error (exit code 2),
never silently ignored.

### Command reference

```
falsify check [paths...] [--json] [--rule ID]... [--disable ID]... [--no-hints] [--list-rules]
falsify dead-asserts [--tests DIR] [--json] [--max-report N] [--timeout S] [-- pytest args]
falsify comment-guards [--tests DIR] [--json] [--timeout S]
common: --root DIR (default: nearest directory with a pyproject.toml), --config FILE
```

Paths given to `check` are relative to the current directory, as your shell completes them,
so `falsify check test_orders.py` works from inside `tests/`. They narrow what is reported,
not what is read: the whole configured project is still read, so a guard called from another
file is still seen as called.

**Exit codes**, for every command: `0` clean, `1` findings, `2` could not check (bad
configuration, a path that does not exist, nothing to check, a file that does not parse, a
suite that did not run, or a suspect that could not be judged). A scan that examined nothing
exits `2`, not `0`: a clean result over zero files would mean nothing. A file that does not
parse (a syntax error, or syntax newer than the Python running falsify) is named and also
exits `2`, and fails `pytest --falsify`; if it is broken on purpose, add it to `exclude`.

`--json` prints a machine-readable report, for CI annotations or dashboards.

#### In CI

```yaml
- run: pip install "./falsify[coverage]"
- run: pytest --falsify
- run: falsify dead-asserts
```

### Honest limits

- **Static rules see what is written, not what runs.** A patch target built at runtime
  (`setattr(obj, name_var, ...)`), a guard called through `getattr(obj, "validate_" + x)`,
  or a lock acquired through a variable of an unrecognized name is invisible to them.
- **Undecidable is never a finding.** A module with a module-level `__getattr__`, writes
  through `globals()`, or a star import from outside your project cannot be resolved from
  its text, so names in it are not reported. That costs some misses and buys no false alarms.
- **`phantom-patch` imports standard-library and installed modules** to resolve targets
  outside your project, the same way the test itself would. Your own modules are only read.
  A module found inside your project that is not under `import_roots` (say, `lib/` added to
  pytest's `pythonpath`) is never imported, so its targets are left undecided; add its
  directory to `import_roots` to have them checked. Packages installed in a virtual
  environment inside the project (`site-packages`) are treated as installed.
- **A name defined in more than one way is not judged.** A function defined differently in
  the branches of an `if`, or rebound after its `def`, has no single signature to check
  calls against. Class attributes set anywhere in your project (`cls.x = ...` in a
  classmethod, `Client.retries = 3` in another module) count as existing.
- **`unwired-guard` is lenient on purpose.** Any reference outside the tests counts as
  wired, including an import that is never called. Methods of classes that inherit from
  something are skipped, because a framework may call an override by name.
- **`unbounded-wait` recognizes common receivers**: variables assigned from `Thread`,
  `Event`, `Condition`, `Popen`, `Queue` and `executor.submit()`, or named like them
  (`t`, `worker`, `done`, `ready`, `q`, `queue`, `future`). A wait on an object it cannot
  identify is not reported.
- **`call-arity` checks only undecorated functions imported by name** from your own
  packages, and only certain errors (too many positional arguments, an unknown keyword).
- **`dead-asserts` needs a normal, single-process run.** Under pytest-xdist or pytest-cov
  pass `-- -p no:xdist -p no:cov`; if coverage measured no test file it exits `2` rather than
  report a clean result. Coverage is collected for test files only.
- **`comment-guards` finds the sources a test reads from the string literals in it** (and
  `inspect.getsource` targets). A path assembled at runtime from variables is not found. Its
  rerun strips whole-line comments only, not trailing comments or Python docstrings. It
  copies the project to a temporary directory, which takes a moment on a large tree, and it
  reruns only the suspect tests.
- **`dead-asserts` recognizes unittest assertions called on `self`.** An assertion helper
  from another library (for example a custom `expect(...)`) is not recognized as an
  assertion.
- **`code_only` for JavaScript and CSS** skips `/*` inside quoted strings. A regular
  expression literal that contains `/*` or a quote character can still confuse it.
- Findings in code you did not write (vendored libraries) are best handled with `exclude`.

### Using it as a library

```python
from falsify import load_config, run_check

result = run_check(load_config("path/to/project"))
for f in result.findings:
    print(f.path, f.line, f.rule, f.message)
```

`falsify.dead_asserts.run(...)` and `falsify.comment_guards.run(...)` return report objects
with the same information the commands print.

---

Docs mirror of the private repository Dominares-Tools/falsify; updated with each release.
