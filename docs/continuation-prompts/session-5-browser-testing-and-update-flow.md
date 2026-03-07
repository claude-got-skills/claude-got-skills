# Session 5 — Claude Capabilities Skill: Browser Testing, Update Flow & Plugin Decision

## Project Overview

We're building a **Claude Capabilities Awareness Skill** for skills.sh — a skill that gives Claude accurate, post-training knowledge about its own capabilities. The skill is at **v1.3.0**, publication-ready, with strong eval results on both Haiku and Sonnet.

**GitHub repo**: `liamjons/claude-capabilities` (being created now — push contents of `publish/claude-capabilities/` directory)

## File Locations

```
.planning/builds/claude-capabilities-skill/
├── claude-capabilities/
│   ├── SKILL.md                     # The skill (v1.3.0, 334 lines, ~4100 tokens)
│   └── references/                  # 5 deep-dive reference files
│       ├── agent-capabilities.md
│       ├── api-features.md
│       ├── claude-code-specifics.md
│       ├── model-specifics.md
│       └── tool-types.md
├── evals/
│   ├── eval_runner.py               # Eval framework (1032 lines)
│   ├── eval-results-*.json          # 7 timestamped result files
│   ├── eval-report-*.md             # Corresponding reports
│   ├── v121-analysis.md             # v1.2.1 analysis
│   └── sonnet-v130-analysis.md      # Sonnet validation results
├── publish/
│   └── claude-capabilities/         # Exact structure for GitHub publication
│       ├── SKILL.md
│       ├── README.md
│       ├── LICENSE (MIT)
│       └── references/              # (same 5 files)
└── CHANGELOG.md
```

Also relevant:
- `distill-monitoring-setup.md` — Distill web monitor configuration (8 monitors for Anthropic pages)
- `.planning/prompts/distill-resolution-prompt.md` — Fix for Distill SELECTION_EMPTY errors on support.claude.com

## Version History

| Version | Key Changes |
|---------|-------------|
| v1.0.0 | Initial SKILL.md + 5 reference files + eval framework |
| v1.1.0 | Trimmed SKILL.md from 11K→4.1K tokens, upgraded eval (18 tests, 6 categories) |
| v1.2.0 | Added LLM judge, synonym keyword groups, negative tests |
| v1.2.1 | Trimmed further, improved keyword coverage |
| **v1.3.0** | **Fixed LLM judge (rubric), hallucination tests (Cat 7), Architecture Decision Patterns section, publication-ready** |

## Eval Results Summary

### Haiku 4.5 (v1.3.0, 1-run with judge)

| Category | Control | Treatment | Lift |
|----------|---------|-----------|------|
| Architecture Decisions | 1.33 | 4.33 | **+225%** |
| Can Claude Do X | 2.0 | 4.67 | **+134%** |
| Implementation Guidance | 2.0 | 5.67 | **+184%** |
| Model Selection | 1.0 | 4.0 | **+300%** |
| Extension Awareness | 1.8 | 4.6 | **+156%** |
| Hallucination Detection | 1.5 | 2.75 | **+83%** |
| Negative (regression) | 5.0 | 4.67 | **-7% (negligible)** |

### Sonnet 4.5 (v1.3.0, 1-run no-judge, almost complete before timeout)

| Category | Control | Treatment | Lift |
|----------|---------|-----------|------|
| Architecture Decisions | 2.0 | 5.33 | **+167%** |
| Can Claude Do X | 3.33 | 9.33 | **+180%** |
| Implementation Guidance | 3.33 | 10.33 | **+210%** |
| Model Selection | 3.0 | 13.0 | **+333%** |
| Extension Awareness | 3.6 | 8.8 | **+144%** |
| Negative (regression) | 9.0 | 9.67 | **+7% (negligible)** |
| Hallucination Detection | 2.0 | ~5.5 | ~**+175%** |

**Key finding**: Lift holds across models. Sonnet shows comparable or stronger lifts than Haiku.

## Session 5 Priorities

This session has **three goals** that feed into each other:

### Part A: Browser Testing on claude.ai (PRIMARY)

**Why this matters**: All our evals inject SKILL.md as a user-turn message *right next to the question*. That's the strongest possible position for context influence. But in real-world usage:
- A **standalone skill** (via skills.sh) loads SKILL.md into the system prompt chain — potentially thousands of tokens before the user's actual question, with conversation turns in between
- A **plugin with a SessionStart hook** puts it even earlier — at the very start of the session

We need to validate that the skill's lift holds when context is injected at the *start* of a conversation, not right next to the question.

**Test Protocol:**

1. **Open claude.ai** in the browser
2. **Baseline test (no skill)**:
   - Start a new conversation
   - Ask: "What models does Claude offer and what are their context windows?"
   - Ask: "Should I use MCP or direct API calls for connecting Claude to my database?"
   - Ask: "Can Claude Code access my browser cookies and browsing history?"
   - Screenshot/note each response
3. **Treatment test (skill at conversation start)**:
   - Start a NEW conversation
   - First message: paste the full SKILL.md content, prefixed with "Use this as reference context for our conversation:" (or similar framing)
   - THEN ask the same 3 questions as separate follow-up messages (not in the same turn)
   - Screenshot/note each response
4. **Treatment test (skill mid-conversation, with dilution)**:
   - Start a NEW conversation
   - Have 2-3 unrelated exchanges first (e.g., "Help me write a Python function to sort a list", then "What's the difference between a list and a tuple?")
   - Then paste SKILL.md content
   - Then ask the 3 test questions
   - Screenshot/note responses — does the dilution from prior context affect quality?
5. **Compare results** across all three conditions

**What we're looking for:**
- Does Treatment (conversation start) match Treatment (right next to question) quality?
- Does signal dilute when SKILL.md is injected after other conversation context?
- Do specific categories hold up better/worse? (Architecture Decisions is our weakest; Model Selection is strongest)

**Decision this drives**: If lift holds at conversation start → standalone skill is fine. If it degrades significantly → we need a plugin with a SessionStart hook that reinforces key facts, or we need to restructure SKILL.md for better "long-distance" retrieval.

### Part B: Distill Update E2E Flow Test

**Why this matters**: We're publishing a skill that claims to provide accurate Claude capabilities. If those capabilities change and the skill isn't updated, it becomes actively harmful (confident wrong answers). We need a proven update mechanism before publishing.

**The Distill monitoring setup** (`distill-monitoring-setup.md`) tracks 8 Anthropic pages. When a change is detected, the flow should be:

1. Distill notifies of a change (email/browser notification)
2. We review the diff to identify what changed
3. We determine if the change affects SKILL.md or reference files
4. We update the relevant files
5. We bump the version
6. We re-run evals to validate no regressions
7. We push to GitHub
8. Users get the update via `npx skills add` (or auto-update in plugin)

**Mock update test:**
1. Pick a realistic mock change — e.g., "Anthropic just released Claude Opus 5.0 with 2M context window and native image generation"
2. Walk through the full update flow:
   - What in SKILL.md needs to change?
   - What reference files need updates?
   - What eval tests need updating?
   - Run evals after the update
   - Revert the mock changes afterward
3. Time the process and document pain points
4. Identify what could be automated

**Existing Distill issues**: Some monitors have SELECTION_EMPTY errors (see `.planning/prompts/distill-resolution-prompt.md`). The support.claude.com pages use Intercom which loads content dynamically via JS. Fix options: enable JS rendering in Distill, switch to local monitoring, or use `title` selector as fallback.

### Part C: Plugin Decision (Based on A & B outcomes)

After browser testing and update flow testing, decide:

| If... | Then... |
|-------|---------|
| Lift holds at conversation start AND update flow is manageable | Ship as standalone skill, defer plugin to v1.4.0 |
| Lift degrades at conversation start | Build plugin with SessionStart hook that reinforces key context |
| Update flow is painful/error-prone | Build plugin with auto-update hook (checks GitHub for new versions) |
| Both degrade | Build plugin with both hooks as priority |

Plugin structure would look like:
```
claude-capabilities-plugin/
├── plugin.json
├── skills/
│   └── claude-capabilities/
│       ├── SKILL.md
│       └── references/
├── hooks/
│   └── session-start.md        # Reinforce key capabilities context
└── agents/                     # (optional) auto-update checker
```

## Environment Notes

- **API key**: User will provide (same key used in Sessions 3-4)
- **Eval runner**: `cd .planning/builds/claude-capabilities-skill/evals && pip install anthropic && python3 eval_runner.py [args]`
- **Eval runner args**:
  - `--model claude-sonnet-4-5-20250929` (or `claude-opus-4-6`, `claude-haiku-4-5-20251001`)
  - `--runs N` (default 3)
  - `--judge` or `--no-judge`
  - `--output-dir PATH` (defaults to same directory as script)
- **Output files**: Timestamped `eval-results-{ts}.json` + `eval-report-{ts}.md` in the evals/ directory
- **Git**: May need `rm .git/index.lock` if git operations fail (mounted filesystem quirk)
- **Sandbox timeout**: 10 minutes. Run multi-run or Opus evals in Claude Code locally instead
- **Browser testing**: Use Claude in Chrome tools to interact with claude.ai — navigate, type prompts, screenshot responses
- **SKILL.md location for pasting**: `.planning/builds/claude-capabilities-skill/claude-capabilities/SKILL.md`

## Git History (skill-related commits)

```
4277fb1 feat(capabilities-skill): v1.3.0 publication-ready directory
c436e19 feat(capabilities-skill): v1.3.0 — fixed judge, hallucination tests, architecture patterns
1c7d9de feat(capabilities-skill): add v1.2.1 eval results and analysis
fcbb4cb feat(capabilities-skill): v1.2.1 — trim SKILL.md, upgrade eval framework
53138c2 feat: Phase 0 scaffolding — monorepo, frontend, backend, Docker, CI
```

## Suggested Workflow

### Step 1 — Read key files
Read `SKILL.md`, `eval_runner.py` (just the test definitions), and `distill-monitoring-setup.md` to build context.

### Step 2 — Browser Testing (Part A)
Follow the test protocol above. This is the most important task — it directly informs whether we need a plugin.

### Step 3 — Analyze browser test results
Compare baseline vs treatment-at-start vs treatment-after-dilution. Document findings.

### Step 4 — Distill E2E Mock Update (Part B)
Run the mock update flow. Time it. Document pain points and automation opportunities.

### Step 5 — Plugin Decision (Part C)
Based on A & B results, decide: standalone skill (ship now) vs plugin (build first).

### Step 6 — GitHub Validation
If repo has been created:
- `npx skills-ref validate ./claude-capabilities` (if skills-ref CLI is available)
- Test installation: `npx skills add liamjons/claude-capabilities`

### Step 7 — Commit results
Commit browser test notes, update flow documentation, and any code changes.

## Key Question to Answer

**Does the skill's effectiveness depend on WHERE in the context window it appears?**

If yes → plugin with SessionStart hook (reinforces context).
If no → standalone skill via skills.sh (simpler, ship now).

This is the single most important question for Session 5.
