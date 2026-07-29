# Contribution 3: fix: prompt_cache_min_tokens inconsistencies for claude-fable-5 and others
Contribution Number: 3
Student: Felix Mathew
Issue: https://github.com/BerriAI/litellm/issues/35011
Status: Phase I In Progress

## Why I Chose This Issue
I chose this issue because it's a great opportunity to understand how LiteLLM manages its model pricing and context window registry. Prompt caching is a massive cost-saving feature for LLM gateways, and fixing inconsistent configuration data guarantees that the caching logic triggers correctly for all supported models. It builds on my familiarity with the LiteLLM codebase from my previous contribution, while letting me tackle a quick, high-impact data consistency bug.

## Understanding the Issue
### Problem Description
There are inconsistencies in `model_prices_and_context_window.json` related to `prompt_cache_min_tokens`. Specifically, four `claude-fable-5` keys disagree with the direct entry, and there are many entries that claim caching support but have no minimum recorded.

### Expected Behavior
Keys for the same model family (like `claude-fable-5`) should have consistent `prompt_cache_min_tokens` values. All entries that claim caching support should explicitly record their `prompt_cache_min_tokens` minimum.

### Current Behavior
Inconsistent values exist, leading to downstream caching features either failing to trigger or using incorrect minimums.

### Affected Components
- `model_prices_and_context_window.json`

Reproduction Process
Environment Setup
[Notes on setting up your local development environment - challenges you faced, how you solved them]

Steps to Reproduce
[Step 1]
[Step 2]
[Observed result]
Reproduction Evidence
Commit showing reproduction: [Link to commit in your fork]
Screenshots/logs: [If applicable]
My findings: [What you discovered during reproduction]
Solution Approach
Analysis
[Your analysis of the root cause - what's causing the issue?]

Proposed Solution
[High-level description of your fix approach]

Implementation Plan
Using UMPIRE framework (adapted):

Understand: [Restate the problem]

Match: [What similar patterns/solutions exist in the codebase?]

Plan: [Step-by-step implementation plan]

[Modify file X to do Y]
[Add function Z]
[Update tests]
Implement: [Link to your branch/commits as you work]

Review: [Self-review checklist - does it follow the project's contribution guidelines?]

Evaluate: [How will you verify it works?]

Testing Strategy
Unit Tests
 Test case 1: [Description]
 Test case 2: [Description]
 Test case 3: [Description]
Integration Tests
 Integration scenario 1
 Integration scenario 2
Manual Testing
[What you tested manually and results]

Implementation Notes
Week [X] Progress
[What you built this week, challenges faced, decisions made]

Week [Y] Progress
[Continue documenting as you work]

Code Changes
Files modified: [List]
Key commits: [Links to important commits]
Approach decisions: [Why you chose certain approaches]
Pull Request
PR Link: [GitHub PR URL when submitted]

PR Description: [Draft or final PR description - much of the content above can be adapted]

Maintainer Feedback:

[Date]: [Summary of feedback received]
[Date]: [How you addressed it]
Status: [Awaiting review / Iterating / Approved / Merged]

Learnings & Reflections
Technical Skills Gained
[What you learned technically]

Challenges Overcome
[What was hard and how you solved it]

What I'd Do Differently Next Time
[Reflection on your process]

Resources Used
[Link to helpful documentation]
[Tutorial or Stack Overflow post that helped]
[GitHub issues or discussions that helped]