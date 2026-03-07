# Session Prompt: Build the Claude Capabilities Awareness Skill

## Context

We've completed a design document for a Claude Capabilities Awareness Skill — a skill that keeps Claude aware of its current capabilities that may not be in its training data. The full design is at `.planning/designs/claude-capabilities-awareness-skill-design.md`.

## What to build

Build the **Phase 1 MVP** — a standalone skill publishable via skills.sh. This means:

### Deliverables

1. **SKILL.md** — the core skill file with YAML frontmatter and capability summary body
2. **references/api-features.md** — detailed API feature docs with code examples, beta headers, platform availability
3. **references/tool-types.md** — all built-in tool types with configuration and usage patterns
4. **references/agent-capabilities.md** — Agent SDK, subagents, skills system, hooks, MCP integration
5. **references/model-specifics.md** — per-model capability matrix, pricing tiers, limits, migration guidance

### Source material

We have ~23 research files in `.planning/research/claude-capabilities/` covering API features, Agent SDK, computer use, hooks, plugins, MCP, structured outputs, and more. These should be condensed and restructured into the skill format — not just copied.

### Key constraints from the design

- **SKILL.md must stay under 500 lines.** Use progressive disclosure — keep the body concise, put detail in references/.
- **Delta-focused.** Emphasise capabilities Claude's training likely missed. Don't restate common knowledge.
- **Structured for scanning.** Use tables and short entries so Claude can quickly locate relevant capabilities.
- **Actionable.** Each entry tells Claude what it CAN DO, not just what exists.
- **Model-aware.** Capabilities vary by model — make this explicit.

### Skill description (triggering)

The description must be "pushy" to ensure Claude loads it when making capability-dependent decisions. Use the description from the design doc Section 3.3 — it covers architectural decisions, implementation approach queries, and explicit "can Claude do X?" questions, with mandatory trigger keywords.

### Writing style

- Use imperative/infinitive form (verb-first instructions), NOT second person
- Include a "Last updated" date and "Covers models through" marker
- The body should have: Current Models table, capability sections grouped by domain, Breaking Changes section, Reference Files pointers with guidance on when to read each

### What NOT to include (out of scope for Phase 1)

- /update-claude-capabilities command
- /publish-claude-capabilities command
- SessionStart or UserPromptSubmit hooks
- capability-matrix.json (Phase 2)
- Automated fetching/diffing scripts (Phase 2)
- README.md, CHANGELOG.md, or other human-facing docs

## Build process

1. Read the design doc at `.planning/designs/claude-capabilities-awareness-skill-design.md` for full context
2. Read the `plugin-dev:skill-development` skill for authoritative guidance on skill structure
3. Read the `skill-creator` skill (at `/sessions/eager-focused-dirac/mnt/.skills/skills/skill-creator/SKILL.md`) for creation best practices
4. Survey the source research files in `.planning/research/claude-capabilities/` to understand what content is available
5. Build the skill in a new directory. Suggested location: `.planning/builds/claude-capabilities-skill/` (working directory) with the final publishable output structured as:
   ```
   claude-capabilities/
   ├── SKILL.md
   └── references/
       ├── api-features.md
       ├── tool-types.md
       ├── agent-capabilities.md
       └── model-specifics.md
   ```
6. After building, use the `plugin-dev:skill-development` skill and/or `skill-creator` to validate the skill structure, description quality, and progressive disclosure architecture

## Validation criteria

- [ ] SKILL.md has valid YAML frontmatter with name, description, and version
- [ ] Description is in third-person "This skill should be used when..." format with specific trigger phrases
- [ ] SKILL.md body is under 500 lines
- [ ] Body uses imperative/infinitive writing style (not second person)
- [ ] All four reference files exist and contain substantive content
- [ ] Reference files are clearly pointed to from SKILL.md with "when to read" guidance
- [ ] Content is delta-focused (post-training features emphasised, common knowledge omitted)
- [ ] Model-specific capabilities are clearly distinguished
- [ ] Information is accurate against source research files
- [ ] Context budget is reasonable (~1,500 tokens for body, ~2,000-4,000 per reference file)
