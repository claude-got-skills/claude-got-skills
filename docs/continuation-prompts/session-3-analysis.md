# Session 3 Analysis: Claude Capabilities Skill

**Date:** 2026-02-10
**Skill version analysed:** v1.2.0
**Author:** Claude (session 3)

---

## 1. Line Budget & Reduction Analysis

### Current Token Costs

| File | Lines | Est. Tokens | Load Behaviour |
|------|-------|-------------|----------------|
| SKILL.md | 344 | ~4,630 | **Always loaded** when skill triggers |
| agent-capabilities.md | 451 | ~2,942 | On-demand (Claude reads when needed) |
| api-features.md | 411 | ~2,869 | On-demand |
| claude-code-specifics.md | 386 | ~3,020 | On-demand |
| model-specifics.md | 312 | ~2,719 | On-demand |
| tool-types.md | 409 | ~2,674 | On-demand |
| **Total** | **2,313** | **~18,854** | — |

**Key finding:** SKILL.md costs ~4,630 tokens per request where the skill triggers. That's roughly 2.3% of a 200K context window — low enough that aggressive reduction isn't critical. The question is whether every section earns its keep.

### Section-by-Section Value Assessment

| Section | Lines | ~Tokens | Eval Impact | Recommendation |
|---------|-------|---------|-------------|----------------|
| Frontmatter + header | 35 | 474 | N/A (metadata) | **Keep** — required |
| Current Models | 13 | 185 | High (Test 4.1: +5 accuracy) | **Keep** — compact, high hit-rate |
| Thinking & Reasoning | 12 | 131 | High (Test 3.1: +6 accuracy) | **Keep** — critical for Opus 4.6 users |
| Context & Memory | 22 | 235 | High (Test 2.1: +5 accuracy, +5 completeness) | **Keep** — memory tool is a blind spot without it |
| Tools & Integration | 25 | 287 | Medium (Tests 2.2, 2.3) | **Keep** — tool search + MCP connector are novel |
| Output & Structure | 18 | 211 | High (Test 3.3: +3 accuracy, +5 completeness) | **Keep** — structured outputs guidance is essential |
| Agent Capabilities | 21 | 265 | Medium (Test 1.2) | **Trim** — 40% overlap with agent-capabilities.md |
| Platform | 10 | 83 | Low (no direct test) | **Keep** — tiny cost, unique content |
| Tool Use Best Practices | 14 | 189 | Low (no direct test) | **Move** to tool-types.md reference |
| Claude Code Capabilities | 39 | 621 | Medium (Test 5.3, 5.5) | **Trim** — heavy overlap with claude-code-specifics.md |
| Choosing Extension Pattern | 32 | 535 | High (Test 5.3: +4 accuracy, +3 completeness) | **Keep** — unique decision framework |
| When to Suggest Extensions | 40 | 568 | High (Test 5.1, behavioural change) | **Keep** — this is the "art of the possible" section |
| Skills Ecosystem Awareness | 25 | 361 | Medium (Test 5.4, 5.5) | **Keep but refine** — ecosystem claims need accuracy fix |
| Breaking Changes | 12 | 135 | High (Test 3.2: +5 accuracy) | **Keep** — critical for migrations |
| Reference Files | 27 | 341 | N/A (navigation) | **Keep** — directs Claude to detailed docs |

### Proposed Reductions

**Move to references (saves ~810 tokens from always-loaded):**

1. **Tool Use Best Practices (14 lines, ~189 tokens)** → Merge into `tool-types.md`
   - Rationale: No eval test directly targets this section. The content is implementation-level guidance that Claude needs when configuring tools, not when making architectural decisions.
   - Risk: Low. Tool-related questions will trigger a reference read anyway.

2. **Claude Code Capabilities — partial trim (reduce from 39 to ~18 lines, save ~310 tokens)**
   - **Keep in SKILL.md**: The extension system hierarchy (lines 198-202) and skills system summary (lines 203-208). These are unique framing that supports the extension decision sections below them.
   - **Move to claude-code-specifics.md**: Agent teams details (lines 176-180), Chrome browser details (lines 182-187), IDE extensions (lines 189-191), CLI key features (lines 193-196). These are pure implementation detail that duplicates the reference.
   - Risk: Medium. Tests 5.3 and 5.5 benefit from agent teams and extension system being always-loaded. But the extension system hierarchy stays, and the decision framework below it is what actually drives the eval lift.

3. **Agent Capabilities — trim (reduce from 21 to ~12 lines, save ~120 tokens)**
   - **Keep**: Agent SDK one-liner, Hooks lifecycle events list, Plugins one-liner, MCP Apps one-liner.
   - **Move**: Subagent details (built-in types, Markdown files with YAML frontmatter), SDK core API details (query(), ClaudeSDKClient). These are implementation details covered by agent-capabilities.md.
   - Risk: Low. The decision framework section ("Choosing the Right Extension Pattern") already captures when to use subagents vs other patterns.

**Estimated result after trimming:**
- SKILL.md: ~300 lines, ~3,820 tokens (17% reduction)
- Savings: ~810 tokens per request where skill triggers

### Verdict on Line Budget

344 lines / ~4,630 tokens is defensible. The skill's +318% accuracy improvement at ~4,630 tokens of overhead is excellent ROI. However, trimming to ~300 lines / ~3,820 tokens would remove pure duplication without sacrificing the sections that actually drive eval improvements. The high-value sections (extension decision framework, proactive suggestions, breaking changes, model matrix) are compact and should stay.

**Recommendation: Do the trim, but don't obsess over it.** The real efficiency question is trigger precision — how often does the skill fire when it shouldn't? A false trigger costs 4,630 wasted tokens. That matters more than shaving 800 tokens from the content.

---

## 2. Ecosystem Accuracy Verification

### skills.sh ✅ CONFIRMED

[skills.sh](https://skills.sh/) exists and functions as described. It's "The Agent Skills Directory" — a public registry for discovering and installing skills for AI agents. Skills are indexed from various GitHub repositories (including anthropics/claude-code, antfu/skills, vercel-labs). Install with a single command.

**SKILL.md accuracy:** The description is correct. No changes needed.

### find-skills ⚠️ NEEDS CORRECTION

The SKILL.md says: *"There is also a `find-skills` skill that searches the skills.sh registry directly from a Claude Code session."*

**Reality:** `find-skills` is part of the [Vercel Labs Skills CLI](https://github.com/vercel-labs/skills), not a Claude Code skill per se. The command is `npx skills find [query]`, which searches the skills.sh registry from the terminal. It's available on [skillstore.io](https://skillstore.io/skills/vercel-labs-find-skills) as a skill that can be installed, but calling it a "meta-skill that searches the registry directly from within a Claude Code session" is misleading.

**Recommended fix:** Change lines 292-293 to:
```
**Skills CLI**: Use `npx skills find [query]` to search the skills.sh registry from the
terminal. Also available as an installable skill for use within Claude Code sessions.
```

### agentskills.io ✅ CONFIRMED

The Agent Skills open standard is published at agentskills.io. Multiple registries reference it.

### Plugin marketplaces ✅ CONFIRMED

Multiple marketplaces exist: SkillsMP (160,000+ skills), skillsdirectory.com, claude-plugins.dev. The concept of organisations hosting private marketplaces of curated plugins is accurate.

### Additional ecosystem resources NOT currently mentioned

The SKILL.md could benefit from mentioning:
- **[Anthropic's official skills repo](https://github.com/anthropics/skills)** — curated example skills
- **Multiple community registries** — SkillsMP, skillsdirectory.com, claude-plugins.dev
- **`npx skills init`** — CLI command to scaffold new skills
- **`npx skills add`** — CLI command to install skills (e.g., `npx skills add vercel-labs/agent-skills@vercel-react-best-practices`)

Whether to add these depends on whether the goal is awareness or action. For a skill about Claude's capabilities, awareness is enough — the user can explore from there.

---

## 3. Content Overlap Analysis

### Pure Duplications (safe to consolidate)

| SKILL.md Content | Reference File | Overlap |
|-----------------|----------------|---------|
| Agent teams details (lines 176-180) | claude-code-specifics.md §Agent Teams | 90%+ identical |
| Chrome browser details (lines 182-187) | claude-code-specifics.md §Chrome | 90%+ identical |
| IDE extensions (lines 189-191) | claude-code-specifics.md §IDE Extensions | Fully duplicated |
| CLI key features (lines 193-196) | claude-code-specifics.md §CLI Reference | Subset of reference |
| Hooks event list (line 137-138) | agent-capabilities.md §Hooks | Identical list |
| Plugin summary (lines 141-142) | agent-capabilities.md §Plugins | Identical concept |
| Tool definition quality (line 163-165) | claude-code-specifics.md §Tool Definition Quality | Identical wording |
| Tool choice control (lines 167-169) | api-features.md + claude-code-specifics.md | Identical |

### SKILL.md Unique Content (DO NOT move)

| Content | Why it's unique |
|---------|----------------|
| Extension decision matrix (lines 216-224) | Adds "Why" column not in any reference |
| Key distinctions (lines 226-237) | Not in any reference — clarifies common confusions |
| Context costs (lines 238-240) | Not in any reference — token awareness guidance |
| When to Suggest Extensions (lines 242-280) | Entirely unique — behavioural change section |
| Skills Ecosystem Awareness (lines 282-305) | Entirely unique — ecosystem awareness |
| Extension system hierarchy (lines 198-202) | Unique layered framing not in reference |

---

## 4. Eval Framework Refinements

### 4.1 Current Eval Limitations

The capabilities-skill-eval.py script uses the Agent SDK's `query()` function, which means:
- **Control vs Treatment differentiation is unclear**: The script doesn't show how the treatment condition gets the skill installed. It calls the same `query()` function for both. The prompt mentions that a separate `eval_runner.py` was used for the v1.2.0 results (using the Anthropic Messages API directly with httpx). This script seems to be the original v1.0.0 design that wasn't actually used.
- **No multi-run support in the runner that produced results**: The v1.2.0 results are single-run.

### 4.2 Multi-Run Variance

**Problem:** Single-run results are noisy. A keyword match of 4/6 could be 2/6 on the next run.

**Recommendation:** Add a `--runs N` flag to the actual eval runner (not this script, which already has it but wasn't used). Run 3 times minimum. Report mean and standard deviation per test.

**Implementation sketch:**
```python
# After collecting N runs per test:
for test_id in tests:
    ctrl_scores = [run["control_accuracy"] for run in runs[test_id]]
    trt_scores = [run["treatment_accuracy"] for run in runs[test_id]]
    print(f"Test {test_id}: ctrl={mean(ctrl_scores):.1f}±{stdev(ctrl_scores):.1f}, "
          f"trt={mean(trt_scores):.1f}±{stdev(trt_scores):.1f}")
```

### 4.3 LLM-as-Judge Scoring

**Problem:** Keyword scoring can't distinguish "budget_tokens is deprecated" (correct) from "use budget_tokens" (incorrect). Both match the keyword "budget_tokens".

**Recommendation:** Add an LLM-judge step that scores each response on a 0-3 rubric per dimension. Use a strong model (Sonnet or Opus) with the existing rubric from the eval script.

**Design:**
```
For each (test, response) pair:
  1. Send to judge model with:
     - The original prompt
     - The response text
     - The expected_advantage description
     - The scoring rubric (Accuracy 0-3, Completeness 0-3, Deprecated 0-1, Actionability 0-2)
  2. Judge returns structured JSON: {accuracy: 2, completeness: 3, deprecated: 1, actionability: 2}
  3. Aggregate across runs
```

**Key benefit:** This catches false positives (mentioning a keyword in the wrong context) and rewards responses that are correct but use different terminology than the keyword list.

**Cost estimate:** 15 tests × 2 conditions × 3 runs = 90 judge calls. At ~500 tokens per call with Haiku as judge, that's ~45K tokens (~$0.05). With Sonnet as judge, ~$0.30. Cheap enough to run routinely.

### 4.4 Test 5.4 (PowerPoint) Analysis

**The problem:** Both control and treatment scored 0/5 on completeness. The treatment scored 4/6 on accuracy (matched: skill, pptx, presentation, document) but missed the completeness keywords: skills.sh, find-skills, npx skills, discover, install.

**Why it failed:** The treatment response correctly identified document generation skills and even recommended them, but framed it as "Built-in Document Skills" rather than directing the user to skills.sh to find/install the pptx skill. The skill content mentions skills.sh in the "When to Suggest Extensions" section, but the model didn't connect "creating a PowerPoint" with "suggest finding an existing skill" — it went straight to "you have built-in skills for this."

**Root cause:** The proactive suggestion triggers in SKILL.md say *"Suggest finding an existing skill when: the task matches a common pattern (document generation, code review, deployment)."* But the model interpreted the pptx skill as already available (it's mentioned in "Pre-built document skills" section), so it didn't suggest discovering it externally.

**Possible fixes:**
1. **Keyword adjustment:** The test's completeness keywords may be too specific. "skills.sh" and "npx skills" are discovery mechanisms, but if the skill is already installed, suggesting discovery is wrong. The treatment response was actually *more correct* than the keywords expected.
2. **Add a clarifying note:** In the "Pre-built document skills" section, add: *"If these skills aren't installed, the user can find them on skills.sh or install via `npx skills add`."*
3. **Rethink the test:** Test 5.4 might be testing the wrong thing. A better completeness test would check whether the response mentions the skill's capabilities (formatting, templates, best practices) rather than the discovery mechanism.

**Recommendation:** Adjust the Test 5.4 completeness keywords to: `["formatting", "template", "best practices", "skill", "install"]` — this tests whether the model knows the skill handles quality concerns, not just that discovery mechanisms exist.

### 4.5 Negative Tests (Regression Detection)

**Problem:** No tests check whether the skill degrades performance on non-capabilities questions.

**Recommended negative test prompts:**
1. "Write a Python function that sorts a list of dictionaries by a specific key." (Should produce identical responses with/without skill)
2. "Explain the difference between TCP and UDP." (General knowledge — skill shouldn't interfere)
3. "Help me debug this React component that's not rendering." (Code task — skill might add noise)

**Scoring:** For negative tests, score similarity between control and treatment. A delta > 10% of response length indicates regression.

### 4.6 Model-Specific Eval Runs

| Model | Expected Outcome | Priority |
|-------|-----------------|----------|
| Haiku 4.5 | Current baseline — strongest lift expected (weakest training knowledge) | Done ✅ |
| Sonnet 4.5 | Should still benefit but smaller delta (better training knowledge) | High |
| Opus 4.6 | Smallest delta expected (strongest model). Tests whether skill adds noise | Medium |

Running Sonnet would validate that the skill helps across model tiers. Running Opus would test the ceiling — if Opus already knows everything in the skill, the overhead isn't justified for Opus-targeted use cases.

---

## 5. Specific Recommendations (Prioritised)

### Do Now (v1.2.1 patch)

1. **Fix find-skills description** — Change from "meta-skill that searches the registry" to "CLI tool (`npx skills find [query]`) that searches skills.sh; also available as an installable skill"
2. **Trim Claude Code Capabilities section** — Move agent teams, Chrome, IDE, CLI details to reference. Keep extension hierarchy and skills system. Saves ~310 tokens.
3. **Trim Agent Capabilities section** — Remove subagent implementation details. Keep the one-liner summaries. Saves ~120 tokens.
4. **Move Tool Use Best Practices to reference** — Merge into tool-types.md. Saves ~189 tokens.

### Do Next (v1.3.0)

5. **Add multi-run support to the actual eval runner** — Run 3x minimum, report mean ± stdev.
6. **Build LLM-as-judge scorer** — Even a simple Haiku-based judge that scores 0-3 per dimension would catch false positives.
7. **Adjust Test 5.4 keywords** — The completeness keywords test for discovery mechanisms when the response correctly identified available skills. Fix the test, not the skill.
8. **Add 3 negative test prompts** — Verify the skill doesn't degrade non-capabilities responses.
9. **Run Sonnet eval** — Validate cross-model benefit.

### Do Later (v1.4.0+)

10. **Build capability-matrix.json** — Machine-readable source of truth for automated diffing.
11. **Build update pipeline** — fetch-docs.sh + diff-capabilities.sh + auto-eval after updates.
12. **Investigate trigger precision** — How often does the skill fire on non-capabilities prompts? False triggers waste more tokens than a slightly oversized SKILL.md.
13. **Consider skill splitting** — If the skill grows beyond ~500 lines, split into a "core" skill (models, breaking changes, extension framework) and a "reference" skill (implementation details). But this is premature at 344 lines.

---

## 6. The eval_runner.py Question

The prompt mentions `eval_runner.py` at the project root as the runner that produced the v1.2.0 results (using Anthropic Messages API with httpx SSL fix). This file doesn't exist at that location — only `capabilities-skill-eval.py` exists in the evals/ directory. The v1.2.0 results were likely produced by a different script or an interactive session. To make evals reproducible, the actual runner should be committed to the repo.

---

## Appendix: Token Budget Context

For perspective on whether ~4,630 tokens is "expensive":

- Haiku 4.5 input cost: ~$0.0046 per request (at $1/M input tokens)
- Sonnet 4.5 input cost: ~$0.014 per request (at $3/M input tokens)
- Opus 4.6 input cost: ~$0.069 per request (at $15/M input tokens)

At 100 requests/day where the skill triggers, that's $0.46/day on Haiku, $1.40/day on Sonnet, $6.90/day on Opus. The accuracy improvement easily justifies this for capability-related queries. The concern would be if the skill triggers on 1,000+ non-capability requests — hence the trigger precision recommendation above.
