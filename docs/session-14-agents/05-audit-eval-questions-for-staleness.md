# 5. Audit eval questions for staleness

The audit is complete. Here is a summary of what was produced:

**File written:** `/private/tmp/claude-501/eval-question-audit.md`

**Key findings across three sections:**

**Section 1 -- 9 questions with outdated assumptions:**
- Q3.2 (Opus 4.6 migration) only tests prefill, should test all 5 breaking changes
- Q5.4 and Q9.6 (document generation) miss Agent Skills API (`pptx_20251013`, etc.) and potential Office add-ins
- Q5.8 (cross-platform skills) missing CoWork as a fourth platform and Agent Skills open standard
- Q7.5 (Sonnet 3.7) -- model is now definitively retired (Feb 2026), no longer ambiguous
- Q8.1 (Desktop automation) should acknowledge Remote Control as a bridge to Claude Code
- Q8.5 (CoWork vs Claude Code) underweights Chrome integration, worktrees, cron, Remote Control
- Q8.10 (multi-step on Claude.ai) should mention claude.ai/code web sessions and `--remote`
- Q2.3 (browser automation) should mention Chrome integration alongside Computer Use API

**Section 2 -- 10 new questions proposed, prioritized:**
1. Code Review managed service (REVIEW.md, $15-25/review, severity levels)
2. Remote Control (`claude remote-control` from phone/browser)
3. claude.ai/code web sessions (`--remote` flag)
4. Cross-device workflows (`--teleport`, session handoff)
5. Chrome browser integration (`--chrome`, extension prerequisites)
6. Slack integration (@Claude, MCP servers)
7. Desktop app vs Claude Code distinction
8. Excel/PowerPoint add-ins with cross-app context
9. JetBrains plugin parity
10. Git worktrees for parallel sessions

**Section 3 -- 6 merge/removal candidates:**
- Q5.4 + Q9.6 (both test document generation)
- Q5.3 + Q8.3 (both test persistent coding standards)
- Q4.1 (Model Selection category has only 1 test -- statistically meaningless +500% lifts; expand to 3 or merge into Category 3)
- Q8.4 + Q8.8 (both test cross-platform feature mapping)

Recommended target: ~62 tests across 10 categories (current 9 + new "New Capabilities" category).

---

**Files referenced:**
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/docs/eval-questions.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/skills/assistant-capabilities/SKILL.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/data/quick-reference.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/skills/assistant-capabilities/references/claude-code-specifics.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/skills/assistant-capabilities/references/agent-capabilities.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-code-capabilities-use-chrome-browser.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-code-capabilities-cli-reference.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-features-overview.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/skills/assistant-capabilities/references/model-specifics.md`

---
*Agent: `adebdba3e496cc302` | Session: `2dcfb391-7b28-4ca1-99a5-f90bc4b7a0f1` | Rows: 75*
*2026-03-13T16:07:54.717Z -> 2026-03-13T16:11:11.543Z*