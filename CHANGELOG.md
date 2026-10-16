# Changelog

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
