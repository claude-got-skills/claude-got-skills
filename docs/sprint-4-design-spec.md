# Sprint 4 Design Spec: Propose SKILL.md Edits from KB Changes

> Generated: 2026-03-14, Session 18
> Status: Design complete, ready for implementation

## Overview

When Sprint 3 updates a KB file, Sprint 4 proposes edits to user-facing files (SKILL.md, references/, quick-reference) with a human review gate. Sprint 4 never writes directly to protected files.

## Content Flow (KB -> References -> SKILL.md)

| Layer | Files | Purpose | Token impact |
|-------|-------|---------|-------------|
| Tier 1 | `data/quick-reference.md` (~100 lines) | SessionStart hook, always loaded | Always loaded |
| Tier 2 | `skills/assistant-capabilities/SKILL.md` (~280 lines) | On-demand when skill invoked | On-demand |
| Tier 3 | `skills/assistant-capabilities/references/*.md` (5 files) | Read by agent for deeper detail | On-demand read |

## Architecture

```
Sprint 3 updates KB file(s)
        |
        v
Sprint 4: propose_skill_edits()
        |
        +-- 1. Identify affected user-facing files (KB_TO_TARGETS mapping)
        +-- 2. Aggregate KB diffs by target file
        +-- 3. Generate proposed edit per target (LLM call with maintenance strategy)
        +-- 4. Write proposals to pipeline/proposals/ (markdown with unified diffs)
        +-- 5. Notify human reviewer (macOS notification)
```

## New Component: `pipeline/propose_edits.py` (~300-400 lines)

### Core API
```python
def propose_skill_edits(
    kb_update_results: list[KBUpdateResult],
    client,
) -> list[ProposalResult]:
```

### Key helpers
- `_aggregate_by_target()` — group KB diffs by target file
- `_generate_proposal()` — single LLM call per target file
- `_format_proposal_report()` — markdown output with unified diffs
- `_save_proposal()` — write to `pipeline/proposals/`

## New Mapping: `KB_TO_TARGETS` in `pipeline/config.py`

Maps each of the 37 KB files to their affected user-facing files:

```python
KB_TO_TARGETS: dict[str, list[str]] = {
    "claude-capabilities-new-in-opus-4-6.md": [
        "skills/assistant-capabilities/SKILL.md",
        "skills/assistant-capabilities/references/model-specifics.md",
        "skills/assistant-capabilities/references/api-features.md",
        "data/quick-reference.md",
    ],
    "claude-code-capabilities-code-review.md": [
        "skills/assistant-capabilities/SKILL.md",
        "skills/assistant-capabilities/references/claude-code-specifics.md",
    ],
    # ... etc
}
```

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Model** | Sonnet 4.6 | Same as Sprint 3, proven quality, human reviews anyway |
| **Output format** | Unified diff in markdown report | Human-readable, git-familiar, preserves review gate |
| **Trigger** | Inline after Sprint 3 | Simplest, proposals appear alongside KB updates |
| **Multi-KB coordination** | Aggregate by target file | One proposal per target, no conflicting diffs |
| **Scope filtering** | Maintenance strategy in prompt | 6-point filter determines what belongs at each tier |

## Proposal Output Format

```markdown
# Skill Edit Proposal — 2026-03-14

## Trigger
KB file `claude-capabilities-new-in-opus-4-6.md` updated by Sprint 3
Source: "Models Overview"
Classification: CRITICAL

## Proposed Changes

### 1. skills/assistant-capabilities/SKILL.md (lines 18-22)
**Rationale:** New model pricing changed.
(unified diff)

### 2. references/model-specifics.md (lines 88-99)
**Rationale:** Pricing section needs update.
(unified diff)

## Review Checklist
- [ ] All proposed changes verified against source
- [ ] Maintenance strategy filter applied correctly
- [ ] No token budget increase in SKILL.md
- [ ] Quick-reference stays under 100 lines
```

## Integration into `freshness_check.py`

- New `--propose-edits` CLI flag (requires `--update-kb`)
- Called after `update_kb_files()` when KB files were actually updated
- Full command: `python -m pipeline.freshness_check --classify --update-kb --propose-edits`

## Maintenance Strategy Filter (embedded in prompt)

1. Load-bearing string (model ID, header, tool type)? → KEEP in SKILL.md
2. Wrong value causes error (400, 404, silent failure)? → KEEP in SKILL.md
3. Architectural guidance not in any single docs page? → KEEP in SKILL.md
4. Prevents common hallucination? → KEEP in SKILL.md
5. Number that changes independently of features? → POINTER only
6. Version number with no functional implication? → POINTER or omit

## Implementation Plan

1. **Draft and test the proposal prompt** (start here — quality depends on this)
2. Create `pipeline/propose_edits.py` with ProposalResult, aggregation, LLM call
3. Add `KB_TO_TARGETS` mapping to `pipeline/config.py`
4. Integrate `--propose-edits` flag into `freshness_check.py`
5. Add macOS notification for proposals
6. Optional: proposal index tracking (pending/applied/rejected)

## Complexity Estimate

~400-500 new lines, medium complexity. Similar pattern to Sprint 3 (`update_kb.py`, 555 lines) but simpler since it generates proposals rather than writing to protected files.

## Key Risks

- **Over-proposal**: Model may propose too many SKILL.md changes. Encode maintenance strategy explicitly.
- **Stale mapping**: `KB_TO_TARGETS` must be updated when files are added/removed. Add validation.
- **Token budget**: Flag proposals that increase SKILL.md line count.
