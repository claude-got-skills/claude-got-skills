# Browser Test Analysis: Skill v1.4.0 on Claude.ai

**Date:** 2026-02-11
**Platform:** Claude.ai (web interface, Opus 4.6)
**Methodology:** 5 identical prompts tested with skill OFF (control) then ON (treatment)
**Evaluator:** Manual observation of thinking panel + response content

## Test Prompts

| # | Prompt | Category |
|---|--------|----------|
| P1 | Document processing pipeline in Python, 50-page docs, best API setup | Architecture / API |
| P2 | Difference between extended thinking and adaptive thinking, when to use each | API Features |
| P3 | Right Claude model for customer support chatbot — fast, cheap, accurate | Model Selection |
| P4 | Code review checklist repeated in every prompt, better way in Claude Code | Extension Patterns |
| P5 | Can Claude remember what we talked about last week in a different conversation? | Product Capabilities |

## Results Summary

| Prompt | Skill Triggered? | Web Search Used? | Quality Δ vs Control |
|--------|-----------------|-----------------|---------------------|
| P1 (doc pipeline) | YES | YES (both) | Similar — marginal |
| P2 (thinking) | YES | YES (both) | Similar — slightly more concise |
| P3 (model selection) | YES | YES (both) | Similar — more specific pricing |
| P4 (code review) | YES (explicit) | NO (treatment), YES (control) | **Treatment notably better** |
| P5 (memory) | UNCLEAR | NO (neither) | Similar — treatment more personalized |

## Detailed Analysis by Dimension

### 1. Trigger Reliability (4/5 confirmed)

The skill triggered on 4 of 5 prompts, confirmed via thinking panel text:

- P1: "Check product self-knowledge for accurate API details" ✅
- P2: "Check product self-knowledge for info on extended thinking and adaptive thinking" ✅
- P3: "Check product self-knowledge for current model details" ✅
- P4: "Check assistant capabilities skill for Claude Code configuration details" ✅ (most explicit)
- P5: "Let me answer this directly based on my product knowledge" ❓ (unclear if skill was read)

P5 is the weak case. The prompt was about cross-conversation memory — a product feature question. The skill description includes "memory tool" as a trigger, but Claude may have relied on its built-in product knowledge rather than reading the skill. This could indicate the memory/product-capability queries are already well-served by Claude's training data.

### 2. Accuracy

Both control and treatment provided accurate information across all 5 prompts. No hallucinations detected in either condition. Key accuracy observations:

- **P1**: Both correctly described Files API, prompt caching, batch processing. Treatment focused on Sonnet 4.5; control also mentioned Haiku 4.5 as budget option.
- **P2**: Both correctly explained extended vs adaptive thinking, deprecation on Opus 4.6, effort parameter values.
- **P3**: Both recommended Haiku 4.5, gave correct pricing. Treatment had slightly more granular pricing ($1/$5 input/output vs general framing).
- **P4**: Treatment had the most accurate and specific details — autoInvoke frontmatter, skills.sh, find-skills meta-skill, CLAUDE.md line limit guidance.
- **P5**: Both accurately described memory system and past chats search. Treatment referenced user's actual saved memories.

### 3. Completeness

| Prompt | Control | Treatment | Winner |
|--------|---------|-----------|--------|
| P1 | Mentioned Sonnet + Haiku, Files API, caching, batch | Mentioned Sonnet, Files API, caching, batch | Tie (control slightly wider model coverage) |
| P2 | Extended + adaptive, effort, deprecation | Same + interleaved thinking is free | Treatment (marginal) |
| P3 | Haiku recommended, tiered approach, Batch API | Same + specific $/M pricing, cache pricing ($0.10/M) | Treatment (marginal) |
| P4 | CLAUDE.md (3 scopes), slash commands, skills | CLAUDE.md vs Skills decision framework, autoInvoke, skills.sh, find-skills, 500-line guideline | **Treatment (significant)** |
| P5 | Memory system, past chats, incognito caveat | Same + remember/forget capability, settings toggle names | Treatment (marginal) |

### 4. Hallucination Prevention

Neither condition produced hallucinations in this test. However, the Opus 4.6 model on Claude.ai has strong self-knowledge and web search access, which means the baseline is already very high. This contrasts with the Haiku 4.5 eval results where control responses frequently hallucinated capabilities.

### 5. Naturalness

Both conditions produced natural, conversational responses. The treatment condition addressed the user by name (Liam) in prompts 1, 2, 4 — likely from Claude's memory system rather than the skill itself. No "skills dump" or awkward forced references to the skill content observed.

## Key Finding: Where the Skill Adds Most Value

The clearest skill advantage was on **P4 (Claude Code extension patterns)** — the one domain where:
1. Web search alone doesn't provide structured, actionable guidance
2. The skill's knowledge-base has unique content not readily available on docs pages
3. Claude's training data likely doesn't cover the rapidly-evolving skills/plugins ecosystem

The treatment response for P4 included:
- **autoInvoke frontmatter syntax** — not easily found via web search
- **skills.sh community registry** — emerging resource
- **find-skills meta-skill** with exact install command
- **"Keep CLAUDE.md under 500 lines"** heuristic
- **Clear decision framework**: "CLAUDE.md = always know this, Skill = know this when relevant"

For API/model questions (P1-P3), the skill's contribution was marginal because Claude.ai already has web search that surfaces Anthropic's docs pages effectively.

## Skill vs Plugin Decision

### Recommendation: **Keep as standalone skill (do NOT convert to plugin)**

Rationale:

1. **On-demand loading is sufficient.** The skill triggers reliably (4/5) when relevant topics come up. There's no need for it to auto-load on every session.

2. **Token efficiency.** At ~4,100 tokens for SKILL.md, loading on-demand saves context for sessions that don't involve capability questions. A plugin with autoInvoke would burn this budget every time.

3. **Marginal lift on well-served topics.** For API and model questions, Claude.ai's web search + training data already gives strong answers. The skill adds the most value for extension patterns and Claude Code specifics — topics that only arise in certain conversations.

4. **No autoInvoke benefit.** The skill's value comes from depth when asked, not from always being present. It's not a system-level instruction that changes Claude's behavior globally (like a CLAUDE.md rule would).

5. **Distribution model.** Skills can be installed with `npx skills add` — simple and lightweight. Plugins add packaging complexity (plugin.json, version management) without a clear benefit for this use case.

### When to reconsider (convert to plugin)

- If the skill needs **hooks** (e.g., PreToolUse to prevent deprecated API patterns)
- If it needs **slash commands** (e.g., `/capabilities update` to refresh knowledge base)
- If it needs **MCP integration** (e.g., fetching live API status or changelog)
- If trigger reliability drops below 3/5 on future evals (suggesting description matching is insufficient)

## Recommendations for v1.5.0

1. **Strengthen P5-style triggers.** Add "cross-conversation", "remember between conversations", "past chats" to the description's trigger keywords.
2. **Lean into the Claude Code niche.** The skill's unique value is in extension patterns, not API reference (which web search handles well). Consider expanding references/claude-code-specifics.md.
3. **Add eval for Claude.ai context.** The current eval runner tests Claude Code (API) context. Consider adding browser-based test prompts to the eval suite to measure web-search-augmented performance.
4. **Consider version pinning.** As the skills ecosystem matures, the skill should pin its compatibility to Claude Code versions to prevent drift.
