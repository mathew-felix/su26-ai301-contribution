# Contribution 3: fix: prompt_cache_min_tokens inconsistencies for claude-fable-5 and others
Contribution Number: 3
Student: Felix Mathew
Issue: https://github.com/BerriAI/litellm/issues/35011
Branch: https://github.com/mathew-felix/litellm/tree/fix-issue-35011
Status: Phase II Complete

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
- **JSON Editing Challenge**: A small but significant challenge encountered was `test_utils.py` falling back to `litellm/model_prices_and_context_window_backup.json` when testing. I had to ensure that both the main root `model_prices_and_context_window.json` and the backup version in the package directory were updated identically to make the tests pass.
- I wrote a small Python script to surgically inject the 29 missing Anthropic variants using the exact JSON dictionary the reviewer provided on the issue. This avoided doing it manually over a 46,000 line file, minimizing the risk of a syntax error.

---

## Code Changes
- `model_prices_and_context_window.json`: Updated 4 Fable-5 variant keys to 512, and backfilled 29 missing Anthropic variant keys.
- `litellm/model_prices_and_context_window_backup.json`: Same modifications to ensure tests and offline mode operate correctly.
- `tests/test_litellm/test_utils.py`: Updated assertions in `test_get_prompt_cache_min_tokens_differs_per_platform_for_same_model` to verify all `claude-fable-5` variants return `512`.

---

## Pull Request
PR Link: *(To be drafted during Phase IV)*

PR Description: *(To be drafted during Phase IV)*

Maintainer Feedback:
*(To be filled in after PR submission)*

Status: Ready for Phase IV (PR Creation)

---

## Learnings & Reflections
*(To be filled in after completion)*

### Technical Skills Gained
*(To be filled in)*

### Challenges Overcome
*(To be filled in)*

### What I'd Do Differently Next Time
*(To be filled in)*

### Resources Used
- https://github.com/BerriAI/litellm/issues/35011
- https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html
- `litellm/utils.py` — `get_prompt_cache_min_tokens()` function
- `tests/test_litellm/test_utils.py` — existing caching tests (lines 4811–4834)