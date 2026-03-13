# Automation Pipeline Design: Source Changed to Skill Updated

**Author:** Liam Jones (with Claude)
**Date:** 2026-03-10
**Status:** Design complete, ready for implementation
**Depends on:** Phase 1 freshness pipeline (built, running twice daily via launchd)

---

## 1. Executive Summary

The current freshness pipeline detects changes but stops there. A human must manually
read the diff report, decide what matters, update knowledge-base files, revise SKILL.md
and references, run the test suite, and publish. This design adds six automation stages
that transform the pipeline from "detection only" into an end-to-end "source changed,
skill updated and validated" workflow, with the human staying in the loop for final
review.

### What Changes

| Stage | Current | Proposed |
|-------|---------|----------|
| Scraping | trafilatura + Jina Reader (13 URLs, code.claude.com broken) | trafilatura + Firecrawl (expanded URLs, clean extractions) |
| Change detection | SHA-256 hash comparison | Same + content-length guard + scrape failure detection |
| Classification | None (raw diffs only) | Claude Haiku classifies as BREAKING / NEW_FEATURE / DEPRECATION / BUG_FIX / COSMETIC |
| KB update | Manual | Auto-write new content to mapped KB files with backup |
| Skill update | Manual | Claude proposes edits to SKILL.md and references (not auto-applied) |
| Validation | Manual test runs | Auto-run test suite, gate on no regressions |
| Notification | macOS notification (exists) | Enhanced: review file + optional git branch |

### Estimated Effort

| Stage | Effort | Priority |
|-------|--------|----------|
| Stage 1: Improved scraping | 3-4 hours | P0 (fixes broken code.claude.com) |
| Stage 2: Scrape noise handling | 1-2 hours | P0 (fixes false positives) |
| Stage 3: Change classification | 2-3 hours | P1 (highest value-add) |
| Stage 4: Automated KB update | 2-3 hours | P1 |
| Stage 5: Proposed skill edits | 3-4 hours | P2 |
| Stage 6: Automated validation | 2-3 hours | P2 |
| Stage 7: Review & notification | 1-2 hours | P2 |
| **Total** | **14-21 hours** | |

---

## 2. Architecture Overview

### Data Flow

```
launchd (09:00, 21:00)
  |
  v
run-freshness-check.sh
  |
  v
freshness_check.py [orchestrator]
  |
  +---> scrape.py -------> [22+ sources via trafilatura/Firecrawl/github_raw]
  |                              |
  |                              v
  +---> noise_filter.py --> [normalize, length-guard, boilerplate detection]
  |                              |
  |                              v
  +---> detect.py ----------> [SHA-256 vs checkpoint.json]
  |                              |
  |                    (if changed)
  |                              v
  +---> classify.py ---------> [Claude Haiku: BREAKING|NEW_FEATURE|DEPRECATION|BUG_FIX|COSMETIC]
  |                              |
  |                    (if BREAKING or NEW_FEATURE)
  |                              v
  +---> kb_update.py --------> [write new content to knowledge-base/ with backup]
  |                              |
  |                              v
  +---> propose_edits.py ----> [Claude proposes SKILL.md + reference file edits]
  |                              |
  |                              v
  +---> validation_gate.py --> [run test suite, check for regressions]
  |                              |
  |                              v
  +---> review.py -----------> [generate review file, create git branch, notify]
```

### Execution Modes

The pipeline supports incremental adoption via flags:

```bash
# Current behavior (detection only, unchanged)
python -m pipeline.freshness_check

# With classification (new)
python -m pipeline.freshness_check --classify

# With KB update (new)
python -m pipeline.freshness_check --classify --update-kb

# Full pipeline (new)
python -m pipeline.freshness_check --classify --update-kb --propose-edits --run-tests

# Dry run of full pipeline (no writes, no git)
python -m pipeline.freshness_check --classify --update-kb --propose-edits --run-tests --dry-run
```

This lets you adopt stages one at a time and fall back if anything breaks.

---

## 3. Stage 1: Improved Scraping

### Problem

The code.claude.com pages (6 of 13 sources) use Next.js with client-side rendering.
Jina Reader returns "Page Not Found" boilerplate for all of them. The checkpoint stores
this boilerplate as the "content," and diffs between runs are just noise within the
boilerplate (e.g., a link changing from "We couldn't find the page" to just the nav
elements). This means:

- **Zero useful monitoring** of 6 Claude Code documentation pages
- **False change detections** every time the 404 boilerplate varies slightly
- **Wasted notification budget** (the 3 "changes" in the latest run were all Jina noise)

Additionally, the support.claude.com release notes URL returns a 404 -- the page has
moved or been reorganized, and Jina Reader faithfully captures the 404 page.

### Solution: Replace Jina Reader with Firecrawl for code.claude.com

Firecrawl renders JavaScript, extracts main content, and returns clean markdown. It
costs 1 credit per scrape (222 credits remaining).

**Credit budget analysis:**
- 6 code.claude.com pages, 2 runs/day = 12 credits/day
- 1 support.claude.com page, 2 runs/day = 2 credits/day
- Total: 14 credits/day = ~15 days of monitoring before credits are exhausted
- After that: purchase more credits (pay-as-you-go) or reduce frequency

**Mitigation for credit conservation:**
- Only re-scrape Firecrawl sources if a lightweight HEAD/hash check suggests change
- Cache Firecrawl results and only re-scrape on a configurable interval (e.g., daily
  instead of twice-daily for low-priority sources)
- Use `--only-main-content` flag to reduce extraction noise

### Implementation: `scrape.py` Changes

Add a new extraction function alongside the existing ones:

```python
import subprocess
import json

# Minimum content length to consider a scrape successful
MIN_CONTENT_LENGTH = 1000

# Firecrawl concurrency limit
FIRECRAWL_SEMAPHORE = 2


def extract_with_firecrawl(url: str, only_main_content: bool = True) -> str | None:
    """Extract content using Firecrawl CLI (JS-rendered pages).

    Firecrawl renders the page in a real browser, extracts content,
    and returns clean markdown. Costs 1 credit per scrape.
    """
    cmd = ["firecrawl", "scrape", url, "markdown", "--json"]
    if only_main_content:
        cmd.append("--only-main-content")

    try:
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            timeout=60,
        )

        if result.returncode != 0:
            logger.error("Firecrawl failed for %s: %s", url, result.stderr)
            return None

        data = json.loads(result.stdout)
        content = data.get("markdown", "") or data.get("content", "")

        if not content or len(content) < MIN_CONTENT_LENGTH:
            logger.warning(
                "Firecrawl returned insufficient content for %s (%d chars)",
                url, len(content) if content else 0,
            )
            return None

        return normalize_content(content)

    except subprocess.TimeoutExpired:
        logger.error("Firecrawl timed out for %s", url)
        return None
    except (subprocess.SubprocessError, json.JSONDecodeError, OSError) as e:
        logger.error("Firecrawl extraction failed for %s: %s", url, e)
        return None
```

Update the extraction router in `extract_content()`:

```python
def extract_content(url: str, page_type: str) -> tuple[str | None, str]:
    if page_type == "claude-code":
        # Firecrawl for JS-rendered code.claude.com pages
        content = extract_with_firecrawl(url)
        if content:
            return content, "firecrawl"
        # Fallback to Jina Reader (unlikely to work, but cheap)
        logger.info("Falling back to Jina Reader for %s", url)
        content = extract_with_jina(url)
        return content, "jina_reader"

    if page_type == "support":
        # Support pages are also JS-rendered
        content = extract_with_firecrawl(url)
        if content:
            return content, "firecrawl"
        content = extract_with_jina(url)
        return content, "jina_reader"

    # api-docs and github unchanged...
```

### Source Registry Expansion

Fix the `kb_files` mapping and add missing sources. The current config.py has 13 sources
with kb_files values that don't match actual filenames in `knowledge-base/`. Here is the
corrected and expanded registry:

```python
SOURCES: list[Source] = [
    # -- HIGH PRIORITY ---------------------------------------------------
    Source(
        url="https://platform.claude.com/docs/en/release-notes/overview",
        name="API Release Notes",
        page_type="api-docs",
        priority="HIGH",
        kb_files=["claude-capabilities-features-overview.md"],  # FIXED: was "api-release-notes.md"
    ),
    Source(
        url="https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md",
        name="Claude Code CHANGELOG",
        page_type="github",
        priority="HIGH",
        kb_files=["claude-code-capabilities-cli-reference.md"],  # FIXED
    ),

    # -- API DOCS (platform.claude.com) -----------------------------------
    Source(
        url="https://platform.claude.com/docs/en/overview",
        name="API Overview",
        page_type="api-docs",
        priority="normal",
        kb_files=["claude-capabilities-features-overview.md"],
    ),
    Source(
        url="https://platform.claude.com/docs/en/build-with-claude/overview",
        name="Build with Claude Overview",
        page_type="api-docs",
        priority="normal",
        kb_files=["claude-capabilities-features-overview.md"],
    ),
    Source(
        url="https://platform.claude.com/docs/en/about-claude/models/overview",
        name="Models Overview",
        page_type="api-docs",
        priority="normal",
        kb_files=["claude-capabilities-context-windows.md"],
    ),
    Source(
        url="https://platform.claude.com/docs/en/build-with-claude/context-windows",
        name="Context Windows",
        page_type="api-docs",
        priority="normal",
        kb_files=["claude-capabilities-context-windows.md"],
    ),

    # -- NEW: API docs not previously monitored ---------------------------
    Source(
        url="https://platform.claude.com/docs/en/build-with-claude/tool-use/overview",
        name="Tool Use Overview",
        page_type="api-docs",
        priority="normal",
        kb_files=["claude-capabilities-implement-tool-use.md"],
    ),
    Source(
        url="https://platform.claude.com/docs/en/build-with-claude/computer-use",
        name="Computer Use",
        page_type="api-docs",
        priority="normal",
        kb_files=["claude-capabilities-computer-use.md"],
    ),
    Source(
        url="https://platform.claude.com/docs/en/build-with-claude/agentic-patterns/agent-sdk",
        name="Agent SDK Overview",
        page_type="api-docs",
        priority="normal",
        kb_files=["claude-capabilities-agent-sdk-overview.md"],
    ),
    Source(
        url="https://platform.claude.com/docs/en/build-with-claude/structured-outputs",
        name="Structured Outputs",
        page_type="api-docs",
        priority="normal",
        kb_files=["claude-capabilities-structured-outputs.md"],
    ),
    Source(
        url="https://platform.claude.com/docs/en/build-with-claude/pdf-support",
        name="PDF Support",
        page_type="api-docs",
        priority="normal",
        kb_files=["claude-capabilities-pdf-support.md"],
    ),

    # -- CLAUDE CODE (code.claude.com -> Firecrawl) -----------------------
    Source(
        url="https://code.claude.com/docs/agent-teams",
        name="Claude Code Agent Teams",
        page_type="claude-code",
        priority="normal",
        kb_files=["claude-code-capabilities-agent-teams.md"],  # FIXED
    ),
    Source(
        url="https://code.claude.com/docs/hooks",
        name="Claude Code Hooks",
        page_type="claude-code",
        priority="normal",
        kb_files=["claude-code-capabilities-automate-with-hooks.md"],  # FIXED
    ),
    Source(
        url="https://code.claude.com/docs/skills",
        name="Claude Code Skills",
        page_type="claude-code",
        priority="normal",
        kb_files=["claude-code-capabilities-skills.md"],  # FIXED
    ),
    Source(
        url="https://code.claude.com/docs/plugins",
        name="Claude Code Plugins",
        page_type="claude-code",
        priority="normal",
        kb_files=["claude-code-capabilities-create-plugins.md"],  # FIXED
    ),
    Source(
        url="https://code.claude.com/docs/cli-reference",
        name="Claude Code CLI Reference",
        page_type="claude-code",
        priority="normal",
        kb_files=["claude-code-capabilities-cli-reference.md"],  # FIXED
    ),
    Source(
        url="https://code.claude.com/docs/extensions",
        name="Claude Code Extensions",
        page_type="claude-code",
        priority="normal",
        kb_files=["claude-code-capabilities-extension-options.md"],  # FIXED
    ),

    # -- NEW: Claude Code docs not previously monitored -------------------
    Source(
        url="https://code.claude.com/docs/mcp",
        name="Claude Code MCP",
        page_type="claude-code",
        priority="normal",
        kb_files=["claude-code-capabilities-mcp.md"],
    ),
    Source(
        url="https://code.claude.com/docs/chrome",
        name="Claude Code Chrome Browser",
        page_type="claude-code",
        priority="normal",
        kb_files=["claude-code-capabilities-use-chrome-browser.md"],
    ),
    Source(
        url="https://code.claude.com/docs/sandboxing",
        name="Claude Code Sandboxing",
        page_type="claude-code",
        priority="normal",
        kb_files=["claude-code-capabilities-sandboxing.md"],
    ),

    # -- SUPPORT (support.claude.com -> Firecrawl) ------------------------
    # NOTE: The original release notes URL returns 404. This may need
    # to be updated when the correct URL is found.
    # Source(
    #     url="https://support.claude.com/en/articles/claude-ai-release-notes",
    #     name="Claude AI Release Notes",
    #     page_type="support",
    #     priority="HIGH",
    #     kb_files=[],
    # ),
]
```

This expands coverage from 13 to ~22 sources, and fixes all `kb_files` mappings to use
the actual filenames in `knowledge-base/`.

**KB files still unmapped** (no direct source URL to monitor):

| KB File | Reason |
|---------|--------|
| `claude-capabilities-agent-sdk-python.md` | SDK-specific; changes tracked via Agent SDK overview + CHANGELOG |
| `claude-capabilities-agent-skills-via-api.md` | Tracked via Build with Claude overview |
| `claude-capabilities-files-api.md` | Tracked via Build with Claude overview |
| `claude-capabilities-mcp-via-api.md` | Tracked via Build with Claude overview |
| `claude-capabilities-memory-tool.md` | Tracked via Build with Claude overview |
| `claude-capabilities-new-in-opus-4-6.md` | Tracked via Models overview + Release Notes |
| `claude-capabilities-programmatic-tool-calling.md` | Tracked via Build with Claude overview |
| `claude-capabilities-search-results-citations.md` | Tracked via Build with Claude overview |
| `claude-code-capabilities-create-custom-subagents.md` | Tracked via Agent Teams page |
| `claude-code-capabilities-plugin-reference.md` | Tracked via Plugins page |
| `claude-code-capabilities-use-cli.md` | Tracked via CLI Reference page |
| `mcp-apps-overview.md` | No single source URL (JS-tabbed page) |

These 12 files are indirectly covered by other monitored pages. When a change is
detected on a parent page, the classification stage can flag related KB files for review.

### Firecrawl Credit Conservation

To avoid burning through 222 credits in 16 days:

```python
@dataclass
class Source:
    url: str
    name: str
    page_type: str
    priority: str
    kb_files: list[str]
    scrape_interval_hours: int = 12  # NEW: per-source interval

    @property
    def uses_firecrawl(self) -> bool:
        return self.page_type in ("claude-code", "support")
```

In `freshness_check.py`, skip Firecrawl sources that were checked recently:

```python
def should_scrape(source: Source, checkpoint: dict, force: bool) -> bool:
    """Check if enough time has elapsed since the last scrape."""
    if force:
        return True
    entry = checkpoint.get("sources", {}).get(source.url)
    if entry is None:
        return True

    last_checked = entry.get("last_checked")
    if not last_checked:
        return True

    from datetime import datetime, timezone, timedelta
    last_dt = datetime.fromisoformat(last_checked)
    elapsed = datetime.now(timezone.utc) - last_dt
    min_interval = timedelta(hours=source.scrape_interval_hours)

    return elapsed >= min_interval
```

With this, Firecrawl sources default to 12-hour intervals (matching the twice-daily
schedule) but can be set to 24-hour intervals to halve credit usage:

- 9 Firecrawl sources at 24h interval = 9 credits/day = 24 days of monitoring
- 9 Firecrawl sources at 12h interval = 18 credits/day = 12 days of monitoring

Recommendation: Set code.claude.com sources to 24h, keep github/api-docs at 12h (free).

---

## 4. Stage 2: Scrape Noise Handling

### Problem

Even with content normalization, several noise sources cause false change detections:

1. **Jina Reader "Page Not Found" pages** (~4,400 chars of nav boilerplate)
2. **Cookie consent banners** ("We use cookies to deliver and improve...")
3. **"Loading..." placeholders** from trafilatura hitting pages before JS completes
4. **Dynamic nav elements** (image hashes, CDN URLs that rotate)

### Solution: Multi-Layer Noise Filter

Create `pipeline/noise_filter.py`:

```python
"""Content noise filtering to reduce false-positive change detections."""

import re
import logging

logger = logging.getLogger(__name__)

# Known boilerplate patterns that indicate scrape failures
SCRAPE_FAILURE_PATTERNS = [
    r"Page Not Found",
    r"Uh oh\. That page doesn.t exist",
    r"We couldn.t find the page",
    r"^Loading\.\.\.\n(Loading\.\.\.\n){3,}",  # Repeated "Loading..." lines
    r"404:\s*Not Found",
]

# Content to strip before hashing (dynamic elements that change between scrapes)
STRIP_PATTERNS = [
    # Cookie consent banners
    r"We use cookies to deliver and improve our services.*?Cookie Policy here\.",
    # Jina Reader metadata headers
    r"^Title:.*?\n\nURL Source:.*?\n\n(?:Warning:.*?\n\n)?Markdown Content:\n",
    # Navigation boilerplate from Jina Reader
    r"\[Skip to main content\].*?\n",
    # CDN image URLs with rotating hashes
    r"!\[Image \d+:.*?\]\(https://(?:mintcdn|downloads\.intercomcdn)\.com/[^\)]+\)",
    # "Was this page helpful?" footer
    r"\nWas this page helpful\?$",
    # Ctrl+I keyboard shortcut artifacts
    r"\nCtrl\+I\n",
]

# Minimum content length for a successful scrape
MIN_CONTENT_LENGTH = 1000


def detect_scrape_failure(content: str, url: str) -> bool:
    """Check if content looks like a scrape failure rather than real page content.

    Returns True if the content appears to be a failure/error page.
    """
    if len(content) < MIN_CONTENT_LENGTH:
        logger.warning(
            "Content too short (%d chars) for %s -- likely scrape failure",
            len(content), url,
        )
        return True

    for pattern in SCRAPE_FAILURE_PATTERNS:
        if re.search(pattern, content, re.MULTILINE | re.DOTALL):
            logger.warning(
                "Scrape failure pattern detected for %s: %s",
                url, pattern[:50],
            )
            return True

    return False


def detect_content_regression(
    new_content: str,
    previous_content: str | None,
    url: str,
    threshold: float = 0.5,
) -> bool:
    """Check if new content is suspiciously shorter than previous content.

    A dramatic length decrease often indicates a scrape failure rather than
    a real content change. Returns True if regression detected.
    """
    if previous_content is None:
        return False

    prev_len = len(previous_content)
    new_len = len(new_content)

    if prev_len == 0:
        return False

    ratio = new_len / prev_len
    if ratio < threshold:
        logger.warning(
            "Content regression for %s: %d -> %d chars (%.0f%% decrease)",
            url, prev_len, new_len, (1 - ratio) * 100,
        )
        return True

    return False


def strip_noise(content: str) -> str:
    """Remove known dynamic/noisy elements before hashing.

    This produces a "stable" version of the content for comparison purposes.
    The full original content is still stored in the checkpoint for diff display.
    """
    cleaned = content
    for pattern in STRIP_PATTERNS:
        cleaned = re.sub(pattern, "", cleaned, flags=re.MULTILINE | re.DOTALL)

    # Collapse resulting multiple blank lines
    cleaned = re.sub(r"\n{3,}", "\n\n", cleaned)
    return cleaned.strip()
```

### Integration with Detection

Modify `freshness_check.py` to use noise filtering:

```python
from .noise_filter import detect_scrape_failure, detect_content_regression, strip_noise

# In the source processing loop:

# After extraction succeeds:
if detect_scrape_failure(content, source.url):
    consecutive = record_scrape_failure(checkpoint, source.url)
    result = SourceResult(
        source=source,
        status="error",
        error_message="Content appears to be a scrape failure (boilerplate/404)",
        consecutive_failures=consecutive,
    )
    report.results.append(result)
    continue

# Check for content regression
previous_content = get_previous_content(checkpoint, source.url)
if detect_content_regression(content, previous_content, source.url):
    consecutive = record_scrape_failure(checkpoint, source.url)
    result = SourceResult(
        source=source,
        status="error",
        error_message=f"Content regression detected ({len(content)} chars vs {len(previous_content)} previously)",
        consecutive_failures=consecutive,
    )
    report.results.append(result)
    continue

# Use noise-stripped content for hashing (but store full content)
stable_content = strip_noise(content)
has_changed, new_hash, old_hash = detect_change(source.url, stable_content, checkpoint)
```

### Impact

This eliminates the three categories of false positives observed in the most recent run:
- **Plugins** diff was 0/-2 within boilerplate ("We couldn't find the page" removed)
- **CLI Reference** diff was 0/-4 within boilerplate (suggestion links removed)
- Both would now be caught by `detect_scrape_failure()` and reported as errors

---

## 5. Stage 3: Change Classification (Claude-Powered)

### Purpose

Not all changes matter equally. A typo fix in the docs should not trigger a skill
update. A new model announcement absolutely should. Classification separates signal
from noise at the semantic level.

### Classification Categories

| Category | Description | Action |
|----------|-------------|--------|
| `BREAKING` | API removal, model retirement, parameter rename, behavior change | Trigger full pipeline (KB + SKILL.md + tests) |
| `NEW_FEATURE` | New model, new tool, new API capability, new parameter | Trigger full pipeline |
| `DEPRECATION` | Model/feature deprecation notice (not yet removed) | Update KB + note in SKILL.md |
| `BUG_FIX` | Fix to existing behavior, no new capabilities | Update KB only |
| `COSMETIC` | Typo, formatting, nav change, doc reorganization | Skip (no update needed) |

### Implementation: `pipeline/classify.py`

```python
"""Claude-powered change classification using Haiku for cost efficiency."""

import json
import logging
import os
from dataclasses import dataclass
from enum import Enum

logger = logging.getLogger(__name__)


class ChangeCategory(Enum):
    BREAKING = "BREAKING"
    NEW_FEATURE = "NEW_FEATURE"
    DEPRECATION = "DEPRECATION"
    BUG_FIX = "BUG_FIX"
    COSMETIC = "COSMETIC"


# Categories that should trigger downstream updates
ACTIONABLE_CATEGORIES = {ChangeCategory.BREAKING, ChangeCategory.NEW_FEATURE, ChangeCategory.DEPRECATION}


@dataclass
class Classification:
    category: ChangeCategory
    confidence: float           # 0.0 to 1.0
    summary: str                # 1-2 sentence summary of the change
    affected_features: list[str]  # e.g., ["structured_outputs", "opus_4.6"]
    skill_sections: list[str]   # Which SKILL.md sections need updating
    raw_response: str           # Full Claude response for debugging


CLASSIFICATION_PROMPT = """You are a documentation change classifier for a Claude capabilities tracking system.

Given a diff between the previous and current versions of an Anthropic documentation page, classify the change into exactly one category:

- BREAKING: API removal, model retirement, parameter rename/move, breaking behavior change
- NEW_FEATURE: New model release, new tool, new API capability, new parameter, new platform support
- DEPRECATION: Model or feature deprecation notice (announced but not yet removed)
- BUG_FIX: Fix to existing documented behavior, SDK bugfix, no new capabilities
- COSMETIC: Typo fix, formatting change, navigation update, doc reorganization, URL change, boilerplate change

Respond with a JSON object (no markdown fences):
{{
  "category": "BREAKING|NEW_FEATURE|DEPRECATION|BUG_FIX|COSMETIC",
  "confidence": 0.0-1.0,
  "summary": "1-2 sentence description of what changed",
  "affected_features": ["feature_name_1", "feature_name_2"],
  "skill_sections": ["Current Models", "Core Capabilities", "Claude Code"]
}}

Rules:
- If the diff contains MULTIPLE changes of different categories, classify by the MOST impactful one.
- If the diff is mostly boilerplate/navigation changes with one real change, classify the real change.
- "Loading..." artifacts and cookie banners are ALWAYS COSMETIC.
- CHANGELOG entries for Claude Code with new features count as NEW_FEATURE.
- Model ID changes (e.g., new alias) count as NEW_FEATURE.
- Price changes count as BREAKING.

Source: {source_name} ({source_url})
KB files: {kb_files}

Diff:
```
{diff_content}
```"""


def classify_change(
    diff_content: str,
    source_name: str,
    source_url: str,
    kb_files: list[str],
) -> Classification | None:
    """Classify a documentation change using Claude Haiku.

    Returns None if classification fails.
    """
    try:
        import anthropic
    except ImportError:
        logger.error("anthropic package not installed -- cannot classify")
        return None

    api_key = os.environ.get("ANTHROPIC_API_KEY")
    if not api_key:
        logger.error("ANTHROPIC_API_KEY not set -- cannot classify")
        return None

    # Truncate very large diffs to control token usage
    max_diff_chars = 8000
    truncated = diff_content[:max_diff_chars]
    if len(diff_content) > max_diff_chars:
        truncated += f"\n\n... (truncated, {len(diff_content) - max_diff_chars} chars omitted)"

    prompt = CLASSIFICATION_PROMPT.format(
        source_name=source_name,
        source_url=source_url,
        kb_files=", ".join(kb_files),
        diff_content=truncated,
    )

    try:
        client = anthropic.Anthropic(api_key=api_key)
        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=512,
            messages=[{"role": "user", "content": prompt}],
        )

        text = response.content[0].text.strip()

        # Parse JSON response
        data = json.loads(text)

        return Classification(
            category=ChangeCategory(data["category"]),
            confidence=float(data.get("confidence", 0.5)),
            summary=data.get("summary", ""),
            affected_features=data.get("affected_features", []),
            skill_sections=data.get("skill_sections", []),
            raw_response=text,
        )

    except (json.JSONDecodeError, KeyError, ValueError) as e:
        logger.error("Failed to parse classification response: %s", e)
        return None
    except Exception as e:
        logger.error("API error during classification: %s", e)
        return None
```

### Cost Analysis

**Claude Haiku 4.5 pricing:** $1/MTok input, $5/MTok output

Per classification call:
- Input: ~2,000 tokens (prompt template) + ~2,000 tokens (diff) = ~4,000 tokens
- Output: ~100 tokens (JSON response)
- Cost per call: $0.004 input + $0.0005 output = **~$0.0045**

Expected usage:
- Average 2-3 real changes per day (excluding cosmetic)
- Even at 13 changes/day (all sources changed): $0.06/day
- Monthly budget: **~$1.80** maximum

This is negligible. We can afford to classify every detected change.

---

## 6. Stage 4: Automated KB Update

### Purpose

When a source changes, write the new scraped content directly to the corresponding
knowledge-base file. This keeps the KB current without manual intervention.

### Implementation: `pipeline/kb_update.py`

```python
"""Automated knowledge-base file updates with backup."""

import logging
import shutil
from datetime import datetime, timezone
from pathlib import Path

from .config import PROJECT_DIR

logger = logging.getLogger(__name__)

KB_DIR = PROJECT_DIR / "knowledge-base"
BACKUP_DIR = PROJECT_DIR / "knowledge-base" / ".backups"


def backup_kb_file(kb_filename: str) -> Path | None:
    """Create a timestamped backup of a KB file before overwriting.

    Returns the backup path, or None if the file doesn't exist.
    """
    source_path = KB_DIR / kb_filename
    if not source_path.exists():
        return None

    BACKUP_DIR.mkdir(exist_ok=True)

    timestamp = datetime.now(timezone.utc).strftime("%Y%m%d-%H%M%S")
    backup_name = f"{source_path.stem}.{timestamp}{source_path.suffix}"
    backup_path = BACKUP_DIR / backup_name

    shutil.copy2(source_path, backup_path)
    logger.info("Backed up %s -> %s", kb_filename, backup_path.name)
    return backup_path


def update_kb_file(
    kb_filename: str,
    new_content: str,
    source_name: str,
    source_url: str,
    dry_run: bool = False,
) -> bool:
    """Update a knowledge-base file with new scraped content.

    Creates a backup first, then writes the new content with a metadata header.

    Returns True if the file was updated (or would be in dry-run mode).
    """
    target_path = KB_DIR / kb_filename

    if dry_run:
        logger.info("[DRY RUN] Would update %s (%d chars)", kb_filename, len(new_content))
        return True

    # Backup existing file
    backup_kb_file(kb_filename)

    # Write new content with metadata header
    header = (
        f"<!-- Source: {source_url} -->\n"
        f"<!-- Scraped: {datetime.now(timezone.utc).isoformat()} -->\n"
        f"<!-- Source name: {source_name} -->\n\n"
    )

    try:
        with open(target_path, "w") as f:
            f.write(header + new_content)
        logger.info("Updated %s (%d chars)", kb_filename, len(new_content))
        return True
    except OSError as e:
        logger.error("Failed to write %s: %s", kb_filename, e)
        return False


def cleanup_old_backups(max_age_days: int = 30) -> int:
    """Remove backup files older than max_age_days. Returns count removed."""
    if not BACKUP_DIR.exists():
        return 0

    cutoff = datetime.now(timezone.utc).timestamp() - (max_age_days * 86400)
    removed = 0

    for backup_file in BACKUP_DIR.iterdir():
        if backup_file.stat().st_mtime < cutoff:
            backup_file.unlink()
            removed += 1

    if removed:
        logger.info("Cleaned up %d old backup(s)", removed)
    return removed
```

### Integration

In `freshness_check.py`, after classification determines a change is actionable:

```python
from .kb_update import update_kb_file, cleanup_old_backups
from .classify import classify_change, ACTIONABLE_CATEGORIES

# After detecting a change and classifying it:
if args.update_kb and classification and classification.category in ACTIONABLE_CATEGORIES:
    for kb_file in source.kb_files:
        update_kb_file(
            kb_filename=kb_file,
            new_content=content,
            source_name=source.name,
            source_url=source.url,
            dry_run=args.dry_run,
        )

# At the end of a run:
if not args.dry_run:
    cleanup_old_backups()
```

### Gitignore

Add to `.gitignore`:
```
knowledge-base/.backups/
```

---

## 7. Stage 5: Proposed Skill/Reference Edits

### Purpose

When a classified change requires skill updates, use Claude to propose specific edits
to SKILL.md and reference files. These are proposals, not auto-applied changes. They are
written to a review file for human inspection.

### Implementation: `pipeline/propose_edits.py`

```python
"""Generate proposed edits to SKILL.md and reference files based on classified changes."""

import json
import logging
import os
from dataclasses import dataclass
from pathlib import Path

from .config import PROJECT_DIR

logger = logging.getLogger(__name__)

SKILL_DIR = PROJECT_DIR / "skills" / "assistant-capabilities"
SKILL_PATH = SKILL_DIR / "SKILL.md"
REFERENCES_DIR = SKILL_DIR / "references"


@dataclass
class ProposedEdit:
    file_path: str          # Relative to project root
    section: str            # Which section to edit
    current_text: str       # The text to replace (for context)
    proposed_text: str      # The replacement text
    rationale: str          # Why this change is needed
    classification_summary: str


EDIT_PROPOSAL_PROMPT = """You are updating a Claude capabilities awareness skill based on documentation changes.

The skill has these files:
- SKILL.md: ~250 lines, always loaded. Contains concise current-state summaries.
- references/model-specifics.md: Model IDs, pricing, capability matrix
- references/api-features.md: API feature details, availability, parameters
- references/tool-types.md: Tool categories and details
- references/agent-capabilities.md: Agent SDK, teams, subagents
- references/claude-code-specifics.md: Claude Code features, CLI, hooks, plugins

Given the following documentation change, propose specific edits to the relevant file(s).

IMPORTANT RULES:
1. Keep SKILL.md under 250 lines. Move details to reference files.
2. Be precise: provide exact old text and new text for each edit.
3. Only change what's necessary -- do not rewrite sections that aren't affected.
4. Use the same formatting style as the existing content.
5. If a change adds a new capability, add it to the appropriate section.
6. If a change deprecates something, update status but keep the entry (for awareness).
7. If you're unsure about a change, say so -- the human reviewer will decide.

Classification: {category} (confidence: {confidence})
Summary: {summary}
Source: {source_name}
Affected features: {affected_features}

Current SKILL.md content:
```
{skill_content}
```

Relevant reference file content ({reference_file}):
```
{reference_content}
```

Documentation diff:
```
{diff_content}
```

Respond with a JSON array of proposed edits:
[
  {{
    "file": "SKILL.md or references/filename.md",
    "section": "section name",
    "old_text": "exact text to find and replace",
    "new_text": "replacement text",
    "rationale": "why this change"
  }}
]

If no edits are needed (the skill already reflects this change), return an empty array: []"""


def load_skill_content() -> str:
    """Load current SKILL.md content."""
    if SKILL_PATH.exists():
        return SKILL_PATH.read_text()
    return ""


def load_reference_content(reference_name: str) -> str:
    """Load a reference file's content."""
    ref_path = REFERENCES_DIR / reference_name
    if ref_path.exists():
        return ref_path.read_text()
    return ""


def determine_reference_file(skill_sections: list[str]) -> str:
    """Map skill sections to the most relevant reference file."""
    section_map = {
        "Current Models": "model-specifics.md",
        "Core Capabilities": "api-features.md",
        "Tools": "tool-types.md",
        "Agent": "agent-capabilities.md",
        "Claude Code": "claude-code-specifics.md",
    }
    for section in skill_sections:
        for key, ref_file in section_map.items():
            if key.lower() in section.lower():
                return ref_file
    return "api-features.md"  # default


def propose_edits(
    classification,  # classify.Classification
    diff_content: str,
    source_name: str,
) -> list[ProposedEdit]:
    """Use Claude to propose specific edits to skill files.

    Returns a list of ProposedEdit objects, or empty list if no edits needed.
    """
    try:
        import anthropic
    except ImportError:
        logger.error("anthropic package not installed")
        return []

    api_key = os.environ.get("ANTHROPIC_API_KEY")
    if not api_key:
        logger.error("ANTHROPIC_API_KEY not set")
        return []

    skill_content = load_skill_content()
    reference_file = determine_reference_file(classification.skill_sections)
    reference_content = load_reference_content(reference_file)

    # Truncate inputs to control costs (Sonnet for higher quality proposals)
    max_diff = 6000
    max_skill = 8000
    max_ref = 6000

    prompt = EDIT_PROPOSAL_PROMPT.format(
        category=classification.category.value,
        confidence=classification.confidence,
        summary=classification.summary,
        source_name=source_name,
        affected_features=", ".join(classification.affected_features),
        skill_content=skill_content[:max_skill],
        reference_file=reference_file,
        reference_content=reference_content[:max_ref],
        diff_content=diff_content[:max_diff],
    )

    try:
        client = anthropic.Anthropic(api_key=api_key)
        # Use Sonnet for edit proposals -- higher quality than Haiku
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}],
        )

        text = response.content[0].text.strip()
        edits_data = json.loads(text)

        edits = []
        for edit_item in edits_data:
            edits.append(ProposedEdit(
                file_path=edit_item["file"],
                section=edit_item.get("section", ""),
                current_text=edit_item.get("old_text", ""),
                proposed_text=edit_item.get("new_text", ""),
                rationale=edit_item.get("rationale", ""),
                classification_summary=classification.summary,
            ))

        logger.info("Generated %d proposed edit(s)", len(edits))
        return edits

    except (json.JSONDecodeError, KeyError) as e:
        logger.error("Failed to parse edit proposals: %s", e)
        return []
    except Exception as e:
        logger.error("API error generating edit proposals: %s", e)
        return []
```

### Cost Analysis for Edit Proposals

**Claude Sonnet 4.6 pricing:** $3/MTok input, $15/MTok output

Per proposal call:
- Input: ~6,000 tokens (prompt + skill + reference + diff)
- Output: ~500 tokens (JSON edits)
- Cost per call: $0.018 input + $0.0075 output = **~$0.026**

Expected usage:
- 1-2 actionable changes per week (most days nothing changes)
- Monthly: ~$0.20-0.40

Could use Haiku instead ($0.0045/call) at the cost of lower-quality proposals. Sonnet
is recommended because edit quality directly affects skill quality, and the cost is
trivial.

---

## 8. Stage 6: Automated Validation

### Purpose

After proposing edits, automatically run the test suite to catch regressions before
any human review. This provides a safety net: even if a proposed edit looks reasonable,
if it degrades test scores, it should be flagged.

### Implementation: `pipeline/validation_gate.py`

```python
"""Run the test suite and gate on no regressions."""

import json
import logging
import subprocess
import sys
from pathlib import Path

from .config import PROJECT_DIR

logger = logging.getLogger(__name__)

RUNNER_SCRIPT = PROJECT_DIR / "evals" / "eval_runner.py"
RESULTS_DIR = PROJECT_DIR / "evals"


def get_latest_results() -> dict | None:
    """Find and load the most recent test results JSON."""
    results_files = sorted(
        RESULTS_DIR.glob("eval-results-*.json"),
        key=lambda p: p.stat().st_mtime,
        reverse=True,
    )
    if not results_files:
        return None

    try:
        with open(results_files[0]) as f:
            return json.load(f)
    except (json.JSONDecodeError, OSError) as e:
        logger.error("Failed to load results: %s", e)
        return None


def extract_scores(results: dict) -> dict[str, float]:
    """Extract per-category scores from results."""
    scores = {}
    for test in results.get("tests", []):
        category = test.get("category", "unknown")
        treatment_score = test.get("treatment_score", 0)
        if category not in scores:
            scores[category] = []
        scores[category].append(treatment_score)

    # Average per category
    return {cat: sum(vals) / len(vals) for cat, vals in scores.items() if vals}


def run_validation(dry_run: bool = False) -> tuple[bool, dict | None, str]:
    """Run the test suite and return (passed, results, summary).

    The gate passes if:
    1. The runner executes successfully
    2. No category score decreases by more than 10%
    3. Overall score does not decrease

    Returns:
        (passed: bool, results: dict | None, summary: str)
    """
    if dry_run:
        return True, None, "[DRY RUN] Tests would run here"

    # Get baseline scores before running
    baseline_results = get_latest_results()
    baseline_scores = extract_scores(baseline_results) if baseline_results else {}

    logger.info("Running test suite...")
    try:
        result = subprocess.run(
            [sys.executable, str(RUNNER_SCRIPT)],
            capture_output=True,
            text=True,
            timeout=600,  # 10 minute timeout
            cwd=str(PROJECT_DIR),
        )

        if result.returncode != 0:
            return False, None, f"Runner failed with exit code {result.returncode}: {result.stderr[:500]}"

    except subprocess.TimeoutExpired:
        return False, None, "Runner timed out after 10 minutes"
    except subprocess.SubprocessError as e:
        return False, None, f"Runner error: {e}"

    # Load new results
    new_results = get_latest_results()
    if not new_results:
        return False, None, "No results file found after running"

    new_scores = extract_scores(new_results)

    # Compare scores
    regressions = []
    for category, new_score in new_scores.items():
        baseline_score = baseline_scores.get(category, 0)
        if baseline_score > 0 and new_score < baseline_score * 0.9:
            regressions.append(
                f"{category}: {baseline_score:.1f} -> {new_score:.1f} "
                f"({(new_score - baseline_score) / baseline_score * 100:+.0f}%)"
            )

    if regressions:
        summary = "REGRESSIONS DETECTED:\n" + "\n".join(f"  - {r}" for r in regressions)
        return False, new_results, summary

    summary_lines = [f"All {len(new_scores)} categories passed (no regressions)"]
    for category, score in sorted(new_scores.items()):
        baseline = baseline_scores.get(category, 0)
        delta = f" ({score - baseline:+.1f})" if baseline else " (new)"
        summary_lines.append(f"  {category}: {score:.1f}{delta}")

    return True, new_results, "\n".join(summary_lines)
```

### Cost Analysis

The test runner uses Haiku for both test responses and LLM-as-judge scoring.

Per test run (33 tests):
- Control: 33 calls at ~1,000 tokens each = ~33K tokens
- Treatment: 33 calls at ~3,000 tokens each (includes SKILL.md) = ~99K tokens
- Judge: 66 calls at ~2,000 tokens each = ~132K tokens
- Total: ~264K input tokens + ~66K output tokens
- Cost: $0.264 input + $0.330 output = **~$0.60 per run**

This runs only when edits are proposed -- maybe once or twice a week. Monthly: ~$5.

---

## 9. Stage 7: Review & Notification

### Purpose

Generate a comprehensive review artifact and notify the human. The review file contains
everything needed to approve or reject the proposed changes.

### Implementation: `pipeline/review.py`

```python
"""Generate review artifacts and optionally create a git branch."""

import logging
import subprocess
from datetime import datetime, timezone
from pathlib import Path

from .config import PROJECT_DIR, REPORTS_DIR

logger = logging.getLogger(__name__)


def generate_review_file(
    classifications: list,      # list of (SourceResult, Classification) tuples
    proposed_edits: list,       # list of ProposedEdit
    test_summary: str,
    tests_passed: bool,
) -> Path:
    """Generate a markdown review file with all proposed changes.

    This is the primary artifact for human review.
    """
    timestamp = datetime.now(timezone.utc).strftime("%Y-%m-%d-%H%M")
    review_path = REPORTS_DIR / f"review-{timestamp}.md"

    lines = [
        "# Automated Update Review",
        "",
        f"**Generated:** {datetime.now(timezone.utc).isoformat()}",
        f"**Validation gate:** {'PASSED' if tests_passed else 'FAILED'}",
        f"**Proposed edits:** {len(proposed_edits)}",
        "",
    ]

    # Classification summary
    lines.append("## Classified Changes")
    lines.append("")
    for result, classification in classifications:
        tag = {
            "BREAKING": "[!!!]",
            "NEW_FEATURE": "[NEW]",
            "DEPRECATION": "[DEP]",
            "BUG_FIX": "[FIX]",
            "COSMETIC": "[---]",
        }.get(classification.category.value, "[???]")

        lines.append(f"### {tag} {result.source.name}")
        lines.append(f"- **Category:** {classification.category.value} (confidence: {classification.confidence:.0%})")
        lines.append(f"- **Summary:** {classification.summary}")
        lines.append(f"- **Affected features:** {', '.join(classification.affected_features)}")
        lines.append(f"- **Skill sections:** {', '.join(classification.skill_sections)}")
        lines.append("")

    # Proposed edits
    if proposed_edits:
        lines.append("## Proposed Edits")
        lines.append("")
        for i, edit in enumerate(proposed_edits, 1):
            lines.append(f"### Edit {i}: {edit.file_path} ({edit.section})")
            lines.append(f"**Rationale:** {edit.rationale}")
            lines.append("")
            lines.append("**Current:**")
            lines.append("```")
            lines.append(edit.current_text)
            lines.append("```")
            lines.append("")
            lines.append("**Proposed:**")
            lines.append("```")
            lines.append(edit.proposed_text)
            lines.append("```")
            lines.append("")

    # Test results
    lines.append("## Validation Results")
    lines.append("")
    lines.append(f"```\n{test_summary}\n```")
    lines.append("")

    # Action items
    lines.append("## Action Items")
    lines.append("")
    if tests_passed and proposed_edits:
        lines.append("- [ ] Review proposed edits above")
        lines.append("- [ ] Apply approved edits to skill files")
        lines.append("- [ ] Commit and push to dev branch")
        lines.append("- [ ] Run manual spot-check on Claude.ai")
    elif not tests_passed:
        lines.append("- [ ] Investigate test regressions")
        lines.append("- [ ] Revise proposed edits")
        lines.append("- [ ] Re-run tests")
    else:
        lines.append("- [ ] No edits proposed -- changes may be cosmetic or already reflected")
    lines.append("")

    content = "\n".join(lines)
    review_path.write_text(content)
    logger.info("Review file saved: %s", review_path)
    return review_path


def create_update_branch(
    review_path: Path,
    classifications: list,
    dry_run: bool = False,
) -> str | None:
    """Create a git branch with KB updates for easy review.

    Returns the branch name, or None if git operations fail.
    """
    timestamp = datetime.now(timezone.utc).strftime("%Y%m%d-%H%M")
    branch_name = f"auto-update/{timestamp}"

    if dry_run:
        logger.info("[DRY RUN] Would create branch: %s", branch_name)
        return branch_name

    try:
        # Check if we're in a git repo and on dev branch
        result = subprocess.run(
            ["git", "rev-parse", "--abbrev-ref", "HEAD"],
            capture_output=True, text=True, cwd=str(PROJECT_DIR),
        )
        current_branch = result.stdout.strip()

        if current_branch != "dev":
            logger.warning("Not on dev branch (on %s) -- skipping branch creation", current_branch)
            return None

        # Create and checkout new branch
        subprocess.run(
            ["git", "checkout", "-b", branch_name],
            capture_output=True, text=True, cwd=str(PROJECT_DIR),
            check=True,
        )

        # Stage KB changes and review file
        subprocess.run(
            ["git", "add", "knowledge-base/", str(review_path)],
            capture_output=True, text=True, cwd=str(PROJECT_DIR),
            check=True,
        )

        # Create summary for commit message
        change_summaries = [c.summary for _, c in classifications if c.summary]
        commit_msg = "auto: update KB from detected documentation changes\n\n"
        commit_msg += "\n".join(f"- {s}" for s in change_summaries[:5])

        subprocess.run(
            ["git", "commit", "-m", commit_msg],
            capture_output=True, text=True, cwd=str(PROJECT_DIR),
            check=True,
        )

        # Switch back to dev
        subprocess.run(
            ["git", "checkout", "dev"],
            capture_output=True, text=True, cwd=str(PROJECT_DIR),
        )

        logger.info("Created branch: %s", branch_name)
        return branch_name

    except subprocess.CalledProcessError as e:
        logger.error("Git operation failed: %s", e.stderr)
        # Try to get back to dev branch
        subprocess.run(
            ["git", "checkout", "dev"],
            capture_output=True, text=True, cwd=str(PROJECT_DIR),
        )
        return None
```

### Enhanced Notification

Update `notify.py` to include classification details:

```python
def notify_classified_changes(
    classifications: list,   # list of (SourceResult, Classification) tuples
    review_path: Path,
    tests_passed: bool,
) -> bool:
    """Send a rich notification about classified changes."""
    actionable = [
        (r, c) for r, c in classifications
        if c.category.value in ("BREAKING", "NEW_FEATURE", "DEPRECATION")
    ]

    if not actionable:
        return send_notification(
            title="Freshness Check: Cosmetic Changes Only",
            message=f"{len(classifications)} change(s) detected, all cosmetic/bugfix. No action needed.",
            sound="",
        )

    breaking = sum(1 for _, c in actionable if c.category.value == "BREAKING")
    new_features = sum(1 for _, c in actionable if c.category.value == "NEW_FEATURE")

    parts = []
    if breaking:
        parts.append(f"{breaking} breaking")
    if new_features:
        parts.append(f"{new_features} new feature(s)")

    subtitle = ", ".join(parts)
    test_status = "Tests: PASSED" if tests_passed else "Tests: FAILED"

    return send_notification(
        title=f"Freshness Check: {len(actionable)} Actionable Change(s)",
        message=f"{subtitle}. {test_status}. Review: {review_path.name}",
        subtitle=subtitle,
        sound="Hero" if breaking else "default",
    )
```

---

## 10. Error Handling Strategy

### Failure Modes and Recovery

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Firecrawl timeout | subprocess.TimeoutExpired | Log error, fall back to Jina Reader (won't work but records attempt), increment failure counter |
| Firecrawl credits exhausted | HTTP 402 or empty response | Log warning, skip Firecrawl sources, continue with free sources. Notify user. |
| Anthropic API error (classification) | Exception from API client | Log error, skip classification, report change as unclassified. Human reviews raw diff. |
| Anthropic API error (edit proposal) | Exception from API client | Log error, skip edit proposals. KB update still proceeds. |
| Test runner crash | subprocess return code != 0 | Log error, report tests as failed. Do not gate -- let human decide. |
| KB file write failure | OSError | Log error, skip this KB file. Other files still updated. |
| Git branch creation failure | subprocess.CalledProcessError | Log error, return to dev branch. Review file still generated. |
| Checkpoint corruption | json.JSONDecodeError | Already handled -- returns empty dict, treats all as first-run |
| Content regression (new << previous) | Length ratio check | Mark as scrape failure, do not update checkpoint or KB |

### Retry Policy

No automatic retries within a single run. The pipeline runs twice daily, so a transient
failure will be retried in 12 hours. For persistent failures:

- After 3 consecutive failures: macOS alert notification (already implemented)
- After 7 consecutive failures: source is flagged in the review file as "monitor unhealthy"
- After 14 consecutive failures: source is auto-disabled (skipped in future runs) until
  manually re-enabled. This prevents credit waste on broken sources.

### Rollback

If a KB update introduces bad content:

```bash
# List backups for a specific KB file
ls knowledge-base/.backups/claude-code-capabilities-skills.*

# Restore from backup
cp knowledge-base/.backups/claude-code-capabilities-skills.20260310-090000.md \
   knowledge-base/claude-code-capabilities-skills.md
```

For skill file edits: since edits are proposed (not auto-applied), rollback is just
"don't apply the proposed edit." If an edit was applied and committed, standard git
revert applies.

---

## 11. Cost Summary

### Monthly Steady-State Costs

| Component | Unit Cost | Frequency | Monthly Cost |
|-----------|-----------|-----------|-------------|
| Firecrawl scraping | 1 credit/scrape | 9 sources x 1-2/day | ~270-540 credits/month |
| Haiku classification | ~$0.005/call | 2-3 changes/day | ~$3-5 |
| Sonnet edit proposals | ~$0.026/call | 1-2/week | ~$0.20 |
| Haiku test runs | ~$0.60/run | 1-2/week | ~$5 |
| **Total API costs** | | | **~$8-10/month** |
| **Firecrawl credits** | | | **~270-540 credits/month** |

### Credit Conservation Strategies (if needed)

1. **Reduce Firecrawl frequency**: Scrape code.claude.com pages daily instead of twice-daily (halves usage)
2. **Conditional scraping**: Only Firecrawl-scrape if the GitHub CHANGELOG (free to check) shows a new release
3. **Free sources only**: Monitor only api-docs (trafilatura, free) and GitHub (free), sacrifice code.claude.com monitoring
4. **Batch scraping**: Use Firecrawl's `crawl` command to scrape all code.claude.com pages in one job

---

## 12. Implementation Order (Recommended)

### Sprint 1: Fix What's Broken (P0) -- 4-6 hours

**Goal:** Eliminate false positives and get real monitoring of code.claude.com pages.

1. **Add Firecrawl extraction to scrape.py** (1.5 hours)
   - Implement `extract_with_firecrawl()`
   - Update `extract_content()` routing for `claude-code` and `support` page types
   - Test with one URL: `firecrawl scrape https://code.claude.com/docs/skills markdown --only-main-content`

2. **Add noise filtering** (1 hour)
   - Create `noise_filter.py` with scrape failure detection, content regression check, and noise stripping
   - Integrate into `freshness_check.py`

3. **Fix config.py source registry** (1.5 hours)
   - Correct all `kb_files` mappings to match actual filenames
   - Add `scrape_interval_hours` field
   - Add credit-conservation logic (`should_scrape()`)
   - Remove or comment out the broken support.claude.com release notes URL

4. **Test end-to-end** (1 hour)
   - Run `python -m pipeline.freshness_check --dry-run`
   - Verify Firecrawl returns real content for code.claude.com pages
   - Verify no false positives from noise

### Sprint 2: Classification (P1) -- 2-3 hours

**Goal:** Automatically classify changes so the human only reviews what matters.

1. **Implement classify.py** (1.5 hours)
   - Classification prompt, Haiku API call, JSON parsing
   - Add `--classify` flag to freshness_check.py

2. **Update report.py** (1 hour)
   - Include classification results in the markdown report
   - Show category, confidence, summary alongside each change
   - Filter report sections by actionability

### Sprint 3: KB Update (P1) -- 2-3 hours

**Goal:** Auto-update knowledge-base files when actionable changes are detected.

1. **Implement kb_update.py** (1.5 hours)
   - Backup + write logic
   - Metadata headers
   - Cleanup of old backups

2. **Add --update-kb flag** (1 hour)
   - Integration in freshness_check.py
   - Only trigger for ACTIONABLE_CATEGORIES
   - Test with dry-run

### Sprint 4: Proposed Edits + Validation Gate (P2) -- 5-7 hours

**Goal:** Full pipeline from detection through validated edit proposals.

1. **Implement propose_edits.py** (2 hours)
   - Edit proposal prompt with SKILL.md + reference context
   - Sonnet API call, JSON parsing
   - Reference file routing

2. **Implement validation_gate.py** (1.5 hours)
   - Subprocess call to the test runner
   - Score comparison against baseline
   - Regression detection

3. **Implement review.py** (1.5 hours)
   - Review file generation
   - Git branch creation (optional)
   - Enhanced notifications

4. **Integration testing** (1.5 hours)
   - End-to-end run with all flags enabled
   - Verify review file quality
   - Test dry-run mode

### Sprint 5: Source Expansion + Polish (P2) -- 2-3 hours

**Goal:** Expand monitoring coverage and add production polish.

1. **Add new source URLs** (1 hour)
   - Tool Use, Computer Use, Agent SDK, Structured Outputs, PDF Support
   - Claude Code MCP, Chrome, Sandboxing pages

2. **Update launchd configuration** (30 minutes)
   - Pass `--classify --update-kb` flags in production runs
   - Leave `--propose-edits --run-tests` for manual invocation initially

3. **Add requirements.txt entries** (15 minutes)
   - `anthropic>=0.40.0`

4. **Documentation** (45 minutes)
   - Update freshness-pipeline-spec.md
   - Add pipeline CLI help text

---

## 13. Open Questions

1. **Firecrawl credit budget**: At ~270-540 credits/month, the current 222 credits last
   about 2-3 weeks. Should we purchase more credits now, or start with daily-only
   scraping for code.claude.com (halving usage)?

2. **Edit auto-application**: Should we ever auto-apply proposed edits (with the
   validation gate as the safety net), or always require human review? Recommendation:
   start with review-only, consider auto-apply after 2-3 months of confidence.

3. **Notification channel**: macOS notifications work when the laptop is open. Should we
   add email (via Gmail MCP) or another channel for when the laptop is sleeping during
   the scheduled run? The review file persists regardless, so this is convenience only.

4. **Sonnet vs Haiku for edit proposals**: Sonnet produces better edits but costs 6x
   more. Given the low frequency (1-2 proposals/week), the cost difference is negligible
   (~$0.20 vs ~$0.04). Recommendation: use Sonnet.

5. **support.claude.com release notes**: The current URL returns 404. Need to find the
   new correct URL. This may be at `support.claude.com/en/articles/12138966-release-notes`
   based on the link visible in the 404 page content. Should be verified before adding.

6. **code.claude.com URL format**: The current URLs use paths like `/docs/skills` but
   Firecrawl may need the full path with locale, e.g., `/docs/en/skills`. Need to test
   both formats.

---

## 14. Success Metrics

After full implementation, the pipeline should achieve:

| Metric | Target | How to Measure |
|--------|--------|---------------|
| False positive rate | < 5% of detected changes | Review reports over 2 weeks |
| Real change detection | > 95% of actual doc changes | Cross-reference with manual checks |
| Time-to-awareness | < 12 hours from source change | Compare source commit timestamps to review file timestamps |
| Validation regression rate | 0% from automated KB updates | Track test scores over time |
| Human review time | < 10 minutes per actionable change | Time spent on review files |
| Credit consumption | < 300 credits/month | Firecrawl credit tracking |

---

## 15. Appendix: Full Data Flow Example

Here is a concrete example of the full pipeline processing a real change:

**Trigger:** The twice-daily launchd job fires at 21:00.

**Step 1 -- Scraping:** The pipeline scrapes all 22 sources. The GitHub CHANGELOG has a
new entry for Claude Code v2.1.73 with a new feature: "Added `--json` output flag to
`claude` CLI." Firecrawl successfully extracts the code.claude.com/docs/cli-reference
page, which now documents the new flag.

**Step 2 -- Noise filtering:** The noise filter strips cookie banners and "Was this page
helpful?" footers. Content length checks pass. No scrape failures detected.

**Step 3 -- Detection:** SHA-256 hashes differ for the CHANGELOG and CLI Reference
sources. Both are marked as `changed`.

**Step 4 -- Classification:** Claude Haiku classifies:
- CHANGELOG change: `NEW_FEATURE` (confidence: 0.92) -- "Claude Code v2.1.73 adds --json output flag for CLI automation"
- CLI Reference change: `NEW_FEATURE` (confidence: 0.88) -- "CLI reference now documents --json flag for machine-readable output"

Both are actionable.

**Step 5 -- KB update:** The pipeline writes:
- New CHANGELOG content to `knowledge-base/claude-code-capabilities-cli-reference.md`
  (after backing up the previous version)
- New CLI Reference content to the same file (or a secondary mapped file)

**Step 6 -- Edit proposal:** Claude Sonnet proposes:
```json
[
  {
    "file": "references/claude-code-specifics.md",
    "section": "CLI Reference",
    "old_text": "## CLI Flags\n\n`--print` ...",
    "new_text": "## CLI Flags\n\n`--print` ...\n`--json` -- Output in JSON format for machine-readable automation\n",
    "rationale": "New --json flag added in v2.1.73 for CLI automation"
  }
]
```

**Step 7 -- Validation gate:** The test runner runs 33 tests. All scores match or exceed
baseline. Gate passes.

**Step 8 -- Review:** A review file is generated at
`pipeline/reports/review-2026-03-10-2100.md` containing the classification, proposed
edit, and test results. A macOS notification fires: "Freshness Check: 2 Actionable
Changes -- 2 new feature(s). Tests: PASSED."

**Human action:** Liam opens the review file the next morning, approves the proposed
edit, applies it with a text editor, runs a quick manual check, and commits to the dev
branch. Total time: 5 minutes.
