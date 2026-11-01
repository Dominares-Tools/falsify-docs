# Changelog

## 1.0.3

Five fixes from a run of falsify over public projects: four rules that reported correct code,
and a scan that failed over one file it could not read. Each has a regression test.

- `phantom-patch` no longer reports a name that `raising=False` or `create=True` creates on
  purpose: a standard-library name (`sys.real_prefix`, `sys.gettotalrefcount` and
  `builtins.__IPYTHON__` differ by platform and Python version, so creating one simulates an
  environment) and a name your own module uses without defining, or looks up with
  `getattr` / `hasattr` (a snippet that calls `DBSession()`, a class that reads
  `self.timeout`). A patch aimed at a renamed or misspelled name in your own code or an
  installed package is still reported.
- `missing-import` leaves alone a module below one that registers modules at run time.
  `requests.packages.urllib3` exists because `requests.packages` writes it into `sys.modules`,
  and a package can serve modules from an import hook it installs. falsify still never imports
  your code, so it reads the modules above the missing one: if one writes to `sys.modules` or
  touches `sys.meta_path` / `sys.path_hooks`, in itself or in a module its `__init__.py`
  imports, it may provide any name below it, and the import is left undecided.
- `unfalsifiable-assert` reports `assert x == x` only when both sides are the same name,
  attribute or literal. `assert Point(1, 2) == Point(1, 2)` and
  `assert hash(key(1)) == hash(key(1))` build an object on each side, so they test `__eq__` and
  `__hash__` and can fail. `assertEqual` and `assertIs` follow the same rule. Most of the 65
  hits in the run were this shape.
- `unwired-guard` no longer reports `@nox.session` functions (nox calls them) or the public
  methods of a class your package exports, listed in an `__all__` or imported by name in a
  package `__init__.py` (the people who use the package call those). A private method, and a
  method of a class you do not export, are still reported.
- A file the running Python cannot parse (a syntax error, or newer syntax such as Python 3.14's
  `t"..."` strings under 3.12) is skipped with a notice instead of making the whole scan exit
  `2`. It is counted as skipped in the summary and as `skipped` in the `--json` report. A scan
  in which no file could be checked still exits `2`, and `pytest --falsify` follows the same
  rule.

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show falsify`, `falsify.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated (Dominares Tools Commercial License, version 1.3).
- One public contact address, support@dominares.org, in the licence and the README.

## 1.0.0

First release.

- `falsify check`: nine static rules. `unfalsifiable-assert`, `phantom-patch`,
  `missing-import`, `call-arity`, `shadowed-import`, `unwired-guard`, `unbounded-wait`,
  `await-under-lock` and `success-from-except`. Text or `--json` output; exit codes 0 / 1 / 2;
  a scan that checked no files, or left a file it could not parse, exits 2. Paths on the
  command line are relative to the current directory.
- `pytest --falsify`: the static rules as part of a pytest session. Findings are printed in
  their own section and fail the session even when every test passed.
- `falsify dead-asserts`: runs the suite under coverage.py and reports assertions that
  never executed: `assert` statements and unittest-style `self.assert*()` / `self.fail()`
  calls. Skip-aware from pytest's own report (per-test skips, helpers reached only
  by skipped tests, module-level skip marks, `importorskip`), judges only tests that passed,
  and excuses `except` / `finally` bodies. Test methods in classes and parametrized tests are
  keyed by their own names.
- `falsify comment-guards`: reruns source-reading tests with the comments stripped from the
  sources they read, in a temporary copy of the project, and names each test that then fails.
  Understands inline reads (`"X" in path.read_text()`), bytes literals and test methods in
  classes. A suspect it could not judge (a timeout, a test pytest could not select) makes the
  result exit 2, not 0.
- `falsify.source_scan.code_only()` / `py_code_only()`: comment strippers for your own
  source-reading tests, refusing formats they cannot strip correctly. Python text comes back
  exactly as written, minus comments and docstrings.
- `[tool.falsify]` configuration with strict validation: unknown keys, unknown rule ids and
  wrong types are errors.
- `# falsify: ok <reason>` excuses a finding; the reason is required.
- The static rules never import your project's code, including packages outside
  `import_roots`; only standard-library and installed packages are imported, to resolve patch
  targets.
