# 06 · Prompt Engineering

> [Home](../README.en.md) ｜ [中文](06-Prompt工程.md) ｜ Prev: [05 Routing & "Agent or Workflow"](05-路由与Agent还是工作流.en.md)

> All prompt snippets here are **rewritten skeletons + generic examples**, not production originals. The focus is the "structure" and "why".

## What sections a detection prompt has

Text-class detection prompts (typo, semantics, punctuation…) share a stable "anatomy":

```
# Role            One or two lines casting the model as a "detect-only, don't-think, don't-solve" scanner
# Input spec      reference (read-only, must not report on it); detection_content (the final text to check)
## Core goal      Precisely state what to detect and what NOT to detect
## Hard bans      No solving / evaluating / scope-creep / reporting on reference
## Workflow       1) Pre-checks (empty content? find-the-error question? image placeholders?)
                  2) Detection (specific error patterns + examples)
                  3) Careful reporting (confidence threshold, often ≥0.7)
## Examples       Positive/negative pairs
## Rules          Severity grading (severe / minor / cases not to report)
## Final output   error_type "must and can only" be this task's class; out-of-scope is never emitted
+ Fixed output JSON structure (shared template across all tasks)
```

Design points:

- **Compress the role into a "scanner"**: make "detect-only, don't solve" explicit, so the model doesn't go solve the problem, give academic critique, or expand the checking scope.
- **reference is read-only**: reference material only aids understanding the question; **reporting on it is banned** — otherwise it would treat background material as the question and report a pile of false errors.
- **Confidence threshold**: write "don't report what you're unsure of" into the prompt (≥0.7) — the first gate for suppressing false positives.
- **Lock error_type**: each task's prompt ends with a hard constraint that the output type can only be its own class, eliminating type drift (see [04](04-错误类别与分类.en.md)).

## Fixed output JSON structure (shared by all tasks)

All detection tasks reuse one output template, for unified downstream parsing, aggregation and storage:

```json
{
  "think": "full reasoning chain: [task analysis]→[execution trace]→[final decision]→[status check] (≤800 chars)",
  "reason": "one-sentence conclusion (≤60 chars)",
  "quality_score": 0.0,
  "overall_confidence": 0.0,
  "has_error": false,
  "total_errors": 0,
  "errors": [
    {
      "error_type": "this task's class",
      "position": "human-readable location",
      "original_text": "the erroneous source text (exactly matching the source, as concise as possible)",
      "anchor_text": "a contiguous anchor for recall location (more context than original_text, no placeholders, easy to search in Word)",
      "correction": "the corrected content",
      "description": "what's wrong (≤2 sentences, no reasoning)",
      "suggestion": "action instruction + correct text (≤3 sentences, no reasoning)",
      "severity": "severe/medium/minor",
      "confidence": 0.0
    }
  ]
}
```

Several engineering details, all hard constraints added after being burned:

- **Ban ASCII double quotes `"` in `think` and `reason`**: otherwise JSON parsing breaks; use single or Chinese quotes to quote.
- **Separate `anchor_text` from `original_text`**: `original_text` aims for precision (display/comparison); `anchor_text` aims for searchability (contiguous, with context, no `<imgN/>` placeholders) to **uniquely locate** the error in the original Word file.
- **"Wording ban" for user-facing fields**: `description / suggestion / position / correction` must not contain internal variable names like `detection_content`, `reference`, `has_error`, nor system markers like `<imgN/>` or "image placeholder" — because placeholders are system-generated, not authored content, so suggesting the user "delete the placeholder" is wrong.
- **`suggestion` bans hesitation words**: no "wait", "let me look again", "could it be" — it must be a clean action instruction + correct text.

## The routing prompt

Routing is a **meta-prompt**: it doesn't detect content, it only decides "which detections to run for this question", outputting a dict of boolean switches where a missing key = false. Its structure, examples and "why fail-closed" are in [05](05-路由与Agent还是工作流.en.md). To reiterate: **the output example must list all task_ids**, or detections will be silently dropped.

## The aggregation prompt

The aggregation layer's prompt hands a question's remaining errors plus the stem to the LLM, which decides `keep / merge / delete` per error, may rewrite descriptions, but **must not change error_type**. It is deliberately written as "merge-first", leaving heavy dedup to the deterministic priority layer. Structure in [08](08-去重与聚合管线.en.md).

## Multilingual: same structure, separate files

The zh/ja/de/fr detection prompts are **one structure, one file each** (e.g. typo has four: `typo` / `ja_spelling` / `de_spelling` / `fren_spelling`). Benefits:

- Each language can encode its own error patterns (Japanese kana/kanji, German capitalization/compounds, French ligatures/agreement), tuned separately;
- With a consistent structure, downstream parsing/aggregation/storage is fully reused — adding a language is just "add a set of files + register tasks".

## Dynamic few-shot: injecting human experience into the prompt

During review, a human can distill a judgment into a **positive/negative case** in the library. At detection time, a query is built from `language + task_id + subject + question_type + detection_content + reference`, the most similar positive/negative cases are recalled via sentence-transformers vector similarity, and `prompt_injector` injects them into the detection prompt to give the LLM a "how we judged similar questions before" reference.

Engineering points:

- **Strong filtering**: language + task_id must match (no cross-language recall); candidates then fall back by subject + question_type.
- **Similarity threshold**: below the threshold (default 0.70) nothing is injected, to avoid "irrelevant examples" misleading the model.
- **Observability**: the detection-detail Excel records whether injection happened, recalled case ids, similarity and latency — to answer "did few-shot fire, did it help" (compare metrics before/after on the evaluation line, see [10](10-量化指标与评测.en.md)).
- **Timeout handling**: local embedding can take >2s per query under high-concurrency review, so the retrieval timeout must be generous (otherwise it mis-fires as a timeout and drops few-shot).

## A philosophy that runs throughout the prompts

**Constrain the LLM's output space to the minimum**: detection locks error_type, routing outputs only booleans, aggregation allows only three actions, and all outputs follow one JSON template. The tighter the constraints, the more stable the parsing, the more testable the regression, the more controllable the false positives — that's the key to engineering "uncontrollable generation" into "reliable pipeline nodes".

---

Next: [07 · Image Review Deep Dive](07-图片审校深挖.en.md) ★
