# Session 19 Handoff: Sprint 4 Implementation, Production Readiness, and Release

## Project Context

We are building and maintaining Claude skills under the **claude-got-skills** GitHub account. The primary skill — **Claude Capabilities Awareness** — gives Claude comprehensive, accurate knowledge about its capabilities across all platforms (Claude.ai, Desktop, Claude Code, CoWork, API).

**GitHub repo**: https://github.com/claude-got-skills/skills
**Private dev repo**: https://github.com/liam-jons/skills-dev
**Local folder**: `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/` — this is the git repo root.
**skills.sh listing**: https://skills.sh/claude-got-skills/skills/assistant-capabilities

---

## Session 18 Completed Work

### 1. Sonnet 4.6 API Eval — COMPLETE ✅

Full 65-test eval with `claude-sonnet-4-6` as both test model and judge. **Treatment wins all 10 categories.**

| Category | Sonnet Ctrl Acc | Sonnet Trt Acc | Sonnet Lift | Haiku Lift (baseline) |
|----------|:---:|:---:|:---:|:---:|
| Architecture Decisions | 1.33 | 3.00 | **+1.67** | +3.34 |
| Can Claude Do X | 2.00 | 4.29 | **+2.29** | +2.14 |
| Competitor Migration | 2.00 | 3.25 | **+1.25** | +1.25 |
| Conversational Platform | 2.42 | 3.33 | **+0.91** | +1.50 |
| Cross-Platform Awareness | 2.75 | 3.92 | **+1.17** | +2.25 |
| Extension Awareness | 1.78 | 4.22 | **+2.44** | +2.22 |
| Hallucination Detection | 0.60 | 2.80 | **+2.20** | +1.20 |
| Implementation Guidance | 2.78 | 4.33 | **+1.55** | +1.77 |
| Model Selection | 1.00 | 7.00 | **+6.00** | +3.00 |
| Negative (No Change) | 5.00 | 5.00 | **0.00** | 0.00 |

**Key findings:**
- Sonnet control ≈ Haiku control (avg accuracy 2.07 vs 2.08) — training data doesn't help capabilities awareness
- Sonnet hallucinates MORE confidently without the skill (control 0.60 vs Haiku 1.60 on hallucination tests)
- Model Selection shows 6x lift on Sonnet (1→7) vs 3x on Haiku
- Token overhead: 283K (comparable to Haiku 308K)
- Report: `evals/eval-report-20260313-223132.md`

### 2. Sprint 3 Pipeline — VERIFIED END-TO-END ✅

All code paths tested:
- `--update-kb` without `--classify`: warning shown correctly
- Full pipeline (23 sources): 18 changes detected, all LOW/MEDIUM, KB update correctly skipped
- KB merge with mock CRITICAL: LLM merge worked (context-windows KB updated accurately)
- Safety: SKILL.md/references untouched (PROTECTED_PATTERNS enforced)
- Error handling: graceful on bad API key, no crashes
- **Finding**: Classifier may under-classify when formatting changes dominate the diff (classified MEDIUM for a GA-status change). Worth tuning in Sprint 4.

### 3. Browser Eval Thinking Panel — FIXED AND LIVE-VERIFIED ✅

**Root cause**: Old `grep 'think|reasoning'` matched user prompt text, returning wrong DOM element (e.g., Settings button).

**Actual DOM structure** (verified live on Claude.ai 2026-03-14):
```
grandparent DIV
  ├── DIV > BUTTON[aria-expanded="true"]  (thinking toggle, summary text)
  ├── SPAN                                (status element, same text)
  └── DIV                                 (expanded thinking content)
```

**Fix** (2 commits):
1. `eeb21fa` — Find `status:` line in snapshot, click adjacent button, multi-strategy JS extraction
2. `52d6e40` — Refined JS to navigate grandparent.children[2+] after live verification

**Verified**: Full thinking content extracted including "Checking product self-knowledge skill" (skill invocation evidence), web search queries, reasoning chain, and "Done" indicator.

Screenshot: `evals/thinking-panel-expanded-verification.png`

### 4. Sprint 4 Design Spec — COMPLETE ✅

Full design specification at `docs/sprint-4-design-spec.md` (270 lines). Key decisions:
- **Model**: Sonnet 4.6 (same as Sprint 3)
- **Output**: Unified diff in markdown report at `pipeline/proposals/`
- **Trigger**: Inline after Sprint 3, gated by `--propose-edits` flag
- **Coordination**: Aggregate by target file (one proposal per target)
- **Key gap identified**: No explicit KB-to-reference mapping exists. Need `KB_TO_TARGETS` in config.py.
- **Recommended start**: Draft and test the proposal prompt first, then build infrastructure.
- **Estimate**: ~400-500 new lines, medium complexity.

---

## Current Git State

```
Branch: dev
Tags: v2.5.0 on main (latest)

main at: f8e4fb0 (codebase-review v1.2.0)
dev at:  5f6b0d2 (Sprint 4 design spec) — 6 commits ahead of main

origin (public):  pushed up to 5f6b0d2
dev (private):    pushed up to 5f6b0d2

Recent commits on dev:
5f6b0d2 Sprint 4 design spec: propose SKILL.md edits from KB changes
52d6e40 Refine thinking panel JS extraction with verified DOM structure
eeb21fa Fix browser eval thinking panel extraction for Claude.ai DOM changes
88b397a Add session 18 handoff: Sonnet eval, Sprint 3 verification, browser refinement
26a27d1 Sprint 3: auto-update KB files from detected documentation changes
b25f7d4 Fix broken pointer URLs and bump codebase-review version
f8e4fb0 codebase-review v1.2.0: implement all 17 improvements (P1-P4)
```

Note: `docs/` and `pipeline/` are in `.gitignore` for origin. Pipeline files are force-added (`git add -f`) and go to dev remote only. Do NOT push pipeline/docs commits to origin.

---

## Session 19 Priority Order

### Priority 1: Sprint 4 Implementation (~2-3 hours)

The design spec is at `docs/sprint-4-design-spec.md`. Follow the recommended approach:

**Step 1: Draft and test the proposal prompt**

Before writing any infrastructure, test the prompt manually:

```python
# Use a KB diff from Sprint 3 testing (context-windows was updated in session 18 test)
# Pass it to Sonnet 4.6 with:
# 1. The KB diff
# 2. The current target file (e.g., references/api-features.md)
# 3. The 6-point maintenance strategy
# Evaluate: does the proposed diff make sense? Is it appropriately scoped?
```

Iterate 2-3 times until proposals consistently apply the strategy:
- SKILL.md changes only for load-bearing strings and error-preventing values
- Reference files get the detail
- Quick-reference gets only critical changes
- No over-proposal

**Step 2: Build infrastructure**

1. Create `pipeline/propose_edits.py` (~300-400 lines):
   - `ProposalResult` dataclass
   - `propose_skill_edits()` main function
   - `_aggregate_by_target()` — group KB diffs by target file
   - `_generate_proposal()` — LLM call per target
   - `_format_proposal_report()` — markdown with diffs
   - `_save_proposal()` — write to `pipeline/proposals/`

2. Add `KB_TO_TARGETS` mapping to `pipeline/config.py` (~50-80 lines):
   - Map all 37 KB files to their target files (see spec section 2.3)
   - Include `get_targets_for_kb()` helper
   - Add validation that all mapped files exist

3. Integrate into `freshness_check.py`:
   - Add `--propose-edits` CLI flag (requires `--update-kb`)
   - Call after `update_kb_files()` when KB files were actually updated

**Step 3: End-to-end test**

```bash
# Force a CRITICAL change and verify full pipeline:
python -m pipeline.freshness_check --classify --update-kb --propose-edits
# Should produce: scrape → detect → classify → update KB → generate proposal
```

### Priority 2: Browser Eval Full Run (~30 min)

The thinking panel fix has been live-verified with Playwright but NOT tested via `browser_eval.sh` in a full A/B run. Run:

```bash
cd /Users/liamj/Documents/development/claude-got-skills/claude-capabilities
./evals/browser_eval.sh --headed  # Full A/B run with visible browser
```

**What to verify:**
- Thinking summary extracted for each prompt (not empty)
- Thinking content contains "product self-knowledge" or "Consulted product knowledge" for treatment
- Response extraction still works (no regressions from thinking panel changes)
- Screenshots captured

**Note**: `fill` takes ~2.5 min per prompt (character-by-character). Total run: ~35 min per condition, ~70 min total.

### Priority 3: Production Readiness — Install Path Verification

Verify the skill installs correctly in clean environments:

1. **Skill install via npx** (test in a clean project directory, NOT this workspace):
   ```bash
   npx skills add claude-got-skills/skills@assistant-capabilities
   ```

2. **Plugin install via CLI**:
   ```bash
   /plugin marketplace add claude-got-skills/skills
   /plugin install claude-got-skills@claude-got-skills
   ```

3. **Claude.ai skill auto-invocation**: Ask a capability question, confirm "Consulted product knowledge" appears in thinking panel (already verified in session 18 live test — the response showed "Checking product self-knowledge skill").

4. **Spot-check 5 key facts** against live docs:
   - Opus 4.6 model ID: `claude-opus-4-6` ✅ (verified in session 18 live test)
   - 1M context: now GA for Opus/Sonnet 4.6, no beta header needed (Sprint 3 detected this change)
   - Structured outputs GA (no beta header needed)
   - Code Review pricing ($15-25/review)
   - Remote Control command (`claude remote-control`)

### Priority 4: Merge dev → main, Tag v2.5.1

After priorities 1-3 are validated:

1. Merge dev → main (6 commits):
   - Browser eval fixes (3 commits) — public
   - Sprint 3 pipeline — private (docs/pipeline gitignored from origin)
   - Session 18 handoff — private
   - Sprint 4 design spec — private

2. Tag as v2.5.1 (browser eval fixes are a patch-level change)

3. Push tag to both remotes

4. Update version in:
   - `.claude-plugin/marketplace.json`
   - `data/quick-reference.md` (if version is mentioned)
   - Any other version references

### Priority 5: Opus 4.6 Eval (Optional)

If time permits, run the eval with Opus 4.6 as test model:

```bash
python evals/eval_runner.py --model claude-opus-4-6 --judge-model claude-sonnet-4-6 --runs 1
```

Must run with `dangerouslyDisableSandbox: true`. Expected to take longer than Sonnet (~60-90 min).

**Why**: Opus users are the highest-value audience. Understanding the skill's value on Opus helps focus content.

---

## Key Technical Context

### Environment
- **API key**: ANTHROPIC_API_KEY in `.env` at repo root (gitignored)
- **Sandbox**: Eval runner and pipeline must run with `dangerouslyDisableSandbox: true`
- **Model IDs**: Only shorthands work for 4.6 (`claude-opus-4-6`, `claude-sonnet-4-6`). No dated variants.
- **Plugin installed locally** — SessionStart hook fires every session in this workspace
- **Skill installed on Claude.ai** — at claude.ai/customize/skills, auto-invokes

### Model Training Data Cutoffs
- Opus 4.6: Reliable May 2025, Training Aug 2025
- Sonnet 4.6: Reliable Aug 2025, Training Jan 2026
- Haiku 4.5: Reliable Feb 2025, Training Jul 2025

### Eval Infrastructure
- **API eval**: `evals/eval_runner.py` — 65 tests, 10 categories. Flags: `--model`, `--judge-model`, `--runs`, `--tier1-only`, `--no-judge`
- **Browser eval**: `evals/browser_eval.sh` — `--auth`, `--control`, `--treatment`, `--headed`, `--yes`. Must run foreground.
- **Judge**: Sonnet 4.6 (`--judge-model claude-sonnet-4-6`). No prefill. Reliable JSON.
- **Haiku baseline**: `evals/eval-report-20260313-201051.md`
- **Sonnet baseline**: `evals/eval-report-20260313-223132.md`
- **Browser baseline**: `evals/browser-eval-report-20260313-210609.md`

### Browser Eval Key Info
- Skills page: `claude.ai/customize/skills` (not Settings > Capabilities)
- `fill` takes ~2.5 min per prompt (character-by-character typing). Total run: ~35 min/condition.
- agent-browser daemon: must be killed between runs. Use `agent-browser --session claude-eval close` (double close).
- Snapshot ref format: `[ref=eN]` — extract with `grep -oE 'ref=e[0-9]+' | sed 's/ref=//'`
- **Thinking panel**: FIXED. Toggle button has summary text, `status:` element follows. Click button → expanded content at grandparent.children[2+]. Button has `aria-expanded="true"` when open.
- Auth state: `evals/claude-ai-auth-state.json` (expires periodically, re-auth with `--auth`)

### Pipeline
- 23 sources in `pipeline/config.py`
- Sprint 1 (scrape + detect) ✅
- Sprint 2 (classify) ✅
- Sprint 3 (auto-update KB) ✅ verified
- Sprint 4 (propose SKILL.md edits) — design spec complete at `docs/sprint-4-design-spec.md`, ready for implementation
- `pipeline/update_kb.py` uses Sonnet 4.6 for merge, 60K char input limit, safety path checks
- **Classifier finding**: Under-classifies when formatting changes dominate the diff

### Git Remotes and Push Rules

| Remote | URL | What to push |
|--------|-----|--------------|
| `origin` | `github.com/claude-got-skills/skills` | **Public repo** — code, evals, skill content, plugin files. No docs, pipeline, or agent outputs. |
| `dev` | `github.com/liam-jons/skills-dev` | **Private repo** — everything from origin PLUS `docs/`, `pipeline/`. |

When committing:
1. **Code/eval changes** → normal `git add` + `git commit` → `git push origin dev` + `git push dev dev`
2. **Docs/pipeline** → `git add -f docs/... pipeline/...` → separate commit → `git push dev dev` only (NOT origin)

### Key Files
```
docs/sprint-4-design-spec.md              — Sprint 4 full design spec (NEW)
pipeline/update_kb.py                     — Sprint 3 KB auto-update
pipeline/freshness_check.py               — main pipeline entry
pipeline/config.py                        — 23 source URLs + kb_files mapping
pipeline/classify.py                      — Haiku classification
data/quick-reference.md                   — Tier 1 content (100 lines, v2.5.0)
skills/assistant-capabilities/SKILL.md    — main skill (280 lines, v2.5.0)
references/api-features.md               — API features
references/model-specifics.md             — model specs
evals/eval_runner.py                      — 65 tests, 10 categories
evals/rubrics.py                          — 65 independent judge rubrics
evals/browser_eval.sh                     — FIXED: thinking panel extraction
evals/eval-report-20260313-201051.md      — v2.5.0 Haiku baseline (Sonnet judge)
evals/eval-report-20260313-223132.md      — v2.5.0 Sonnet baseline (Sonnet judge)
evals/browser-eval-report-20260313-210609.md — browser A/B baseline
.claude-plugin/marketplace.json           — v2.5.0, codebase-review v1.2.0
```

### Maintenance Strategy (v2.5.0 Decision Framework)
When adding new content to the skill, apply this test:
1. Load-bearing string (model ID, header, tool type)? → KEEP (bootstrapping)
2. Wrong value causes error (400, 404, silent failure)? → KEEP (error prevention)
3. Architectural guidance not in any single docs page? → KEEP (unique advisory)
4. Prevents common hallucination? → KEEP (anti-hallucination)
5. Number that changes independently of features? → POINTER (price, rate limit)
6. Version number with no functional implication? → POINTER or omit
