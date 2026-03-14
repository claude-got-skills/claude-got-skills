# Sprint 4 Design Spec: Propose SKILL.md Edits from KB Changes

> Generated: 2026-03-14, Session 18
> Status: Design complete, ready for implementation
> Source: Opus 4.6 research agent analysis of full pipeline codebase

---

## 1. Current Content Flow Analysis (KB -> References -> SKILL.md)

**Architecture overview:**

The content hierarchy is three-tiered:

| Layer | Files | Purpose | Token impact |
|-------|-------|---------|-------------|
| Tier 1 | `data/quick-reference.md` (~100 lines) | Loaded via SessionStart hook every session | Always loaded |
| Tier 2 | `skills/assistant-capabilities/SKILL.md` (~280 lines) | Loaded on-demand when skill is invoked | On-demand |
| Tier 3 | `skills/assistant-capabilities/references/*.md` (5 files) | Read by agent when deeper detail needed | On-demand read |

**Source-to-KB mapping (many-to-one):**

23 monitored sources in `pipeline/config.py` map to KB files via the `Source.kb_files` field. This is primarily a one-to-one mapping (one source URL maps to one KB file), with exceptions:
- 3 sources all map to `claude-capabilities-features-overview.md` (API Release Notes, API Overview, Build with Claude Overview)
- 2 sources map to `claude-code-capabilities-desktop.md` (Desktop App, Desktop Quickstart)
- 2 sources map to `claude-code-capabilities-ci-cd.md` (GitHub Actions, GitLab CI/CD)
- 2 sources map to `claude-capabilities-new-in-opus-4-6.md` (Models Overview, Pricing Details)

There are 37 KB files total but only ~18 unique KB filenames in the source registry. The remaining ~19 KB files (e.g., `claude-capabilities-implement-tool-use.md`, `claude-capabilities-files-api.md`, `mcp-apps-overview.md`, etc.) are not tied to any monitored source and were likely manually created.

**KB-to-Reference mapping (many-to-one, manually maintained):**

There is NO explicit mapping from KB files to reference files or SKILL.md sections. This is a key gap. The relationship is implicit:

| Reference file | KB files that feed into it |
|----------------|--------------------------|
| `api-features.md` | structured-outputs, files-api, memory-tool, context-windows, search-results-citations, programmatic-tool-calling, new-in-opus-4-6, pricing |
| `tool-types.md` | computer-use, implement-tool-use, programmatic-tool-calling, mcp-via-api |
| `agent-capabilities.md` | agent-sdk-overview, agent-sdk-python, agent-skills-via-api, automate-with-hooks, create-plugins, create-custom-subagents, mcp, plugin-reference, skills, mcp-apps-overview |
| `claude-code-specifics.md` | code-review, remote-control, web-sessions, slack, agent-teams, use-chrome-browser, cli-reference, jetbrains, vscode, desktop, ci-cd, sandboxing, use-cli, extension-options, skills |
| `model-specifics.md` | new-in-opus-4-6, pricing, features-overview |

SKILL.md then synthesizes from all 5 references into concise summaries. Quick-reference (`data/quick-reference.md`) is a further distillation of SKILL.md.

**Content filtering via the Maintenance Strategy:**

The v2.5.0 maintenance strategy defines what belongs at each tier:

1. **KEEP in SKILL.md** if: load-bearing string (model ID, header, tool type), wrong value causes error (400/404), architectural guidance not in any single doc page, prevents common hallucination
2. **POINTER only** if: number that changes independently (price, rate limit), version number with no functional implication
3. **Reference files** contain: code examples, detailed configurations, migration guides, compatibility matrices
4. **Quick-reference** contains: the most critical subset for Tier 1 (always-loaded context)

## 2. Proposed Sprint 4 Architecture

### 2.1 High-Level Flow

```
Sprint 3 updates KB file(s)
        |
        v
Sprint 4: propose_skill_edits()
        |
        +-- 1. Identify affected user-facing files
        |      (KB -> reference mapping + SKILL.md section mapping)
        |
        +-- 2. For each affected file, generate proposed edit
        |      (LLM call with KB diff, current file, maintenance strategy)
        |
        +-- 3. Write proposals to pipeline/proposals/ directory
        |      (markdown report with unified diffs)
        |
        +-- 4. Notify human reviewer
               (macOS notification + summary in pipeline report)
```

### 2.2 New File: `pipeline/propose_edits.py`

**Core function:**

```python
def propose_skill_edits(
    kb_update_results: list[KBUpdateResult],
    client,
) -> list[ProposalResult]:
```

Triggered after Sprint 3's `update_kb_files()` returns results with status "updated" or "created".

### 2.3 KB-to-Target Mapping Registry

A new mapping is needed in `pipeline/config.py`. This is the critical missing piece. Proposed approach:

```python
# KB file -> which user-facing files it can affect
KB_TO_TARGETS: dict[str, list[str]] = {
    "claude-capabilities-new-in-opus-4-6.md": [
        "skills/assistant-capabilities/SKILL.md",          # Current Models, Breaking Changes sections
        "skills/assistant-capabilities/references/model-specifics.md",
        "skills/assistant-capabilities/references/api-features.md",
        "data/quick-reference.md",
    ],
    "claude-code-capabilities-code-review.md": [
        "skills/assistant-capabilities/SKILL.md",          # Platform Overview section
        "skills/assistant-capabilities/references/claude-code-specifics.md",
    ],
    # ... etc for all 37 KB files
}
```

This is a manual mapping that must be maintained, but it is small (37 entries) and changes rarely (only when new KB files or reference files are added).

### 2.4 Proposal Prompt Design

The LLM prompt for generating proposals needs three inputs:
1. **The KB diff** (from `KBUpdateResult.diff` -- already generated by Sprint 3)
2. **The current target file** (SKILL.md, reference, or quick-reference)
3. **The maintenance strategy rules** (embedded in prompt, not a separate file)

The prompt should instruct the model to:
- Apply the maintenance strategy filter (what belongs vs what's a pointer)
- Preserve the target file's structure and token budget
- Output ONLY a unified diff (not the full rewritten file)
- Explain each proposed change in 1-2 sentences (the "why")

### 2.5 Proposal Output Format

Each proposal should be a markdown file in `pipeline/proposals/`:

```
pipeline/proposals/
  proposal-20260314-123456.md
```

Format:

```markdown
# Skill Edit Proposal -- 2026-03-14

## Trigger
KB file `claude-capabilities-new-in-opus-4-6.md` updated by Sprint 3
Source: "Models Overview" (https://platform.claude.com/docs/en/about-claude/models/overview)
Classification: CRITICAL

## Proposed Changes

### 1. skills/assistant-capabilities/SKILL.md (lines 18-22)

**Rationale:** New model pricing changed from $5/$25 to $6/$30 per MTok for Opus 4.6.

- old: $5/$25 per MTok
+ new: $6/$30 per MTok

### 2. skills/assistant-capabilities/references/model-specifics.md (lines 88-99)

**Rationale:** Pricing section needs update to reflect new Opus 4.6 rates.

(unified diff)

### 3. data/quick-reference.md (line 8)

**Rationale:** Quick reference Opus pricing line outdated.

(unified diff)

## Review Checklist
- [ ] All proposed changes verified against source
- [ ] Maintenance strategy filter applied correctly
- [ ] No token budget increase in SKILL.md
- [ ] Quick-reference stays under 100 lines
```

## 3. Key Design Decisions with Trade-offs

### Decision 1: Model Choice for Proposal Generation

| Option | Pros | Cons |
|--------|------|------|
| **Opus 4.6** | Highest quality diffs, best at understanding maintenance strategy nuance, can reason about multi-file consistency | ~$5/$25 per MTok, ~5-10x more expensive per proposal than Sonnet |
| **Sonnet 4.6** (recommended) | Good quality, same model as Sprint 3 merge, ~$3/$15 per MTok, already proven in KB merge | May miss subtle maintenance strategy decisions |
| **Haiku 4.5** | Cheapest ($1/$5 per MTok) | Too simple for diff generation across files, high error rate on JSON parsing (known issue from judge eval) |

**Recommendation: Sonnet 4.6** -- same as Sprint 3, proven quality. The proposal is reviewed by a human anyway, so perfection is not required. Falls back gracefully to Opus if Sonnet is unavailable (using the `MERGE_MODEL` pattern from `update_kb.py`). Add a `--proposal-model` flag for override.

### Decision 2: Output Format -- Diff vs Full File vs Interactive

| Option | Pros | Cons |
|--------|------|------|
| **Unified diff in markdown report** (recommended) | Human-readable, git-familiar, easy to review, can be applied with `git apply` | Requires human to manually apply changes |
| **Full rewritten files** | Can be applied automatically after review | Harder to review (must diff against current), model may introduce unwanted changes outside the relevant section |
| **Interactive Claude Code prompt** | Most ergonomic, human reviews in-context | Requires a running Claude Code session, not automatable on a schedule |

**Recommendation: Unified diff in markdown report**. The report goes to `pipeline/proposals/` and the human reviewer reads it, then applies the changes manually (or asks Claude Code to apply them). This is the simplest implementation and preserves the human review gate.

### Decision 3: Trigger Mechanism

| Option | Pros | Cons |
|--------|------|------|
| **Inline after Sprint 3** (recommended) | Simplest, proposals appear alongside KB updates | Adds latency to pipeline run (1-3 extra LLM calls) |
| **Separate scheduled run** | Can batch multiple KB updates into one proposal | Requires separate state tracking of "unproposed" KB updates |
| **Manual flag (`--propose-edits`)** | Maximum control, human decides when | Easy to forget, KB and proposals drift apart |

**Recommendation: Inline after Sprint 3**, gated by the existing `--update-kb` flag. Add a new `--propose-edits` flag (requires `--update-kb`) for explicit opt-in, but also allow `--update-kb` to automatically generate proposals when KB files are actually updated.

### Decision 4: Handling Multiple KB Changes Affecting the Same Target File

When multiple KB files are updated in a single pipeline run and they all affect the same reference file (e.g., 3 different KB files all map to `api-features.md`), the proposals must be coordinated.

| Option | Pros | Cons |
|--------|------|------|
| **Single aggregated proposal per target file** (recommended) | Coherent, no conflicting diffs, single review unit | Requires aggregating all KB diffs before generating the proposal |
| **One proposal per KB update** | Simpler implementation | Can produce conflicting diffs for the same file, harder to review |

**Recommendation: Aggregate by target file.** Group all KB diffs that affect the same target file, then generate one proposal per target file with all relevant KB changes included in the context.

### Decision 5: Scope of Proposals (SKILL.md vs References vs Quick-Reference)

The three-tier architecture means a single KB change could affect all three levels. The maintenance strategy determines what goes where:

- **SKILL.md changes**: Only for load-bearing strings, error-preventing values, new capability summaries, or deprecations. Most KB updates will NOT need SKILL.md changes.
- **Reference changes**: Most KB updates will result in reference file proposals.
- **Quick-reference changes**: Only when SKILL.md changes, and only for the most critical subset.

The proposal prompt must encode this filtering logic so the model doesn't propose unnecessary SKILL.md or quick-reference changes.

## 4. Implementation Plan

### Phase 1: Core Implementation (~2-3 hours)

1. **Create `pipeline/propose_edits.py`** (~300-400 lines estimated):
   - `ProposalResult` dataclass (target_file, rationale, diff, status)
   - `propose_skill_edits()` main function
   - `_aggregate_by_target()` helper to group KB diffs by target file
   - `_generate_proposal()` single LLM call per target file
   - `_format_proposal_report()` markdown output
   - `_save_proposal()` write to `pipeline/proposals/`

2. **Add `KB_TO_TARGETS` mapping** to `pipeline/config.py` (~50 lines):
   - Map all 37 KB files to their target files
   - Include a `get_targets_for_kb()` helper function

3. **Integrate into `freshness_check.py`**:
   - Add `--propose-edits` CLI flag
   - Call `propose_skill_edits()` after `update_kb_files()` when KB files were actually updated
   - Log proposal summary

4. **Keep `PROTECTED_PATTERNS` in `update_kb.py`** as-is. Sprint 4 generates *proposals*, never writes to protected files. The safety gate remains.

### Phase 2: Polish (~1 hour)

5. **Notification**: Add macOS notification for proposals ("2 skill edit proposals generated -- review at pipeline/proposals/")
6. **Proposal indexing**: Write a `proposals/index.json` tracking proposal status (pending/applied/rejected) for future automation
7. **Tests**: Unit tests for `_aggregate_by_target()`, `_is_proposal_needed()` (maintenance strategy filter)

## 5. Complexity Estimate

| Component | Estimated Lines | Complexity | Notes |
|-----------|----------------|------------|-------|
| `propose_edits.py` | 300-400 | Medium | Similar pattern to `update_kb.py` |
| `KB_TO_TARGETS` mapping | 50-80 | Low | Manual mapping, straightforward |
| `freshness_check.py` integration | 20-30 | Low | Same pattern as Sprint 3 integration |
| Proposal prompt engineering | N/A | Medium-High | Getting the maintenance strategy filter right in the prompt is the hardest part |
| Total | ~400-500 new lines | **Medium** | Sprint 3 was 555 lines; Sprint 4 is comparable but simpler (no file writes to protected paths) |

## 6. Recommended Approach

**Start with the prompt.** The quality of Sprint 4 depends entirely on the proposal prompt getting the maintenance strategy right. Before writing any infrastructure:

1. Draft the proposal prompt with the 6-point maintenance strategy embedded
2. Test it manually: take a recent KB diff (from Sprint 3 testing), pass it to Sonnet 4.6 with the current SKILL.md and a reference file, and evaluate whether the proposed diff is appropriate
3. Iterate on the prompt 2-3 times until proposals consistently apply the strategy correctly (no unnecessary SKILL.md changes, references get the detail, quick-reference gets only critical changes)
4. Then build the infrastructure around the proven prompt

**Key risks:**
- **Over-proposal**: The model may propose too many SKILL.md changes. Mitigate by encoding the maintenance strategy explicitly and reviewing the first 5-10 proposals carefully.
- **Stale mapping**: `KB_TO_TARGETS` must be updated when KB files or references are added/removed. Mitigate with a validation function that checks all mapped files exist.
- **Token budget**: SKILL.md has a ~280 line budget. Proposals that increase line count should be flagged. Add a line-count check to the proposal output.

## 7. Key Files

Pipeline files:
- `pipeline/update_kb.py` -- Sprint 3 implementation, model for Sprint 4
- `pipeline/freshness_check.py` -- orchestrator to integrate into
- `pipeline/config.py` -- source registry, needs KB_TO_TARGETS mapping
- `pipeline/classify.py` -- classification system (Sprint 2)

Target files (what Sprint 4 would propose changes to):
- `skills/assistant-capabilities/SKILL.md`
- `skills/assistant-capabilities/references/api-features.md`
- `skills/assistant-capabilities/references/tool-types.md`
- `skills/assistant-capabilities/references/agent-capabilities.md`
- `skills/assistant-capabilities/references/model-specifics.md`
- `skills/assistant-capabilities/references/claude-code-specifics.md`
- `data/quick-reference.md`
