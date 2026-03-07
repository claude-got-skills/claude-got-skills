# Capabilities Pipeline — Phase 2 Requirements

**Author:** Liam Jones
**Date:** 10 February 2026
**Status:** Draft
**Depends on:** Phase 1 MVP skill (built), Distill monitors (8 configured)

---

## 1. Overview

Phase 2 bridges the gap between "Distill detects a change" and "the skill gets updated."
Currently the pipeline is fully manual: Distill sends an email notification, Liam reads
the change, and manually edits the skill files. Phase 2 adds the `/update-claude-capabilities`
command, `capability-matrix.json` source of truth, and fetch/diff scripts to make this
semi-automated — Claude does the heavy lifting, Liam reviews and approves.

### Target Pipeline

```
Distill Monitors (8 pages)
    │ email notification
    ▼
Liam runs: /update-claude-capabilities [--source changed-page] [--dry-run]
    │
    ▼
Command workflow:
  1. fetch-docs.sh — retrieve current content from all monitored URLs
  2. diff-capabilities.sh — compare fetched content vs capability-matrix.json
  3. Claude interprets diff — identify new features, deprecations, status changes
  4. Claude updates files — SKILL.md, references/, capability-matrix.json, changelog
  5. Verify — line count, JSON validity, format consistency
  6. Present summary — show Liam what changed and why
    │
    ▼
Liam reviews changes → approves → publishes via skills.sh or plugin marketplace
```

---

## 2. Workstream 1: Fix Distill SELECTION_EMPTY Errors

### Problem

Four monitors return SELECTION_EMPTY errors because support.claude.com uses Intercom's
help center platform, which loads article content dynamically via JavaScript. Distill's
cloud monitors perform HTTP fetches by default — no JS rendering — so the CSS selector
`article[class*="intercom-force-break"]` matches nothing.

### Affected Monitors

| # | Monitor Name | URL | Selector | Status |
|---|-------------|-----|----------|--------|
| 3 | Features & Capabilities | support.claude.com/en/collections/... | `section[data-testid="main-content"]` | At risk |
| 4 | Claude.ai Release Notes | support.claude.com/en/articles/... | `article[class*="intercom-force-break"]` | SELECTION_EMPTY |
| 5 | Claude in Excel | support.claude.com/en/articles/... | `article[class*="intercom-force-break"]` | Likely affected |
| 6 | Claude in PowerPoint | support.claude.com/en/articles/... | `article[class*="intercom-force-break"]` | SELECTION_EMPTY |
| 7 | Getting Started with Cowork | support.claude.com/en/articles/... | `article[class*="intercom-force-break"]` | SELECTION_EMPTY |

### Fix: Option A (Recommended) — Enable JavaScript Rendering + Delay

For each affected monitor:

1. Open the monitor's edit/options screen
2. Find the rendering/advanced settings (gear icon next to selector area, or JSON config)
3. Set `"dynamic": true` — tells Distill to use a headless browser (JS rendering)
4. Set `"delay": 10` (seconds) — gives Intercom content time to render
5. Save and test

**Note:** JS rendering on cloud checks may require a paid Distill plan (Starter or above).
Free plans may be limited to local monitoring with JS rendering.

### Fix: Option B (Fallback) — Less Dynamic Selector

If the current Distill plan does not support cloud JS rendering:

- Change selector to `title` (always in HTML head, no JS needed)
- Optionally combine with `meta[name="description"]`
- Downside: less granular monitoring — catches any page metadata change

### Fix: Option C (Last Resort) — Switch to Local Monitoring

Change affected monitors from "Cloud - Distill's Servers" to "Local" monitoring:

- Local monitoring runs in the browser extension — always has JS
- Downside: requires browser to be open; won't check when computer is off

### Verification Steps

After applying fixes to all affected monitors:

1. Force-check each monitor (right-click → "Check now")
2. Verify:
   - Error badge clears (no SELECTION_EMPTY)
   - Status shows "ON" (active)
   - Content preview populates with actual article content
3. Also verify the unaffected monitors still work:
   - Monitors 1-2 (docs.anthropic.com): use `main` selector, server-rendered — should be fine
   - Monitor 8 (GitHub CHANGELOG): uses `article.markdown-body`, server-rendered — should be fine

### Distill Error Reference

Source: https://distill.io/docs/web-monitor/troubleshooting-errors/#selection_empty

Key parameters:
- `"dynamic": true` — enable JS rendering (headless browser) for cloud checks
- `"delay": 10` — seconds to wait after page load before checking selector
- `"ignoreEmptyText": false` — get notified if content becomes empty
- Proxy options (Shared Pool, Dedicated) can help with IP-based blocking

---

## 3. Source Coverage Gap Analysis

### Current State

The Distill monitors and the planned `/update` fetch script watch **different sets of pages**.
This gap must be closed in Phase 2.

| Source URL | Distill Monitor | Fetch Script |
|-----------|:-:|:-:|
| docs.anthropic.com/en/docs/about-claude/models/overview | ✅ | ✅ |
| docs.anthropic.com/en/docs/about-claude/models (release notes) | ✅ | ✅ |
| support.claude.com — Features & Capabilities collection | ✅ | ❌ |
| support.claude.com — Claude.ai Release Notes | ✅ | ❌ |
| support.claude.com — Claude in Excel | ✅ | ❌ |
| support.claude.com — Claude in PowerPoint | ✅ | ❌ |
| support.claude.com — Getting Started with Cowork | ✅ | ❌ |
| github.com — Claude Code CHANGELOG | ✅ | ✅ |
| docs.anthropic.com/en/docs/build-with-claude/overview | ❌ | ✅ |
| docs.anthropic.com/en/changelog | ❌ | ✅ |
| code.claude.com/docs (Claude Code docs) | ❌ | ❌ |

### Resolution Plan

**A. Add support.claude.com URLs to fetch-docs.sh:**

```bash
SUPPORT_SOURCES=(
  "https://support.claude.com/en/collections/10404134-features-capabilities"
  "https://support.claude.com/en/articles/10399539-claude-ai-release-notes"
  "https://support.claude.com/en/articles/11559749-claude-in-excel"
  "https://support.claude.com/en/articles/11559826-claude-in-powerpoint"
  "https://support.claude.com/en/articles/11559587-getting-started-with-cowork"
)
```

These require JS rendering to fetch content (same issue as Distill). The `/update` command
should use Claude's WebFetch tool (which renders JS) rather than curl.

**B. Add new Distill monitors for currently unmonitored pages:**

| Page | Why |
|------|-----|
| docs.anthropic.com/en/changelog | API changelog catches changes not in release notes |
| docs.anthropic.com/en/docs/build-with-claude/overview | Build overview — framework for capabilities |
| docs.anthropic.com/en/docs/build-with-claude/computer-use | Computer use is active beta, changes frequently |
| docs.anthropic.com/en/docs/build-with-claude/tool-use | Tool use patterns evolve |
| code.claude.com/docs/changelog | Claude Code release notes (if dedicated page exists) |

**C. Add code.claude.com sources to fetch-docs.sh:**

The skill now covers Claude Code capabilities (agent teams, Chrome browser, CLI). The
fetch script needs these sources:

```bash
CLAUDE_CODE_SOURCES=(
  "https://code.claude.com/docs/agent-teams"
  "https://code.claude.com/docs/chrome"
  "https://code.claude.com/docs/cli-reference"
  "https://code.claude.com/docs/changelog"
)
```

---

## 4. Build Items

### Item 1: capability-matrix.json

**Design doc section:** §6
**Priority:** High — this is the diffing source of truth

Create the machine-readable JSON file that captures all capabilities. Schema already
defined in the design doc. Initial population from the current SKILL.md + reference files.

**Tasks:**
- [ ] Create `assets/capability-matrix.json` using schema from design doc §6
- [ ] Populate with all current models, features, tools, deprecations
- [ ] Add Claude Code capabilities section (agent teams, Chrome, CLI)
- [ ] Validate JSON schema
- [ ] Write schema validation script (`validate-matrix.sh`)

### Item 2: fetch-docs.sh

**Design doc section:** §5.2
**Priority:** High — entry point for the update workflow

Expand the script to cover ALL monitored URLs (not just docs.anthropic.com + GitHub).

**Tasks:**
- [ ] Add support.claude.com URLs (5 pages)
- [ ] Add code.claude.com URLs (4 pages)
- [ ] Handle JS-rendered pages via WebFetch markers
- [ ] Add error handling for failed fetches
- [ ] Add cache management (timestamp, TTL)
- [ ] Test with dry-run output

### Item 3: diff-capabilities.sh

**Design doc section:** §5.3
**Priority:** High — the intelligence layer

Compare fetched content against capability-matrix.json and produce a structured diff.

**Tasks:**
- [ ] Parse capability-matrix.json into comparable format
- [ ] Compare against cached doc content
- [ ] Categorize changes: new features, status changes, deprecations, new models
- [ ] Output structured diff for Claude to interpret
- [ ] Handle partial fetch failures gracefully

### Item 4: /update-claude-capabilities command

**Design doc section:** §5.1
**Priority:** High — the main workflow

The slash command that orchestrates the entire update process.

**Tasks:**
- [ ] Create command markdown file with frontmatter
- [ ] Implement --dry-run flag (report changes without modifying files)
- [ ] Implement --source flag (filter to specific source types)
- [ ] Integrate fetch → diff → interpret → update → verify pipeline
- [ ] Add backup/rollback support (design doc §5.4)
- [ ] Add line count verification (SKILL.md < 500 lines)
- [ ] Add JSON validation for capability-matrix.json
- [ ] Add changelog entry generation

### Item 5: Error Handling & Rollback

**Design doc section:** §5.4
**Priority:** Medium — safety net

**Tasks:**
- [ ] Implement timestamped backup before modifications
- [ ] Implement restore-from-backup on verification failure
- [ ] Add 30-day backup retention cleanup
- [ ] Handle partial update failures (some files updated, others not)
- [ ] Add --rollback flag to manually restore from a specific backup

### Item 6: Plugin Structure (plugin.json)

**Design doc section:** §3.2
**Priority:** Medium — enables distribution

Wrap the skill + command into a plugin for marketplace distribution.

**Tasks:**
- [ ] Create `.claude-plugin/plugin.json` manifest
- [ ] Organize directory structure per design doc §3.2
- [ ] Test plugin installation via `claude plugin install`
- [ ] Verify skill namespacing (`/claude-capabilities:...`)
- [ ] Test command availability after installation

### Item 7: UserPromptSubmit Hook (Optional)

**Design doc section:** §11
**Priority:** Low — optimization for frequent users

Lightweight context injection on every user prompt to remind Claude about capabilities.

**Tasks:**
- [ ] Create hooks.json with UserPromptSubmit configuration
- [ ] Write minimal reminder prompt (< 200 tokens)
- [ ] Test that it triggers skill loading when capability-related
- [ ] Verify it doesn't over-fire on unrelated prompts
- [ ] Make optional/configurable in plugin settings

---

## 5. Evaluation Framework Integration

### Pre-Update Baseline

Before any `/update` run, the evaluation tests (see `.planning/builds/claude-capabilities-skill/evals/`)
should be run to establish a baseline score. After the update, re-run to verify no regression.

**Evaluation checkpoints:**
- [ ] Run evals before Phase 2 development begins (baseline)
- [ ] Run evals after capability-matrix.json is populated
- [ ] Run evals after /update command is functional
- [ ] Run evals after each real update cycle (ongoing)

### Acceptance Criteria for Phase 2

1. `/update-claude-capabilities --dry-run` successfully fetches all monitored URLs and
   produces a structured diff without modifying any files
2. `/update-claude-capabilities` successfully updates skill files when real changes exist
3. Backup and rollback work correctly
4. All Distill monitors are active (no SELECTION_EMPTY errors)
5. Source coverage gap is closed (fetch script covers all monitored URLs)
6. Evaluation scores do not regress after an update cycle
7. SKILL.md stays under 500 lines after update
8. capability-matrix.json validates against schema after update

---

## 6. Implementation Order

| Phase | Items | Estimated Effort |
|-------|-------|-----------------|
| 2a | Fix Distill monitors (Workstream 1) | 30 minutes |
| 2b | Create capability-matrix.json (Item 1) | 2-3 hours |
| 2c | Build fetch-docs.sh (Item 2) | 2-3 hours |
| 2d | Build diff-capabilities.sh (Item 3) | 2-3 hours |
| 2e | Build /update command (Item 4) | 3-4 hours |
| 2f | Error handling & rollback (Item 5) | 1-2 hours |
| 2g | Plugin structure (Item 6) | 1-2 hours |
| 2h | UserPromptSubmit hook (Item 7) | 1 hour |
| 2i | Integration testing | 2-3 hours |

**Total estimated effort:** 15-21 hours across multiple sessions.

**Recommended session sequence:**
1. Fix Distill monitors (2a) — do immediately, browser-based
2. Build capability-matrix.json + fetch/diff scripts (2b-2d) — one session
3. Build /update command + error handling (2e-2f) — one session
4. Plugin structure + hook + integration testing (2g-2i) — one session

---

## 7. Open Questions

1. **Distill plan level:** Does the current Distill plan support cloud JS rendering? This
   determines whether Option A or Option B/C is viable for the SELECTION_EMPTY fix.

2. **code.claude.com monitoring:** Should we add Distill monitors for Claude Code docs
   (code.claude.com)? The skill now covers Claude Code capabilities, but these pages
   aren't currently monitored.

3. **Automatic vs semi-automatic:** Should the `/update` command auto-commit changes, or
   always require manual review? The design doc implies manual review (recommended for
   now).

4. **Community contributions:** When (if ever) should capability-matrix.json be open-sourced
   for community PRs? Phase 3 scope, but worth planning the schema for extensibility.
