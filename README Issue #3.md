# Contribution 3: fix: prompt_cache_min_tokens inconsistencies for claude-fable-5 and others
Contribution Number: 3
Student: Felix Mathew
Issue: https://github.com/BerriAI/litellm/issues/35011
Branch: https://github.com/mathew-felix/litellm/tree/fix-issue-35011
Status: In Review

## Why I Chose This Issue
I chose this issue because it's a great opportunity to understand how LiteLLM manages its model pricing and context window registry. Prompt caching is a massive cost-saving feature for LLM gateways, and fixing inconsistent configuration data guarantees that the caching logic triggers correctly for all supported models. It builds on my familiarity with the LiteLLM codebase from my previous contribution, while letting me tackle a focused, high-impact data consistency bug.

This issue is meaningful because the `prompt_cache_min_tokens` field is used at runtime by router logic (e.g., `is_prompt_caching_valid_prompt`) to decide whether to route a request to a caching-enabled deployment. Wrong or missing values cause caching to silently fail — users pay full price for tokens that should have been cached.

## Understanding the Issue

### Problem Description
`model_prices_and_context_window.json` is the single source of truth for all model capability metadata in LiteLLM. The file has two categories of bugs related to the `prompt_cache_min_tokens` field:

1. **Value inconsistency across claude-fable-5 family keys**: The base entry `"claude-fable-5"` sets `prompt_cache_min_tokens: 512`, but the Bedrock-routed variants (`anthropic.claude-fable-5`, `us.anthropic.claude-fable-5`, `eu.anthropic.claude-fable-5`, `global.anthropic.claude-fable-5`) all set `prompt_cache_min_tokens: 1024`. The `azure_ai/claude-fable-5`, `vertex_ai/claude-fable-5`, and `vertex_ai/claude-fable-5@default` keys are missing the field entirely.

2. **Missing field on 29 derived Anthropic entries**: Across the file, there are 29 specific Anthropic model variants (like `azure_ai/claude-haiku-4-5` or `openrouter/anthropic/claude-opus-4.5`) that declare `"supports_prompt_caching": true` but are missing `prompt_cache_min_tokens`. Because they are Anthropic models, their minimums are well-known and already exist under different keys in the same file.

### Expected Behavior
- All `claude-fable-5` variant keys should share the same `prompt_cache_min_tokens: 512` value.
- The 29 targeted Anthropic variants should have their `prompt_cache_min_tokens` filled in using the exact values from their base counterparts (e.g., Haiku = 4096, Sonnet/Opus = 1024/2048/4096 depending on the specific version).

### Current Behavior
- `claude-fable-5` → `512`
- `anthropic.claude-fable-5` → `1024` (disagrees)
- `azure_ai/claude-fable-5` → **MISSING**
- `vertex_ai/claude-fable-5` → **MISSING**
- `vertex_ai/claude-fable-5@default` → **MISSING**
- 26 other Anthropic variants → **MISSING**

Downstream, this causes `get_prompt_cache_min_tokens()` to fall back to the hardcoded default (`1024`) for missing entries, which is incorrect for models like Haiku 4.5 (which should be 4096) or Fable-5 (which should be 512).

### Affected Components
- `model_prices_and_context_window.json` (claude-fable-5 keys and the 29 targeted missing keys)
- `litellm/utils.py`
- `tests/test_litellm/test_utils.py`

---

## Reproduction Process

### Environment Setup
No special environment setup was needed beyond having the repository cloned and Python 3 available. 

- **Repo**: `git clone git@github.com:mathew-felix/litellm.git`
- **Branch**: `git checkout -b fix-issue-35011 upstream/main`
- **Python**: 3.11

### Steps to Reproduce
1. Clone the repository and check out `upstream/main`.
2. Run the reproduction script: `python3 reproduce_35011.py`
3. Observe Part 1 output: 5 of 8 `claude-fable-5` keys have inconsistent or missing `prompt_cache_min_tokens`.
4. Observe Part 2 output: The 29 targeted Anthropic entries are missing their `prompt_cache_min_tokens` values.

### Reproduction Evidence
- Reproduction script commit: https://github.com/mathew-felix/litellm/commit/f5479aad0f (updated in subsequent commit to target the 29 keys)
- Branch (buggy code, no fix): https://github.com/mathew-felix/litellm/tree/fix-issue-35011

### My Findings
- The base `claude-fable-5` correctly has `512` (Anthropic's documented minimum). The Bedrock variants mistakenly have `1024`.
- The 29 missing entries are all clearly derived from Anthropic base models, meaning we can backfill their values exactly without guessing. (Guessing is dangerous because setting it too high disables caching entirely for prefixes that would have worked).

---

## Solution Approach

### Analysis
The Fable-5 mismatch occurred because the Bedrock variants were likely copy-pasted from older Claude models where Bedrock required a 1024 minimum. The 29 missing entries were added without explicitly defining the minimum, leaving them to fall back to the 1024 default, which is wrong for models like Haiku (4096).

### Proposed Solution
1. Unify all 8 `claude-fable-5` keys to `prompt_cache_min_tokens: 512`.
2. Backfill the 29 missing Anthropic keys with their exact known minimums (provided by the reviewer).

### Implementation Plan (UMPIRE Framework)

**Understand:**
We need to unify Fable-5 keys to `512`, and backfill 29 specific Anthropic variant keys. We should *not* backfill the other ~470 non-Anthropic models.

**Match:**
The pattern for this fix exists within the JSON file itself (other keys for the same models have the correct values) and in `test_utils.py`.

**Plan:**
1. **Unify claude-fable-5 family**: Set `prompt_cache_min_tokens: 512` on all 8 `claude-fable-5` keys.
2. **Backfill 29 targeted entries**: Update `model_prices_and_context_window.json` for the 29 keys with the exact values provided by the reviewer (e.g., `azure_ai/claude-haiku-4-5` to 4096).
3. **Update tests**: Update `test_get_prompt_cache_min_tokens_differs_per_platform_for_same_model` to assert `512` for all Fable-5 keys.
4. **Integrate reviewer test**: Add the reviewer's test that asserts every key for a given model agrees on its minimum.

**Implement:** *(Phase III Complete)*
- Branch: https://github.com/mathew-felix/litellm/tree/fix-issue-35011
- Commits will be linked here as work progresses.
- [Key Commit: fix(utils): unify claude-fable-5 min tokens and backfill 29 missing Anthropic caches](https://github.com/mathew-felix/litellm/commit/c1a4f75c41)

**Review:**
- [x] All `claude-fable-5` keys agree on `prompt_cache_min_tokens: 512`
- [x] All 29 previously missing Anthropic entries now have correct, provider-documented values
- [x] `reproduce_35011.py` exits with code 0 (no bugs detected)
- [x] `make lint` passes (Black, Ruff, basedpyright)
- [x] `make test-unit` passes with no regressions
- [ ] PR targets the latest `litellm_oss_daily_YYYY_MM_DD` branch, not `main`
- [ ] CLA already signed from previous contribution

**Evaluate:**
Run `python3 reproduce_35011.py` after the fix. Expected output: no `[BUG]` lines, exit code 0. Additionally run `uv run pytest tests/test_litellm/test_utils.py -v -k "prompt_cache_min_tokens"` to confirm all relevant tests pass.

---

## Testing Strategy

### Unit Tests
- **Test case 1**: `test_get_prompt_cache_min_tokens_differs_per_platform_for_same_model` — updated assertion so all `claude-fable-5` keys assert `512` instead of the current mixed values.

### Manual Testing
- Ran `python3 reproduce_35011.py` before and after the fix to confirm the script moved from exit code 1 → 0.

---

## Implementation Notes

### Week 1 Progress

**What I built:**

Unified the `claude-fable-5` family on `512` and backfilled the 29 Anthropic variant keys Tanisha-Katara identified on the issue.

**Files modified:**
- `model_prices_and_context_window.json` — updated 4 Fable-5 Bedrock keys to 512, backfilled 29 missing Anthropic variant keys
- `litellm/model_prices_and_context_window_backup.json` — same modifications, kept in sync
- `tests/test_litellm/test_utils.py` — updated `test_get_prompt_cache_min_tokens_differs_per_platform_for_same_model` to assert `512` for all Fable-5 keys

**Key commit:** [`c1a4f75c41`](https://github.com/mathew-felix/litellm/commit/c1a4f75c41) — fix(utils): unify claude-fable-5 min tokens and backfill 29 missing Anthropic caches

**Challenges faced:**
- `test_utils.py` was pulling from `litellm/model_prices_and_context_window_backup.json` rather than the root file during testing. Had to keep both files in sync by hand for every edit.
- Wrote a small Python script to inject the 29 missing Anthropic variants using the exact dictionary the reviewer posted, rather than editing a 46,000-line file by hand.

### Week 2 Progress

**What I built:**

Caught and fixed a scope drift in the backfill: a branch cleanup (hard reset + cherry-pick onto `upstream/litellm_internal_staging`, followed by an amend) had let 12 extra keys into the change that weren't part of Tanisha's approved 29 — `github_copilot/claude-haiku-4.5`, `github_copilot/claude-opus-4.5`, `github_copilot/claude-sonnet-4.5`, `gmi/anthropic/claude-opus-4.5`, `gmi/anthropic/claude-sonnet-4.5`, `perplexity/anthropic/claude-opus-4-6`, `perplexity/anthropic/claude-opus-4-7`, `perplexity/anthropic/claude-opus-4-5`, `perplexity/anthropic/claude-sonnet-4-5`, `perplexity/anthropic/claude-haiku-4-5`, `vertex_ai/claude-opus-4-1`, and `vertex_ai/claude-opus-4-1@20250805`.

I went back through the diff key by key against the exact 29-entry list Tanisha posted, confirmed the 12 extras weren't on it, and reverted just those, keeping the 29 approved keys and the 4 Fable-5 fixes untouched. Re-verified line by line afterward that the file matched her list exactly, with zero keys added beyond it.

**Files modified:**
- `model_prices_and_context_window.json` — removed `prompt_cache_min_tokens` from the 12 out-of-scope keys
- `litellm/model_prices_and_context_window_backup.json` — same removal, kept in sync

**Key commit:** [`5239e748dc`](https://github.com/mathew-felix/litellm/commit/5239e748dc) — fix(utils): narrow prompt_cache_min_tokens backfill to reviewer-approved 29 keys

**Challenges faced:**
- The extras were easy to miss on a surface read of the diff, since `git show`'s default 3-line context sometimes attributes a hunk to the wrong model key when the actual key name falls outside the visible context window. Confirmed the real additions by diffing the full JSON key-by-key against the branch's parent commit instead of trusting the unified diff text directly.
- Also ran the full `tests/test_litellm/test_*.py` glob (matching the `Unit Tests: MCP, Secrets, Containers & Misc` CI job that was failing in the PR) to make sure the revert didn't break anything else. Found 5 failing tests total, none related to this change: one flaky retry-count test and four `gpt-5.4` backup/main drift tests, both confirmed present on the commit before this fix too.

---

## Code Changes
- `model_prices_and_context_window.json`: Updated 4 Fable-5 variant keys to 512, backfilled the 29 approved Anthropic variant keys, then reverted 12 out-of-scope keys that had slipped in during a branch cleanup.
- `litellm/model_prices_and_context_window_backup.json`: Same modifications to ensure tests and offline mode operate correctly.
- `tests/test_litellm/test_utils.py`: Updated assertions in `test_get_prompt_cache_min_tokens_differs_per_platform_for_same_model` to verify all `claude-fable-5` variants return `512`.
- `tests/test_litellm/test_prompt_cache_min_tokens_aliases.py`: Added Tanisha-Katara's regression test file as provided, confirming every alias resolves to its documented minimum and agrees with its canonical model.

---

## Pull Request
PR Link: https://github.com/BerriAI/litellm/pull/35378

PR Description (updated after the Week 2 follow-up, to fold in Tanisha's point about why this bug matters):
```markdown
## TLDR

Problem this solves:

Anthropic models served through reseller aliases had wrong or missing prompt cache minimums. The four Bedrock `claude-fable-5` keys (`anthropic.`, `us.`, `eu.`, `global.`) were set to `1024` while the base `claude-fable-5` entry correctly has `512`. Separately, 29 Anthropic model variants across azure_ai, openrouter, snowflake, vercel_ai_gateway and vertex_ai declare `supports_prompt_caching: true` but carry no `prompt_cache_min_tokens` at all, so `get_prompt_cache_min_tokens()` silently falls back to the wrong default for each of them.

How it solves it:

Sets all 8 `claude-fable-5` keys to `512`, Anthropic's documented minimum. Fills in the 29 missing entries using values that already exist in the file under each model's base key, so nothing here is guessed.

## Why this isn't cosmetic

Both directions of a wrong minimum fail silently, which is basically why 29 of these sat unset for this long without anyone catching it. Set the minimum too high and `is_prompt_caching_valid_prompt` says no to a prompt that would have cached just fine, so you skip caching that would have worked. Set it too low, or just leave it unset so it falls back to the default, and the function says yes, the caching marker goes out, and Anthropic quietly processes the request without caching it at all. No error comes back. You just get `cache_creation_input_tokens: 0` and a bill that looks completely normal. There's nothing in the response that tells you caching didn't happen.

## Relevant issues

Fixes #35011

## Type

🐛 Bug Fix

## Changes

Two commits here. The first (`c1a4f75c41`) fixes the 4 Bedrock fable-5 keys and backfills the 29 approved Anthropic variants, plus updates `test_utils.py` to assert `claude-fable-5` resolves to `512` everywhere. The second (`5239e748dc`) walks back 12 extra keys that had slipped into the first commit outside the approved list during a branch cleanup, and adds the reviewer's own regression test file (`tests/test_litellm/test_prompt_cache_min_tokens_aliases.py`) so this specific class of bug can't come back unnoticed.
```

Maintainer Feedback:
- Pre-PR feedback from contributor/maintainer Tanisha-Katara was incorporated into the scope and plan, including her exact list of 29 keys and her regression test file.
- 2026-08-03: Caught a scope drift on my own before requesting another review — a branch cleanup had let 12 out-of-scope keys into the backfill. Pushed a follow-up commit narrowing the change back to exactly Tanisha's 29 keys.
- 2026-08-03 (later): Tanisha independently re-verified the follow-up commit (`5239e748dc`) rather than taking the commit message at face value — confirmed all 29 keys carry her exact values, all 8 `claude-fable-5` keys read `512`, and all 12 out-of-scope keys (github_copilot, gmi, perplexity, both vertex_ai/claude-opus-4-1 entries) are unset again. She also replayed her test's assertions directly against the JSON herself. She flagged that the silent, bidirectional nature of this failure mode (wrong-high skips valid caching, wrong-low/missing ships a caching marker that Anthropic silently ignores) is the actual reason this fix matters, and asked that it be visible to a reviewer rather than left implicit. Updated the PR description with a "Why this isn't cosmetic" section covering it, and replied on the issue thanking her for the re-check.
- Awaiting a maintainer merge.

Status: In Review

---

## Learnings & Reflections

### Technical Skills Gained
- **Model Registry Architecture**: Deepened understanding of how LiteLLM maintains capability metadata and token pricing in `model_prices_and_context_window.json`.
- **Package Dist & Test Environment Dual-Mapping**: Discovered that LiteLLM unit tests rely on `litellm/model_prices_and_context_window_backup.json` inside python site-packages, requiring parallel maintenance of backup configuration files.
- **Precision Data Validation**: Learned techniques for programmatically auditing and backfilling large (46k+ line) JSON registries without introducing syntax errors or unintended mutations.

### Challenges Overcome
- **Scoping & Avoiding Destructive Guesses**: Initial analysis targeted all 499 missing entries, but reviewer feedback pointed out that setting guessed minimums for non-Anthropic models breaks prompt caching. Successfully pivoted to a narrow, high-confidence 29-entry Anthropic backfill.
- **Pytest Backup File Discrepancy**: Unit tests initially failed after modifying root JSON because `get_model_cost_map()` loaded `litellm/model_prices_and_context_window_backup.json`. Identified the root cause and synchronized edits across both files.
- **Scope Drift During Branch Cleanup**: A hard reset and cherry-pick onto `upstream/litellm_internal_staging`, followed by an amend, quietly let 12 keys outside the reviewer's approved list into the change. Caught it by diffing the actual JSON contents key-by-key against the branch's parent commit rather than trusting the unified diff's context lines, and reverted just those 12 in a separate follow-up commit.

### What I'd Do Differently Next Time
- Check whether package distribution files or bundled backups exist alongside root configuration files before running tests.
- Seek reviewer alignment on broad data cleanup issues earlier in the contribution process.
- After any hard reset / cherry-pick / amend on a branch, re-diff the actual data (not just the unified diff text) against the reviewer's original approved list before pushing, since branch surgery is exactly where scope creep sneaks in unnoticed.
- Put the "why this matters" explanation in the PR description from the start, not just in test docstrings. Tanisha had to point out that the silent, bidirectional failure mode is what makes this fix non-cosmetic; that belongs somewhere a reviewer sees it before reading a single line of the diff.

### Resources Used
- https://github.com/BerriAI/litellm/issues/35011
- https://github.com/BerriAI/litellm/pull/35378
- https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html
- `litellm/utils.py` — `get_prompt_cache_min_tokens()` function
- `tests/test_litellm/test_utils.py` — existing caching tests