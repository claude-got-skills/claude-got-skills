# Chrome CDP Browser Eval Findings — 2026-03-16

## Eval Overview

- **Date:** 2026-03-16, 16:38–17:06
- **Backend:** Chrome CDP (new, 5-6x faster than agent-browser)
- **Prompts:** 15 (selected from 65-test API eval suite)
- **Categories:** Hallucination Detection (2), Can Claude Do X (3), Extension Awareness (2), Cross-Platform Awareness (2), Model Selection (1), Architecture Decisions (1), Implementation Guidance (1), Competitor Migration (1), Conversational Platform (2)
- **Total time:** 28.4 minutes (30 prompt cycles, avg ~37s each)
- **Success rate:** 30/30 prompts completed, 0 errors

## Headline Results

### Skill Trigger Rate: 10/15 (67%) treatment prompts

| Which Skill | Count | Prompts |
|---|---|---|
| **assistant-capabilities** (ours) | 8 | P1, P3, P4, P8, P9, P10, P13, P15 |
| **product-self-knowledge** (built-in) | 2 | P2, P12 |
| Not triggered | 5 | P5, P6, P7, P11, P14 |

### Web Search Reduction

| Condition | Prompts using web search | Rate |
|---|---|---|
| Control (skill OFF) | P2, P4, P5, P9, P13 | **5/15 (33%)** |
| Treatment (skill ON) | P5 only | **1/15 (7%)** |

**The skill reduced web searches by 80%** — Claude used the skill's authoritative content instead of searching the web.

## Best Showcase Examples for Release Video

### Example 1: P8 — "How do I invoke a skill by name on Claude.ai?"
**Category:** Cross-Platform Awareness

**Control (no skill):** Vague answer about skills being "triggered based on description matching" — lacks specifics about the actual mechanism.

**Treatment (with skill):** Precise answer: "on Claude.ai, you can't invoke a skill directly by name" — cites platform documentation, explains the description-based routing mechanism accurately.

**Why it's good for video:** Shows the skill preventing incorrect advice about a nuanced platform feature.

---

### Example 2: P9 — "Can I build a multi-step workflow on Claude.ai?"
**Category:** Cross-Platform Awareness

**Control:** Needed web search, eventually pointed to Claude Code but took longer and was less structured.

**Treatment:** Immediately identified the workflow requires Claude Code, not Claude.ai. No web search needed. More concise and actionable.

**Why it's good for video:** Demonstrates platform-aware guidance — the skill knows what each platform can and can't do.

---

### Example 3: P4 — "Hundreds of MCP tools overwhelming context window"
**Category:** Can Claude Do X

**Control:** Used web search to find information about tool management features.

**Treatment:** Directly referenced tool filtering/deferred tools from the skill's knowledge. No web search. Faster, more authoritative.

**Why it's good for video:** Shows the skill eliminating web search latency by having authoritative API knowledge ready.

---

### Example 4: P1 — "Does Claude remember things between conversations?"
**Category:** Hallucination Detection

**Control:** Gave a correct but general answer based on system prompt knowledge.

**Treatment:** Referenced the assistant-capabilities skill explicitly in thinking, provided more specific details about memory mechanics (background updates, Incognito mode, memory_user_edits tool).

**Why it's good for video:** The thinking panel shows "Let me also check my assistant-capabilities skill" — visible proof of skill invocation.

---

### Example 5: P13 — "What's the equivalent of custom GPTs in Claude?"
**Category:** Competitor Migration

**Control:** Used web search to find Claude equivalents to custom GPTs.

**Treatment:** Answered from skill knowledge without web search. More comprehensive feature mapping.

**Why it's good for video:** Migration guidance without relying on potentially outdated web results.

---

### Example 6: P10 — "Which model for 200+ page documents?"
**Category:** Model Selection

**Control:** Used product-self-knowledge (built-in), gave reasonable advice.

**Treatment:** Used assistant-capabilities (our skill), provided more detailed model comparison with specific context window sizes and pricing considerations.

**Why it's good for video:** Direct model recommendation comparison — treatment is more specific and actionable.

## Patterns Observed

### Where the skill adds most value
1. **Platform-specific questions** (P8, P9) — knowing what each Claude surface can/can't do
2. **Feature awareness** (P3, P4) — detailed knowledge of memory, tool search, deferred tools
3. **Migration guidance** (P13) — mapping competitor features to Claude equivalents
4. **Model selection** (P10) — specific parameters, pricing, context windows

### Where the skill doesn't trigger (and why)
1. **General dev questions** (P6, P7) — pre-commit hooks, Slack workflows — these are general knowledge, not Claude-specific
2. **Architecture questions** (P11) — QA system design — domain knowledge, not product knowledge
3. **File generation** (P14) — Claude already knows its own artifact capabilities well
4. **Code Review** (P5) — too new (launched March 9), both conditions searched the web

### Built-in skill overlap (P2, P12)
Two prompts triggered `product-self-knowledge` instead of our skill:
- P2: Sonnet 3.7 deprecation/migration
- P12: Opus 4.6 prefilling breaking change

Both are migration/deprecation topics. The built-in skill may have stronger trigger affinity for these. Worth investigating whether our skill's description should be tuned to capture these.

## Technical Performance

| Metric | Value |
|---|---|
| Average prompt cycle | ~37s |
| Response detection | Feedback buttons (primary), Copy+no-Stop (secondary) |
| Text input | `document.execCommand('insertText')` — ProseMirror compatible |
| Submit | Click `button[aria-label="Send message"]` |
| Response extraction | Via `action-bar-copy` ancestor walk |
| Thinking extraction | Via `button[aria-expanded]` within assistant message scope |
| Skill toggle | Automated via `/customize/skills` DOM interaction |
| Speed vs agent-browser | ~5.5x faster (28 min vs ~135 min projected) |

## Raw Data Location

- Results JSON: `evals/browser-eval-results.json` (gitignored)
- Full report: `evals/browser-eval-report-20260316-170639.md` (gitignored)
- Run directory: `evals/browser-eval-run-20260316-163813/` (gitignored)
- Screenshots: `evals/browser-eval-run-20260316-163813/screenshots/` (gitignored)
