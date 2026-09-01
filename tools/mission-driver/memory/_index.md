---
scope: mission-driver
kind: self-memory-index
lesson_count: 10
updated: 2026-09-01
---

# Mission-Driver Self-Memory Index

Top procedural rules the mission-driver engine injects into draft / execute /
closure-audit prompts via `{{selfMemoryIndex}}`. This file is **runtime-generated
and maintained by `mission-driver analyze`** (the `--analyze-run` postmortem
command). It consolidates durable lessons under a consolidate-don't-accumulate
protocol.

This template copy ships **empty** — rules accumulate as real runs are analyzed.

## Rules

_No rules promoted yet — 10 lessons exist in `lessons.md` (L001–L010). Promotion
bar: high-severity AND recurring (`count >= 2`). L004 is `count: 3` but SEV3.
L007 is SEV3 `count: 2`. L008 (SEV1, held-plan `held` outcome), L009 (SEV2,
DEEP_AUDIT no-op when audit prompts unconfigured) and L010 (SEV2, concurrent
missions sharing a Maven localRepository) are new from the 2026-09-01 starter
run — all `count: 1`, promote on recurrence. Top rules will be lifted here once
a high-severity lesson recurs across runs._
