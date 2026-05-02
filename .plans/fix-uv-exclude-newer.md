# Investigation: `exclude-newer = "7 days"` in `pyproject.toml`

## TL;DR — Not a bug

Initial hypothesis was wrong. `exclude-newer = "7 days"` is **valid uv syntax**
in uv 0.11.0+. The TOML parse warning we observed was caused by the sandbox
having an old uv (0.8.17) pre-installed, not by anything wrong in the repo.
**No upstream PR is warranted.** The `fix` commit was reverted.

## What we initially observed

Running `setup-hermes.sh` produced:

```
warning: Failed to parse `pyproject.toml` during settings discovery:
  TOML parse error at line 176, column 17
      |
  176 | exclude-newer = "7 days"
      |                 ^^^^^^^^
  failed to parse year in date "7 days": failed to parse "7 da" as year
  (a four digit integer): invalid digit, expected 0-9 but got ' '
```

…and `uv sync --locked` failed (output suppressed by `2>/dev/null`), so the
script fell back to `uv pip install -e ".[all]"`.

## Initial (wrong) diagnosis

I assumed `"7 days"` was a placeholder mistakenly committed, that uv only
ever accepted ISO 8601 timestamps for `exclude-newer`, and that this was
causing both the parse warning and the lockfile failure. I removed the line.

## Why that was wrong

### Verification step 1 — uv changelog
Searching the upstream uv `CHANGELOG.md` revealed:

- **0.11.8**: "Use a sentinel timestamp for relative `exclude-newer` and
  `exclude-newer-package` values in lockfiles"
- **0.11.4**: "Recompute relative `exclude-newer` values during
  `uv tree --outdated`"

→ Relative values for `exclude-newer` are a real, supported feature.

### Verification step 2 — direct test against uv 0.11.8
Installed `uv==0.11.8` to `/tmp/uvnew` and ran a minimal `pyproject.toml`:

```toml
[tool.uv]
exclude-newer = "7 days"
```

Result: `uv lock` ran cleanly with `EXIT=0`, no warning.

For comparison, a deliberately-invalid value:
```toml
[tool.uv]
exclude-newer = "garbage"
```

Yielded the authoritative error message — which itself documents the accepted
formats:

> `garbage` could not be parsed as a valid exclude-newer value (expected a
> date like `2024-01-01`, a timestamp like `2024-01-01T00:00:00Z`, or
> **a duration like `3 days` or `P3D`**)

So `"7 days"` matches the documented "duration" form.

### Verification step 3 — does the line actually affect anything on old uv?
Compared `uv tree` output with the line present (HEAD~3) vs. removed (HEAD)
on uv 0.8.17:

- Both: identical resolver error (`yc-bench` requires Python >= 3.12)
- Both: `EXIT=1`
- The only behavioral difference of the line on old uv is the parse warning
  itself; resolution is unaffected.

### Verification step 4 — what was the real cause of the lockfile failure?
The hidden `uv sync --locked` error (revealed by re-running without `2>/dev/null`):

> Because only `yc-bench{python_full_version >= '3.12'}==0.1.0` is available
> and the current Python version (3.11.15) does not satisfy Python>=3.12 …

This is unrelated to `exclude-newer`. It is a real but separate issue with
the `yc-bench` extra in the lockfile being incompatible with the Python 3.11
target that `setup-hermes.sh` provisions.

## Actual root cause of what we saw

The sandbox had **uv 0.8.17** pre-installed. `setup-hermes.sh` reuses any
existing uv on `PATH` (or in `~/.local/bin` / `~/.cargo/bin`) without
checking a minimum version. uv 0.8.17 predates relative-duration support,
so it warned on the (otherwise-valid) `"7 days"` value.

The `uv sync --locked` failure was a coincident, separate problem with the
`yc-bench` extra requiring Python >= 3.12.

## Criteria I should have applied before claiming "bug"

For a config-syntax claim to be "buggy in upstream" I should have required
all of:

1. The syntax is rejected on the **latest released** version of the tool,
   not just whatever happens to be installed locally.
2. Upstream documentation explicitly disallows the syntax.
3. The syntax change is the **sole** cause of the downstream symptom
   (verified by isolated reproduction).
4. Git history shows the line was added by mistake or never functioned.

In this case 1, 2, and 3 all failed verification.

## What (if anything) is worth a PR

Possibly: a small improvement to `setup-hermes.sh` to require a minimum uv
version (e.g. `>= 0.11.0`) so users with stale uv don't get confusing
parse warnings about features their uv predates. That's a different and
much smaller change than what I originally proposed and would need its own
investigation.

The `yc-bench` Python-version mismatch in the lockfile is also a separate
real issue worth filing, independent of any of the above.

## Status

- The `fix` commit (`d85a4ec`) has been reverted.
- This document is kept as a record of the misdiagnosis and the verification
  process that caught it.
