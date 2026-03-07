# Design Document: Claude Capabilities Awareness Skill

**Author:** Liam Jones
**Date:** 10 February 2026
**Status:** Draft
**Version:** 0.1.0

---

## 1. Problem Statement

Claude's training data has a cutoff, which means it doesn't always recognise its own current capabilities. Features released after training — adaptive thinking, 1M context windows, the Files API, Memory tool, MCP connector, Tool Search, and others — are unknown to Claude unless explicitly injected into context. This creates a concrete problem: Claude designs around limitations that no longer exist, under-utilises its own capabilities, and gives outdated guidance to developers building on the API.

A capabilities awareness skill would bridge this gap by injecting a condensed, current summary of Claude's capabilities at the point they're most needed — when Claude is making architectural or implementation decisions.

---

## 2. Design Decision: Skill vs Plugin

### Recommendation: Plugin (with standalone skill as MVP)

The full solution requires three components: a skill (capabilities awareness), a command (/update-claude-capabilities), and potentially a second command (/publish-claude-capabilities). Only a plugin can bundle all three.

However, the MVP should be a standalone skill publishable via skills.sh, with the plugin structure added in v2.

**Rationale:**

| Factor | Standalone Skill | Plugin |
|--------|-----------------|--------|
| Distribution | skills.sh (available now) | Marketplace (limited availability) |
| Capabilities | Awareness injection only | Awareness + update + publish commands |
| Maintenance | Manual updates to SKILL.md | Semi-automated via /update command |
| Complexity | Single file + references | Full plugin structure |
| Time to ship | Days | Weeks |

**Phased approach:**

- **Phase 1 (MVP):** Standalone skill via skills.sh. Manual updates. Covers 90% of the value.
- **Phase 2:** Plugin wrapping the skill, adding /update-claude-capabilities command.
- **Phase 3:** /publish-claude-capabilities command, community contribution workflow.

---

## 3. Skill Architecture

### 3.1 Directory Structure

```
claude-capabilities/
├── SKILL.md                          # Core skill (< 500 lines)
├── references/
│   ├── api-features.md               # API capabilities detail
│   ├── tool-types.md                 # All built-in tool types
│   ├── agent-capabilities.md         # Agent SDK, subagents, skills
│   ├── model-specifics.md            # Per-model capabilities and limits
│   └── changelog.md                  # What changed in each update
├── scripts/
│   ├── fetch-docs.sh                 # Fetch latest Anthropic docs
│   └── diff-capabilities.sh          # Diff against current knowledge
└── assets/
    └── capability-matrix.json        # Machine-readable capability data
```

### 3.2 Plugin Structure (Phase 2)

```
claude-capabilities-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── claude-capabilities/
│       ├── SKILL.md
│       ├── references/
│       │   ├── api-features.md
│       │   ├── tool-types.md
│       │   ├── agent-capabilities.md
│       │   └── model-specifics.md
│       ├── scripts/
│       │   ├── fetch-docs.sh
│       │   └── diff-capabilities.sh
│       └── assets/
│           └── capability-matrix.json
├── commands/
│   ├── update-claude-capabilities.md
│   └── publish-claude-capabilities.md
└── hooks/
    └── hooks.json                    # Optional: SessionStart context injection
```

### 3.3 Triggering Strategy

The skill description is the primary trigger mechanism. It should be "pushy" to ensure Claude loads it when making capability-dependent decisions.

```yaml
---
name: claude-capabilities
description: >
  Current Claude model capabilities, API features, and tool types that may
  not be in training data. Use this skill whenever Claude is designing a
  system architecture, choosing between implementation approaches, advising
  on what Claude can or cannot do, discussing API features, evaluating
  whether a capability exists, building agents or tools, or any situation
  where knowledge of Claude's current capabilities would improve the
  response. Also trigger when the user asks "can Claude do X?",
  "does Claude support Y?", "what models are available?", or discusses
  context windows, tool use, computer use, MCP, extended thinking,
  structured outputs, or agent patterns. MANDATORY TRIGGERS: Claude
  capabilities, API features, model comparison, context window, tool use,
  agent SDK, what can Claude do, Claude limitations, model selection.
version: 1.0.0
---
```

This description ensures triggering in the right contexts without over-firing on general conversation.

---

## 4. Content Format Design

### 4.1 Design Principles

1. **Concise over comprehensive.** The SKILL.md body must stay under 500 lines. Detailed reference material goes in references/.
2. **Delta-focused.** Emphasise what's new or surprising — things Claude's training likely missed. Don't restate common knowledge.
3. **Structured for scanning.** Use tables and short entries so Claude can quickly locate relevant capabilities.
4. **Actionable over informational.** Each entry should tell Claude what it can DO, not just what EXISTS.
5. **Model-aware.** Capabilities vary by model — make this explicit.

### 4.2 SKILL.md Body Structure

The core skill file follows this structure:

```markdown
---
name: claude-capabilities
description: [as above]
version: 1.0.0
---

# Claude Capabilities Awareness

This skill contains current Claude capabilities that may not be in
training data. When making architectural decisions, recommending
approaches, or answering questions about what Claude can do, consult
this information to ensure accuracy.

**Last updated:** 2026-02-10
**Covers models through:** Claude Opus 4.6

## Current Models

| Model | ID | Context | Max Output | Key Capabilities |
|-------|----|---------|------------|------------------|
| Opus 4.6 | claude-opus-4-6 | 200K (1M beta) | 128K | Adaptive thinking, max effort |
| Sonnet 4.5 | claude-sonnet-4-5-20250929 | 200K | 64K | Extended thinking, interleaved thinking |
| Haiku 4.5 | claude-haiku-4-5-20251001 | 200K | 64K | Fast, cost-efficient |
| Opus 4.5 | claude-opus-4-5-20251101 | 200K | 32K | High capability |

## Key Capabilities (Post-Training)

### Thinking & Reasoning
- **Adaptive thinking** (Opus 4.6): `thinking: {type: "adaptive"}` —
  Claude decides when/how much to think. Replaces budget_tokens
  (deprecated on Opus 4.6). Automatically enables interleaved thinking.
- **Effort parameter** (GA): `effort: "low"|"medium"|"high"|"max"` —
  controls token usage vs thoroughness. No beta header required. "max"
  is Opus 4.6 only.
- **128K output tokens** (Opus 4.6): doubled from 64K. Requires
  streaming for large max_tokens values.

### Context & Memory
- **1M token context window** (beta): available on all current models.
  Requires usage tier 3+. Beta header required.
- **Compaction API** (beta): server-side context summarisation for
  infinite conversations. Automatic when context approaches limit.
- **Context editing** (beta): automatic context management with
  configurable strategies (clearing tool results, managing thinking
  blocks).
- **Memory tool** (beta): cross-conversation memory with store/retrieve.
  Client-side persistence. Commands: view, create, str_replace, insert,
  delete, rename.
- **Prompt caching**: 5-minute (all platforms) and 1-hour (API, Azure)
  durations available.

### Tools & Integration
- **Built-in tools**: Bash, Text Editor, Computer Use, Web Search, Web
  Fetch, Code Execution, Memory, Tool Search.
- **Tool Search** (beta): scale to thousands of tools with dynamic
  discovery via regex search.
- **MCP Connector** (beta): connect to remote MCP servers directly from
  Messages API without separate client.
- **Programmatic tool calling** (beta): call tools programmatically from
  code execution containers — reduces latency for multi-tool workflows.
- **Fine-grained tool streaming** (GA): stream tool parameters without
  buffering. No beta header required.
- **Computer use**: desktop automation via screenshots + mouse/keyboard.
  Model-specific tool versions (computer_20251124 for Opus 4.6).

### Output & Structure
- **Structured outputs** (GA on API): guaranteed JSON schema conformance.
  Two approaches: JSON outputs and strict tool use. Parameter moved to
  `output_config.format` (old `output_format` deprecated).
- **Files API** (beta): upload/download/manage files without
  re-uploading. Supports PDFs, images, text files.
- **Citations/Search results**: natural citations for RAG with source
  attribution from search_result content blocks.

### Agent Capabilities
- **Agent SDK**: Python SDK for building custom agents with query(),
  ClaudeSDKClient, streaming, hooks, custom tools.
- **Agent Skills**: pre-built (PowerPoint, Excel, Word, PDF) and custom
  skills with progressive disclosure.
- **Custom subagents**: system prompts, tool restrictions, built-in types
  (Explore, Plan, general-purpose).
- **Hooks**: SessionStart, PreToolUse, PostToolUse, Stop, SubagentStop,
  UserPromptSubmit, PreCompact, Notification events.

### Platform
- **Data residency** (Opus 4.6+): `inference_geo: "us"|"global"` per
  request. US-only priced at 1.1x.
- **Batch processing**: 50% cost reduction for async batch requests.

## Breaking Changes (Opus 4.6)

- **Prefill removed**: assistant message prefilling returns 400 error.
  Use structured outputs or system prompts instead.
- **budget_tokens deprecated**: use adaptive thinking + effort parameter.
- **output_format deprecated**: moved to output_config.format.
- **Tool parameter quoting**: may produce different JSON string escaping.
  Use standard JSON parsers.

## Reference Files

For detailed information, read the appropriate reference file:

- `references/api-features.md` — detailed API feature documentation
  with code examples, beta headers, and platform availability
- `references/tool-types.md` — all built-in tool types with
  configuration, parameters, and usage patterns
- `references/agent-capabilities.md` — Agent SDK, subagents, skills
  system, hooks, and MCP integration
- `references/model-specifics.md` — per-model capability matrix,
  pricing, limits, and migration guidance
```

### 4.3 Context Budget Analysis

| Component | Estimated Tokens | When Loaded |
|-----------|-----------------|-------------|
| Description (metadata) | ~150 | Always in system prompt |
| SKILL.md body | ~1,500 | When skill triggers |
| Single reference file | ~2,000-4,000 | On-demand |
| Total worst-case | ~6,000 | Rare — most queries need body only |

This is well within acceptable bounds. The body at ~1,500 tokens is comparable to other production skills (docx: ~2,000, pptx: ~1,800).

---

## 5. Update Command Design

### 5.1 Command: /update-claude-capabilities

```markdown
---
description: Fetch latest Claude capabilities from Anthropic docs and update the skill
allowed-tools: Read, Write, Edit, Bash(curl:*), Bash(diff:*), Bash(node:*), WebFetch, WebSearch
argument-hint: [--dry-run] [--source api|changelog|all]
---

Update the claude-capabilities skill with the latest information from
Anthropic's documentation.

## Process

1. **Fetch current documentation**
   Run: `bash ${CLAUDE_PLUGIN_ROOT}/skills/claude-capabilities/scripts/fetch-docs.sh`
   This fetches from:
   - https://docs.anthropic.com/en/docs/about-claude/models/overview
   - https://docs.anthropic.com/en/docs/build-with-claude (feature pages)
   - https://docs.anthropic.com/en/changelog
   - https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md

2. **Diff against current knowledge**
   Run: `bash ${CLAUDE_PLUGIN_ROOT}/skills/claude-capabilities/scripts/diff-capabilities.sh`
   This compares fetched docs against the current capability-matrix.json
   and highlights:
   - New features not in current skill
   - Changed capabilities (limits, availability, GA status)
   - Deprecated features
   - New models

3. **Interpret changes**
   Review the diff output. For each change, determine:
   - Is this a capability Claude's training data likely covers?
   - Does this change what Claude can recommend or do?
   - What's the impact level (high/medium/low)?

4. **Update skill files**
   If not --dry-run:
   - Update SKILL.md body with new/changed capabilities
   - Update relevant reference files
   - Update capability-matrix.json
   - Update version number and "Last updated" date
   - Update changelog.md

5. **Verify**
   Read the updated SKILL.md and confirm it:
   - Stays under 500 lines
   - Maintains the established format
   - Accurately reflects the changes
   - Doesn't duplicate information already in training data
```

### 5.2 Fetch Script Design

```bash
#!/bin/bash
# fetch-docs.sh — Fetch latest Anthropic documentation
# Outputs fetched content to stdout for AI interpretation

CACHE_DIR="${CLAUDE_PLUGIN_ROOT}/skills/claude-capabilities/.cache"
mkdir -p "$CACHE_DIR"

SOURCES=(
  "https://docs.anthropic.com/en/docs/about-claude/models/overview"
  "https://docs.anthropic.com/en/docs/build-with-claude/overview"
  "https://docs.anthropic.com/en/changelog"
)

for url in "${SOURCES[@]}"; do
  filename=$(echo "$url" | sed 's/[^a-zA-Z0-9]/_/g').md
  echo "--- SOURCE: $url ---"
  # Actual fetching delegated to Claude's WebFetch tool
  # Script outputs markers for AI to fill in
  echo "FETCH_NEEDED: $url"
  echo "CACHE_TO: $CACHE_DIR/$filename"
  echo "---"
done
```

In practice, the /update command uses Claude's WebFetch and WebSearch tools to retrieve content, with the script providing structure. The AI interpretation step is essential — raw doc diffs need judgment about significance.

### 5.3 Diff Script Design

```bash
#!/bin/bash
# diff-capabilities.sh — Compare fetched docs against current knowledge

SKILL_DIR="${CLAUDE_PLUGIN_ROOT}/skills/claude-capabilities"
MATRIX="$SKILL_DIR/assets/capability-matrix.json"
CACHE_DIR="$SKILL_DIR/.cache"

if [ ! -f "$MATRIX" ]; then
  echo "ERROR: No capability-matrix.json found"
  exit 1
fi

echo "=== CURRENT CAPABILITIES ==="
cat "$MATRIX" | python3 -c "
import json, sys
data = json.load(sys.stdin)
print(f'Models: {len(data.get(\"models\", []))}')
print(f'Features: {len(data.get(\"features\", []))}')
print(f'Tools: {len(data.get(\"tools\", []))}')
print(f'Last updated: {data.get(\"lastUpdated\", \"unknown\")}')
"

echo ""
echo "=== CACHED DOCS ==="
for f in "$CACHE_DIR"/*.md; do
  [ -f "$f" ] && echo "  $(basename "$f"): $(wc -l < "$f") lines"
done

echo ""
echo "AI_REVIEW_NEEDED: Compare cached docs against capability-matrix.json"
echo "Look for: new features, changed limits, new models, deprecations"
```

### 5.4 Error Handling & Rollback

The /update command must be resilient to failures at any stage and provide rollback capability.

**Failure points and handling:**

1. **Fetch step fails** (network errors, docs unavailable):
   - If WebFetch/WebSearch cannot retrieve docs, stop immediately and report which sources failed
   - Do not proceed to diffing or updating
   - Advise user to try again later or verify network connectivity
   - Exit with non-zero status code

2. **Partial update completion** (e.g., SKILL.md updated but reference files not):
   - The backup strategy below prevents this, but if detected:
   - Halt all remaining updates
   - Restore from backup
   - Report which file(s) failed and why
   - Ask user to investigate or retry

3. **Verification fails** (updated SKILL.md exceeds 500 lines, invalid JSON in capability-matrix.json, etc.):
   - Do not commit any changes
   - Restore all files from backup
   - Report what failed validation and why
   - Suggest remediation (e.g., "SKILL.md is 520 lines; trim by 20 lines or increase limit")

**Backup strategy:**

Before any file modifications, the /update command creates a timestamped backup:

```bash
BACKUP_DIR="${CLAUDE_PLUGIN_ROOT}/skills/claude-capabilities/.cache/backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Backup all files that will be modified
cp SKILL.md "$BACKUP_DIR/"
cp -r references/ "$BACKUP_DIR/"
cp assets/capability-matrix.json "$BACKUP_DIR/"
cp assets/changelog.md "$BACKUP_DIR/"

echo "Backup created: $BACKUP_DIR"
```

**Rollback procedure:**

If any verification step fails or user requests rollback:

```bash
# Restore from backup
cp "$BACKUP_DIR/SKILL.md" .
cp -r "$BACKUP_DIR/references/" .
cp "$BACKUP_DIR/capability-matrix.json" assets/
cp "$BACKUP_DIR/changelog.md" assets/

echo "Rolled back to: $BACKUP_DIR"
```

Backups are retained for 30 days, allowing post-update recovery if issues surface after publication.

---

## 6. Capability Matrix Format

The machine-readable capability-matrix.json serves as the source of truth for diffing:

```json
{
  "schemaVersion": "1.0.0",
  "lastUpdated": "2026-02-10",
  "skillVersion": "1.0.0",
  "models": [
    {
      "name": "Claude Opus 4.6",
      "id": "claude-opus-4-6",
      "contextWindow": 200000,
      "contextWindowBeta": 1000000,
      "maxOutputTokens": 128000,
      "extendedThinking": true,
      "adaptiveThinking": true,
      "prefillSupport": false,
      "effortLevels": ["low", "medium", "high", "max"],
      "releaseDate": "2025-02"
    },
    {
      "name": "Claude Sonnet 4.5",
      "id": "claude-sonnet-4-5-20250929",
      "contextWindow": 200000,
      "contextWindowBeta": 1000000,
      "maxOutputTokens": 64000,
      "extendedThinking": true,
      "adaptiveThinking": false,
      "prefillSupport": false,
      "effortLevels": ["low", "medium", "high"],
      "releaseDate": "2025-09"
    },
    {
      "name": "Claude Haiku 4.5",
      "id": "claude-haiku-4-5-20251001",
      "contextWindow": 200000,
      "contextWindowBeta": 1000000,
      "maxOutputTokens": 64000,
      "extendedThinking": false,
      "adaptiveThinking": false,
      "prefillSupport": false,
      "effortLevels": ["low", "medium", "high"],
      "releaseDate": "2025-10"
    },
    {
      "name": "Claude Opus 4.5",
      "id": "claude-opus-4-5-20251101",
      "contextWindow": 200000,
      "contextWindowBeta": 1000000,
      "maxOutputTokens": 32000,
      "extendedThinking": true,
      "adaptiveThinking": false,
      "prefillSupport": false,
      "effortLevels": ["low", "medium", "high"],
      "releaseDate": "2025-11"
    }
  ],
  "features": [
    {
      "name": "Adaptive Thinking",
      "status": "ga",
      "models": ["claude-opus-4-6"],
      "betaHeader": null,
      "addedInVersion": "1.0.0",
      "description": "Claude dynamically decides when and how much to think"
    },
    {
      "name": "Extended Thinking",
      "status": "ga",
      "models": ["claude-opus-4-6", "claude-sonnet-4-5-20250929", "claude-opus-4-5-20251101"],
      "betaHeader": null,
      "addedInVersion": "1.0.0",
      "description": "Multi-stage reasoning mode with explicit thinking blocks"
    },
    {
      "name": "1M Context Window",
      "status": "beta",
      "models": ["all"],
      "betaHeader": "required",
      "addedInVersion": "1.0.0",
      "description": "Extended context window requiring usage tier 3+"
    },
    {
      "name": "Effort Parameter",
      "status": "ga",
      "models": ["all"],
      "betaHeader": null,
      "addedInVersion": "1.0.0",
      "description": "Control token usage vs thoroughness with low/medium/high/max levels"
    },
    {
      "name": "Memory Tool",
      "status": "beta",
      "models": ["all"],
      "betaHeader": "required",
      "addedInVersion": "1.0.0",
      "description": "Cross-conversation memory with store/retrieve commands"
    }
  ],
  "tools": [
    {
      "name": "Computer Use",
      "type": "built-in",
      "status": "ga",
      "description": "Desktop automation via screenshots, mouse, and keyboard"
    },
    {
      "name": "Tool Search",
      "type": "built-in",
      "status": "beta",
      "description": "Dynamic tool discovery via regex search for handling thousands of tools"
    },
    {
      "name": "Web Search",
      "type": "built-in",
      "status": "ga",
      "description": "Real-time web search capability"
    },
    {
      "name": "MCP Connector",
      "type": "built-in",
      "status": "beta",
      "description": "Connect to remote MCP servers directly from Messages API"
    }
  ],
  "deprecations": [
    {
      "feature": "budget_tokens",
      "model": "claude-opus-4-6",
      "replacement": "adaptive thinking + effort parameter",
      "deprecatedInVersion": "1.0.0",
      "sunsetDate": "2026-08-10"
    },
    {
      "feature": "output_format parameter",
      "model": "all",
      "replacement": "output_config.format",
      "deprecatedInVersion": "1.0.0",
      "sunsetDate": "2026-06-10"
    },
    {
      "feature": "Assistant message prefill",
      "model": "claude-opus-4-6",
      "replacement": "Use structured outputs or system prompts",
      "deprecatedInVersion": "1.0.0",
      "sunsetDate": "2026-05-10"
    }
  ]
}
```

---

## 7. Versioning Strategy

### Recommendation: Independent semantic versioning

The skill should NOT be tied to Claude model versions. Models release on Anthropic's schedule; the skill updates whenever any capability changes.

**Versioning scheme:** MAJOR.MINOR.PATCH

- **MAJOR**: Structural changes to the skill format, or a new model generation
- **MINOR**: New capabilities, new models, significant feature changes
- **PATCH**: Corrections, clarifications, status changes (beta → GA)

**Examples:**
- 1.0.0 → Initial release covering through Opus 4.6
- 1.1.0 → New model released (e.g., Sonnet 4.6)
- 1.1.1 → Feature moved from beta to GA
- 2.0.0 → Major restructure or Claude 5.x generation

The `lastUpdated` date in SKILL.md provides temporal context. The `skillVersion` in capability-matrix.json enables programmatic tracking.

---

## 8. Distribution Strategy

### Phase 1: skills.sh (Immediate)

Publish as a standalone skill:

```bash
# User installation
claude skills install claude-capabilities
```

The skill works immediately — no commands, no hooks, just the SKILL.md and reference files injected into context when triggered.

**Update process (manual):** maintainer updates the skill files, publishes new version to skills.sh. Users get updates on next install or via `claude skills update`.

### Phase 2: Plugin via Marketplace

Bundle into a plugin with:
- The skill (unchanged)
- /update-claude-capabilities command
- Optional SessionStart hook for lightweight context injection

```json
{
  "name": "claude-capabilities",
  "version": "1.0.0",
  "description": "Keep Claude aware of its current capabilities",
  "author": {
    "name": "Liam Jones",
    "email": "liamjons@hotmail.com"
  },
  "keywords": ["capabilities", "awareness", "models", "api", "tools"]
}
```

### Phase 3: Community Contributions

Open-source the capability-matrix.json and accept PRs. The /update command helps maintainers stay current; community members can submit additions via standard git workflows.

---

## 9. Maintenance Model

### Who Updates

**Primary maintainer** (initially the author) runs /update-claude-capabilities periodically or when Anthropic announces changes.

**Community contributors** can submit PRs to update capability-matrix.json. The maintainer reviews and publishes.

### Update Frequency

| Trigger | Action |
|---------|--------|
| New Claude model release | MINOR version bump, same day |
| Feature GA announcement | PATCH version bump, within a week |
| New beta feature launch | MINOR version bump, within a week |
| Quarterly review | Run /update command, check for drift |

### Quality Gates

Before publishing any update:
1. SKILL.md stays under 500 lines
2. capability-matrix.json validates against schema
3. No duplication of information commonly in training data
4. All code examples are syntactically correct
5. Cross-reference with official Anthropic docs

---

## 10. What This Skill Does NOT Cover

To keep scope manageable and context budget lean:

- **Third-party plugins/skills** — project-specific, not universal
- **Pricing details** — change frequently, easily searched
- **Rate limits** — vary by tier, easily searched
- **SDK installation/setup** — stable, in training data
- **Prompt engineering** — separate domain, well-documented
- **Anthropic company/product information** — not capability-related

---

## 11. Optional: UserPromptSubmit Hook

For the plugin version, an optional UserPromptSubmit hook could inject a minimal capabilities reminder:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "If this request involves building with Claude, designing agents, choosing API features, or answering questions about what Claude can do, load the claude-capabilities skill for current information. Key updates since training: Opus 4.6 with adaptive thinking, 1M context (beta), 128K output, Memory tool, Tool Search, MCP Connector, Compaction API, Files API. Prefill is removed on Opus 4.6. budget_tokens is deprecated in favour of effort parameter."
          }
        ]
      }
    ]
  }
}
```

**Why UserPromptSubmit instead of SessionStart:** UserPromptSubmit fires only when the user submits a prompt, making it lightweight and context-preserving. SessionStart fires on every session initialization, which would inject this reminder regardless of whether the prompt is capability-related, wasting context. UserPromptSubmit is more efficient—it only consumes context when a user actually needs capabilities guidance, not on every session.

---

## 12. MVP Specification

### Minimum Viable Skill (Phase 1)

**Deliverables:**
1. SKILL.md — core capability summary (< 500 lines)
2. references/api-features.md — detailed API feature reference
3. references/tool-types.md — all built-in tool types
4. references/agent-capabilities.md — Agent SDK and agent patterns
5. references/model-specifics.md — per-model capability matrix

**Derived from:** the 23 existing research files in .planning/research/claude-capabilities/, condensed and restructured.

**Success criteria:**
- Claude correctly identifies capabilities it wouldn't otherwise know about
- Context budget impact is < 2,000 tokens for typical queries
- Skill triggers appropriately (not too aggressively, not too rarely)
- Information is accurate and current as of publication date

**Not in MVP:**
- /update-claude-capabilities command
- /publish-claude-capabilities command
- SessionStart hook
- capability-matrix.json (added in Phase 2 for diffing)
- Automated fetching scripts

---

## 13. Risk Analysis

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Skill becomes stale | High | Medium | Regular update cadence, /update command in Phase 2 |
| Context budget bloat | Medium | High | Strict 500-line limit on SKILL.md, progressive disclosure |
| Over-triggering | Low | Medium | Specific trigger keywords, "MANDATORY TRIGGERS" list |
| Under-triggering | Medium | High | Pushy description, broad keyword coverage |
| Inaccurate information | Low | High | Cross-reference with official docs, community review |
| Conflicts with training data | Medium | Low | Focus on post-training features only |

---

## 14. Next Steps

1. **Build MVP skill** — condense the 23 research files into the SKILL.md + 4 reference files
2. **Test triggering** — verify the description triggers appropriately across realistic prompts
3. **Publish to skills.sh** — make available to Claude users
4. **Gather feedback** — monitor usage and accuracy reports
5. **Build plugin (Phase 2)** — add /update command and capability-matrix.json
6. **Open-source** — enable community contributions
