# Claude Capabilities Skill — Continuation Prompt

Paste the below into a new session. It contains full context for continuing work on the capabilities skill after two prior sessions.

---

## Prompt

I'm continuing work on the **Claude Capabilities Awareness Skill** — a skill that gives Claude accurate post-training knowledge about its own capabilities, extension patterns, and the skills ecosystem. We've completed two sessions so far and the skill is producing strong eval results. I need your help with refinement, gap analysis, and next steps.

### Project Location

Everything is in my mounted folder under `.planning/builds/claude-capabilities-skill/`:

```
claude-capabilities/
├── SKILL.md                          (344 lines, v1.2.0 — the core skill file)
└── references/
    ├── api-features.md               (411 lines — API calls, headers, code examples)
    ├── tool-types.md                 (409 lines — built-in tool configs)
    ├── agent-capabilities.md         (451 lines — Agent SDK, subagents, hooks, plugins)
    ├── model-specifics.md            (312 lines — per-model matrix, pricing, migrations)
    └── claude-code-specifics.md      (386 lines — agent teams, Chrome, CLI, skills system)

evals/
├── capabilities-skill-eval.py        (original 10-prompt eval script, not used now)
├── README.md                         (eval design docs)
├── eval-results-20260210-221143.json (v1.1.0 baseline, 10 tests)
├── eval-results-20260210-222404.json (v1.2.0 results, 15 tests — CURRENT)
├── eval-report-20260210-221143.md    (v1.1.0 human-readable report)
└── eval-report-20260210-222404.md    (v1.2.0 human-readable report — CURRENT)
```

The eval runner script is at `evals/eval_runner.py` (uses Anthropic Messages API directly with httpx SSL fix for sandbox proxy). Note: you'll need `ANTHROPIC_API_KEY` set and `pip install anthropic httpx[socks]`.

Design docs: `.planning/designs/claude-capabilities-awareness-skill-design.md` (Phase 1-3 design), `.planning/designs/capabilities-pipeline-phase2-requirements.md` (Phase 2 requirements including Distill fix documentation).

### What's Been Done

**Session 1 (v1.0.0 → v1.1.0):**
- Built the initial skill from 7 research files (claude-capabilities-* and claude-code-capabilities-*)
- Added Tool Use Best Practices and Claude Code Capabilities sections to SKILL.md
- Created `references/claude-code-specifics.md` (new reference file)
- Built eval script with 10 blind test prompts across 4 categories
- Created Phase 2 requirements document

**Session 2 (v1.1.0 → v1.2.0):**
- Ran baseline evals: v1.1.0 showed +28 accuracy keywords, +20 completeness (10 tests, Haiku 4.5)
- Added three new sections to SKILL.md based on user feedback:
  1. **Choosing the Right Extension Pattern** — decision framework table + key distinctions
  2. **When to Suggest Extensions (Proactive Guidance)** — behavioural triggers for when Claude should suggest building/finding skills instead of just completing the task
  3. **Skills Ecosystem Awareness** — skills.sh, find-skills, plugin marketplaces, pre-built document skills
- Added 5 new eval prompts (category 5: Extension Awareness) testing meta-level behaviour
- Ran v1.2.0 evals (15 tests): **+54 accuracy keywords (+318%), +27 completeness (+245%)**
- Updated frontmatter triggers and description to cover extension patterns

### Key Eval Results (v1.2.0)

| Metric | Control | Treatment | Delta |
|--------|---------|-----------|-------|
| Accuracy keywords | 17/96 (18%) | 71/96 (74%) | +54 (+318%) |
| Completeness keywords | 11/82 (13%) | 38/82 (46%) | +27 (+245%) |
| Deprecated patterns | 1 | 2 | +1 (noise) |
| Token overhead | 8,150 | 81,570 | +73,420 |

Biggest lifts: Test 3.2 (prefill migration, ctrl 1/6 → trt 6/6), Test 3.1 (thinking config, 1/7 → 7/7), Test 2.1 (memory tool, 0/5 → 5/5), Test 5.5 (plugin distribution, 0/6 → 6/6), Test 5.3 (CLAUDE.md vs skill, 2/6 → 6/6).

### What Needs Doing Next

#### 1. Line Budget & Reduction Analysis (Priority)

The 500-line SKILL.md limit was a made-up figure, not a hard constraint. The real constraint is: what's the optimal amount of always-loaded context that maximises Claude's accuracy without wasting tokens or adding noise? Current state:

- SKILL.md: 344 lines (always loaded when skill triggers)
- References: 1,969 lines total (loaded on-demand when Claude reads them)

Questions to answer:
- **Is 344 lines too much for SKILL.md?** Some content might be better moved to reference files, reducing always-loaded cost. Candidates for demotion: the Claude Code Capabilities section (37 lines) could arguably move entirely to `claude-code-specifics.md`. The Tool Use Best Practices section (13 lines) could move to `tool-types.md`.
- **Is anything redundant?** The extension decision framework in SKILL.md partially overlaps with content in `claude-code-specifics.md`. Is the duplication worth the always-loaded benefit?
- **What's the actual token cost?** Run a token count on SKILL.md to understand the per-request overhead. Then compare accuracy gains vs token cost to find the efficient frontier.
- **Could any reference files be consolidated or trimmed?** `agent-capabilities.md` (451 lines) is the largest. Does it contain content that's now covered by `claude-code-specifics.md`?

#### 2. Eval Refinements

- **Run multiple times for variance**: Current results are single-run. Run 3-5x to measure consistency. The eval_runner.py doesn't currently support multiple runs natively — it would need a `--runs` flag.
- **Try Sonnet as eval model**: Haiku results show the skill works for a weaker model. Does Sonnet still benefit, or does its stronger training knowledge reduce the delta?
- **Keyword scoring limitations**: The scorer can't distinguish "budget_tokens is deprecated" (correct) from "use budget_tokens" (incorrect). Consider adding a simple LLM-as-judge scoring step, where a second model scores each response against the rubric. This would give more nuanced accuracy/completeness/actionability scores.
- **Test 5.4 (PowerPoint) weakness**: Both conditions scored 0/5 on completeness — neither mentioned skills.sh or find-skills for discovering the pptx skill. The skill content mentions these but the model didn't surface them for this specific prompt. May need stronger wording in the ecosystem section, or the keyword set may need adjustment.
- **Add negative tests**: Prompts where the skill should NOT change the answer (e.g., "write me a Python function to sort a list"). Verify the skill doesn't cause regression on non-capabilities questions.

#### 3. Proactive Suggestion Tuning

The "When to Suggest Extensions" section works (test 5.1 showed the proposal-skill suggestion) but could be stronger:
- Test with real-world prompts from non-technical users to see if the suggestions feel natural vs preachy
- Consider whether the "How to raise it" examples are sufficient or need expansion
- The signal phrases ("I always...", "every time I...") could be tested more systematically

#### 4. Ecosystem Accuracy Verification

I wrote the skills.sh and find-skills content based on the user's description. This should be verified:
- Does skills.sh actually exist and work as described? Web search to confirm.
- Is find-skills an actual published skill? Verify and get the exact install command.
- What other ecosystem resources exist that the skill should mention?

#### 5. Phase 2 Pipeline Items (Lower Priority)

These are documented in `.planning/designs/capabilities-pipeline-phase2-requirements.md`:
- Build `capability-matrix.json` (machine-readable source of truth for diffing)
- Build `/update-claude-capabilities` command
- Build `fetch-docs.sh` and `diff-capabilities.sh` scripts
- Close the source coverage gap (Distill monitors vs fetch script watch different pages)
- Integrate eval framework into the update pipeline (auto-run evals after updates)

#### 6. Distill Monitor Status

A separate session was started to resolve SELECTION_EMPTY errors on 4-5 Distill monitors. The prompt is at `.planning/prompts/distill-resolution-prompt.md`. Check whether this was completed successfully. If not, the fixes are documented in the Phase 2 requirements doc Section 2.

### Research Files (Source Material)

The 7 research files used to build the skill are in the user's uploads or mounted folder. They follow a naming convention:
- `claude-capabilities-implement-tool-use.md` — General tool use (66KB)
- `claude-code-capabilities-agent-teams.md` — Agent teams (22KB)
- `claude-code-capabilities-cli-reference.md` — CLI commands/flags (26KB)
- `claude-code-capabilities-create-plugins.md` — Plugin authoring (17KB)
- `claude-code-capabilities-extension-options.md` — Extension decision framework, the official "Extend Claude Code" docs page (11KB)
- `claude-code-capabilities-skills.md` — Skill authoring (33KB)
- `claude-code-capabilities-use-chrome-browser.md` — Browser automation (11KB)

The `extension-options.md` file is particularly important — it's the official Anthropic documentation for the extension decision framework, and the SKILL.md section was derived from it.

### Context on User's Goals

The user (Liam) is building this skill to solve a specific problem: Claude's training data doesn't include post-training capabilities, so it gives outdated or wrong answers about what Claude can do. The skill closes that gap.

But the user's bigger insight (from session 2) is that the skill's highest value isn't just factual accuracy — it's changing Claude's behaviour to think at the meta level. Instead of just completing a task, Claude should recognise when suggesting a skill, hook, or plugin would serve the user better long-term. This is especially valuable for non-technical users who don't know the extension ecosystem exists. The user described this as helping users see the "art of the possible."

The eval results validate this: the extension awareness tests (category 5) showed dramatic improvements, with control responses saying things like "there's no built-in way to share Claude configuration" while treatment responses correctly identified plugins as the solution.

### How to Start

1. Read SKILL.md to understand current state
2. Read the latest eval report (`eval-report-20260210-222404.md`) to see actual control vs treatment responses
3. Start with the line budget / reduction analysis — this is the most impactful next step since it determines the skill's efficiency
4. Then move to eval refinements (multi-run, Sonnet, Opus, LLM-judge)
5. Verify ecosystem claims (skills.sh, find-skills)
