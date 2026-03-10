# Session 12 Handoff: v2.2.0 Released, Pipeline Fixed, Cross-Platform Evals Live

## Project Context

We are building and maintaining Claude skills under the **claude-got-skills** GitHub account. The primary skill — **Claude Capabilities Awareness** — gives Claude comprehensive, accurate knowledge about its capabilities across all platforms (Claude.ai, Desktop, Claude Code, CoWork, API).

**GitHub repo**: https://github.com/claude-got-skills/skills
**Skill status**: v2.2.0 on `main`. Dev branch synced.
**Local folder**: `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/` — this is the git repo root.
**skills.sh listing**: https://skills.sh/claude-got-skills/skills/assistant-capabilities (1 install, Snyk WARN expected after softening)

---

## Session 11 Completed Work

### 1. skills.sh Publication ✅

- Skill is live on skills.sh (telemetry-driven, no submission process)
- Install command: `npx skills add claude-got-skills/skills@assistant-capabilities`
- Verified working for global and local installs across 8+ agents (Claude Code, Cursor, Copilot, Codex, etc.)
- "Claude" name restriction only applies to Claude.ai/Desktop ZIP upload, not skills.sh

### 2. Public Repo Cleanup ✅ (Major)

- Removed 76,059 lines of dev artifacts from public repo (docs/, launchd/, pipeline/, monitoring/, 20 eval reports)
- All personal info (email, paths, Gmail label ID) removed from tracked files
- Files remain on disk (gitignored) for migration to private dev repo
- Clean public repo: skills/, .claude-plugin/, evals/ (scripts only), knowledge-base/, README, LICENSE

### 3. Directory Rename ✅

- `skills/claude-capabilities/` → `skills/assistant-capabilities/` to match frontmatter name
- marketplace.json path and name updated
- Eval runner path updated

### 4. v2.1.72 Breaking Change Fix ✅

- Effort levels simplified: `max` removed, now only `low`/`medium`/`high`
- Updated SKILL.md + 3 reference files (api-features.md, model-specifics.md, agent-capabilities.md)
- Also added: ExitWorktree tool, Agent tool model parameter, /plan description, CLAUDE_CODE_DISABLE_CRON

### 5. Snyk Security Audit Fix ✅

- Softened hook shell command examples (removed explicit bash command JSON)
- Removed `dangerouslyDisableSandbox` mention
- Softened browser login state references
- Expected result: FAIL → WARN (matching competitor)

### 6. Skill Review (plugin-dev:skill-reviewer) ✅

- 5 issues found, 3 critical/major fixed:
  - Stale code example (`code_execution_20250825` → `code_execution_20260120`)
  - Effort:max inconsistency resolved in model-specifics.md
  - Model ID/Alias column swap fixed for Opus 4.6 and Sonnet 4.6
- Remaining: Description optimization (see below), hook events list inconsistency (minor)

### 7. Cross-Platform Eval Tests ✅

- 10 new tests in Category 8: Cross-Platform Awareness (43 total, 8 categories)
- Tests verify skill helps users discover capabilities on OTHER platforms
- Eval results: Treatment accuracy 3.90 vs Control 1.90 (+105% lift)
- Standout: 8.8 (full extension matrix) 1→6, 8.1 (Desktop→Code) 2→5
- Weak: 8.7 (Desktop memory) 2→2 (no lift), 8.10 (Claude.ai→agents) 0→2

### 8. Pipeline Fix: code.claude.com Scraping ✅ (Major)

- **Root cause discovered**: All code.claude.com URLs had moved from `/docs/X` to `/docs/en/X` — old URLs return 404
- Replaced Jina Reader with Firecrawl CLI for all code.claude.com pages
- Added boilerplate/scrape-failure detection (catches "Page Not Found" content)
- Fixed all 13 `kb_files` mappings (previously used fabricated filenames)
- Results: 6 pages went from "Page Not Found" (~4K chars) to real content (16K-98K chars)
- Removed stale support.claude.com source (returns 404, page moved)

### 9. Competitor Analysis ✅

- jezweb/claude-skills: 591 stars, 219 weekly installs, 24 skills across 9 plugins
- Their claude-capabilities: 48-line SKILL.md + 4 refs (350 lines total) vs our 259 + 5 refs (2,742 lines)
- Content is 6.9x less comprehensive, covers only 2 platforms (Claude AI, Code)
- Install numbers likely inflated by cross-agent indexing (uniform ~180 per agent)
- Stars driven by breadth (24 skills) not depth of claude-capabilities

### 10. Automation Pipeline Design ✅

- 1,792-line design document at `docs/automation-pipeline-design.md`
- 7-stage pipeline: scrape → noise filter → detect → classify → KB update → propose edits → validate
- Implementation plan: 5 sprints, 14-21 hours total
- Sprint 1 (P0) partially complete: Firecrawl integration done, URL fixes done, KB mapping fixed

---

## Current Git State

```
Branch: main (up to date with origin)
Dev: synced with main
Tag: v2.2.0

Commit history (session 11 work):
1ba8109 (HEAD -> dev, tag: v2.2.0, origin/main, origin/dev, main) Fix eval runner skill path after directory rename
c7229fd Fix skill review findings: stale code example, effort/model inconsistencies
08c1253 Add 10 cross-platform awareness eval tests (Category 8, 43 total)
2ea35d9 Clean public repo: rename skill, fix effort levels, remove dev artifacts
```

Note: Pipeline fixes (Firecrawl, URL fixes, KB mapping) are on disk but NOT committed (pipeline/ is gitignored from public repo). These files are local development infrastructure.

---

## Session 12 Priority Order

### 1. Browser Eval Rewrite (~2 hrs)

The browser_eval.sh script has multiple issues:
- Cloudflare blocks headless Chrome intermittently (browser fingerprinting)
- Claude.ai's input field selectors have changed (snapshot grep patterns outdated)
- Interactive prompts don't work from background processes
- Auth URL wait pattern was wrong (fixed: added `https://claude.ai/` pattern)
- Auth state load required `--state` flag on open, not separate `state load` command (fixed)

**Auth state IS saved** (manually via `agent-browser state save` after headed login).

**Recommended approach**: Fix agent-browser in **headed mode** (not Playwright — Playwright MCP is single-instance so blocks parallel work). The fixes needed:
1. Run entire eval headed (`--headed` flag on all `ab` calls, not just auth) — solves Cloudflare
2. Update input field snapshot grep patterns for current Claude.ai UI (take fresh headed snapshot, update regex)
3. Add `--yes` flag to skip interactive confirmation prompts (or pipe "y")

The core eval logic (navigate, type, wait, extract) is solid — only auth/selectors broke.

Manual fallback (run from terminal):
```bash
./evals/browser_eval.sh --auth    # Re-auth if needed
./evals/browser_eval.sh --control # Run with skill OFF (needs terminal for confirmation)
./evals/browser_eval.sh --treatment # Run with skill ON
./evals/browser_eval.sh --report-only # Generate report
```

### 2. Description Optimization (~1 hr)

The skill-reviewer flagged the description as too long (~680 chars, "MANDATORY TRIGGERS" anti-pattern). Use skill-creator's `run_loop.py` to empirically optimize:

1. Generate 20 trigger eval queries (10 should-trigger, 10 should-not)
2. Review with user via HTML template
3. Run optimization loop: `python -m scripts.run_loop --eval-set <eval.json> --skill-path skills/assistant-capabilities --model claude-opus-4-6 --max-iterations 5`
4. Apply best description

Suggested shorter description (~480 chars) from skill-reviewer:
```
Use when answering questions about Claude capabilities, models, pricing, or platform availability where training data may be outdated. Also use when designing system architecture, choosing extension patterns (CLAUDE.md vs skills vs hooks vs plugins), building agents, configuring tools, or advising what Claude can or cannot do. Covers API features, structured outputs, adaptive thinking, MCP connector, computer use, Files API, memory tool, code execution, agent SDK, agent teams, streaming, vision, PDFs, rate limits, Claude Code, Claude.ai, Desktop, CoWork, skills.sh
```

### 3. Strengthen Weak Cross-Platform Tests (~1 hr)

Tests 8.7 (Desktop memory, 0% lift) and 8.10 (Claude.ai→agents, low treatment score) reveal content gaps. Investigate and fix:
- 8.7: SKILL.md may not explicitly connect memory options across platforms
- 8.10: Need clearer guidance that multi-step agent workflows require Claude Code

### 4. Pipeline Sprint 1 Completion (~2 hrs)

Firecrawl integration and URL fixes are done. Remaining Sprint 1 work:
- Run a full (non-dry-run) pipeline execution to update checkpoints with real content
- Verify the 2100 scheduled run uses new Firecrawl scraping
- Monitor Firecrawl credit consumption (6 pages × 2 runs/day = 12 credits/day)
- Consider adding credit conservation (only re-scrape on HEAD request change)

### 5. Pipeline Sprint 2: Change Classification (~2-3 hrs)

Add Claude Haiku-powered change classification:
- When change detected, classify as BREAKING/NEW_FEATURE/DEPRECATION/BUG_FIX/COSMETIC
- Only BREAKING and NEW_FEATURE trigger notifications
- Estimated cost: ~$0.005/classification call
- See `docs/automation-pipeline-design.md` Stage 3 for implementation details

### 6. Private Dev Repo Creation (~30 min)

Create `claude-got-skills/skills-dev` on GitHub for archived artifacts:
- Move continuation prompts, design docs, specs, analysis, treatment/control samples
- Update memory files to reference the new repo
- Keep pipeline files local (they're machine-specific)

### 7. Multi-Run Eval for Variance (~30 min)

```bash
python3 evals/eval_runner.py --runs 3
```

The two single runs from session 11 showed stable cross-platform scores (3.9 treatment in both). A 3-run eval would confirm variance is low.

### 8. Claude.ai ZIP Packaging + Testing (~1 hr)

Package SKILL.md as ZIP for Claude.ai/Desktop upload:
1. Verify frontmatter has no colons, no block scalars, name is `assistant-capabilities`
2. Create ZIP, upload to Settings > Capabilities > Skills
3. Test cross-platform prompts from eval tests
4. Verify auto-invocation triggers

---

## Key Technical Context

### Environment

- **API key**: ANTHROPIC_API_KEY in `.env` at repo root (gitignored)
- **Python**: eval_runner.py needs `anthropic` package (installed globally)
- **Firecrawl**: CLI v1.2.1 installed, authenticated via shell env (not .env), pay-as-you-go (~216 credits remaining after pipeline test)
- **agent-browser**: v0.13.0 installed. Auth state expired (Cloudflare cookies). Re-auth: `./evals/browser_eval.sh --auth`
- **Freshness pipeline**: Files on disk but gitignored. Launchd agent loaded, runs 09:00/21:00. Now uses Firecrawl for code.claude.com pages.
- **Skill installed globally**: `~/.agents/skills/assistant-capabilities/` (symlinked to `~/.claude/skills/`)

### Eval Results (v2.2.0, 43 tests)

| Category | Tests | Ctrl | Trt | Lift |
|----------|-------|------|-----|------|
| Architecture Decisions | 3 | 1.67 | 4.33 | +159% |
| Can Claude Do X | 5 | 2.00 | 4.60 | +130% |
| Implementation Guidance | 8 | 2.50 | 4.50 | +80% |
| Model Selection | 1 | 1.00 | 5.00 | +400% |
| Extension Awareness | 8 | 2.00 | 4.50 | +125% |
| Hallucination Detection | 5 | 1.80 | 3.20 | +78% |
| Cross-Platform Awareness | 10 | 1.90 | 3.90 | +105% |
| Negative (regression) | 3 | 5.00 | 5.00 | 0% |

### Competitor: jezweb/claude-skills

- 591 stars, 219 weekly installs, `claude-capabilities` name on skills.sh
- 48-line SKILL.md + 350 lines references (6.9x less than ours)
- Covers 2 platforms (Claude AI, Claude Code) vs our 4
- No eval infrastructure, no freshness pipeline
- Stars from breadth (24 skills in 9 plugins), not capabilities depth

### Automation Pipeline Architecture

Design doc: `docs/automation-pipeline-design.md` (1,792 lines)

Current state: Sprint 1 partially complete
- ✅ Firecrawl integration for code.claude.com
- ✅ URL fixes (/docs/ → /docs/en/)
- ✅ KB file mapping corrections
- ✅ Boilerplate/scrape failure detection
- ⬜ Full pipeline run with new scraping (checkpoint update)
- ⬜ Sprint 2: Claude-powered change classification
- ⬜ Sprint 3: Automated KB updates
- ⬜ Sprint 4: Proposed edits + eval validation gate
- ⬜ Sprint 5: Source expansion (13→22 URLs)

### Available Quality Tools

- **writing-skills/testing-skills-with-subagents**: TDD pressure testing for skills (RED-GREEN-REFACTOR)
- **skill-creator**: Full eval framework with subagent testing, benchmark viewer, description optimization (`run_loop.py`)
- **plugin-dev/skill-reviewer**: Structured quality review agent (already run, critical issues fixed)

### File Inventory

```
SKILL.md:                          259 lines
references/api-features.md:       662 lines
references/agent-capabilities.md:  493 lines  (was 509, hook example simplified)
references/claude-code-specifics.md: 542 lines
references/tool-types.md:         467 lines
references/model-specifics.md:    309 lines  (was 310, effort:max row removed)
Total skill content:               2,732 lines

Eval tests:     43 (8 categories)
Eval reports:   2 from session 11 (on disk, gitignored)
Knowledge-base: 28 source files (last refreshed 2026-03-09)
Pipeline:       8 Python files + shell wrapper (on disk, gitignored, Firecrawl-enabled)
```
