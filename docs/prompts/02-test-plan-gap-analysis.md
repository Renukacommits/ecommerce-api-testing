# Prompt 02 — Test Plan Gap Analysis

**Tool:** Claude / ChatGPT  
**Date:** Throughout April–May 2026  
**Stage:** Test Planning (used repeatedly across modules)

---

## Context

After independently designing test scenarios for each module, AI was used to challenge the plan — not to generate it. The goal was to surface gaps that might have been missed, then evaluate each gap critically before deciding whether to act on it.

## Prompt Pattern Used

> "I am a QA engineer testing the [module] endpoints of the DummyJSON API. Here are the test scenarios I have designed: [list of scenarios]. What edge cases, boundary conditions, or negative scenarios might I have missed? Do not generate test code — only identify gaps in coverage."

## AI Output (Example: Carts Module)

Prompt fed to Claude with the Get All Carts, Get Single Cart, and Get By User scenario list.

**Gaps Claude identified:**
- skip parameter behaviour when skip exceeds total records → validated and added (self-identified defect: API returns 200 with empty array and skip=209, an impossible position)
- Combination of limit + skip parameters → evaluated, added as a scenario
- Response time assertion → considered, added as a baseline assertion

**Gaps Claude missed:**
- The skip > total defect was not flagged by Claude — it focused on limit behaviour only. Identified independently during testing.

## Evaluation

**Partially accepted.** Claude's gap analysis improved coverage in boundary and combination scenarios. However, Claude's suggestions were not complete — the most significant defect in the Carts module (skip > total returning 200 with impossible skip value) was self-identified, not AI-identified.

**Key learning:** AI gap analysis is useful for prompting structured thinking, but does not replace exploratory testing and independent investigation.

## Outcome

Gap analysis used as a checklist input, not as a definitive coverage list. All AI-suggested gaps were independently validated against actual API behaviour before adding test cases.
