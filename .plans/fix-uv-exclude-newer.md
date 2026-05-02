# Bug Fix: Invalid `exclude-newer` Value in `[tool.uv]`

## Summary

`pyproject.toml` sets `exclude-newer = "7 days"` under `[tool.uv]`. uv's
`exclude-newer` field accepts only ISO 8601 timestamps (e.g.
`"2024-03-01T00:00:00Z"`), not relative durations. The value `"7 days"`
fails to parse and causes uv to emit a warning and skip the setting entirely,
which in turn makes `uv sync --locked` fail and fall back to an unchecked
`uv pip install`.

## Reproduction

```
$ uv venv venv --python 3.11
warning: Failed to parse `pyproject.toml` during settings discovery:
  TOML parse error at line 176, column 17
      |
  176 | exclude-newer = "7 days"
      |                 ^^^^^^^^
  failed to parse year in date "7 days": failed to parse "7 da" as year
  (a four digit integer): invalid digit, expected 0-9 but got ' '
```

Every `uv` invocation (not just `sync`) that reads `pyproject.toml` emits
this warning. Additionally, `setup-hermes.sh` catches the resulting non-zero
exit from `uv sync --locked` and falls back to `uv pip install -e ".[all]"`,
which skips hash verification entirely — undermining the security benefit the
lockfile was meant to provide.

## Root Cause

`[tool.uv] exclude-newer` is documented as requiring an RFC 3339 / ISO 8601
timestamp string. Relative durations are not a supported syntax in any
released version of uv. See upstream docs:
https://docs.astral.sh/uv/reference/settings/#exclude-newer

The value `"7 days"` appears to be a placeholder or draft value that was
never replaced with an actual date before being committed.

## Fix

Remove the invalid `exclude-newer` line from `[tool.uv]`.

```diff
 [tool.uv]
-exclude-newer = "7 days"
```

**Why not replace with a date?** `exclude-newer` is most useful as a
reproducibility pin during development — freezing the resolution to a known
point in time. A hardcoded date in the repo would go stale quickly and
confuse contributors installing weeks or months later. If the intent is to
prevent very-recently-released packages from being pulled in, that concern is
already addressed by the `uv.lock` lockfile: `uv sync --locked` installs
exactly what the lockfile specifies, with hash verification, regardless of
what is newest on PyPI.

## Impact

| Before fix | After fix |
|---|---|
| uv emits TOML parse warning on every invocation | No warning |
| `uv sync --locked` exits non-zero | `uv sync --locked` succeeds |
| `setup-hermes.sh` silently falls back to unchecked pip install | Lockfile-verified install used as intended |

## Files Changed

- `pyproject.toml` — remove `exclude-newer = "7 days"` from `[tool.uv]`

---

## Draft PR Description (for NousResearch/hermes-agent)

### What does this PR do?

Removes an invalid `exclude-newer = "7 days"` value from `[tool.uv]` in
`pyproject.toml`. uv requires an ISO 8601 timestamp for this field, not a
relative duration string. The malformed value causes uv to emit a parse
warning on every invocation and — more importantly — causes `uv sync --locked`
to fail, so `setup-hermes.sh` silently falls back to an unchecked
`uv pip install`, bypassing the hash-verified lockfile install that was
intended.

### Related Issue

N/A (no existing issue — this was discovered during initial dev setup via
`setup-hermes.sh`).

### Type of Change

- [x] 🐛 Bug fix (non-breaking change that fixes an issue)

### Changes Made

- `pyproject.toml`: removed `exclude-newer = "7 days"` from `[tool.uv]`
  (line 176). The lockfile (`uv.lock`) already provides reproducible,
  hash-verified installs, so the setting is not needed.

### How to Test

1. Clone the repo and run `./setup-hermes.sh`
2. Confirm no uv TOML parse warnings appear in the output
3. Confirm the output shows `Dependencies installed (lockfile verified)`
   rather than `Lockfile install failed … falling back to pip install`

Alternatively, run directly:
```bash
uv sync --all-extras --locked
# Should exit 0 with no parse warnings
```

### Checklist

#### Code
- [x] Commit message follows Conventional Commits (`fix: ...`)
- [x] PR contains only changes related to this fix
- [x] `pytest tests/ -q` — run before submitting
- [x] Tested on: Linux (Ubuntu 24.04)

#### Documentation & Housekeeping
- [x] No documentation changes needed — this is a one-line config fix
- [x] N/A for `cli-config.yaml.example`, `CONTRIBUTING.md`, `AGENTS.md`
- [x] Cross-platform impact: fix is beneficial on all platforms (uv is
      cross-platform and the parse failure occurs everywhere)
