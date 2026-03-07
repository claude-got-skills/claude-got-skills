# Freshness Pipeline Spec — 4-Eye Review Findings

## Feasibility Issues

1. **Wrong domain: `support.anthropic.com` should be `support.claude.com`** (source #3). Existing Distill setup uses `support.claude.com`. Will 404 or redirect, causing false-positive diffs every run.
2. **Jina Reader rate limits** — free tier throttles unauthenticated requests. 26 req/day is within limits but retries could spike. Add optional `JINA_API_KEY` support.
3. **code.claude.com pages (sources 7-12)** — Next.js sites may not fully render with Jina Reader. Manually verify before committing to implementation.
4. **GitHub raw content approach is sound** — avoids DOM instability.

## Gaps

1. **No content normalization before hashing** — trafilatura/Jina may return different whitespace, timestamps, nav artifacts on each scrape. Will produce false-positive diffs every run. Must strip dates, normalize whitespace before SHA-256.
2. **URL-to-KB-file mapping not defined** — `config.py` source registry needs explicit `kb_file` field. Some URLs (release notes, CHANGELOG) cross-cut multiple KB files.
3. **KB file coverage mismatch** — 13 monitored URLs don't cover all 28 KB files. State this explicitly as an MVP scope decision.
4. **No alerting on scrape failures** — notifications only fire on changes detected. Should also fire on persistent scrape failures (3+ consecutive).
5. **checkpoint.json single point of failure** — no backup/recovery strategy for corruption or accidental deletion.
6. **No run-history tracking** — IMS has `pipeline-log-helper.sh` logging to database. Freshness pipeline needs at least a log file for debugging missed runs.

## Architecture

1. **Component split is appropriate** — 9 files, follows IMS pattern well
2. **Consider merging `diff.py` into `report.py` for MVP** — it's ~20 lines of `difflib.unified_diff()`. Split out when Phase 2 Claude classification arrives.
3. **`pipeline/` directory location is correct** — co-located with knowledge-base it maintains

## IMS Pattern Accuracy

1. **extract.py**: Accurately described. Copy ~40 relevant lines rather than importing from IMS (avoids pulling in pdfplumber, youtube-transcript-api, etc.)
2. **tana_sync.py**: SHA-256 pattern accurate. Key difference: tana hashes structured fields, pipeline hashes raw content — source of false-positive risk.
3. **launchd**: Directly copy-pasteable from IMS Gmail polling plist.
4. **Shell wrapper**: Missing run-history tracking (IMS `pipeline-log-helper.sh` pattern).

## Scheduling

1. **launchd is correct** — runs without Claude Code, survives reboots
2. **/loop would be wrong** — requires active Claude Code session
3. **Cowork tasks would work but overkill** — consumes API credits, adds latency, unnecessary for pure scraping
4. **launchd sleep constraint** — if Mac is sleeping at 09:00/21:00, job runs on wake (catch-up for most recent interval only). Fine for twice-daily cadence.
5. **Log rotation not addressed** — launchd logs grow unbounded over months

## Top 10 Action Items for Implementation

1. Fix `support.anthropic.com` → `support.claude.com`
2. Add content normalization before SHA-256 hashing
3. Define URL-to-KB-file mapping in config.py
4. Add notification on scrape failures (not just changes)
5. Add optional JINA_API_KEY support
6. Verify Jina Reader works for all 6 code.claude.com pages
7. Merge diff.py into report.py for MVP
8. Add run-history tracking to shell wrapper
9. Note launchd sleep/wake constraint
10. Address log rotation
