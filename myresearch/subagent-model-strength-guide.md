# Subagent Model Strength Guide

This document explains the recommended model strength tiers for each Superpowers subagent and the reasoning behind each recommendation.

> **For detailed configuration examples**, see [`opencode.config.example.json`](../opencode.config.example.json).

---

## Model Strength Tiers

We define four model strength tiers, ordered from most to least capable:

| Tier | Description | Example Models |
| ------ | ------------- | ---------------- |
| **Highest** | Maximum reasoning, judgment, and code understanding. Reserved for the most critical review tasks. | `claude-opus-4-8`, `gpt-5.6-sol` |
| **High** | Strong analytical and coding ability. Suitable for complex implementation and high-stakes review. | `qwen3.8-max-preview`, `deepseek-v4-pro` |
| **Standard** | Solid general-purpose capability. Handles well-specified tasks and document review reliably. | `mimo-v2.5-pro`, `qwen3.7-plus`, `claude-sonnet-4-6`, `gpt-5.4` |
| **Economical** | Cost-effective for narrow, well-scoped tasks with minimal ambiguity. | `MiniMax-M2.7`, `qwen3.6-flash`, `glm-5.2` |

---

## Recommendations by Subagent

### `sp-sub-implementer` — **Standard**

Implements a single task from a Superpowers plan using TDD. Reads the task brief, writes a failing test, implements the minimal code, runs tests, and commits.

**Why Standard:** When a task brief contains complete specifications and code structure, implementation is largely transcription plus testing. A standard model handles this reliably. For tasks requiring architectural judgment or broad codebase understanding, consider upgrading to High on a per-task basis.

---

### `sp-sub-task-reviewer` — **>= Implementer**

Reviews one task implementation for spec compliance and code quality. Produces two independent verdicts: Spec Compliance and Code Quality.

**Why >= Implementer:** Review requires equal or greater judgment than implementation. A reviewer must verify claims against the diff, detect subtle spec deviations, and assess code quality — all of which demand at least the same reasoning capacity as the implementer. Using a weaker model for review than for implementation risks missing issues the implementer introduced. As a rule, the reviewer should never be weaker than the implementer it is reviewing.

---

### `sp-sub-re-reviewer` — **< Task Reviewer**

Re-reviews a fix round: verifies each finding from the previous review was addressed and checks the fix diff for new breakage.

**Why < Task Reviewer:** Re-review is a narrow, scoped task. The re-reviewer only checks whether specific, already-identified findings were resolved and whether the fix diff introduced new problems. It does not perform a full re-review of untouched code. This bounded scope allows a cheaper model to succeed. However, it should not be the weakest tier — some judgment is still required to assess whether a fix truly addresses a finding.

---

### `sp-sub-final-reviewer` — **Highest**

Whole-branch code review after all tasks are complete. Evaluates plan alignment, code quality, architecture, testing, and production readiness across the entire change.

**Why Highest (mandatory):** This is the final gate before merge. The final reviewer evaluates the entire branch across six dimensions — including cross-task consistency, architectural soundness, and production readiness — that no other subagent checks. A missed issue here reaches production. This role must use the most capable available model, not the session default. Do not downgrade this one.

---

### `sp-sub-fixer` — **>= Implementer**

Fixes Critical and Important findings from a task review. Makes precise edits to address specific findings without introducing new issues.

**Why >= Implementer:** Fixing requires understanding the original intent, the reviewer's criticism, and the codebase well enough to make targeted changes. If the original implementer struggled (e.g., the task required more reasoning than the implementer's model could provide), the fixer should be upgraded to a stronger model. At minimum, it must match the implementer's strength.

---

### `sp-sub-spec-reviewer` — **Standard**

Reviews a spec document for completeness, consistency, clarity, scope, and YAGNI compliance. Read-only.

**Why Standard:** Spec review is primarily textual analysis — checking for contradictions, ambiguities, placeholders, and scope creep. It does not involve code reasoning. A standard model has sufficient judgment to identify structural problems in a spec document without needing the strongest tier.

---

### `sp-sub-plan-reviewer` — **Standard**

Reviews an implementation plan for completeness, spec alignment, task decomposition, and buildability. Read-only.

**Why Standard:** Like spec review, plan review is document-level analysis. The reviewer checks whether the plan covers the spec, whether task boundaries are clear, and whether an engineer could follow it without getting stuck. This requires solid reasoning about task decomposition but does not involve code, making standard models sufficient.

---

## Quick Reference

| Subagent | Recommended Strength | Can Be Downgraded? |
| ---------- | --------------------- | -------------------- |
| `sp-sub-implementer` | Standard | Yes, to Economical for trivial transcription tasks |
| `sp-sub-task-reviewer` | >= Implementer | No weaker than the implementer it reviews |
| `sp-sub-re-reviewer` | < Task Reviewer | Yes, but not below Economical |
| `sp-sub-final-reviewer` | **Highest** | **Never** |
| `sp-sub-fixer` | >= Implementer | No weaker than the implementer it fixes |
| `sp-sub-spec-reviewer` | Standard | Yes, for simple specs |
| `sp-sub-plan-reviewer` | Standard | Yes, for simple plans |

---

## Core Principles

1. **Review must be >= implementation.** A weaker reviewer cannot reliably catch issues a stronger implementer introduced.

2. **Scope determines cost.** Narrow, well-bounded tasks (re-review, spec review) can use cheaper models. Broad, high-stakes tasks (final review, task review) demand stronger models.

3. **Turn count beats token price.** Cheaper models often take 2-3x more turns on multi-step work, costing more overall. Use at least Standard as the floor for implementers working from prose descriptions.

4. **Always specify the model explicitly.** An omitted model inherits the session default — often the most expensive — which silently defeats the purpose of tiered dispatch.

5. **The final review is sacred.** It is the last line of defense before code reaches production. Always dispatch it on the most capable available model.
