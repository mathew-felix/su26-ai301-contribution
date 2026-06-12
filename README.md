# Contribution 1: fix: split auth/timeout/connection error classes in signals_agent_registry

**Contribution Number:** 1  
**Student:** Felix Mathew  
**Issue:** https://github.com/prebid/salesagent/issues/1433  
**Status:** Phase I Complete

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
