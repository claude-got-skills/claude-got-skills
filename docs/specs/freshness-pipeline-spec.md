# Freshness Pipeline Spec: Anthropic Docs Monitoring

**Status:** Spec complete, ready for implementation
**Replaces:** Distill.io monitoring (hit free tier limits, SELECTION_EMPTY errors)
**Location:** `pipeline/` directory in the skills repo
**Schedule:** Twice daily via launchd (09:00, 21:00)
**Estimated effort:** 6-7 hours across 1-2 sessions

## Architecture

Self-hosted Python pipeline that scrapes Anthropic docs, detects changes via SHA-256 hashing, generates change reports, and notifies via macOS notifications. Reuses IMS patterns (extract.py, tana_sync.py, launchd scheduling).

### Key Design Decisions

- **Location: Skills repo** (not IMS or standalone) — pipeline's sole purpose is maintaining the skill
- **Scheduling: launchd** (not /loop or Cowork) — runs without Claude Code, survives reboots
- **Auto-update: No** — generates reports for human review; skill quality depends on editorial judgment
- **Scraping: trafilatura + Jina Reader** — Python-native, no MCP dependency for scheduled runs

## Components

| File | Responsibility |
|------|----------------|
| `pipeline/config.py` | Source URL registry (13 sources), path constants, env loading |
| `pipeline/scrape.py` | Content extraction (trafilatura for server-rendered, Jina Reader for JS-rendered) |
| `pipeline/detect.py` | SHA-256 change detection, checkpoint.json persistence |
| `pipeline/diff.py` | Unified diff generation (Phase 2: Claude-powered classification) |
| `pipeline/report.py` | Markdown change reports with impact analysis |
| `pipeline/notify.py` | macOS notifications via osascript |
| `pipeline/freshness_check.py` | Main orchestrator (entry point) |
| `pipeline/run-freshness-check.sh` | Launchd wrapper (env, venv, logging) |
| `launchd/com.claude-capabilities.freshness-check.plist` | Schedule definition |

## Source URL Registry (13 sources)

| # | URL | Type | Priority |
|---|-----|------|----------|
| 1 | platform.claude.com/docs/en/release-notes/overview | api-docs | HIGH |
| 2 | platform.claude.com/docs/en/overview | api-docs | normal |
| 3 | support.anthropic.com/en/articles/claude-ai-release-notes | support | HIGH |
| 4 | platform.claude.com/docs/en/build-with-claude/overview | api-docs | normal |
| 5 | platform.claude.com/docs/en/about-claude/models/overview | api-docs | normal |
| 6 | github.com/anthropics/claude-code/blob/main/CHANGELOG.md | github | HIGH |
| 7-12 | code.claude.com/docs/{agent-teams,hooks,skills,plugins,cli-reference,extensions} | claude-code | normal |
| 13 | platform.claude.com/docs/en/build-with-claude/context-windows | api-docs | normal |

## Scraping Strategy by Page Type

- **api-docs** (platform.claude.com): trafilatura primary, Jina Reader fallback
- **support** (support.anthropic.com): Jina Reader only (JS-rendered via Intercom)
- **github**: Raw content URL (raw.githubusercontent.com) for pure markdown
- **claude-code** (code.claude.com): Jina Reader (Next.js, JS-rendered)

## Data Flow

```
launchd (twice daily)
  → run-freshness-check.sh
    → freshness_check.py
      → scrape.py (13 sources)
      → detect.py (SHA-256 vs checkpoint.json)
      → diff.py (unified diff for changed sources)
      → report.py → pipeline/reports/freshness-YYYY-MM-DD.md
      → notify.py (macOS notification if changes)
      → update checkpoint.json
```

## MVP vs Future

**MVP (implement now):**
- 13 source URLs, trafilatura + Jina Reader scraping
- SHA-256 change detection with checkpoint.json
- Raw unified diffs (no Claude classification)
- Markdown reports with basic section mapping
- macOS notifications, launchd scheduling

**Phase 2:**
- Claude-powered diff classification (new_feature, deprecation, status_change)
- Auto-update knowledge-base files (--update-kb flag)
- Impact analysis mapping changes to SKILL.md sections
- Auto-run evals after KB update

**Phase 3:**
- Auto-PR generation on dev branch
- New page detection from collection pages
- Historical trend tracking
- Health dashboard

## CLI Usage

```bash
python pipeline/freshness_check.py              # Full check
python pipeline/freshness_check.py --dry-run     # Check without updating checkpoint
python pipeline/freshness_check.py --source URL  # Check single source
python pipeline/freshness_check.py --force       # Re-check all (ignore recency)
python pipeline/freshness_check.py --quiet       # Suppress notifications
```

## Dependencies

- `trafilatura>=1.6.0` — HTML content extraction
- `requests>=2.31.0` — HTTP client for Jina Reader
- Python 3.10+, macOS (for launchd + osascript)
- `.env` with ANTHROPIC_API_KEY (Phase 2 only)

## Implementation Order

1. config.py (30 min) — source registry, paths
2. scrape.py (1 hr) — extraction per page type
3. detect.py (45 min) — hashing, checkpoint
4. diff.py (30 min) — unified diff
5. report.py (45 min) — markdown reports
6. notify.py (15 min) — macOS notifications
7. freshness_check.py (1 hr) — orchestrator
8. run-freshness-check.sh (15 min) — wrapper
9. launchd plist (15 min) — schedule
10. Manual testing (1 hr) — all 13 sources
