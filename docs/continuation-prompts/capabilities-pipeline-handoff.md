# Capabilities Pipeline: Fix, Enhance, Test, Close the Loop

## Context

We've built a Claude Capabilities Awareness Skill (Phase 1 MVP) and set up 8 Distill Web Monitors watching Anthropic documentation pages for changes. The skill is built and ready to publish. The monitors are running and will send email notifications when content changes.

**What's missing:** the connective tissue between "Distill detects a change" and "the skill gets updated," plus fixes needed for monitoring errors, new content that needs incorporating, and evaluation tests to prove the skill works.

This session covers four workstreams:

1. Fix Distill SELECTION_EMPTY errors
2. Incorporate new capabilities research into the skill
3. Design and build blind evaluation tests
4. Document the Phase 2 close-the-loop requirements

## Read These Files

1. `.planning/designs/claude-capabilities-awareness-skill-design.md` — The full design document (Phase 1-3)
2. `.planning/builds/claude-capabilities-skill/claude-capabilities/SKILL.md` — The built skill (182 lines)
3. `.planning/builds/claude-capabilities-skill/claude-capabilities/references/` — All 4 reference files

## Workstream 1: Fix Distill SELECTION_EMPTY Errors

### The Problem

Four monitors are returning SELECTION_EMPTY errors:
- Anthropic — Getting Started with Cowork
- Anthropic — Claude in PowerPoint
- Anthropic — Claude.ai Release Notes
- (Likely also) Anthropic — Claude in Excel

All four monitor support.claude.com pages using the CSS selector `article[class*="intercom-force-break"]`.

### Root Cause

support.claude.com uses Intercom's help center platform, which loads article content dynamically via JavaScript. Distill's cloud monitors (`Cloud - Distill's Servers`) perform HTTP fetches by default — they don't render JavaScript. The selector matches when the browser extension previews the page (because the browser has JS), but the cloud server gets a page where the content hasn't been rendered yet.

### Fix Options (in order of preference)

**Option A: Enable JavaScript rendering + add delay**

In each affected monitor's config:
1. Open the monitor's options
2. Click the gear icon (⚙️) next to the selector area, or edit the JSON config
3. Set `"dynamic": true` — this tells Distill to use a headless browser (JS rendering) instead of a plain HTTP fetch
4. Set `"delay": 10` (or 10-15 seconds) — this gives the page time to render Intercom content
5. Save and test

This is the recommended fix. It may require a paid Distill plan for cloud-based JS rendering.

**Option B: Switch to a less dynamic selector**

If JS rendering isn't available on the current plan, try monitoring a higher-level container that exists in the initial HTML:
- Try `title` (the page title is in the HTML head, always present)
- Try `meta[name="description"]` (also in HTML head)
- Combine: monitor the title tag alongside the article content

Downside: less granular — you'll get notified on any title change, not specific content changes.

**Option C: Switch affected monitors to Local monitoring**

Change from "Cloud - Distill's Servers" to "Local" monitoring for these 4 monitors. Local runs in the browser, which always renders JS. Downside: requires the browser to be open, and won't check when the computer is off.

### Verification

After applying the fix, force-check each monitor (right-click → "Check now" or similar) and verify:
- The error clears
- Status shows "ON" without error badge
- Content preview populates (no longer shows "Preview will be available soon after this is run")

### Also Check

The other monitors should be verified too — even if they aren't showing SELECTION_EMPTY currently:
- Monitor 3 (Features & Capabilities collection) uses `section[data-testid="main-content"]` — this is also a support.claude.com page and may have the same issue
- Monitors 1-2 (API Overview, API Release Notes) use `main` on docs.anthropic.com — these are likely fine as docs.anthropic.com is server-rendered
- Monitor 8 (Claude Code CHANGELOG) uses `article.markdown-body` on GitHub — GitHub server-renders markdown, so this should be fine

### Distill Error Documentation

Reference: https://distill.io/docs/web-monitor/troubleshooting-errors/#selection_empty

Key info from research:
- SELECTION_EMPTY means the selector matched nothing when the page was fetched
- Setting `"dynamic": true` enables JS rendering (headless browser) for cloud checks
- Adding `"delay": 10` (seconds) gives dynamic content time to load
- Proxy options (Shared Pool, Dedicated) can help with IP-based blocking
- `"ignoreEmptyText": false` if you want to be notified when content becomes empty

## Workstream 2: Incorporate New Capabilities Research

### What Changed

Seven research files were created or updated today (10 Feb 2026) AFTER the skill was built. These use the new naming convention that distinguishes API vs Claude Code capabilities:

| File | Status | Content |
|------|--------|---------|
| `claude-capabilities-implement-tool-use.md` | New (66KB) | General tool use implementation guide |
| `claude-code-capabilities-agent-teams.md` | New (22KB) | Multi-agent orchestration, team patterns |
| `claude-code-capabilities-cli-reference.md` | Updated (26KB) | CLI commands, flags, options |
| `claude-code-capabilities-create-plugins.md` | Updated (17KB) | Plugin authoring guide |
| `claude-code-capabilities-extension-options.md` | New (23KB) | VS Code, JetBrains IDE integrations |
| `claude-code-capabilities-skills.md` | Updated (33KB) | Skill authoring, structure, best practices |
| `claude-code-capabilities-use-chrome-browser.md` | New (11KB) | Browser automation in Claude Code |

### Naming Convention

The user has reorganised research files into three categories:
- `claude-capabilities-*` → API-level capabilities (work across all interfaces)
- `claude-code-capabilities-*` → Claude Code-specific capabilities (CLI, plugins, extensions)
- Other → Platform-level (MCP Apps, etc.)

### What the Skill Currently Covers vs What's New

The built skill's `references/agent-capabilities.md` (451 lines) covers Agent SDK, hooks, plugins, subagents, MCP, and skills — but from a condensed perspective. The new research files add significant depth on:

1. **Agent teams** — Multi-agent orchestration patterns, team composition, parallel execution
2. **Chrome browser use** — Claude Code's browser automation via MCP (distinct from API-level computer use)
3. **Extension options** — VS Code integration, JetBrains support, configuration per-IDE
4. **CLI reference** — Full command reference (not just SDK programmatic usage)
5. **Tool use implementation** — Broader implementation guide beyond what's in tool-types.md

### Task

Review each of the 7 new/updated files and determine:
1. What information is genuinely new (not already in the skill)?
2. Does it belong in the SKILL.md body (< 500 line budget) or a reference file?
3. Should `references/agent-capabilities.md` be expanded, or should a new reference file be added (e.g., `references/claude-code-specifics.md`)?

**Constraints:**
- SKILL.md must stay under 500 lines (currently 182 — significant budget remaining)
- Reference files should stay around 400-450 lines each
- Only include post-training information that Claude's training data likely missed
- The skill is for ALL Claude users, not just Claude Code users — so Claude Code specifics should be clearly marked as such

### Consideration: Skill Scope Decision

The current skill focuses on API and agent capabilities. Adding Claude Code-specific content (CLI, extensions, browser use) expands the scope. This is a deliberate decision:

- **For expanding:** Claude Code users are the primary audience for a skill (skills only work in Claude Code). Chrome browser use, plugin authoring, and agent teams are exactly the kind of post-training knowledge these users need.
- **Against expanding:** Keeps the skill focused. Claude Code-specific docs are already available via `plugin-dev:*` skills and may duplicate.

Recommend: Expand with a light touch. Add a "Claude Code Capabilities" section to SKILL.md body with key items, and either expand `references/agent-capabilities.md` or create a new `references/claude-code-specifics.md` reference file for the detail.

## Workstream 3: Evaluation Tests

### Goal

Create a set of blind test prompts that can be run in Claude Code with subagents to measure whether the capabilities skill makes a meaningful difference. The prompts must NOT mention the skill, capabilities awareness, or hint at what's being tested. They should be realistic tasks that a developer would actually ask.

### Test Design

**Method:** Each prompt is run twice as a subagent task:
- **Control:** Standard Claude Code session (no skill installed)
- **Treatment:** Session with the claude-capabilities skill installed

Compare responses for accuracy, completeness, and whether deprecated/outdated approaches are recommended.

### Test Categories and Prompts

#### Category 1: Architecture Decisions (Should Benefit from Knowing Current Capabilities)

```
Test 1.1: "I'm building a document analysis pipeline that needs to process
100-page PDFs. The system should extract key information, cross-reference
it with a knowledge base, and produce a structured summary. What
architecture would you recommend? Keep it simple — this is for a small
team."

Expected skill advantage: Should recommend using 1M context window
(beta) to process entire documents in one pass rather than chunking.
Should mention Files API for upload. Should consider effort parameter
for cost control.
```

```
Test 1.2: "I need to build a quality assurance system that reviews
AI-generated content before it goes to customers. The system should
check for accuracy, tone, and compliance with our style guide. How
would I architect this?"

Expected skill advantage: Should recommend subagents or Agent SDK
with appropriate model selection (Haiku for fast checks, Sonnet/Opus
for nuanced review). Should mention hooks for automation. Without
the skill, may recommend older multi-agent patterns.
```

```
Test 1.3: "We want to add AI features to our Next.js app. Users should
be able to ask questions about their data and get real-time streaming
responses. What's the simplest way to integrate Claude into this?"

Expected skill advantage: Should mention fine-grained tool streaming
(GA), structured outputs for typed responses, and possibly MCP
Connector for direct server integration. May recommend Agent SDK
for backend.
```

#### Category 2: "Can Claude Do X?" (Answers Changed Since Training)

```
Test 2.1: "Can I get Claude to remember things between separate
conversations? I'm building a tool where Claude helps with ongoing
projects and it's frustrating that it loses context each time."

Expected skill advantage: Should describe the Memory tool (beta)
with specific details — commands (view, create, str_replace, insert,
delete, rename), client-side persistence requirement, beta header.
Without the skill, may say this isn't possible or suggest workarounds.
```

```
Test 2.2: "Is there a way to make Claude use tools without me having
to define every single one upfront? I have hundreds of MCP tools and
the context window overhead is killing me."

Expected skill advantage: Should describe Tool Search (beta) with
dynamic discovery via regex, and mention the 10% auto-activation
threshold. Should also mention MCP Connector for remote servers.
Without the skill, may suggest manual tool filtering.
```

```
Test 2.3: "I want Claude to help fill out web forms and navigate
browser interfaces for data entry tasks. Is that possible through
the API?"

Expected skill advantage: Should describe Computer Use with specific
tool version (computer_20251124 for Opus 4.6 with zoom), requirements
(screenshot handling, coordinate system), and current status. May also
mention Claude Code's Chrome browser use capability.
```

#### Category 3: Implementation Guidance (Should Avoid Deprecated Patterns)

```
Test 3.1: "I'm using Claude Opus 4.6 and want to control how much
'thinking' it does on different types of requests. Some are simple
lookups, others are complex analysis. How do I configure this?"

Expected skill advantage: Should recommend adaptive thinking
(type: "adaptive") + effort parameter. Should NOT recommend
budget_tokens (deprecated on Opus 4.6). Should mention that "max"
effort is Opus 4.6 exclusive.
```

```
Test 3.2: "I have a working integration with Claude that uses assistant
message prefilling to start responses in a specific format. I want to
upgrade to Opus 4.6. Anything I should know?"

Expected skill advantage: Should flag that prefill is REMOVED on
Opus 4.6 (returns 400 error), and recommend structured outputs or
system prompts as alternatives. Without the skill, may not flag
this breaking change.
```

```
Test 3.3: "I want Claude to output JSON that strictly matches my
schema — not 'best effort' but guaranteed. How do I set this up?"

Expected skill advantage: Should describe structured outputs (GA)
with the correct parameter path (output_config.format), mention
the old output_format is deprecated, and provide both JSON output
and strict tool use approaches with SDK helpers.
```

#### Category 4: Model Selection (Should Know Current Landscape)

```
Test 4.1: "I need to choose a Claude model for my application. It needs
to handle long documents (200+ pages), produce detailed analysis,
and keep costs reasonable. What would you recommend?"

Expected skill advantage: Should present the current model matrix
accurately — Opus 4.6 with 128K output and 1M context (beta), vs
Sonnet 4.5 with 64K output and interleaved thinking, vs Haiku 4.5
for cost efficiency. Should mention effort parameter for cost control.
```

### Scoring Rubric

For each test, score the response on:

| Criterion | Score | Definition |
|-----------|-------|------------|
| Accuracy | 0-3 | 0: Incorrect/outdated. 1: Partially correct. 2: Correct. 3: Correct with specifics (parameter names, headers, versions). |
| Completeness | 0-3 | 0: Misses the key capability. 1: Mentions it vaguely. 2: Covers main points. 3: Covers main points + edge cases/caveats. |
| Avoids Deprecated | 0-1 | 0: Recommends deprecated approach. 1: Uses current approach. |
| Actionability | 0-2 | 0: Vague. 1: Gives direction. 2: Gives specific code/config. |

**Max score per test: 9. Total across 10 tests: 90.**

### Running the Tests

Use Claude Code subagents for isolation:

```python
# Control (no skill)
async for event in query(
    prompt=TEST_PROMPT,
    options={
        "model": "claude-sonnet-4-5-20250929",
        "max_turns": 1,
        "allowed_tools": [],  # No tools — pure knowledge test
    }
):
    # Capture response

# Treatment (with skill installed)
# Same parameters, but skill is present in the session
```

Important:
- Use the same model for both runs
- No tool access (we're testing knowledge, not tool use)
- Single turn only
- Run each prompt 2-3 times to account for variance
- Score independently (blind to condition if possible)

## Workstream 4: Close-the-Loop Documentation

### The Pipeline As-Is

```
┌─────────────────┐     email      ┌─────────────┐     manual      ┌──────────────┐
│ Distill Monitors │ ──────────>   │    Liam      │ ──────────>    │  Skill Files │
│ (8 pages)        │  notification │              │   editing      │  (published)  │
└─────────────────┘               └─────────────┘               └──────────────┘
```

### The Pipeline Target (Phase 2)

```
┌─────────────────┐     email      ┌─────────────┐    runs         ┌──────────────────┐
│ Distill Monitors │ ──────────>   │    Liam      │ ──────>        │ /update-claude-   │
│ (8 pages)        │  notification │              │                │  capabilities     │
└─────────────────┘               └──────┬──────┘               │                  │
                                          │                      │ 1. fetch docs    │
                                          │                      │ 2. diff vs JSON  │
                                          │                      │ 3. interpret     │
                                          │                      │ 4. update files  │
                                          │                      │ 5. verify        │
                                          └───── reviews ──────> │ 6. publish       │
                                                                  └──────────────────┘
```

### Phase 2 Build Items

These are documented in the design doc (Sections 5-6) but listed here for tracking:

| # | Item | Design Doc Section | Notes |
|---|------|--------------------|-------|
| 1 | `capability-matrix.json` | §6 | Machine-readable source of truth. Schema defined. |
| 2 | `fetch-docs.sh` | §5.2 | **Needs expanding** — currently only fetches from docs.anthropic.com + GitHub. Must also include support.claude.com URLs that Distill monitors. |
| 3 | `diff-capabilities.sh` | §5.3 | Compares fetched content against capability-matrix.json. |
| 4 | `/update-claude-capabilities` command | §5.1 | The main workflow. Includes --dry-run and --source flags. |
| 5 | Error handling & rollback | §5.4 | Backup strategy, restore procedure, 30-day retention. |
| 6 | Plugin structure (`plugin.json`) | §3.2 | Wraps skill + command + optional hook. |
| 7 | UserPromptSubmit hook | §11 | Lightweight context injection. Optional. |

### Source Coverage Gap (Important)

The Distill monitors and the `/update` command's fetch script watch **different sets of pages**:

| Source | Distill | /update Script |
|--------|---------|---------------|
| docs.anthropic.com/en/docs/about-claude/models/overview | ✅ Monitor 1 | ✅ In script |
| docs.anthropic.com/en/docs/about-claude/models (release notes) | ✅ Monitor 2 | ✅ (via changelog) |
| support.claude.com — Features & Capabilities collection | ✅ Monitor 3 | ❌ NOT in script |
| support.claude.com — Claude.ai Release Notes | ✅ Monitor 4 | ❌ NOT in script |
| support.claude.com — Claude in Excel | ✅ Monitor 5 | ❌ NOT in script |
| support.claude.com — Claude in PowerPoint | ✅ Monitor 6 | ❌ NOT in script |
| support.claude.com — Getting Started with Cowork | ✅ Monitor 7 | ❌ NOT in script |
| github.com — Claude Code CHANGELOG | ✅ Monitor 8 | ✅ In script |
| docs.anthropic.com/en/docs/build-with-claude/overview | ❌ No monitor | ✅ In script |
| docs.anthropic.com/en/changelog | ❌ No monitor | ✅ In script |

**Action needed for Phase 2:** When building `fetch-docs.sh`, add the support.claude.com URLs. Alternatively, the `/update` command prompt should instruct Claude to also check what Distill has recently flagged (perhaps by reading a local notifications log).

**Action needed for Distill:** Consider adding monitors for:
- `docs.anthropic.com/en/changelog` (API changelog — not currently monitored, but in the fetch script)
- `docs.anthropic.com/en/docs/build-with-claude/overview` (build overview — not monitored)

### Additional Distill Monitors to Consider

Based on the research files and what the skill covers, these pages would be valuable additions:

| Page | Why |
|------|-----|
| docs.anthropic.com/en/changelog | Catches API changes not in release notes |
| docs.anthropic.com/en/docs/build-with-claude/computer-use | Computer use is beta and changing |
| docs.anthropic.com/en/docs/build-with-claude/tool-use | Tool use patterns evolve |
| support.claude.com/en/articles (Claude Code release notes, if a dedicated page exists) | Claude Code capabilities |

## Output Expectations

This session should produce:

1. **Fixed Distill monitors** (or clear documentation of what to change if browser access is needed)
2. **Updated skill files** incorporating the 7 new/updated research files
3. **Evaluation test script** — ready to run, with all 10 prompts and scoring rubric
4. **Phase 2 requirements document** — Clean task list with the source coverage gap resolved

Save outputs to:
- Updated skill: `.planning/builds/claude-capabilities-skill/claude-capabilities/` (in place)
- Evaluation tests: `.planning/builds/claude-capabilities-skill/evals/`
- Phase 2 requirements: `.planning/designs/capabilities-pipeline-phase2-requirements.md`
