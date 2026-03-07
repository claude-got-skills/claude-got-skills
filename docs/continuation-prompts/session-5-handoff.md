# Session 5 Handoff: Claude Capabilities Skill

## Project Context

We are building a **Claude Capabilities Awareness skill** — a Claude Code skill that gives Claude accurate, post-training knowledge about its own capabilities, API features, extension patterns, and model specifics. The skill addresses the gap between Claude's training data cutoff and the rapidly evolving feature set.

**GitHub repo**: `claude-got-skills/claude-capabilities` (NEW — previously planned under `liam-jons` which has been fully replaced)

**STATUS: Skill is now LIVE on Claude.ai** — successfully uploaded to Claude settings → capabilities page.

## Current State

### Skill Version: 1.3.0

The skill is functionally complete with:
- **SKILL.md** (~335 lines): Core capabilities reference covering models, thinking, context, tools, outputs, agents, platform, extension patterns, architecture decisions, breaking changes
- **5 reference files**: api-features.md, tool-types.md, agent-capabilities.md, model-specifics.md, claude-code-specifics.md
- **README.md**: Installation, eval results, structure
- **LICENSE**: MIT (copyright: claude-got-skills contributors)

### Key Directories

- **Working copy**: `.planning/builds/claude-capabilities-skill/claude-capabilities/`
- **Publish copy**: `.planning/builds/claude-capabilities-skill/publish/claude-capabilities/` — This is the version to push to GitHub. All liam-jons references have been replaced with claude-got-skills.
- **Evals**: `.planning/builds/claude-capabilities-skill/evals/` — 8 eval runs total

### Frontmatter: Hard-Won Constraints for Claude.ai Capabilities Page

The Claude settings capabilities page validator is significantly stricter than Claude Code's local skill parser. Through trial and error, we discovered these constraints:

**Constraint 1 — No YAML block scalars**: The `>` and `|` folded/literal block scalars are not supported. Description must be a single-line value.

**Constraint 2 — No colons in description**: The `:` character within the description value breaks the parser. We replaced `MANDATORY TRIGGERS:` with `MANDATORY TRIGGERS -` and removed all other colons from the description text.

**Constraint 3 — Character limit on description**: The original ~150-word description was too long. It was shortened by removing the preamble ("Current Claude model capabilities, API features...") and some trigger phrases.

**Constraint 4 — "Claude" is a reserved word in the `name` field**: `name: claude-capabilities` was rejected. Changed to `name: assistant-capabilities`. Note: "Claude" CAN appear in the description text, just not in the `name` field.

**Constraint 5 — Only allowed top-level keys**: `name`, `description`, `license`, `allowed-tools`, `compatibility`, `metadata`. No `version` or custom keys.

**Constraint 6 — No nested objects**: `metadata:` with nested `version:` was rejected despite `metadata` being an allowed key.

**Current frontmatter (LIVE, working):**
```yaml
---
name: assistant-capabilities
description: This skill should be used whenever Claude is designing a system architecture, choosing between implementation approaches, advising on what Claude can or cannot do, discussing API features, evaluating whether a capability exists, building agents or tools, helping users choose between extension patterns (CLAUDE.md vs skills vs hooks vs plugins), or any situation where knowledge of Claude's current capabilities would improve the response. MANDATORY TRIGGERS - Discussing Claude capabilities, API features, model comparison, context window, tool use, agent SDK, what can Claude do, Claude limitations, model selection, structured outputs, MCP connector, computer use, files API, memory tool, adaptive thinking, effort parameter, code execution, Claude Code, agent teams, chrome browser, CLI reference, skills, plugins, subagents, hooks, automate this, repeatable workflow, extension pattern, CLAUDE.md vs skill, hook vs skill, skills.sh, find skills, skill marketplace.
---
```

**IMPORTANT NOTE**: Despite `name: assistant-capabilities` in frontmatter, the system loads it as `claude-capabilities` (visible in the skills list in the system reminder). This appears to be because the Claude.ai capabilities page may use the repo/folder name or a separate identifier rather than the frontmatter `name` field for display. The `license` field was also dropped from the final version to minimise frontmatter complexity.

## Eval Results Summary

### Haiku 4.5 Eval (v1.3.0) — eval-results-20260211-010400.json
- **22 test prompts across 7 categories**
- Control (no skill): 52.1% accuracy, 36.4% completeness
- Treatment (with skill): **87.8% accuracy, 80.0% completeness**
- Lift: +35.7% accuracy, +43.6% completeness

**Category breakdown (treatment accuracy):**
| Category | Accuracy | Notes |
|----------|----------|-------|
| Can Claude Do X | 100% | Perfect |
| Implementation Guidance | 100% | Perfect |
| Model Selection | 100% | Perfect |
| Negative (regression) | 100% | No interference with non-capability queries |
| Extension Awareness | 86% | Good, room for improvement |
| Hallucination Detection | 87.5% | Effectively prevents confident-wrong answers |
| Architecture Decisions | 50.5% | Weakest — answers architecture but doesn't bridge to Claude capabilities |

### Opus 4.6 Eval — eval-results-20260211-112210.json
- Separate run on Opus. Results show similar patterns but with higher baseline due to Opus's stronger reasoning.
- Token overhead: ~97,623 tokens

### Previous eval runs (v1.2.1 → v1.3.0 progression)
- 6 earlier runs in the evals directory tracking iterative improvement
- See `v121-analysis.md` and `sonnet-v130-analysis.md` for version-specific analyses

## Open Work Items

### 1. ~~IMMEDIATE: Confirm Frontmatter Fix Works~~ ✅ DONE
The skill is now live on Claude.ai. The frontmatter constraints are documented above for future reference.

### 2. Browser Testing (Continuation from Session 4) — TOP PRIORITY
- The skill is now loaded on Claude.ai — we can now do live browser testing
- Previous attempt hit a Usage Policy API error — this needs to be retried
- **Goal**: Compare Claude's responses with and without the skill loaded in a real browser session on claude.ai
- Test prompts to use: the same categories from the eval (architecture decisions, "can Claude do X?", model selection, extension awareness, hallucination-prone questions)
- **Approach**: Use natural, user-like phrasing rather than eval-style prompts to avoid triggering content filters
- **Key comparison**: Does the skill trigger reliably on implicit capability questions (like architecture decisions), or does it only trigger on explicit "can Claude do X?" style questions?

### 3. Deploy Subagents for Iterative Review
**Three parallel review tracks needed:**

**Track A — Skill Content Review**: Read the full publish copy (SKILL.md + all 5 references). Identify:
- Outdated or potentially inaccurate claims
- Sections that could be more concise (token optimization)
- Gaps in coverage (features mentioned in evals as missing)
- Opportunities to move content from SKILL.md body to references (progressive disclosure)

**Track B — Eval Analysis Deep Dive**: Read ALL 8 eval result JSONs and reports. Identify:
- Patterns in what fails across runs (not just the latest)
- Whether architecture decisions failures are structural (skill content issue) or trigger-related (description issue)
- Token usage trends across versions
- Suggestions for new test cases based on gaps

**Track C — Token Optimization**: Measure current token costs and identify reduction opportunities:
- SKILL.md body token count vs the ~1,500-2,000 word ideal from skill-development best practices
- Reference file sizes and whether any can be trimmed
- Whether the shortened description (post character-limit fix) still triggers as reliably as the longer version
- Impact of removing `license` field

### 4. Session-Start vs User-Prompt Loading Question
This is the key strategic question for the plugin requirements:

**Current behavior**: The skill loads when Claude detects relevant trigger phrases in a user prompt (on-demand). On Claude.ai, it's loaded via the capabilities page (always available as a skill).

**Alternative**: Load at session start (always available, like CLAUDE.md) via a plugin with `autoInvoke`.

**Why this matters**:
- If the skill needs to be available for architecture decisions (which are often implicit, not triggered by keywords), session-start loading may be necessary
- But session-start loading costs ~4,100 tokens every session, even irrelevant ones
- The browser testing comparison will help determine this — if Claude's responses are significantly better with the skill pre-loaded vs triggered on-demand, that justifies the token cost
- This decision determines whether the delivery vehicle is a **standalone skill** (on-demand) or a **plugin with auto-invoke** (session-start)

### 5. GitHub Publishing
- Repo `claude-got-skills/claude-capabilities` has been created
- The publish directory is ready — push contents to the repo
- **Note**: The publish copy has `name: assistant-capabilities` in frontmatter (required by Claude.ai validator). For GitHub/skills.sh publishing where this restriction may not apply, we could potentially use the original `name: claude-capabilities`.
- Consider whether to also publish to skills.sh registry
- The README already has eval results and installation instructions

### 6. Usage Policy Error Investigation
- The browser testing hit: "Claude Code is unable to respond to this request, which appears to violate our Usage Policy"
- This occurred when testing capability-related prompts through the browser
- May need to rephrase test prompts to avoid triggering content filters
- Try using more natural, user-like phrasing rather than eval-style prompts

## File References

| File | Purpose |
|------|---------|
| `.planning/builds/claude-capabilities-skill/publish/claude-capabilities/` | **PUBLISH-READY** — push to GitHub |
| `.planning/builds/claude-capabilities-skill/claude-capabilities/` | Working copy (synced with publish) |
| `.planning/builds/claude-capabilities-skill/evals/capabilities-skill-eval.py` | Eval runner script |
| `.planning/builds/claude-capabilities-skill/evals/eval_runner.py` | Eval infrastructure |
| `.planning/builds/claude-capabilities-skill/evals/eval-results-*.json` | Raw eval results (8 runs) |
| `.planning/builds/claude-capabilities-skill/evals/eval-report-*.md` | Human-readable eval reports |
| `.planning/builds/claude-capabilities-skill/session-3-analysis.md` | Session 3 analysis notes |
| `.planning/builds/claude-capabilities-skill/session-4-continuation.md` | Session 4 continuation notes |
| `.planning/builds/claude-capabilities-skill/session-5-handoff.md` | This file |

## Session 5 Priorities (Recommended Order)

1. ~~Confirm frontmatter fix~~ ✅ DONE — skill is live on Claude.ai
2. **Browser testing** — This is now unblocked. Test the skill live on Claude.ai with natural prompts. Compare responses to baseline (could temporarily disable skill for control, or use a separate browser profile)
3. **Push to GitHub** — `claude-got-skills/claude-capabilities` from publish directory
4. **Deploy 3 parallel subagents** for content review, eval deep dive, and token optimization
5. **Analyze browser test results** against eval results — determine if on-demand triggering is sufficient or if session-start loading is needed
6. **Make decision on plugin requirements** based on loading analysis
7. **Iterate on skill content** based on subagent findings
