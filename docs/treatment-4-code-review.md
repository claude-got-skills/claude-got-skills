# Treatment Test - Prompt 4: Code Review Checklist in Claude Code
**Skill status**: ON
**Prompt**: "I've been writing the same code review checklist into every prompt. There must be a better way to do this in Claude Code?"

## Key Observations
- **Skill triggered**: YES — "Check assistant capabilities skill for Claude Code configuration details"
- Thinking explicitly says "Let me check the assistant-capabilities skill for the most accurate info"
- Two options presented: CLAUDE.md vs Skills
- CLAUDE.md: project root or ~/.claude/CLAUDE.md, loaded every session
- Skills: .claude/skills/ (project) or ~/.claude/skills/ (global), Agent Skills standard
- Frontmatter example with autoInvoke: true
- Key rule of thumb: "CLAUDE.md = always know this, Skill = know this when relevant"
- Recommended autoInvoke: true skill for code review use case
- CLAUDE.md guidance: keep under ~500 lines
- Mentioned skills.sh community registry
- Mentioned find-skills meta-skill: `npx skills add vercel-labs/skills@find-skills`
- Addressed Liam by name

## Comparison to Control
- Control also mentioned CLAUDE.md (3 scopes), slash commands, and skills
- Treatment was MORE STRUCTURED in presenting the choice (CLAUDE.md vs Skill)
- Treatment included autoInvoke frontmatter, which control may not have
- Treatment mentioned skills.sh and find-skills - specific, actionable details
- Treatment gave the "keep CLAUDE.md under 500 lines" guideline
- **Treatment was notably better on this prompt** - more actionable, clearer decision framework
