# Session 15 Handoff: Content Update, Pipeline Sprint 2, and v2.4.0 Release

## Project Context

We are building and maintaining Claude skills under the **claude-got-skills** GitHub account. The primary skill — **Claude Capabilities Awareness** — gives Claude comprehensive, accurate knowledge about its capabilities across all platforms (Claude.ai, Desktop, Claude Code, CoWork, API).

**GitHub repo**: https://github.com/claude-got-skills/skills
**Private dev repo**: https://github.com/liam-jons/skills-dev
**Local folder**: `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/` — this is the git repo root.
**skills.sh listing**: https://skills.sh/claude-got-skills/skills/assistant-capabilities

---

## Session 14 Completed Work

### 1. 3-Model × 55-Test Baselines ✅

All three models evaluated on full 55-test suite with old SKILL.md-based judge. Keyword accuracy results:

| Category | Haiku Ctrl→Trt (Lift) | Sonnet Ctrl→Trt (Lift) | Opus Ctrl→Trt (Lift) |
|----------|----------------------|------------------------|---------------------|
| Architecture | 1.33→3.67 (+176%) | 1.33→3.00 (+125%) | 1.00→2.67 (+167%) |
| Can Claude Do X | 1.80→4.20 (+133%) | 2.40→4.80 (+100%) | 3.00→4.60 (+53%) |
| Implementation | 2.25→4.62 (+105%) | 2.62→4.75 (+81%) | 3.12→4.62 (+48%) |
| Model Selection | 2.00→5.00 (+150%) | 0.00→5.00 (+∞) | 1.00→6.00 (+500%) |
| Extension | 1.62→4.12 (+154%) | 2.12→4.38 (+107%) | 2.50→4.25 (+70%) |
| Hallucination | 1.20→3.00 (+150%) | 1.20→2.80 (+133%) | 2.00→2.60 (+30%) |
| Cross-Platform | 2.00→4.10 (+105%) | 2.60→4.10 (+58%) | 2.90→4.30 (+48%) |
| Negative | 5.00→5.00 (0%) | 4.67→4.33 (-7%) | 5.00→5.00 (0%) |
| Cat 9: Conversational | 2.33→3.75 (+61%) | 2.50→3.58 (+43%) | 3.00→3.92 (+31%) |

Reports at:
- Haiku: `evals/eval-report-20260310-193001.md`
- Sonnet: `evals/eval-report-20260310-200219.md`
- Opus: `evals/eval-report-20260310-173601.md` (session 13, 43 tests for cats 1-8)

### 2. Independent Rubric System ✅ (Circular Bias Confirmed)

Built `evals/rubrics.py` with 213 factual claims across 55 tests, sourced from knowledge-base docs (NOT SKILL.md). Modified `judge_score()` to use per-test rubrics with SKILL.md fallback.

**Key finding:** The old judge inflated treatment scores by ~0.7-1.0 points on average. With the independent rubric:
- Treatment accuracy dropped from ~3.0 to ~2.0 across the board
- Control accuracy slightly increased (esp. hallucination: 1.5→2.0)
- **The lift pattern still holds** — skill genuinely helps, but by less than old judge claimed
- Category 9 (Conversational) barely affected (-0.09) — most honest category
- Haiku as judge has high N/A rate — **use Sonnet as judge** in future runs

Rubric-validated reports:
- Opus: `evals/eval-report-20260310-204847.md`
- Sonnet: `evals/eval-report-20260310-212132.md`
- Haiku: `evals/eval-report-20260310-214031.md`

### 3. Browser Eval Fixes ✅

- Added `--headed` flag (visible browser for debugging eval runs)
- Added `--yes` flag (skip interactive confirmations for unattended runs)
- Fixed input field grep pattern (added "Write your prompt to Claude")
- Improved response extraction (snapshot saving, broader CSS selectors)
- **Still needs**: actual headed test run to capture real response detection selectors

### 4. Doc Gap Analysis ✅ (Significant New Features Missing)

Scraped current docs and found features missing from our skill/plugin:

| Feature | Status in Skill |
|---------|----------------|
| **Code Review** (managed PR service, REVIEW.md, severity levels, $15-25/review) | MISSING |
| **Remote Control** (`claude remote-control`, continue from phone/browser) | CLI ref only, no detail |
| **claude.ai/code** (web-based Claude Code sessions) | MISSING |
| **Slack integration** (@Claude → PRs) | MISSING |
| **Desktop app** (standalone, not Claude Desktop) | MISSING |
| **Excel/PowerPoint add-ins** (cross-app context, March 11 update) | MISSING |
| **JetBrains plugin** (IntelliJ, PyCharm, WebStorm) | Brief mention only |
| `/teleport`, `/desktop` commands | MISSING |
| Chrome browser integration | In claude-code-specifics.md ✅ |

**Docs structure note**: `code.claude.com/docs/` is a separate docs site for Claude Code. `platform.claude.com/docs/` is the API docs site. Both coexist — the docs did NOT move.

### 5. Pipeline Source Gaps ✅

Current `pipeline/config.py` monitors 12 sources. Missing pages to add:

```python
# Missing from SOURCES list — add to pipeline/config.py:
# code.claude.com/docs/en/:
#   code-review, remote-control, slack, desktop, desktop-quickstart,
#   claude-code-on-the-web, jetbrains, vs-code, github-actions, gitlab-ci-cd
# platform.claude.com/docs/en/:
#   about-claude/pricing (full pricing details)
# Product changelog:
#   anthropic.com/news or similar (for Excel/PowerPoint add-in updates)
```

### 6. Eval Question Audit ✅

Full audit at `/private/tmp/claude-501/eval-question-audit.md`. Summary:

**9 questions with outdated assumptions:**
- Q3.2: Only tests prefill, should test all 5 breaking changes
- Q5.4: Misses Agent Skills (pptx/xlsx via API)
- Q5.8: Missing CoWork as 4th platform
- Q7.5: Sonnet 3.7 now definitively retired (Feb 2026)
- Q8.1: Missing Remote Control as bridge for Desktop users
- Q8.5: Missing Chrome, background tasks, worktrees
- Q8.10: Missing claude.ai/code web sessions
- Q2.3: Missing Chrome integration for browser automation
- Q9.6: Missing Agent Skills and Office add-ins

**10 new questions proposed:**
1. Code Review managed service
2. Remote Control
3. claude.ai/code web sessions
4. Cross-device workflows (/teleport, /desktop, /rc)
5. Chrome integration (browser automation)
6. Slack integration
7. Desktop app vs Claude Desktop distinction
8. Excel/PowerPoint add-ins with cross-app context
9. JetBrains plugin awareness
10. Worktrees for parallel sessions

**5 merge/restructure candidates:**
- Q5.4 + Q9.6 (both test document generation)
- Q5.3 + Q8.3 (both test persistent coding standards)
- Q4.1: Expand to 3 tests or merge into Cat 3
- Q8.4 + Q8.8 (both test cross-platform mapping)
- Recommended target: ~62 tests across 10 categories

### 7. IMS Review ✅ (New Eval Questions from Real User Patterns)

Searched the IMS (2200+ content items from UK SME). Key findings:

**Best new eval question ideas from IMS patterns:**
1. Bid/tender automation — users don't realize Claude can process Q&A libraries via Projects + long context
2. Competitor migration (ChatGPT → Claude) — users carry wrong assumptions about Code Interpreter equivalence
3. "Can Claude replace our tool?" — tests nuanced over/under-promising about SaaS replacement
4. Multi-file cross-reference analysis — extends Q9.1 to harder multi-document scenario
5. Team access and data isolation — users don't know about Team/Enterprise plans
6. Small team AI adoption guide — practical starting point for non-technical 10-person companies

**Recommended new category**: "Competitor Migration / Feature Comparison" — no existing tests cover ChatGPT/Copilot/Gemini comparison questions.

---

## Current Git State

```
Branch: dev (3 commits ahead of main)
Tag: v2.3.0 on main

Commits on dev not on main:
b61041c Fix browser eval auth: use manual ENTER confirmation instead of URL detection
20c8981 Fix incorrect model IDs: dated 4.6 variants don't exist on API
eb41f42 Fix quick-reference gaps and add SMB-based conversational eval tests

Uncommitted changes on dev:
- evals/rubrics.py (NEW — independent judge rubrics)
- evals/eval_runner.py (rubric import + judge_score update)
- evals/browser_eval.sh (--headed, --yes, input/extraction fixes)
- data/quick-reference.md (no net change — tier fix reverted)
- docs/eval-questions.md (NEW — all 55 questions for review)
```

---

## Session 15 Priority Order

### Phase 1: Data Accuracy (blocks everything else) (~3-4 hrs)

#### 1a. Add Missing Sources to Pipeline Config (~30 min)
Add 11+ new URLs to `pipeline/config.py` SOURCES list. Create corresponding KB file mappings.

#### 1b. Run Full Scrape (~30 min)
Run one-off scrape of all sources (existing + new) to capture current state. This creates/updates KB files.

#### 1c. Create New KB Files (~1 hr)
For genuinely new pages (Code Review, Remote Control, Slack, etc.), create KB files from scraped content.

#### 1d. Update Skill Content (~1.5 hrs)
Update these files with new feature coverage:
- `data/quick-reference.md` — add Code Review, Remote Control, claude.ai/code
- `skills/assistant-capabilities/SKILL.md` — platform availability table, extension patterns
- `references/claude-code-specifics.md` — Code Review, Remote Control, Chrome (expand), Slack, Desktop app, JetBrains, /teleport, /desktop
- `references/api-features.md` — Agent Skills expansion, Office add-ins
- `references/model-specifics.md` — training data cutoffs (Opus: Aug 2025, Sonnet: Jan 2026, Haiku: Jul 2025)

### Phase 2: Sprint 2 — Classification Pipeline (~2-3 hrs)

Spec at `docs/automation-pipeline-design.md` lines 1626-1637:

1. **Implement `pipeline/classify.py`** (1.5 hrs) — Haiku classification prompt, JSON parsing
2. **Add `--classify` flag** to `pipeline/freshness_check.py`
3. **Update `pipeline/report.py`** — include classification results, filter by actionability

Also add source discovery capability — detect new pages in docs index that aren't tracked.

### Phase 3: Eval Updates (~2-3 hrs)

1. **Fix 9 outdated questions** — update prompts and/or rubrics per audit
2. **Add 10 new questions** — Code Review, Remote Control, chrome, etc.
3. **Implement 4 merges** — reduce overlap
4. **Add 4+ IMS-derived questions** — bid automation, competitor migration, team access
5. **Refresh rubrics** — `evals/rubrics.py` needs updating after KB changes
6. **Consider Sonnet as judge** — add `--judge-model claude-sonnet-4-6` support (already works via CLI flag)

### Phase 4: Validation (~1.5 hrs)

1. **Run API evals** — all 3 models on updated test suite with Sonnet judge
2. **Browser eval headed test** — capture real DOM selectors, run single test
3. **Verify Tier 1 injection** — SessionStart hook works with updated quick-reference

### Phase 5: Release (~30 min)

1. **Commit all changes** — atomic commits per logical unit
2. **Merge dev → main**
3. **Tag v2.4.0** — significant content update + eval methodology improvement
4. **Push to GitHub** — both repos
5. **Verify install paths** — `npx skills add` and `/plugin marketplace add`

### Phase 6: Description Re-validation (if time permits)

Use skill-creator's `run_loop.py` for empirical trigger testing in clean environment.

---

## Key Technical Context

### Environment
- **API key**: ANTHROPIC_API_KEY in `.env` at repo root (gitignored)
- **Sandbox**: Eval runner must run with `dangerouslyDisableSandbox: true`
- **Model IDs**: Only shorthands work for 4.6 (`claude-opus-4-6`, `claude-sonnet-4-6`)
- **Plugin installed locally** — SessionStart hook fires every session
- **Browser auth**: State at `evals/claude-ai-auth-state.json` (2026-03-10, may be expired)

### Pipeline Config Location
`pipeline/config.py` — SOURCES list (line 67). Each Source has: url, name, page_type, priority, kb_files.

### Rubric System
`evals/rubrics.py` — RUBRICS dict mapping test ID → list of factual claims. Imported at top of `eval_runner.py`, merged into TESTS at line 1257. Judge uses rubric when `test.get("rubric")` is truthy, else falls back to SKILL.md.

### Eval Question Audit File
Already copied to `docs/eval-question-audit.md`. Also available at `/private/tmp/claude-501/eval-question-audit.md`.

### IMS Integration
Knowledge Hub accessible via MCP tools (`mcp__claude_ai_knowledge-hub__*`). Contains 2200+ items — primarily bid response library for UK SME. No AI-specific content but reveals real SMB task patterns. Search showed zero results for "Claude", "ChatGPT", "AI capabilities".

### Key Files to Update
```
pipeline/config.py                    — add missing sources
data/quick-reference.md               — Tier 1 content (~95 lines)
skills/assistant-capabilities/SKILL.md — main skill (~270 lines)
references/claude-code-specifics.md   — Claude Code features
references/api-features.md            — API features
references/model-specifics.md         — model specs
evals/eval_runner.py                  — test definitions (line 82+)
evals/rubrics.py                      — independent judge rubrics
```

### The Automation Gap
The pipeline detects changes but doesn't auto-apply them:
- Sprint 1 (DONE): Scrape → Detect → Report
- Sprint 2 (TODO): Classify changes by actionability
- Sprint 3 (TODO): Auto-update KB files
- Sprint 4 (TODO): Propose edits to SKILL.md with validation gate
- **Missing entirely**: Source discovery (detect new doc pages not being tracked)

Even with Sprint 2-4, new features like Code Review require manual addition of source URLs to `config.py`. The pipeline can't create new KB files for pages it doesn't know about.

### Product Changelog Monitoring
The Excel/PowerPoint add-in update (March 11) wasn't caught because the pipeline doesn't monitor the product changelog. Add `https://www.anthropic.com/news` or equivalent as a HIGH priority source.

### Subagent Outputs (saved)
All 5 subagent outputs from session 14 saved to `docs/session-14-agents/`:
1. `01-research-browser-eval-selectors.md` — DOM analysis, what's broken, fix plan
2. `02-research-judge-rubric-implementation.md` — current judge_score() analysis, rubric design
3. `03-draft-judge-rubrics-from-kb.md` — 213 claims across 55 tests from KB docs
4. `04-review-ims-for-eval-improvements.md` — IMS search results, 10 suggested eval questions
5. `05-audit-eval-questions-for-staleness.md` — full audit: 9 outdated, 10 new, 5 merges
Also includes `prompts.md` with the prompts used to launch each agent.

### Model Training Data Cutoffs (from docs scrape)
- Opus 4.6: Reliable May 2025, Training Aug 2025
- Sonnet 4.6: Reliable Aug 2025, Training Jan 2026
- Haiku 4.5: Reliable Feb 2025, Training Jul 2025
These are NOT in our current model-specifics.md — add during Phase 1d.

### Tier Discrepancy
KB doc says 1M context requires "tier 4". Our references say "tier 3+ (previously tier 4+)". References likely more current (manual update after docs changed). Rubrics were corrected to say "tier 3+". Verify during KB re-scrape.

### Competitor Migration Category — User Priority
User explicitly endorsed the competitor migration category. Real-world observation: people moving from ChatGPT carry wrong assumptions (e.g. "Code Interpreter" vs Claude's code execution). High priority for v2.4.0 eval expansion.

### Skill-Creator for Description Validation
When updating the description for new capabilities (Code Review, Remote Control), use the skill-creator skill's `run_loop.py` eval for empirical trigger testing. Don't manually edit the description — use the proper eval workflow.

### Excel/PowerPoint Add-ins Update (March 11, 2026)
"We've improved our Claude for Excel and Claude for PowerPoint add-ins. They can now share the full context of your conversation, so every action Claude takes in one application is informed by everything that's happened in the other. We also added support for skills in the add-ins, and the ability for Amazon Bedrock, Google Cloud's Vertex AI, or Microsoft Foundry users to connect to them via an LLM gateway."
Key points: cross-app context sharing, skills support, LLM gateway for third-party providers.

### Browser Eval Current State
- Auth state may be expired (saved 2026-03-10)
- Script has `--headed` and `--yes` flags now
- Input field discovery improved but response extraction still uses guessed CSS selectors
- Need a headed test run to capture real response DOM elements
- `eval echo` in prompt getter has shell metacharacter expansion risk (low priority)
- The P1_control.json from last attempt failed with "Input field not found"
