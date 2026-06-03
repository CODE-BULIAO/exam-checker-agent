# 05 · Routing & "Agent or Workflow"

> [Home](../README.en.md) ｜ [中文](05-路由与Agent还是工作流.md) ｜ Prev: [04 Error Categories](04-错误类别与分类.en.md)

This chapter answers a question interviewers will probe: **"You call it an agent, but it looks more like a workflow — where exactly does the LLM make autonomous decisions?"** — answered honestly.

## The conclusion first

It is a **deterministic multi-stage pipeline** with **two constrained LLM decision points** embedded in it:

1. **Routing**: decide per question "which detections to run".
2. **Error aggregation**: decide `keep / merge / delete` for a question's multiple reports.

The LLM **does not drive control-flow branching, does not self-replan, and does not dynamically decide what to do next**. Strictly speaking it's closer to an "LLM-in-the-loop workflow" than an "autonomous agent". Below, the two decision points, then why it's still called an agent and how it could evolve.

## Decision point one: per-question routing

### What it does

Given a question's structured info, the LLM outputs a set of boolean switches deciding which detections to run downstream, plus skip reasons.

**Input (rewritten example):**

```json
{
  "language": "zh",
  "question_type": "single choice",
  "question_text": "(first 200 chars of stem, possibly truncated)",
  "has_options": true,
  "subject": "Physics",
  "section_title": "II. Multiple choice",
  "reference": "(first 200 chars of reference)"
}
```

**Output (rewritten example):**

```json
{
  "typo_check": true,
  "ambiguity_check": true,
  "options_check": true,
  "punctuation_check": true,
  "mismatch_check": true,
  "missing_check": false,
  "unsolvability_check": true,
  "answer_leak_check": true,
  "image_consistency_check": false,
  "skip_reasons": {
    "missing_check": "no figure/table/material reference and no blank markers",
    "image_consistency_check": "stem has no 'see figure / as shown' reference"
  }
}
```

### Three key designs

- **Missing key = false**: the routing prompt is explicit — any missing key in the output is treated as `false`, skipping that detection. **This is a deliberate "fail-closed" choice**, but it also means: when editing the routing prompt, always include every task_id in the output example, or you'll silently drop detections.
- **Better over-check than under-check**: the rule is "default true, set false only when clearly inapplicable". When unsure, output true. Routing exists to cut the **obviously** wasted calls, not to prune aggressively.
- **Special skip rules take precedence**: e.g. "upstream parsing incomplete, the stem is only a number/numbering path" → set all detections false; "term-definition-type ultra-short term question" → skip ambiguity/type/missing/unsolvable/answer-leak, keep typo.

### Payoff

Routing directly cuts wasted LLM calls on inapplicable questions (e.g. non-choice questions skip the option check, figure-less questions skip the image check). Across multi-task × multi-question batches, such pruning saves a meaningful number of calls.

## Decision point two: error aggregation

After a question is checked by multiple tasks, there are several possibly-overlapping reports. After deterministic priority aggregation, if more than one remains, an LLM aggregation layer decides an action per error:

- `keep`: keep as-is
- `merge`: merge with others and write one unified description
- `delete`: delete

**Hard constraints**: must not modify `error_type`; the action can only be one of these three; and **failures fall back without dropping errors** (on LLM error or unmatchable ids, keep the originals). This layer is **deliberately weakened to "merge-first"**, leaving heavy dedup to the deterministic priority layer. See [08](08-去重与聚合管线.en.md).

## Why these two use the LLM and the rest is deterministic

| Stage | Implementation | Why |
|---|---|---|
| Step order (parse→review→plagiarism) | Fixed | The order is naturally deterministic, no reasoning needed |
| Parse flattening, numbering location | Deterministic | Precision required; can't let the LLM improvise |
| Routing (which detections) | LLM | "What should this question be checked for" needs semantic understanding |
| Single detection output | LLM (constrained) | Detection is semantic; but error_type is locked |
| Same-location same-root-cause dedup | Deterministic priority table | Clear rules; must be reproducible & auditable |
| Semantic merging | LLM (constrained) | Cross-wording merges need judgment, but only merge/keep/delete |

**Principle**: use reproducible rules wherever you can (auditable, regression-testable); reserve the LLM for "needs semantic understanding, rules can't fully cover", and **constrain the LLM's output space to the minimum** (boolean switches / three actions / locked error_type). That's why it's "stable" and also why it's "not very agentic" — a trade-off, not a flaw.

## So why still call it an agent

- At the "per-question" granularity it really does let the LLM make **autonomous judgments** (what to check, how to merge these reports), rather than running every question through one dead ruleset.
- It has **multiple collaborating LLM roles**: skeleton extraction during parsing, routing, the various detections, image re-review, aggregation — like a "review team" with a division of labor.
- It has **human-in-the-loop + memory accumulation**: human review results can be distilled into the case library (dynamic few-shot) and flow back into later detection prompts.

But be honest with the interviewer: **it currently lacks "autonomous planning / dynamic re-routing / goal-driven multi-round self-correction"**. Positioning it as "a reliable workflow with LLM decision points" is more credible than overclaiming "fully autonomous agent" — and shows engineering judgment.

## How to evolve toward a more agentic form

- **Self-feedback after routing**: when detection finds many of a certain issue, dynamically attach related detections, or deep-check suspicious questions a second time (currently one-shot and fixed).
- **Aggregation upgraded to tool-using adjudication**: let the aggregation layer call "re-run a task" rather than only merge/delete.
- **Autonomous remediation on parse failure**: when parse confidence is low, let the LLM autonomously choose "switch strategy & re-extract / flag for human" rather than a fixed serial fallback.
- **Budget-aware scheduling**: autonomously decide detection depth by token budget (see [10](10-量化指标与评测.en.md)).

---

Next: [06 · Prompt Engineering](06-Prompt工程.en.md)
