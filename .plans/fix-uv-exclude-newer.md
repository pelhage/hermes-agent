# Bug Fix: Invalid `exclude-newer` Value in `[tool.uv]`

## Summary

`pyproject.toml` sets `exclude-newer = "7 days"` under `[tool.uv]`. uv's
`exclude-newer` field accepts only ISO 8601 timestamps (e.g.
`"2024-03-01T00:00:00Z"`), not relative durations. The value `"7 days"`
fails to parse and causes uv to emit a warning on every invocation.

## What this fix does and does NOT do

**Does fix:** the TOML parse warning emitted by uv on every command:
```
warning: Failed to parse `pyproject.toml` during settings discovery:
  TOML parse error at line 176, column 17
      |
  176 | exclude-newer = "7 days"
      |                 ^^^^^^^^
  failed to parse year in date "7 days": failed to parse "7 da" as year
  (a four digit integer): invalid digit, expected 0-9 but got ' '
```

**Does NOT fix:** the `uv sync --locked` failure seen in `setup-hermes.sh`.
That failure has a separate root cause (see below).

## Root Cause (parse warning)

`[tool.uv] exclude-newer` is documented as requiring an RFC 3339 / ISO 8601
timestamp string. Relative durations are not supported. See upstream docs:
https://docs.astral.sh/uv/reference/settings/#exclude-newer

The value `"7 days"` appears to be a placeholder that was never replaced
with an actual date before being committed.

## Side Effect of This Fix

Removing `exclude-newer` changes the effective timestamp cutoff from the
value uv stored in the lockfile (`0001-01-01T00:00:00Z`, the epoch fallback
uv used when the original value failed to parse) to "no cutoff". This causes
uv to print:

```
Ignoring existing lockfile due to removal of timestamp cutoff: `global: 0001-01-01T00:00:00Z`
```

and re-resolve from scratch. This is expected and harmless — the lockfile was
already generated without a meaningful cutoff — but worth knowing.

## Separate Issue: `uv sync --locked` Failure During Setup

The `uv sync --locked` failure in `setup-hermes.sh` is unrelated to
`exclude-newer`. The actual error (hidden by `2>/dev/null` in the script) is:

```
× No solution found when resolving dependencies for split
  (markers: python_full_version >= '3.12'):
  ╰─▶ Because only yc-bench{python_full_version >= '3.12'}==0.1.0 is
      available and the current Python version (3.11.15) does not satisfy
      Python>=3.12, we can conclude that all versions of
      yc-bench{python_full_version >= '3.12'} cannot be used.
```

The lockfile contains a `yc-bench` dependency that requires Python >= 3.12.
The setup script targets Python 3.11, causing an unsatisfiable resolution.
This is a separate bug that requires its own fix (e.g. constraining the
lockfile to supported Python versions, or removing `yc-bench` from the
extras if it is not needed).

## Files Changed

- `pyproject.toml` — remove `exclude-newer = "7 days"` from `[tool.uv]`

---

## Draft PR Description (for NousResearch/hermes-agent)

### What does this PR do?

Removes an invalid `exclude-newer = "7 days"` value from `[tool.uv]` in
`pyproject.toml`. uv requires an ISO 8601 timestamp for this field; relative
durations like `"7 days"` are not supported. The malformed value causes uv to
emit a TOML parse warning on every invocation.

Note: the `uv sync --locked` failure visible in `setup-hermes.sh` is a
separate issue caused by a `yc-bench` extra in the lockfile that requires
Python >= 3.12 — it is not addressed by this PR.

### Related Issue

N/A (no existing issue — discovered during initial dev setup via
`setup-hermes.sh`).

### Type of Change

- [x] 🐛 Bug fix (non-breaking change that fixes an issue)

### Changes Made

- `pyproject.toml`: removed `exclude-newer = "7 days"` from `[tool.uv]`
  (line 176). This eliminates the recurring TOML parse warning.

### Known Side Effect

Removing `exclude-newer` changes uv's stored timestamp cutoff, which causes
uv to print `Ignoring existing lockfile due to removal of timestamp cutoff`
and re-resolve on the next `uv sync`. This is expected — the prior value was
epoch (`0001-01-01T00:00:00Z`) because the original string was unparseable —
and does not change dependency resolution.

### How to Test

1. Clone the repo
2. Run any uv command, e.g. `uv venv venv --python 3.11`
3. Confirm no TOML parse warning appears

### Checklist

#### Code
- [x] Commit message follows Conventional Commits (`fix: ...`)
- [x] PR contains only changes related to this fix
- [ ] `pytest tests/ -q` — to be run before submitting
- [x] Tested on: Linux (Ubuntu 24.04), uv 0.8.17

#### Documentation & Housekeeping
- [x] No documentation changes needed — one-line config fix
- [x] N/A for `cli-config.yaml.example`, `CONTRIBUTING.md`, `AGENTS.md`
- [x] Cross-platform: uv is cross-platform; the parse failure occurs on all platforms
