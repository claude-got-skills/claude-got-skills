# 3. Draft judge rubrics from KB

The file is valid. All 55 tests covered, with 3-5 claims per test.

Here is a summary of what was done:

**Output file**: `/private/tmp/claude-501/rubrics_draft.py`

**Coverage**: All 55 tests across 9 categories, with 3-5 factual claims per test (213 total claims).

**Source methodology**: Each claim was cross-referenced against the 28 knowledge-base files in `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/`. The most frequently used sources were:
- `claude-capabilities-new-in-opus-4-6.md` -- for model specs, deprecations, fast mode, adaptive thinking
- `claude-capabilities-context-windows.md` -- for 1M context, tier requirements, pricing
- `claude-capabilities-features-overview.md` -- for feature availability, tool categories, caching
- `claude-capabilities-memory-tool.md` -- for memory tool GA status, client-side operation, commands
- `claude-capabilities-structured-outputs.md` -- for output_config.format, strict mode, SDK helpers
- `claude-capabilities-pdf-support.md` -- for 100-page/32MB limits, processing pipeline
- `claude-code-capabilities-skills.md` -- for skill locations, invocation control, /loop
- `claude-code-capabilities-automate-with-hooks.md` -- for hook events, deterministic execution
- `claude-code-capabilities-create-plugins.md` -- for plugin structure, namespacing, distribution

**Design decisions by category**:
- **Categories 1-5, 7-9**: Claims reference specific numbers, parameter names, model IDs, and feature availability from Anthropic docs
- **Category 6 (Negative tests)**: Claims are domain-specific (Python, TCP/UDP, React) as instructed, not about Claude capabilities
- **Category 7 (Hallucination)**: Claims focus on what IS true (e.g., "memory tool is GA, not beta") to help the judge detect confident-but-wrong answers
- **Category 9 (Conversational users)**: Claims are written to be evaluable for both technical and non-technical responses

---

**Files referenced:**
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/eval_runner.py`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-context-windows.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-files-api.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-features-overview.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-new-in-opus-4-6.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-memory-tool.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-computer-use.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-structured-outputs.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-pdf-support.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-agent-sdk-overview.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-implement-tool-use.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-mcp-via-api.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-programmatic-tool-calling.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-search-results-citations.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-code-capabilities-skills.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-code-capabilities-automate-with-hooks.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-code-capabilities-create-plugins.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-code-capabilities-sandboxing.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-code-capabilities-mcp.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-code-capabilities-extension-options.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/mcp-apps-overview.md`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/claude-capabilities-agent-skills-via-api.md`

---
*Agent: `aded199bd1ab6c9ac` | Session: `2dcfb391-7b28-4ca1-99a5-f90bc4b7a0f1` | Rows: 117*
*2026-03-10T19:14:47.807Z -> 2026-03-10T19:19:48.362Z*