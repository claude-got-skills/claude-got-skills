# Eval Question Audit (2026-03-13)

Audit of 55 eval test questions against current Claude capabilities, scraped docs findings, and skill content.

---

## 1. Questions with Outdated Assumptions

### 1.1 - Q3.2: Opus 4.6 prefill assumption is correct but question framing could be sharper

**Current text:** "I have a working integration with Claude that uses assistant message prefilling to start responses in a specific format. I want to upgrade to Opus 4.6. Anything I should know?"

**Issue:** Not outdated per se, but the question only tests prefill awareness. The migration from Sonnet 4.5 to Opus 4.6 involves five breaking changes (prefill removed, budget_tokens deprecated, output_format deprecated, interleaved thinking header deprecated, tool parameter quoting changes). The question should test awareness of all of them or be explicitly scoped.

**Suggested fix:** Widen to: "I have a working integration using Claude Sonnet 4.5 with assistant message prefilling, `budget_tokens`, and `output_format`. I want to upgrade to Opus 4.6. What breaking changes should I watch for?"

---

### 1.2 - Q5.4: PowerPoint question misses Agent Skills (API)

**Current text:** "I need to create a PowerPoint presentation about our quarterly results. Is there a good way to do this with Claude?"

**Issue:** The skill and scraped docs now document **Agent Skills** (beta) for document generation -- `pptx_20251013`, `xlsx_20251013`, `docx_20251013`, `pdf_20251013` -- available via the API with code execution. Claude.ai also has native artifact-based document creation. The rubric should expect mention of both the API agent skills approach and the Claude.ai artifacts approach, not just "use a skill" or "write Python."

**Suggested fix:** Keep the question but update the rubric to require mention of Agent Skills (API-level pptx generation) and/or Claude.ai's built-in document creation. Also update to include Excel/PowerPoint add-in awareness if those are now live (see Section 2 for new test on add-ins).

---

### 1.3 - Q5.8: Cross-platform skill portability -- partially outdated

**Current text:** "I built a custom skill for Claude Code and it works great. Now my colleague wants to use it on Claude.ai (the web app) and another teammate uses Claude Desktop. How do I get the same skill working across all three platforms?"

**Issue:** The skill content now documents that Claude.ai and Desktop install skills as ZIP via Settings > Capabilities > Skills. CoWork also supports skills via plugins. The question is still valid but should have its rubric updated to include CoWork as a fourth platform, and to mention the Agent Skills open standard (agentskills.io) which enables cross-tool portability beyond just Claude platforms.

**Suggested fix:** Keep question. Update rubric to expect: (1) ZIP upload for Claude.ai/Desktop, (2) filesystem for Claude Code, (3) plugins for CoWork, (4) Agent Skills open standard for broader portability.

---

### 1.4 - Q7.5: Sonnet 3.7 retirement status

**Current text:** "I have a production system using Claude Sonnet 3.7 (claude-3-7-sonnet). A teammate says we need to migrate urgently. Is Sonnet 3.7 still available? What should we move to?"

**Issue:** The skill states Sonnet 3.7 was **retired in Feb 2026**. This is now a factual question with a definitive answer, not a hallucination trap. The test should verify the model correctly states it's retired and recommends Sonnet 4.6 (or Sonnet 4.5 as a closer migration path). The model ID format `claude-3-7-sonnet` is also outdated naming convention.

**Suggested fix:** Keep question -- it now tests whether the model knows about the retirement. Ensure rubric scores positively for: (1) confirming Sonnet 3.7 is retired, (2) recommending Sonnet 4.5 or 4.6, (3) noting Haiku 3.5 is also retired and Haiku 3 retires Apr 2026.

---

### 1.5 - Q8.1: Claude Desktop capabilities -- needs update for new features

**Current text:** "I'm using Claude Desktop for my daily work. I need to set up automated tasks -- like running linting after every file edit and monitoring a service every 5 minutes. Is any of this possible?"

**Issue:** Claude Desktop now has more capabilities than when this was written. The skill still correctly states Desktop doesn't support hooks, background tasks, or code execution. But the new **Remote Control** feature means a Desktop user could potentially trigger Claude Code sessions from their phone/browser. The question should still test the "Desktop can't do this, you need Claude Code" boundary, but the rubric should also accept mention of Remote Control as a bridge.

**Suggested fix:** Keep question. Update rubric to accept: (1) Desktop can't do hooks/background tasks, (2) Claude Code is needed, (3) optionally mentioning Remote Control as a way to access Claude Code capabilities from other devices.

---

### 1.6 - Q8.5: CoWork vs Claude Code comparison -- may underweight new Claude Code features

**Current text:** "My team uses Claude in CoWork. We have skills and MCP integrations through plugins. But I keep hearing about Claude Code features like 'subagents', 'hooks', and 'agent teams'. What are we missing by being on CoWork instead of Claude Code?"

**Issue:** Claude Code has added significant new features since this question was written: Chrome browser integration, Remote Control, web sessions (`--remote`), `/teleport`, `--from-pr`, worktrees, cron scheduling, and more. The rubric should expect mention of the browser automation gap (Chrome integration is Claude Code only) and cross-device features.

**Suggested fix:** Keep question. Expand rubric to expect mention of: (1) Chrome browser integration, (2) background tasks + /loop + cron, (3) worktrees, (4) Remote Control / web sessions as Claude Code advantages over CoWork.

---

### 1.7 - Q8.10: Multi-step workflows on Claude.ai -- new options exist

**Current text:** "I'm on Claude.ai and I want to build a multi-step workflow where Claude reviews code, runs tests, and then creates a PR. Each step needs different instructions. Is this possible on Claude.ai, or do I need something else?"

**Issue:** With **claude.ai/code** (web-based Claude Code sessions) and the `--remote` flag, there's now a bridge between Claude.ai and Claude Code capabilities. Users on claude.ai can access full Claude Code functionality through web sessions. The expected answer should mention this path, not just "you need Claude Code CLI."

**Suggested fix:** Keep question. Update rubric to expect mention of: (1) Claude.ai alone can't do this, (2) Claude Code is needed, (3) **claude.ai/code** provides web-based Claude Code sessions accessible from claude.ai, (4) `--remote` flag can create web sessions from CLI.

---

### 1.8 - Q2.3: Browser automation via API -- Chrome integration exists now

**Current text:** "I want Claude to help fill out web forms and navigate browser interfaces for data entry tasks. Is that possible through the API?"

**Issue:** The question asks specifically about "through the API" which is correctly answered by Computer Use. But the Chrome integration (Claude Code + Chrome extension) is now a much more practical answer for browser automation. The rubric should accept both paths.

**Suggested fix:** Keep question wording (it says "through the API" which keeps Computer Use as the primary answer). Update rubric to also accept mention of Chrome browser integration via Claude Code as an alternative that may be more practical for form-filling use cases.

---

### 1.9 - Q9.6: Document generation capabilities expanded

**Current text:** "I need to create a PowerPoint presentation for a client pitch and an Excel spreadsheet with project costings. Can Claude actually generate these file types, or can it only do text?"

**Issue:** Agent Skills now support native pptx, xlsx, docx, pdf generation via the API. Claude.ai can create artifacts including document types. The rubric should expect awareness of these concrete capabilities rather than vague "use Python libraries" answers. The scraped docs mention **Excel/PowerPoint add-ins with cross-app context sharing** which is a new capability not yet in the skill content.

**Suggested fix:** Keep question. Update rubric to expect: (1) Claude.ai artifacts can create basic documents, (2) API Agent Skills (pptx_20251013, xlsx_20251013) for programmatic generation, (3) mention of Excel/PowerPoint add-ins if those are documented as live.

---

## 2. New Questions Needed

### Priority 1: Code Review (managed service)

**Proposed Q (Code Review service):**
"I want Claude to automatically review pull requests in our GitHub repo. I've heard there's a managed code review service. How does it work, what does it cost, and how do I configure it with a REVIEW.md file?"

**Rationale:** Code Review is a significant new product offering ($15-25/review, REVIEW.md configuration, severity levels). No existing question tests awareness of this managed service. Tests knowledge of pricing, configuration mechanism, and distinction from DIY code review via Claude Code hooks/skills.

---

### Priority 2: Remote Control

**Proposed Q (Remote Control):**
"I use Claude Code on my work laptop but sometimes I need to continue a session from my phone when I'm away from my desk. Is there a way to control my running Claude Code session remotely?"

**Rationale:** Remote Control (`claude remote-control`) enables controlling Claude Code from claude.ai or the Claude mobile app while it runs locally. This is a cross-device workflow that none of the existing 55 questions cover. Tests awareness of the `remote-control` command and its requirements.

---

### Priority 3: claude.ai/code (web sessions)

**Proposed Q (Web sessions):**
"I want to give a non-technical colleague access to Claude Code capabilities without them installing anything locally. They just need to run some data processing scripts. Is there a web-based option?"

**Rationale:** `claude.ai/code` provides web-based Claude Code sessions. The `--remote` flag creates sessions accessible via browser. No existing question directly tests awareness of this capability. Also tests knowledge of `--teleport` (resuming web sessions locally).

---

### Priority 4: Cross-device workflows (/teleport, --remote)

**Proposed Q (Cross-device / teleport):**
"I started a Claude Code session on my work machine to fix a bug, but I need to leave the office. Can I pick up the same session on another device? What are my options?"

**Rationale:** Tests awareness of multiple cross-device features: (1) `--remote` to create web sessions, (2) `--teleport` to resume web sessions locally, (3) `claude remote-control` for live control, (4) session resumption via `--resume` / `--from-pr`. These are significant Claude Code features with no eval coverage.

---

### Priority 5: Chrome integration (browser automation)

**Proposed Q (Chrome integration):**
"I'm building a web app with Claude Code and I want to test my changes by having Claude interact with the actual browser -- clicking buttons, filling forms, checking for visual bugs. How do I set that up?"

**Rationale:** Chrome browser integration (`--chrome`, Claude in Chrome extension) is a major Claude Code feature. Existing Q2.3 asks about API browser automation (Computer Use), but no question tests the Chrome integration specifically. This tests: prerequisites (Chrome extension v1.0.36+), usage (`--chrome` flag, `/chrome` command), capabilities (live debugging, data extraction, GIF recording), limitations (Chrome/Edge only, not Brave/Arc, no WSL).

---

### Priority 6: Slack integration

**Proposed Q (Slack integration):**
"My team uses Slack heavily. I want Claude to interact with our Slack workspace -- reading messages, posting updates, maybe even creating PRs from Slack threads. What are my options?"

**Rationale:** The scraped docs mention @Claude in Slack triggering PR creation. This tests awareness of MCP-based Slack integration and the distinction between: (1) Slack MCP server (via Claude Code), (2) @Claude Slack bot (managed integration), (3) Claude.ai MCP connectors for Slack.

---

### Priority 7: Desktop app vs Claude Desktop distinction

**Proposed Q (Desktop app naming):**
"What's the difference between the 'Claude Desktop' app and the 'Claude Code' desktop extension for VS Code? Are they the same thing? Can they both use MCP servers and skills?"

**Rationale:** Users frequently confuse Claude Desktop (the standalone app for claude.ai-like experience with MCP) and Claude Code (the CLI/IDE extension for development). Existing Q9.7 touches on platform choice but doesn't test the specific Desktop vs Code distinction. This tests: (1) Claude Desktop = conversational UI with Projects, MCP via settings, skill ZIP upload, (2) Claude Code = developer tool with full CLI, filesystem skills, hooks, subagents, (3) different MCP configuration methods, (4) different skill installation methods.

---

### Priority 8: Excel/PowerPoint add-ins with cross-app context

**Proposed Q (Office add-ins):**
"I spend a lot of time working in Excel and PowerPoint. I heard Claude has add-ins for these apps. Can Claude work directly inside my spreadsheets and presentations? Does it know about content across both apps?"

**Rationale:** The scraped docs mention Excel/PowerPoint add-ins with cross-app context sharing. If this feature is live, it represents a significant capability for business users. No existing question covers in-app integrations. Tests awareness of: (1) add-in availability, (2) cross-app context sharing, (3) what operations Claude can perform within Office apps.

---

### Priority 9: JetBrains plugin awareness

**Proposed Q (JetBrains):**
"My team is split between VS Code and IntelliJ/PyCharm. We want everyone using Claude Code with the same plugins, hooks, and skills. Does Claude Code work the same in JetBrains IDEs?"

**Rationale:** The JetBrains plugin is documented in the skill under IDE Extensions as beta. No existing question tests awareness of JetBrains support. Tests: (1) JetBrains plugin exists (beta), (2) feature parity with CLI/VS Code (same tools, skills, hooks, MCP), (3) shared settings via `~/.claude/settings.json`.

---

### Priority 10: Worktrees for parallel sessions

**Proposed Q (Worktrees):**
"I want to work on two features at the same time with Claude Code -- one in the main branch and one in a feature branch. But Claude keeps getting confused about which files to edit. Is there a way to isolate these sessions?"

**Rationale:** Git worktrees (`--worktree` / `-w`) enable isolated parallel Claude Code sessions. This is a practical developer workflow feature with no eval coverage. Tests: (1) `--worktree` flag, (2) subagent `isolation: "worktree"`, (3) `EnterWorktree`/`ExitWorktree` tools.

---

## 3. Questions to Consider Removing or Merging

### 3.1 Merge candidates: Q5.4 and Q9.6 (both test document generation)

**Q5.4:** "I need to create a PowerPoint presentation about our quarterly results."
**Q9.6:** "I need to create a PowerPoint presentation for a client pitch and an Excel spreadsheet with project costings."

**Overlap:** Both test whether Claude knows it can generate pptx/xlsx files. Q9.6 is broader (mentions Excel too) and targets a non-developer persona.

**Recommendation:** Merge into Q9.6 (keep the non-developer framing as it's more useful for Cat 9). Repurpose Q5.4 to test a different extension pattern -- for example, the new Office add-ins capability.

---

### 3.2 Merge candidates: Q2.1 and Q9.5 (both test cross-conversation memory)

**Q2.1:** "Can I get Claude to remember things between separate conversations?"
**Q9.5:** "Every time I start a new chat with Claude, I have to re-explain my company..."

**Overlap:** Both test whether Claude knows about cross-conversation persistence mechanisms (Projects, CLAUDE.md, Memory tool). The difference is persona: Q2.1 is developer-oriented, Q9.5 is business user.

**Recommendation:** Keep both but ensure rubrics are differentiated. Q2.1 should expect technical answers (Memory tool API, CLAUDE.md, skills). Q9.5 should expect accessible answers (Projects, starred instructions, no jargon). If rubrics already differ appropriately, no action needed.

---

### 3.3 Merge candidates: Q5.3 and Q8.3 (both test persistent coding standards)

**Q5.3:** "I want Claude to always know about our company's API conventions..."
**Q8.3:** "I want to store our company's coding standards and style guide so Claude always knows them..."

**Overlap:** Nearly identical use case. Q5.3 is Claude Code-focused (expects CLAUDE.md), Q8.3 is API-focused (expects system prompt embedding).

**Recommendation:** Merge into a single question that spans platforms: "I want Claude to always know our API conventions. I use Claude Code, but my team also uses the API directly. How do I set this up for both?" This tests cross-platform awareness in one question instead of two.

---

### 3.4 Potential removal: Q4.1 (Model Selection -- single test, statistically meaningless)

**Q4.1:** "I need to choose a Claude model for my application. It needs to handle long documents (200+ pages), produce detailed analysis, and keep costs reasonable."

**Issue:** Category 4 has only 1 test, producing +500%/+600% lifts that are statistically meaningless. The question itself is fine, but the category structure inflates results.

**Recommendation:** Either (a) add 2-3 more model selection questions to make the category meaningful, or (b) merge Q4.1 into Category 3 (Implementation Guidance) and dissolve Category 4. Adding more model selection questions is preferred -- for example: "I need to process 10,000 customer support tickets per day. Which model should I use and how do I minimize costs?" and "I'm building a coding assistant that needs to reason about complex algorithms. Which model gives the best results?"

---

### 3.5 Consider merging: Q8.4 and Q8.8 (both test cross-platform feature mapping)

**Q8.4:** "I use Claude Code... A colleague showed me something on Claude.ai where they had a 'Project' with uploaded documents... What are these features?"
**Q8.8:** "I want to understand all the different ways I can extend and customize Claude."

**Overlap:** Both ask for cross-platform feature comparison. Q8.4 is narrower (Projects, artifacts), Q8.8 is comprehensive.

**Recommendation:** Keep Q8.8 (comprehensive overview is valuable). Consider repurposing Q8.4 to focus on a specific cross-platform migration scenario rather than general comparison. For example: "I've been using Claude.ai Projects with uploaded docs. I want to move to Claude Code. How do I replicate my Projects setup?"

---

### 3.6 Consider merging: Q7.2 and Q2.1 / Q9.5 (all test memory/persistence)

**Q7.2:** "Does Claude remember things between conversations by default?"

**Overlap with Q2.1 and Q9.5:** All three test cross-conversation memory, though Q7.2 is in the hallucination category (expected answer: no, not by default).

**Recommendation:** Keep Q7.2 in hallucination category -- it tests a fundamentally different thing (whether the model incorrectly claims it has default memory). The overlap is acceptable because the categories serve different purposes.

---

## Summary

| Action | Count | Details |
|--------|-------|---------|
| Questions with outdated assumptions | 9 | Q3.2, Q5.4, Q5.8, Q7.5, Q8.1, Q8.5, Q8.10, Q2.3, Q9.6 |
| New questions proposed | 10 | Code Review, Remote Control, claude.ai/code, cross-device, Chrome, Slack, Desktop vs Code, Office add-ins, JetBrains, Worktrees |
| Merge/remove candidates | 6 | Q5.4+Q9.6, Q2.1+Q9.5, Q5.3+Q8.3, Q4.1 (expand or dissolve), Q8.4+Q8.8, Q7.2 (keep) |

### Recommended target: 60 tests across 10 categories

If all merges are applied (net -3 questions) and all new questions added (+10), the suite grows from 55 to 62 tests. With Category 4 expansion (+2), that's 64 tests. Consider whether to cap at 60 for eval runtime reasons.

### Suggested new category structure

| Category | Current Count | After Audit |
|----------|--------------|-------------|
| 1. Architecture Decisions | 3 | 3 |
| 2. Can Claude Do X | 5 | 5 |
| 3. Implementation Guidance | 8 | 8 (absorb Q5.3 merge) |
| 4. Model Selection | 1 | 3 (add 2 new) |
| 5. Extension Awareness | 8 | 7 (merge Q5.4 into Q9.6, merge Q5.3 into Q8.3) |
| 6. Negative (No Change Expected) | 3 | 3 |
| 7. Hallucination Detection | 5 | 5 |
| 8. Cross-Platform Awareness | 10 | 10 (merge Q8.4+Q8.8, add Chrome, Desktop vs Code, Worktrees) |
| 9. Conversational Platform Users | 12 | 12 (merge Q9.6+Q5.4, add Office add-ins) |
| 10. New Capabilities (new) | 0 | 6 (Code Review, Remote Control, claude.ai/code, cross-device, Slack, JetBrains) |

**Total: ~62 tests**
