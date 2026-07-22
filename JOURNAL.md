# Journal — Issue #153

## Issue Selected

**Issue #153:** Faithfulness checker crashes when a context chunk has `text: None`
**Link:** https://github.com/ascherj/pathreview/issues/153
**Tier:** 1 (Starter — good for first-time contributors)

## Why This Issue Fits Me

I'm new to navigating a codebase this large, so I deliberately looked for a Tier 1 issue rather than a Tier 2 or 3 one — there's no extra credit for picking something harder, and I wanted my first contribution to be something I could fully understand end-to-end rather than something I'd have to partially guess at.

This issue fit that goal well: the bug report already identified the exact root cause (a `.get()` default-value gotcha) and gave exact reproduction steps and a named failing test. That meant I could focus my effort on verifying and understanding the fix deeply, rather than spending most of my time just locating the problem. It's scoped to a single function in a single file (`rag/evaluator/faithfulness_checker.py`), which matched the "single file/config" description of Tier 1 issues.

## "Is This Right For Me?" Checklist Reasoning

**Understanding the issue:** I can explain this without re-reading it: `dict.get(key, default)` only applies the default when the key is missing, not when it's present with value `None`. Before the fix, passing a context chunk like `{"text": None}` crashes the whole faithfulness check with a `TypeError`. After the fix, the same input degrades gracefully and returns a valid float score instead.

**Tier fit:** This is my first open source contribution, so Tier 1 was the right call rather than reaching for Tier 2 or 3 to "challenge myself." The issue lives entirely in one function in one file, matching the Tier 1 description exactly.

**Codebase readiness:** I located and read the exact function (`FaithfulnessChecker.check()`), not just the file, and could reason through the fix (`chunk.get("text") or ""`) before writing any code. I also read the full test file (`tests/unit/test_faithfulness_checker.py`) end-to-end, including the specific failing test (`test_none_context_chunk_text`) referenced in the issue.

**Scope and time:** The issue had 41 existing comments, but claims are non-exclusive and my grade comes from my own artifacts, so I didn't let that discourage me from picking it. I estimated this at the lower end of the 3–6 hour Tier 1 range for the core fix itself, though my actual time this week was higher due to unrelated local environment setup issues (BIOS/virtualization and two Docker service bugs), which I've documented separately. No blockers or dependencies were noted on the issue.

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