# Journal — Issue #153

## Problem Summary

`FaithfulnessChecker.check()` crashes with a `TypeError` when a context chunk's `text` key is explicitly set to `None`, rather than missing entirely.

The bug is in this line:

```python
context_text = " ".join([
    chunk.get("text", "") for chunk in context_chunks
])
```

`dict.get(key, default)` only returns `default` when `key` is **absent** from the dictionary. If `key` exists but its value is `None`, `.get()` returns `None` — the default is never applied. When `context_chunks` contains a chunk like `{"text": None}`, this line collects `None` into the list being joined, and `" ".join([...])` raises:

TypeError: sequence item 0: expected str instance, NoneType found


This matches the reproduction steps in the issue exactly.

## Why This Matters

This is a realistic failure mode, not just a contrived edge case: a chunk with `text: None` could easily come from an upstream ingestion or retrieval step that returns a chunk record without content (e.g. a failed extraction, a placeholder, or a null field in a database row). The faithfulness checker should degrade gracefully in that case, not crash the whole review pipeline.

## Scope

- **File affected:** `rag/evaluator/faithfulness_checker.py`
- **Function affected:** `FaithfulnessChecker.check()`, specifically the `context_text` concatenation step
- **Tier:** 1 (single-file, single-line logic fix)
- **Existing test:** `test_none_context_chunk_text` in `tests/unit/test_faithfulness_checker.py` — already written, was failing before the fix, passes after

## Fix

Changed:

```python
chunk.get("text", "")
```

to:

```python
chunk.get("text") or ""
```

`chunk.get("text")` returns `None` whether the key is missing or explicitly `None`. `None or ""` normalizes both cases to an empty string, so the `join()` call always receives strings.

## Verification

Ran the full test file before and after the fix using `git stash` / `git stash pop` to isolate the effect of the change:

**Without the fix (stashed):**
- `test_none_context_chunk_text` → **FAILED** with the exact `TypeError` described in the issue
- 3 other tests failed for unrelated reasons (see below)

**With the fix (applied):**
- `test_none_context_chunk_text` → **PASSED**
- `test_missing_text_key_in_chunk` (a related but different case — missing key entirely) → **PASSED**, confirming no regression on the pre-existing "missing key" handling
- Same 3 unrelated tests still failed, identically, in both runs

## Out-of-Scope Findings

While testing, I found 3 pre-existing test failures unrelated to this issue:

- `test_partial_support_returns_middle_score`
- `test_multiple_context_chunks`
- `test_multiple_claims_varying_support`

All three fail because `_is_supported()` returns `False` even when there's clear keyword overlap between claim and context — this points to a bug in the stop-word filtering or the "≥2 meaningful tokens" threshold, not in anything touched by my fix. I confirmed this by stashing my change and re-running the tests: all 3 failed identically with and without my fix, proving they're independent of the `None`-handling bug in #153.

I'm leaving these out of this PR since they're outside the scope of #153 and would need their own investigation into the scoring/overlap logic.

## Branch / Commit

- Branch: `fix/153-faithfulness-none-context-text`
- Commit: `df03eb8` — `fix(rag): handle None context chunk text in faithfulness checker`