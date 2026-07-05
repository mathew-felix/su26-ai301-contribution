# Contribution 1: fix: split auth/timeout/connection error classes in signals_agent_registry

**Contribution Number:** 1  
**Student:** Felix Mathew  
**Issue:** https://github.com/prebid/salesagent/issues/1433  
**Status:** Phase IV Complete

---

## Why I Chose This Issue

I chose issue #1433 from the prebid/salesagent repository because handling network resilience and SDK exceptions is a core competency for Backend and Cloud Engineers. Currently, the system flattens distinct network failures into a single generic error, which hinders observability. By decoupling these exception classes, I will gain hands-on experience improving backend error handling and system logging, which are critical for debugging production microservices.

Additionally, as a recent M.S. Computer Science graduate targeting backend infrastructure roles, working on a massive, enterprise-backed project like Prebid allows me to navigate production-grade architecture. This issue perfectly aligns with my goals to strengthen my Python API skills and understand how large distributed systems handle network failures and timeouts gracefully.

---

## Understanding the Issue

### Problem Description

In `signals_agent_registry.py` (lines 219-233), the code currently catches four distinct network and SDK exceptions (like Authentication errors, Timeouts, and Connection issues) but squashes them into one generic `AdCPAdapterError`. This makes it impossible for the system or developers to know the actual root cause of a network failure.

### Expected Behavior

The signals agent registry should maintain the specificity of the errors. It should raise distinct, specific error classes (e.g., `ADCPAuthenticationError`, `ADCPTimeoutError`, `ADCPConnectionError`) so the caller knows the exact nature of the failure, mirroring how errors are already handled in the `creative_registry`.

### Current Behavior

All four distinct SDK exceptions are flattened and re-raised as a single, generic `AdCPAdapterError`, wiping out the specific context of the network failure. 

### Affected Components

- `signals_agent_registry.py` (specifically the exception handling block around lines 219-233).
- Potentially the exception definition classes if new ones need to be imported or mirrored from the creative registry.

---

## Reproduction Process

### Environment Setup

The project uses `uv` for dependency management and requires Python 3.12+. The setup process was straightforward once I located `uv`:

1. Cloned the fork: `git clone git@github.com:mathew-felix/salesagent.git && cd salesagent`
2. Checked `.python-version` — project requires Python 3.12
3. Installed dependencies: `uv sync --group dev` (downloaded ~241 packages; first run took ~5 minutes)
4. No `.env` file was needed to run unit tests — only integration/e2e tests require Docker and a running Postgres instance

**Challenge:** `uv` was not on the system PATH by default on macOS. Fixed by using the full path `~/Library/Python/3.9/bin/uv run ...` for all commands, or prefixing: `PATH="$HOME/Library/Python/3.9/bin:$PATH" make quality`.

**No Docker was needed to reproduce this issue** — the bug lives entirely in the exception-handling logic and is reproducible with unit tests and a small Python script.

### Steps to Reproduce

The bug is in `src/core/signals_agent_registry.py` lines 219–233. When the AdCP library raises `ADCPAuthenticationError` (invalid credentials to a signals agent), the code catches it but re-raises it as the wrong type.

**Method A — Run the existing unit test (confirms the wrong type is currently expected):**

```bash
PATH="$HOME/Library/Python/3.9/bin:$PATH" uv run pytest \
  tests/unit/test_signals_agent_registry.py::TestSignalsAgentRegistry::test_get_signals_from_agent_handles_auth_error \
  -v
```

The test currently asserts `pytest.raises(AdCPAdapterError)` and passes — this *is* the bug. The test was written to match the broken behavior.

**Method B — Run a direct reproduction script to observe the wrong exception type:**

```bash
PATH="$HOME/Library/Python/3.9/bin:$PATH" uv run python - <<'EOF'
import asyncio
from unittest.mock import AsyncMock, Mock
from adcp.exceptions import ADCPAuthenticationError as LibADCPAuthError
from src.core.exceptions import AdCPAdapterError, AdCPAuthenticationError
from src.core.signals_agent_registry import SignalsAgent, SignalsAgentRegistry

async def reproduce():
    registry = SignalsAgentRegistry()
    agent = SignalsAgent(
        agent_url="https://example.com/mcp",
        name="Test Agent",
        auth={"type": "bearer", "credentials": "bad-token"},
        auth_header="Authorization",
    )
    mock_client = Mock()
    mock_agent_client = Mock()
    mock_agent_client.get_signals = AsyncMock(
        side_effect=LibADCPAuthError(
            "Invalid credentials", agent_id="test", agent_uri="https://example.com"
        )
    )
    mock_client.agent = Mock(return_value=mock_agent_client)

    try:
        await registry._get_signals_from_agent(
            mock_client, agent, brief="test", tenant_id="tenant-1"
        )
    except Exception as e:
        print(f"Exception type raised : {type(e).__name__}")
        print(f"  status_code          : {e.status_code}")
        print(f"  recovery             : {e.recovery}")
        print()
        print("Expected: AdCPAuthenticationError (status=401, recovery=terminal)")
        print("Actual  : AdCPAdapterError        (status=502, recovery=transient)")

asyncio.run(reproduce())
EOF
```

**Expected output:**
```
Exception type raised : AdCPAdapterError
  status_code          : 502
  recovery             : transient

Expected: AdCPAuthenticationError (status=401, recovery=terminal)
Actual  : AdCPAdapterError        (status=502, recovery=transient)
```

**What this proves:** An authentication failure (bad credentials) is being reported to the caller as a 502 Service Unavailable with `recovery=transient`, telling the system to retry. The correct behavior is a 401 with `recovery=terminal`, telling the caller to fix their credentials rather than retry.

The same bug exists for `ADCPTimeoutError` and `ADCPConnectionError` — both get squashed into `AdCPAdapterError(502)` instead of `AdCPServiceUnavailableError(503)`.

### Reproduction Evidence

- **Working branch:** https://github.com/mathew-felix/salesagent/tree/fix-issue-1433
- **Bug location:** [`src/core/signals_agent_registry.py` lines 219–233](https://github.com/mathew-felix/salesagent/blob/fix-issue-1433/src/core/signals_agent_registry.py#L219-L233)
- **My findings:** The four `except` blocks all re-raise to `AdCPAdapterError`, but each should map to a specific type. The correct mapping already exists in `creative_agent_registry.py` (lines 493–504) and just needs to be applied here.

---

## Solution Approach

### Analysis

**Root cause:** In `src/core/signals_agent_registry.py`, the import on line 37 only brings in `AdCPAdapterError` from `src.core.exceptions`. The four `except` blocks on lines 219–233 all re-raise every ADCP library exception as this one generic type, regardless of the actual failure:

```python
# Current (broken) — all four blocks look like this:
except ADCPAuthenticationError as e:
    raise AdCPAdapterError(f"Authentication failed: {e.message}") from e  # ← wrong type
```

This erases the semantic difference between:
- `ADCPAuthenticationError` → should be `AdCPAuthenticationError` (HTTP 401, `recovery=terminal` — caller must fix credentials, never retry)
- `ADCPTimeoutError` → should be `AdCPServiceUnavailableError` (HTTP 503, `recovery=transient` — safe to retry)
- `ADCPConnectionError` → should be `AdCPServiceUnavailableError` (HTTP 503, `recovery=transient` — safe to retry)
- `ADCPError` (catch-all) → keep as `AdCPAdapterError` (HTTP 502)

### Proposed Solution

Apply the same exception mapping that already exists in `creative_agent_registry.py` (lines 493–504) to `signals_agent_registry.py`. The fix is a 2-file change: update the imports and the four `except` clauses.

### Implementation Plan

**Understand:** The signals agent registry catches all ADCP exceptions and re-raises them as `AdCPAdapterError(502, transient)`. Authentication failures should instead raise `AdCPAuthenticationError(401, terminal)` and network failures should raise `AdCPServiceUnavailableError(503, transient)`. The wrong recovery hint causes callers to retry requests that will never succeed (bad credentials).

**Match:** `src/core/creative_agent_registry.py` lines 493–504 already implements the correct 4-way exception mapping. `AdCPAuthenticationError` and `AdCPServiceUnavailableError` are already defined in `src/core/exceptions.py` — no new classes needed.

**Plan:**
1. In `src/core/signals_agent_registry.py` line 37: expand the import to add `AdCPAuthenticationError` and `AdCPServiceUnavailableError`
2. In `src/core/signals_agent_registry.py` lines 219–233: replace all four `raise AdCPAdapterError(...)` with the correct type per exception:
   - `ADCPAuthenticationError` → `raise AdCPAuthenticationError(...)`
   - `ADCPTimeoutError` → `raise AdCPServiceUnavailableError(...)`
   - `ADCPConnectionError` → `raise AdCPServiceUnavailableError(...)`
   - `ADCPError` (catch-all) → keep `raise AdCPAdapterError(str(e.message))` (matches creative registry)
3. In `tests/unit/test_signals_agent_registry.py`: update the existing auth error test to expect `AdCPAuthenticationError` instead of `AdCPAdapterError`, and add two new tests for the timeout and connection error mappings

**Implement:** https://github.com/mathew-felix/salesagent/tree/fix-issue-1433 — commit [`2947c77`](https://github.com/mathew-felix/salesagent/commit/2947c77e)

**Review:** Checked against `CONTRIBUTING.md` — used Conventional Commits format (`fix: ...`). Ran `make quality` (ruff formatting, ruff lint, mypy typecheck, unit tests, duplication baseline — all passed, exit code 0). PR title will follow the same `fix:` prefix.

**Evaluate:**
- Ran `pytest tests/unit/test_signals_agent_registry.py -v` — 9/9 passed including the updated auth test and two new tests
- Ran `make quality` — all checks passed (0 new duplicate blocks, 0 mypy errors, 0 lint violations)
- Ran the reproduction script — now correctly raises `AdCPAuthenticationError(401, terminal)` instead of `AdCPAdapterError(502, transient)`

---

## Implementation Notes

### Week 1 Progress

**What I built:**

Updated `src/core/signals_agent_registry.py` to properly map ADCP library exceptions to specific internal exception types, matching the established pattern in `creative_agent_registry.py`.

**Files modified:**
- `src/core/signals_agent_registry.py` — expanded import on line 37 to include `AdCPAuthenticationError` and `AdCPServiceUnavailableError`; updated the four `except` blocks (lines 219–233) to raise the correct exception type per failure
- `tests/unit/test_signals_agent_registry.py` — updated the existing auth error test to assert `AdCPAuthenticationError` (was `AdCPAdapterError`); added two new tests for `ADCPTimeoutError` → `AdCPServiceUnavailableError` and `ADCPConnectionError` → `AdCPServiceUnavailableError`

**Key commit:** [`2947c77`](https://github.com/mathew-felix/salesagent/commit/2947c77e) — fix: map ADCP exceptions to specific types in signals_agent_registry

**Challenges faced:**
- The project uses `uv` for dependency management which was not on the system PATH by default on macOS — worked around by prefixing `PATH="$HOME/Library/Python/3.9/bin:$PATH"` to all commands
- No Docker required for this fix — the bug and tests are entirely unit-testable without spinning up the full stack

**Approach decisions:**
- Chose a minimal 2-file change with no refactoring — the CLAUDE.md project guide explicitly warns against DRY-extracting shared helpers unless duplication increases (the baseline was already set with the creative registry having the same pattern)
- Kept log messages consistent with the existing wording to avoid noisy diffs

---

## Testing Strategy

**New tests added** (`tests/unit/test_signals_agent_registry.py`):
- `test_get_signals_from_agent_handles_auth_error` — updated to assert `AdCPAuthenticationError` (401, terminal) is raised when `ADCPAuthenticationError` comes from the library
- `test_get_signals_from_agent_handles_timeout_error` — new: asserts `AdCPServiceUnavailableError` (503) is raised on `ADCPTimeoutError`
- `test_get_signals_from_agent_handles_connection_error` — new: asserts `AdCPServiceUnavailableError` (503) is raised on `ADCPConnectionError`

**Results:** 9/9 unit tests passed. Full `make quality` passed (ruff, mypy, duplication check, unit suite).

---

## Pull Request

**PR Link:** https://github.com/prebid/salesagent/pull/1483

**PR Description:** Fixed exception mapping in `signals_agent_registry.py` so that `ADCPAuthenticationError`, `ADCPTimeoutError`, and `ADCPConnectionError` are re-raised as their correct internal types (`AdCPAuthenticationError`, `AdCPServiceUnavailableError`) instead of being flattened into the generic `AdCPAdapterError`. Matches the existing pattern in `creative_agent_registry.py`.

**Maintainer Feedback:**
- PR reviewed and approved by maintainers

**Status:** Merged ✅

---

## Learnings & Reflections

### Technical Skills Gained
- Learned how exception hierarchies carry semantic meaning (HTTP status codes, retry hints) in a real production Python codebase
- Gained hands-on experience navigating a large open source project with strict architecture guards (`make quality`, pre-commit hooks, AST-scanning tests)
- Understood the importance of matching existing patterns in a codebase rather than inventing new ones

### Challenges Overcome
- `uv` was not on the system PATH by default on macOS — resolved by locating the binary and prefixing commands
- No Docker was required for this fix, which simplified the reproduction and testing workflow significantly

### What I'd Do Differently Next Time
- Set up the full Docker stack earlier to be ready for integration tests in case they were needed
- Look for existing patterns in the codebase before planning the fix — the answer was already in `creative_agent_registry.py`

