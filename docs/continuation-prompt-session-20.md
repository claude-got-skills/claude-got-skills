# Session 20: Chrome CDP Eval Implementation & 15-Prompt A/B Run

## Project Context

We are building and maintaining Claude skills under the **claude-got-skills** GitHub account. The primary skill — **Claude Capabilities Awareness** (`assistant-capabilities`) — gives Claude comprehensive, accurate knowledge about its capabilities across all platforms.

**GitHub repo**: https://github.com/claude-got-skills/claude-got-skills
**Private dev repo**: https://github.com/liam-jons/skills-dev
**Local folder**: `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/` — this is the git repo root.

---

## Session 19 Completed Work

### 1. Sprint 4: Pipeline Proposal Engine — COMPLETE ✅

Full edit proposal pipeline (`pipeline/propose_edits.py`, 438 lines). When Sprint 3 updates KB files, Sprint 4 generates human-reviewed edit proposals for SKILL.md, references, and quick-reference.

- `KB_TO_TARGETS` mapping: 39 KB files → 7 target files
- Prompt tuned across 4 edge cases (formatting-only, mixed importance, multi-KB, pricing)
- **Pricing excluded from proposals** — deferred to live docs lookup (the standard Anthropic skill handles this)
- `--propose-edits` CLI flag integrated into `freshness_check.py`

### 2. Browser Eval: 5-Prompt A/B Run — COMPLETE ✅

Full A/B run on Claude.ai (control + treatment). Findings:
- Skill triggered on 4/5 prompts
- Thinking extraction fixed (removed `-i` flag from snapshot)
- Screenshot timeout capped at 10s (was hanging ~150s)
- Report: `evals/browser-eval-report-20260314-173801.md`

### 3. Browser Eval Expanded to 15 Prompts — COMPLETE ✅

Replaced 5 prompts with 15 selected from the 65-test eval suite for maximum category coverage:
- 2x Hallucination Detection, 3x Can Claude Do X, 2x Extension Awareness
- 2x Cross-Platform Awareness, 2x Conversational Platform
- 1x each: Model Selection, Architecture, Implementation, Competitor Migration
- Automated skill toggle via JS on `/customize/skills` page

### 4. Chrome CDP Proof of Concept — COMPLETE ✅

Confirmed 6-7x speedup over agent-browser:
- `document.execCommand('insertText')` for ProseMirror: 0.4s vs 150s
- Click `button[aria-label="Send message"]` to submit
- Poll `snap` for "Give positive feedback" to detect completion
- Full prompt cycle: ~30-50s (vs 260-350s with agent-browser)
- **Projected 15-prompt A/B time: ~25 min** (vs ~135 min)

### 5. `/btw` Command Added to Skill — COMPLETE ✅

- Added to SKILL.md Claude Code section
- Added section + bundled commands table entry in `claude-code-specifics.md`
- Two new pipeline sources: `interactive-mode`, `commands` (25 total sources)

### 6. Chrome CDP Eval Spec — COMPLETE ✅

990-line implementation spec at `docs/chrome-cdp-eval-spec.md` covering:
- 8 core functions with PoC-validated code
- Automated skill toggle
- 3-tier skill invocation detection (our skill vs built-in product knowledge)
- Error handling, migration path, ~5 hour implementation estimate

---

## Current Git State

```
Branch: dev
Tags: v2.5.0 on main (latest release)

origin (public):  github.com/claude-got-skills/claude-got-skills
dev (private):    github.com/liam-jons/skills-dev

Recent commits on dev:
6bb1a6a Re-add pipeline files (private dev remote only)
97a6930 Remove pipeline files from public repo (internal only)
a2c6397 Chrome CDP eval spec (dev remote only)
c19bc7d Add /btw command to skill, restore pipeline, add 2 new sources
ea95d50 Update all references for repo rename skills → claude-got-skills
386d6b1 Rebrand repo for multi-plugin marketplace launch
8f87ff6 Remove internal development artifacts from public repo
1e661a1 Expand browser eval to 15 prompts with automated skill toggle
7f423b1 Browser eval A/B report
8e5b6db Fix browser eval thinking extraction and screenshot timeout
61713d9 Sprint 4: propose skill/reference edits from KB changes
```

---

## Session 20 Priority: Chrome CDP Eval Implementation + 15-Prompt A/B Run

### Implementation Approach

**Sequential sub-agent phases.** Each phase is handled by a sub-agent that completes before the next launches. This ensures quality over speed.

### Phase 1: Core Script Scaffold (~1 hour)

Create `evals/browser_eval_cdp.sh` with:
- Configuration (CDP script path, prompts from existing `browser_eval.sh`)
- Helper functions: `cdp_cmd`, `cdp_discover_target`
- Core functions: `cdp_navigate`, `cdp_type`, `cdp_submit`
- Submit uses `document.execCommand('insertText')` + click `button[aria-label="Send message"]`

**Key PoC discoveries to encode:**
- `type` command (Input.insertText) does NOT trigger ProseMirror — must use `document.execCommand('insertText')` via eval
- Enter key via CDP doesn't submit — must click Send button
- Send button only appears after ProseMirror detects content via execCommand
- Single quotes in prompt text must be escaped for JS

### Phase 2: Response Detection & Extraction (~1 hour)

Add:
- `cdp_wait_for_response` — poll snap for "Give positive feedback" / "Give negative feedback"
- `cdp_extract_response` — parse accessibility snap for response text
- `cdp_extract_thinking` — find thinking button, expand, extract content

**Response detection:** The snap shows `[button] Give positive feedback` and `[button] Give negative feedback` when response is complete. Check for these via `snap | grep`.

**Thinking extraction:** The snap shows `[button] Thinking about <summary>` followed by thinking content as `[StaticText]` elements.

### Phase 3: Skill Toggle & Run Orchestration (~1 hour)

Add:
- `cdp_toggle_skill` — navigate to `/customize/skills`, click skill, toggle `input[aria-label="Enable skill"]`
- `run_condition` — loop through all 15 prompts
- Full run mode: toggle OFF → control → toggle ON → treatment → assemble → report

**Skill toggle PoC-validated flow:**
1. Navigate to `https://claude.ai/customize/skills`
2. Click the skill name button in the sidebar
3. `eval`: find `input[aria-label="Enable skill"]`, check `.checked`, click if needed
4. Verify new state

### Phase 4: Integration & Testing (~1-2 hours)

- Wire up JSON output (same format as existing `browser_eval.sh`)
- Reuse `browser_eval_report.py` for report generation
- Run a 3-prompt smoke test to verify end-to-end
- Run the full 15-prompt A/B eval

### Post-Eval

- Analyze results, compare with API eval data
- If results are good: merge dev → main, tag v2.5.1 (or v2.6.0)
- Update skill on Claude.ai with latest version

---

## Critical Technical Context

### Skill Invocation Detection

**IMPORTANT:** Claude.ai has its own built-in "product self-knowledge" / "product knowledge" skill. Our skill is "assistant-capabilities". These are DIFFERENT.

In thinking panel text:
- **Our skill:** Look for `assistant-capabilities`, `reading assistant capabilities`, `check the assistant-capabilities skill`
- **Built-in platform skill:** `product self-knowledge`, `product knowledge skill`, `Recalled product knowledge`
- Only "assistant-capabilities" counts as our skill triggering

### Git Remotes and Push Rules

**CRITICAL: Never `git add -f pipeline/` or `git add -f docs/` in commits pushed to origin.**

| Remote | URL | What to push |
|--------|-----|--------------|
| `origin` | `github.com/claude-got-skills/claude-got-skills` | **Public** — code, evals, skill content, plugin files only |
| `dev` | `github.com/liam-jons/skills-dev` | **Private** — everything PLUS `docs/`, `pipeline/` |

The `.gitignore` excludes `pipeline/` and `docs/` from normal adds. Only force-add these in commits destined for the `dev` remote.

### Chrome CDP Setup

- **CLI:** `/Users/liamj/.agents/skills/chrome-cdp/scripts/cdp.mjs`
- **Requires:** Chrome with remote debugging enabled (`chrome://inspect/#remote-debugging`)
- **Requires:** Node.js 22+
- **Target discovery:** `node cdp.mjs list` → find Claude.ai tab by URL
- **Commands:** `eval`, `snap`, `click`, `nav`, `type`, `shot`, `stop`

### Environment
- **API key**: ANTHROPIC_API_KEY in `.env` at repo root
- **Sandbox**: Eval runner and pipeline must run with `dangerouslyDisableSandbox: true`
- **Model IDs**: Only shorthands work for 4.6 (`claude-opus-4-6`, `claude-sonnet-4-6`)

### Key Files
```
docs/chrome-cdp-eval-spec.md              — Full implementation spec (990 lines)
evals/browser_eval.sh                      — Current 15-prompt eval (agent-browser backend)
evals/browser_eval_report.py               — Report generator (shared, reuse as-is)
evals/browser-eval-report-20260314-173801.md — Latest A/B report (5 prompts)
skills/assistant-capabilities/SKILL.md     — Main skill file
pipeline/config.py                         — 25 sources, 39 KB targets
pipeline/propose_edits.py                  — Sprint 4 proposal engine
```
