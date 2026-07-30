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

2. **Missing field on 499 other entries**: Across the entire file, 499 model entries declare `"supports_prompt_caching": true` but do not define `prompt_cache_min_tokens` at all. This includes Amazon Nova models, many Bedrock Anthropic entries, and Azure/Codex models.

### Expected Behavior
- All `claude-fable-5` variant keys should share the same `prompt_cache_min_tokens` value.
- Every entry with `"supports_prompt_caching": true` should explicitly define `prompt_cache_min_tokens`.

### Current Behavior
- `claude-fable-5` → `512`
- `anthropic.claude-fable-5` → `1024` (disagrees)
- `azure_ai/claude-fable-5` → **MISSING**
- `vertex_ai/claude-fable-5` → **MISSING**
- `vertex_ai/claude-fable-5@default` → **MISSING**
- 499 other entries with `supports_prompt_caching: true` → **MISSING**

Downstream, this causes the `get_prompt_cache_min_tokens()` utility (in `litellm/utils.py`) to fall back to the hardcoded default (`DEFAULT_MINIMUM_PROMPT_CACHE_TOKEN_COUNT = 1024`) for all missing entries, which is incorrect for models like Nova that use lower or higher thresholds.

### Affected Components
- `model_prices_and_context_window.json` (line 1361, 1397, 1433, 1469, 2728, 11811, 36647, 36677, and 499 more scattered entries)
- `litellm/utils.py` — contains `get_prompt_cache_min_tokens()` which reads this file
- `tests/test_litellm/test_utils.py` — line 4820: `test_get_prompt_cache_min_tokens_differs_per_platform_for_same_model` currently asserts `claude-fable-5 == 512` and `anthropic.claude-fable-5 == 1024` (will need updating after fix)

---

## Reproduction Process

### Environment Setup
No special environment setup was needed beyond having the repository cloned and Python 3 available. The reproduction is purely a JSON audit — no LLM API keys or running server required.

- **Repo**: `git clone git@github.com:mathew-felix/litellm.git`
- **Branch**: `git checkout -b fix-issue-35011 upstream/main`
- **Python**: 3.11 (system Python, no virtualenv needed for the reproduction script)

No errors were encountered during setup.

### Steps to Reproduce
1. Clone the repository and check out `upstream/main`.
2. Run the reproduction script at the root of the repository:
   ```bash
   python3 reproduce_35011.py
   ```
3. Observe Part 1 output: 5 of 8 `claude-fable-5` keys have inconsistent or missing `prompt_cache_min_tokens`.
4. Observe Part 2 output: 499 entries declare `supports_prompt_caching: true` with no `prompt_cache_min_tokens` defined.

### Reproduction Evidence
- Reproduction script commit: https://github.com/mathew-felix/litellm/commit/f5479aad0f
- Branch (buggy code, no fix): https://github.com/mathew-felix/litellm/tree/fix-issue-35011

### Observed Output
```
=================================================================
PART 1: claude-fable-5 prompt_cache_min_tokens values
=================================================================
  [OK]    'claude-fable-5'                                    supports_prompt_caching=True  prompt_cache_min_tokens=512
  [OK]    'anthropic.claude-fable-5'                          supports_prompt_caching=True  prompt_cache_min_tokens=1024
  [OK]    'global.anthropic.claude-fable-5'                   supports_prompt_caching=True  prompt_cache_min_tokens=1024
  [OK]    'us.anthropic.claude-fable-5'                       supports_prompt_caching=True  prompt_cache_min_tokens=1024
  [OK]    'eu.anthropic.claude-fable-5'                       supports_prompt_caching=True  prompt_cache_min_tokens=1024
  [BUG]   'azure_ai/claude-fable-5'                           supports_prompt_caching=True  prompt_cache_min_tokens=MISSING
  [BUG]   'vertex_ai/claude-fable-5'                          supports_prompt_caching=True  prompt_cache_min_tokens=MISSING
  [BUG]   'vertex_ai/claude-fable-5@default'                  supports_prompt_caching=True  prompt_cache_min_tokens=MISSING

  BUG CONFIRMED: 2 distinct values found across claude-fable-5 keys: [512, 1024]

=================================================================
PART 2: Entries with supports_prompt_caching=true but no prompt_cache_min_tokens
=================================================================
  Total entries with supports_prompt_caching=true but missing prompt_cache_min_tokens: 499

  BUG CONFIRMED: 499 entries claim caching support but have no minimum token count set.
```

### My Findings
- The direct entry `"claude-fable-5"` was set to `512` — consistent with Anthropic's documented minimum for the fable-5 family.
- The Bedrock-specific variants (`anthropic.*`) were set to `1024`, which appears to be carried over from older Claude 3 entries where Bedrock required a higher minimum.
- Azure AI and Vertex AI variants were never given the field at all when they were added to the registry.
- The 499 missing entries span Amazon Nova, many Bedrock-routed Anthropic entries, Azure Codex models, and others — these were likely added without a defined Anthropic-style minimum, so they were left blank.

---

## Solution Approach

### Analysis
The root cause is a lack of enforcement: there is no validation step that ensures newly added caching-capable models also specify their minimum token count. Each provider has different documented minimums — there is no universal default that works correctly for all of them.

### Proposed Solution
Two targeted changes to `model_prices_and_context_window.json`:
1. Unify all `claude-fable-5` keys to `prompt_cache_min_tokens: 512` (matching the authoritative base entry and Anthropic's published docs).
2. Backfill the 499 missing entries with the correct per-provider minimum, sourced from each provider's official caching documentation.

### Implementation Plan (UMPIRE Framework)

**Understand:**
`model_prices_and_context_window.json` has two classes of bugs: value disagreement within the `claude-fable-5` family, and completely missing `prompt_cache_min_tokens` on 499 entries that declare `supports_prompt_caching: true`. The `get_prompt_cache_min_tokens()` function in `litellm/utils.py` reads this field at runtime, so incorrect or missing values cause silent caching failures.

**Match:**
The pattern for this type of fix already exists in the repository:
- The `claude-opus-4-5`, `claude-sonnet-4-5` entries in the Bedrock section serve as reference: they correctly set provider-specific minimums (4096, 1024 respectively).
- The existing test `test_get_prompt_cache_min_tokens_differs_per_platform_for_same_model` in `tests/test_litellm/test_utils.py` tests this exact function and will be the test to update/extend.

**Plan:**
1. **Unify claude-fable-5 family**: Set `prompt_cache_min_tokens: 512` on all 8 `claude-fable-5` keys in `model_prices_and_context_window.json` (lines 1395, 1431, 1467, 1503, 2756, 36675, and the `@default` variant). The value `512` is correct because Anthropic's documentation specifies 512 for this model family.
2. **Backfill 499 missing entries**: Group the 499 missing entries by provider prefix and apply the correct minimum per provider:
   - Amazon Nova models → `1024` (AWS documented minimum)
   - Bedrock-routed Anthropic models → `1024` (Bedrock platform minimum)
   - Azure AI / Vertex AI Anthropic models → `512` (same as direct Anthropic access)
   - Other providers → research and apply documented minimums
3. **Update affected tests**: Update `test_get_prompt_cache_min_tokens_differs_per_platform_for_same_model` to assert that all `claude-fable-5` variants return `512`.
4. **Run `make lint` and `make test-unit`** to verify no regressions.

**Implement:** *(placeholder — Phase III)*
- Branch: https://github.com/mathew-felix/litellm/tree/fix-issue-35011
- Commits will be linked here as work progresses.

**Review:**
- [ ] All `claude-fable-5` keys agree on `prompt_cache_min_tokens: 512`
- [ ] All 499 previously missing entries now have correct, provider-documented values
- [ ] `reproduce_35011.py` exits with code 0 (no bugs detected)
- [ ] `make lint` passes (Black, Ruff, basedpyright)
- [ ] `make test-unit` passes with no regressions
- [ ] PR targets the latest `litellm_oss_daily_YYYY_MM_DD` branch, not `main`
- [ ] CLA already signed from previous contribution

**Evaluate:**
Run `python3 reproduce_35011.py` after the fix. Expected output: no `[BUG]` lines, exit code 0. Additionally run `uv run pytest tests/test_litellm/test_utils.py -v -k "prompt_cache_min_tokens"` to confirm all relevant tests pass.

---

## Testing Strategy

### Unit Tests
- **Test case 1**: `test_get_prompt_cache_min_tokens_differs_per_platform_for_same_model` — update assertion so all `claude-fable-5` keys assert `512` instead of the current mixed values.
- **Test case 2**: New parametrized test asserting `get_prompt_cache_min_tokens("azure_ai/claude-fable-5") == 512`.
- **Test case 3**: New parametrized test asserting `get_prompt_cache_min_tokens("vertex_ai/claude-fable-5") == 512`.

### Manual Testing
- Run `python3 reproduce_35011.py` before and after the fix to confirm the script moves from exit code 1 → 0.

---

## Implementation Notes
*(To be filled in during Phase III)*

---

## Code Changes
*(To be filled in during Phase III)*

---

## Pull Request
PR Link: *(To be filled in during Phase IV)*

PR Description: *(To be drafted during Phase IV)*

Maintainer Feedback:
*(To be filled in after PR submission)*

Status: Awaiting implementation

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