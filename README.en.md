# Multilingual Exam Review Agent · exam_checker_agent

> 🌐 English (WIP) ｜ [中文](README.md)

> A full-stack review platform that splits an entire exam paper into structured questions, runs per-question content / image-text / plagiarism checks, and produces reviewable reports.

> 📌 **About this repo**: a **desensitized design archive** of `exam_checker_agent` — Markdown notes and diagrams only, **no runnable source code, no company exam data, no verbatim prompts**. All prompt snippets are **rewritten skeletons** with generic examples.

The full English translation is in progress. For now, please read the **[Chinese version](README.md)**, which contains the complete documentation set:

- [01 · Background & Goals](docs/01-背景与目标.md)
- [02 · System Architecture](docs/02-系统架构.md) — dual web stack (FastAPI + in-process Flask bridge), AppRuntime DI, SQLite migrations
- [03 · End-to-End Flow](docs/03-端到端流程.md) — five-stage parsing → routing → detection → plagiarism
- [04 · Error Categories](docs/04-错误类别与分类.md) — 20 detection tasks across 4 languages, unified type mapping
- [05 · Routing & "Agent or Workflow"](docs/05-路由与Agent还是工作流.md) — an honest take on the two constrained LLM decision points
- [06 · Prompt Engineering](docs/06-Prompt工程.md) — detection prompt anatomy, fixed JSON schema, few-shot
- [07 · Image Review Deep Dive](docs/07-图片审校深挖.md) ★ — four-stage multimodal pipeline + text-only re-review
- [08 · Dedup & Aggregation](docs/08-去重与聚合管线.md)
- [09 · Retry & Resilience](docs/09-重试与韧性.md)
- [10 · Metrics & Evaluation](docs/10-量化指标与评测.md) — confusion matrix, P/R/F1/F2/FPR, token accounting
- [11 · Tech Stack & Engineering Decisions](docs/11-技术栈与工程决策.md)

---

*Design archive, desensitized, by the author.*
