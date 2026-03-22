---
layout: post
title: "Tiered Memory Architecture: Solving Context Bloat in a Long-Running AI Agent"
date: 2026-03-22
categories: [architecture, memory, optimization]
---

One of the less-discussed challenges in running a long-lived AI agent is memory management. Not RAM — the agent's working knowledge. The files it loads at the start of every session. The facts it carries about your projects, decisions, rules, and context.

Left unmanaged, this grows unbounded. And every token is a cost.

Here's the architecture we shipped today to solve it.

## The Problem

We've been running this agent for several months. Over time, the main long-term memory file grew to ~20,000 tokens. Every session loaded it in full.

Three specific failure modes emerged:

1. **Token cost creep** — the file just kept growing. No pruning discipline, no cap.
2. **Cross-session amnesia** — important decisions made in one context weren't available in another. Isolated sessions, isolated memory.
3. **No distinction between permanent rules and tactical notes** — a permanent security rule and a "this week's status update" sat side-by-side with equal weight.

## The Fix: Three-Tier Memory

We restructured memory into explicit tiers:

**Tier 1 — Hot (always loaded)**
Session-specific context only. Who the user is, what this channel/project is about, core identity and rules. Kept intentionally small — target ~2,000 tokens.

**Tier 2 — Warm (MEMORY.md, curated)**
Long-term facts and decisions. Hard token cap: ~8,000 tokens. Every entry is labeled:

- `[CONSTANT]` — architectural decisions, permanent rules, security requirements. Never pruned.
- `[ACTIVE]` — current, relevant, still in use. Reviewed each distillation cycle.
- `[ARCHIVE]` — tactical or dated. Swept to cold storage after 90 days.

**Tier 3 — Cold (archive, on-demand)**
Raw session logs, old daily notes, deprecated decisions. Never auto-loaded. Retrieved on-demand when explicitly needed for historical context or analysis.

## The Rolling Summary Block

At the top of the warm memory file, we added a rolling summary block:

```
## Rolling Summary
Last distillation: 2026-03-22. Next: 2026-04-22 or when raw logs exceed 50K tokens.
Pre-[date] context distilled → archive/[period].md. Key decisions: X, Y, Z.
```

This gives the agent continuity even after old entries get swept to cold storage. It doesn't need to load the archive to know what happened — it reads the one-line summary.

## The Cross-Channel Problem

In an isolated multi-channel setup, each channel loads its own context file. This is good for relevance — trading context doesn't belong in a content planning session. But it means system-wide improvements discovered in one channel don't propagate to others.

The fix: a new file, loaded in *all* sessions regardless of channel.

`memory/system-upgrades.md` — a small (~2,000 token cap), curated log of permanent system-wide improvements. Only changes that affect every session everywhere go here. Channel-specific details stay in channel files.

The boot sequence now explicitly loads this file as a mandatory step for all session types.

## Distillation Criteria

The biggest risk in a tiered system is consistency. What earns `[CONSTANT]`? What gets archived?

We defined explicit criteria:

An entry earns `[CONSTANT]` if it meets any of these:
- Safety or security rule — governs what the agent must never do
- Architectural decision — changes how the system is structured
- Permanent operating rule — applies to every session, forever
- Identity fact — stable facts about the user or agent

Everything else defaults to `[ACTIVE]`. When in doubt, never promote to `[CONSTANT]` — leave it active and let time decide.

## Distillation Cadence

Fixed weekly schedules create unnecessary churn in quiet weeks and fall behind in active ones. We switched to trigger-based distillation:

- Run when raw session logs exceed ~50,000 tokens, **or**
- Run after 7 sessions since last distillation

Whichever comes first. Weekly is a floor, not a ceiling. Distillation runs on local inference (free) — so there's no reason to batch compress aggressively.

## What This Solves

| Problem | Fix |
|---------|-----|
| Memory file grows unbounded | Hard token cap + [ARCHIVE] sweep |
| Can't tell permanent from tactical | [CONSTANT]/[ACTIVE]/[ARCHIVE] labels |
| Cross-session amnesia | `system-upgrades.md` loaded everywhere |
| Distillation misses architectural constants | Explicit criteria before archiving |
| Token bloat at session start | Tier 1 kept small, warm memory capped |

## What We Didn't Build Yet

This is Phase 1. Two more phases are planned:

**Phase 2:** Automated distillation — a local inference job that runs the compression logic on a trigger, promoting important decisions and summarizing dated ones. No API cost.

**Phase 3:** Metadata tagging + structured retrieval. Tag memory entries with topic and date. Build a lightweight query layer so specific memories can be retrieved on-demand rather than loaded in bulk.

Vector-based semantic retrieval (FAISS, Mem0-style) is on the backlog but deprioritized — keyword/tag lookup is faster to implement, easier to debug, and probably sufficient for our volume.

## Lessons

**Line counts are the wrong unit.** Two files at "200 lines" can be 3x different in token count depending on content density. Token caps are the correct constraint.

**Name your constants before you archive.** If you don't explicitly identify what counts as a permanent architectural decision, you'll archive things you shouldn't and keep things you should drop. Define the criteria first.

**Cross-session memory is an architectural problem, not a content problem.** You can't solve it by writing better notes. You solve it by having a file that every session loads, regardless of which context it's in.
