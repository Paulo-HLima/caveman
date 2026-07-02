---
name: cavecrew
description: >
  Decision guide for delegating to caveman-style subagents. Tells the main
  thread WHEN to spawn `cavecrew-investigator` (locate code), `cavecrew-builder`
  (1-2 file edit), or `cavecrew-reviewer` (diff review) instead of doing the
  work inline or using vanilla `Explore`. Subagent output is caveman-compressed
  so the tool-result injected back into main context is ~60% smaller — main
  context lasts longer across long sessions.
  Trigger: "delegate to subagent", "use cavecrew", "spawn investigator/builder/reviewer",
  "save context", "compressed agent output".
---

Cavecrew = three subagent presets that emit caveman output. Same job as Anthropic defaults (`Explore`, edit-style agents, reviewer); difference is the tool-result they return is compressed, so main context shrinks per delegation.

## When to use cavecrew vs alternatives

| Task | Use |
|---|---|
| "Where is X defined / what calls Y / list uses of Z" | `cavecrew-investigator` |
| Same but you also want suggestions/architecture commentary | `Explore` (vanilla) |
| Surgical edit, ≤2 files, scope obvious | `cavecrew-builder` |
| New feature / 3+ files / cross-cutting refactor | Main thread or `feature-dev:code-architect` |
| Review diff, branch, or file for bugs | `cavecrew-reviewer` |
| Deep code review with rationale + alternatives | `Code Reviewer` (vanilla) |
| One-line answer you already know | Main thread, no subagent |

Rule of thumb: **if you'd want the subagent's output in 1/3 the tokens, pick cavecrew. If you'd want prose, pick vanilla.**

## Model routing (cost tiering)

Each agent runs on a fixed model tier so cheap work never burns Opus quota. Main thread stays Opus 4.8 (reasoning + orchestration); it delegates *execution* down-tier.

| Agent / thread | Model | Why this tier |
|---|---|---|
| main thread | Opus 4.8 | Orchestration, architecture, hard/exploratory debug, decisions |
| `cavecrew-reviewer` | Opus (inherits) | Bug-finding is reasoning — keep flagship |
| `cavecrew-builder` | Sonnet 5 | Mechanical edit ≤2 files — Sonnet ~5x cheaper than Opus, quality ample |
| `cavecrew-investigator` | Haiku | Locate = pattern match — cheapest tier is enough |

**Delegate down-tier ONLY when the task is:**
- **Self-contained** — every fact the agent needs fits in the spawn prompt. No back-and-forth (each round = new cold spawn = lost savings).
- **Clear output** — you can name the exact deliverable up front (a `file:line` list, a diff receipt).
- **Bounded** — investigator: one search intent; builder: ≤2 files AND site already known.

**Do NOT delegate — keep on Opus main — when:**
- The fix isn't known yet (exploratory debug).
- Architecture / design / tradeoff calls.
- Task only makes sense with the main thread's accumulated context.

Net effect: Opus reasons + decides, cheaper tiers execute volume → weekly Opus budget spent only where reasoning is the product. Overhead caveat: a spawn re-reads context cold, so a *single* tiny edit main could do in one `Edit` may cost more delegated than inline — delegate for *volume* and *context-offload*, not one-liners.

## Why this exists (the real win)

Subagent tool results get injected into main context verbatim. A vanilla `Explore` that returns 2k tokens of prose costs 2k tokens of main-context budget every time. The same finding from `cavecrew-investigator` returns ~700 tokens. Across 20 delegations in one session that's the difference between context exhaustion and finishing the task.

## Output contracts

What main thread can rely on per agent:

**`cavecrew-investigator`**
```
<Header>:
- path:line — `symbol` — short note
totals: <counts>.
```
Or `No match.` Always file-path-first, line-number-attached, backticked symbols. Safe to grep with `path:\d+`.

**`cavecrew-builder`**
```
<path:line-range> — <change ≤10 words>.
verified: <re-read OK | mismatch @ path:line>.
```
Or one of: `too-big.` / `needs-confirm.` / `ambiguous.` / `regressed.` (terminal first token).

**`cavecrew-reviewer`**
```
path:line: <emoji> <severity>: <problem>. <fix>.
totals: N🔴 N🟡 N🔵 N❓
```
Or `No issues.` Findings sorted file → line ascending.

## Chaining patterns

**Locate → fix → verify** (most common):
1. `cavecrew-investigator` returns site list.
2. Main thread picks 1-2 sites, hands paths to `cavecrew-builder`.
3. `cavecrew-reviewer` audits the diff.

**Parallel scout** (when investigation is broad):
Spawn 2-3 `cavecrew-investigator` calls in one message (different angles: defs vs callers vs tests). Aggregate in main thread.

**Single-shot edit** (when site is already known):
Skip investigator. Hand exact path:line to `cavecrew-builder` directly.

## What NOT to do

- Don't use `cavecrew-builder` when you don't already know the file. Spawn investigator first or main thread will eat tokens passing context.
- Don't chain `cavecrew-investigator → cavecrew-builder` for a 5-file refactor. Builder will return `too-big.` and you'll have wasted a turn.
- Don't ask `cavecrew-reviewer` for "general feedback" — it returns findings only, no architecture opinions. Use `Code Reviewer` for that.
- Don't expect prose. Cavecrew output is structured, sometimes terse to the point of cryptic. If a human will read it directly, paraphrase.

## Auto-clarity (inherited)

Subagents drop caveman → normal English for security warnings, irreversible-action confirmations, and any output where fragment ambiguity could be misread. Resume caveman after.
