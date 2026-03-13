# Subagent Prompts

## 1. Research browser eval selectors

I need to understand the current state of the browser eval script and what needs updating. Thoroughly explore:

1. Read `evals/browser_eval.sh` in `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/` — focus on:
   - Lines 184-196 and 228-261 (response detection selectors)
   - The `run_prompt` function
   - How headed vs headless mode works (ab vs ab_headed)
   - The --yes flag handling (or lack thereof)
   - How responses are currently detected and extracted

2. Check if there are any related config files or helper scripts in the evals/ directory

3. Report back with:
   - The exact current selectors used for response detection
   - What needs to change based on the known working selectors (input: `textbox "Write your prompt to Claude"`, model: `button "Sonnet 4.6 Extended"`)
   - The specific code sections that need editing
   - Any other issues you spot

---

## 2. Research judge rubric implementation

I need to understand the current eval judge implementation to plan an independent rubric system. Thoroughly explore in `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/`:

1. Read `evals/eval_runner.py` — focus on:
   - The `judge_score()` function (or similar) — how does it currently score responses?
   - How SKILL.md content is passed to the judge
   - The test definition format — what fields does each test have?
   - How control vs treatment scoring works

2. Read the test definitions (likely in the eval runner or a separate file) — look at all 55 tests across 9 categories

3. Check `docs/` for any discussion of the independent rubric approach (Option B)

4. Report back with:
   - The exact current judge prompt/system message
   - The current test definition schema
   - Where rubric fields would need to be added
   - How `judge_score()` would need to change
   - Any existing discussion of Option B approach

---

## 3. Draft judge rubrics from KB

You are drafting independent judge rubrics for an eval system. The current eval has a circular bias problem: the judge uses SKILL.md (the skill being tested) as "ground truth" to score responses. We need per-test rubrics sourced from Anthropic's official documentation instead.

**Your task**: Read all 55 test definitions from `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/eval_runner.py` (the TESTS list starts at line 81), then cross-reference against the knowledge-base documents in `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/` to write 3-5 factual claims per test.

**Output format**: Write a Python file at `/private/tmp/claude-501/rubrics_draft.py` containing a dictionary mapping test IDs to rubric lists:

```python
RUBRICS = {
    "1.1": [
        "Claude supports up to 1M token context with the context-1m-2025-08-07 beta header",
        "Files API supports uploads up to 500MB per file",
        "1M context requires usage tier 3 or higher",
    ],
    "1.2": [
        ...
    ],
    # etc for all 55 tests
}
```

**Rules**:
1. Each rubric claim must be a verifiable fact from Anthropic's documentation (the knowledge-base files)
2. Do NOT copy claims from SKILL.md — source them from knowledge-base/*.md files
3. Each claim should be concrete and specific (include numbers, parameter names, model IDs where relevant)
4. For Category 6 (Negative tests — tests about Python, TCP/UDP, React that should NOT change with the skill), write rubrics about the domain topic, not about Claude capabilities
5. For Category 7 (Hallucination Detection), focus on what IS true vs common misconceptions
6. 3-5 claims per test, prioritizing the most important facts for that question

Start by listing the knowledge-base files, then read the test definitions, then systematically work through each test cross-referencing the relevant KB docs.

---

## 4. Review IMS for eval improvements

You have access to a Knowledge Hub (IMS) with 2200+ content items via MCP tools. Your task is to search the IMS for content that could help improve evaluations for a Claude capabilities awareness skill.

The skill helps Claude accurately answer questions about its own capabilities. We need eval questions that test real user confusion points — situations where Claude can do something out-of-the-box but users don't know, or where competing products cause confusion about what Claude can vs can't do.

Search the IMS using these strategies:
1. Search for content about "Claude" or "AI capabilities" or "AI tools"
2. Search for content about competing products (ChatGPT, Gemini, Copilot, etc.) that might reveal confusion points
3. Search for content about document processing, automation, code review, or AI workflows
4. Search for content about user pain points with AI tools

Use the knowledge-hub search tools (mcp__claude_ai_knowledge-hub__search_knowledge_base, mcp__claude_ai_knowledge-hub__search_qa_library, mcp__claude_ai_knowledge-hub__get_content_items) to explore.

For each useful finding, note:
- The content item title/ID
- What eval question it suggests
- Why this is a real user confusion point
- Whether it maps to an existing eval category or needs a new one

Output a structured list of suggested new eval questions with rationale. Focus on questions that test genuine user confusion, not technical API details.

---

## 5. Audit eval questions for staleness

Review all 55 eval test questions at `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/docs/eval-questions.md` and audit them against current Claude capabilities.

Also read:
- The latest scraped docs findings: Code Review (managed PR review service with REVIEW.md, severity levels, $15-25/review), Remote Control (continue sessions from phone/browser), Chrome integration (browser automation), claude.ai/code (web-based Claude Code), Desktop app, Slack integration (@Claude → PRs), /teleport command, JetBrains plugin, Excel/PowerPoint add-ins with cross-app context sharing
- The current skill content at `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/skills/assistant-capabilities/SKILL.md`
- The quick reference at `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/data/quick-reference.md`

Produce three lists:

**1. Questions with outdated assumptions** — questions that reference capabilities, model names, or platform limitations that have changed. For each, note what's outdated and suggest the fix.

**2. New questions needed** — questions for features/capabilities not currently tested. Prioritize:
- Code Review (managed service)
- Remote Control
- claude.ai/code (web sessions)
- Chrome integration
- Slack integration
- Excel/PowerPoint add-ins
- Cross-device workflows (/teleport, /desktop, /rc)
- Desktop app vs Claude Desktop distinction

**3. Questions to consider removing or merging** — any that overlap significantly or test edge cases that aren't worth the eval time.

Write the output to `/private/tmp/claude-501/eval-question-audit.md`.

---

*Session: `2dcfb391-7b28-4ca1-99a5-f90bc4b7a0f1`*