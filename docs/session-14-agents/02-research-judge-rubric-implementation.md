# 2. Research judge rubric implementation

I now have all the information needed. Here is the complete analysis.

---

## Analysis: Current Judge Implementation and Independent Rubric Plan

### 1. Current Judge Prompt / System Message

The judge function is `judge_score()` at **line 1310** of `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/eval_runner.py`.

**Judge system prompt** (lines 1329-1342):

```python
judge_system = f"""You are an expert evaluator scoring AI responses about Claude's capabilities.

IMPORTANT: Use the following reference document as GROUND TRUTH for what Claude can actually do.
Responses that mention specific features, parameter names, headers, or patterns from this
reference are MORE accurate than generic responses, even if the generic response sounds confident.

## Ground Truth Reference (Claude Capabilities)
{skill_body}

## END OF REFERENCE

Score the response below against this ground truth. A response that mentions specific
API headers, model IDs, tool type strings, or extension patterns from the reference
is more accurate than one that gives vague or generic advice."""
```

**Judge user prompt** (lines 1344-1363):

```python
judge_prompt = f"""## Original Prompt
{test['prompt']}

## Expected Knowledge
A good answer should include concepts like: {expected_knowledge}

## Response to Evaluate
{response}

## Scoring Rubric
Rate each dimension 0-3:
- **Accuracy** (0-3): 0=Incorrect/outdated info, 1=Partially correct but vague, 2=Correct and specific, 3=Correct with exact parameter names, headers, versions from the reference
- **Completeness** (0-3): 0=Misses the key capability entirely, 1=Mentions it vaguely, 2=Covers main points with some specifics, 3=Main points + edge cases + practical guidance
- **Actionability** (0-2): 0=Vague suggestions, 1=Gives direction, 2=Gives specific code/config the user can use

Also check:
- **Avoids Deprecated** (0-1): 1=Uses current approaches per reference, 0=Recommends deprecated patterns

Return ONLY a JSON object (keep reasoning under 15 words):
{{"accuracy": N, "completeness": N, "actionability": N, "avoids_deprecated": N, "reasoning": "brief"}}"""
```

**The circular bias**: The full SKILL.md content (`skill_body`) is injected as "GROUND TRUTH" in the judge's system prompt. This means:
- Treatment responses that parrot SKILL.md content score higher because the judge sees them matching the reference
- The eval measures "how well does the response match SKILL.md" rather than "how factually correct is the response"
- This inflates treatment lift numbers

**How skill content reaches the judge**: The `skill_body` variable is loaded at the start of `main()` (line 1588) via `load_skill_content()`, which reads and strips frontmatter from SKILL.md. The same `skill_body` is passed both to `run_test()` (for treatment system message) and to `judge_score()` (for the judge rubric). The judge receives it identically for both control and treatment scoring.

**The `expected_knowledge` derivation** (lines 1319-1326): Takes the first 3 accuracy keywords and first 3 completeness keywords from the test's `scoring_keywords`, using the first synonym from each group as a label. This gives the judge a hint about what to look for, but it's secondary to the full SKILL.md being the ground truth.

### 2. Current Test Definition Schema

Each test is a dictionary with exactly 3 fields:

```python
{
    "id": str,                    # e.g. "1.1", "7.5", "9.12"
    "category": str,              # e.g. "Architecture Decisions", "Hallucination Detection"
    "prompt": str,                # The user question sent to the model
    "scoring_keywords": {
        "accuracy": list,         # Keywords/synonym-groups the response SHOULD contain
        "completeness": list,     # Additional keywords for thoroughness scoring
        "deprecated_patterns": list,  # Keywords the response should NOT contain
    },
}
```

Keyword entries can be either:
- `str` -- exact substring match (case-insensitive)
- `list[str]` -- synonym group, any match counts as one hit

There are **55 tests across 9 categories**:
- Cat 1: Architecture Decisions (3 tests: 1.1-1.3)
- Cat 2: Can Claude Do X (5 tests: 2.1-2.5)
- Cat 3: Implementation Guidance (8 tests: 3.1-3.8)
- Cat 4: Model Selection (1 test: 4.1)
- Cat 5: Extension Awareness (8 tests: 5.1-5.8)
- Cat 6: Negative / No Change Expected (3 tests: 6.1-6.3)
- Cat 7: Hallucination Detection (5 tests: 7.1-7.5)
- Cat 8: Cross-Platform Awareness (10 tests: 8.1-8.10)
- Cat 9: Conversational Platform Users (12 tests: 9.1-9.12)

### 3. Dual Scoring System

The eval runs **two independent scoring mechanisms** per test:

**A. Keyword scoring** (`keyword_score()`, line 1275): Pure substring matching. Counts how many accuracy/completeness keywords appear in the response, and how many deprecated patterns appear. Returns counts and percentages. This is objective but brittle.

**B. LLM-as-judge scoring** (`judge_score()`, line 1310): Calls the judge model with SKILL.md as ground truth. Returns accuracy (0-3), completeness (0-3), actionability (0-2), avoids_deprecated (0-1). This is the circular one.

Both control and treatment responses go through the same judge with the same SKILL.md ground truth. The judge prefill trick (line 1368: `{"role": "assistant", "content": "{"}`) forces JSON output.

### 4. Where Rubric Fields Would Need to Be Added

Based on the session-14 handoff document at `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/docs/continuation-prompts/session-14-handoff.md` (lines 165-171), the plan is:

**In the test definitions** (lines 81-1253 of eval_runner.py): Add a `rubric` field to each test dict containing 3-5 factual claims sourced from Anthropic docs, not from SKILL.md. Example:

```python
{
    "id": "7.5",
    "category": "Hallucination Detection",
    "prompt": "...",
    "scoring_keywords": { ... },
    "rubric": [
        "Claude 3.7 Sonnet was retired in February 2026",
        "Current Sonnet models are Claude Sonnet 4.5 and Claude Sonnet 4.6",
        "Migration requires updating model IDs and handling breaking changes like prefill removal",
    ],
}
```

### 5. How `judge_score()` Would Need to Change

The key change is replacing the SKILL.md ground truth with the per-test rubric:

**Current** (line 1329-1342): System prompt injects the entire `skill_body` (~270+ lines of SKILL.md) as ground truth.

**Required changes**:

1. **Replace the system prompt**: Instead of `{skill_body}`, use `test["rubric"]` -- the 3-5 factual claims for that specific test.

2. **Change the `judge_system` template** to something like:
   ```
   You are an expert evaluator scoring AI responses about Claude's capabilities.
   
   Use the following FACTUAL CLAIMS as your scoring rubric. These are verified facts
   from Anthropic's official documentation:
   
   {formatted_rubric_claims}
   
   Score the response based on whether it aligns with these facts.
   ```

3. **Update the function signature**: The `skill_body` parameter could become optional or be removed. The rubric data comes from `test["rubric"]` instead.

4. **Fallback behavior**: For tests that don't yet have a `rubric` field, the function could fall back to the current SKILL.md approach. This allows incremental migration.

5. **The `expected_knowledge` derivation** (lines 1319-1326) could be replaced or supplemented by the rubric claims, since those are more descriptive than keyword labels.

6. **The scoring dimensions** (accuracy 0-3, completeness 0-3, actionability 0-2, avoids_deprecated 0-1) can stay the same -- the rubric changes only affect what counts as "correct," not the scoring scale.

### 6. Existing Discussion of Option B

The only explicit discussion is in `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/docs/continuation-prompts/session-14-handoff.md`:

- **Line 95**: "Eval methodology has circular bias: Judge receives SKILL.md as 'ground truth' -- needs independent rubric"
- **Lines 165-171**: The implementation plan:
  - Add `rubric` field to each test with 3-5 factual claims from Anthropic docs (not skill content)
  - Modify `judge_score()` to use rubric instead of full SKILL.md
  - Re-run eval with new rubric for unbiased scores
  - References "session 13 discussion" but that discussion is not saved as a document -- it exists only in the MEMORY.md notes

The MEMORY.md also captures the rationale (under "Eval Methodology Issues"):
- "Circular judge bias: Judge receives SKILL.md as 'ground truth' -- scores parrot skill content, not user helpfulness"
- "Fix planned: Option B -- independent per-test rubric with factual claims from Anthropic docs, not skill content"

No other docs, scripts, or code files contain discussion of this approach.

### 7. Key Implementation Considerations

- **55 tests need rubrics**: Each needs 3-5 factual claims sourced from the 28 knowledge-base files in `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/knowledge-base/`. This is the bulk of the work.
- **Category 6 (Negative tests)** don't test Claude capabilities, so they may not need rubrics, or need domain-specific rubrics (Python, TCP/UDP, React).
- **Category 7 (Hallucination)** tests are the highest-value targets for independent rubrics since they test factual correctness about what Claude can/cannot do.
- **The keyword scoring is unaffected** by this change -- it remains as a parallel, objective measure.
- **Re-running evals** after the rubric change will show whether the reported lift numbers were genuinely inflated or mostly valid.

---

**Files referenced:**
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/eval_runner.py`
- `/Users/liamj/.claude/projects/-Users-liamj-Documents-development-claude-got-skills/2dcfb391-7b28-4ca1-99a5-f90bc4b7a0f1/tool-results/toolu_01XrFvG6xyD752nwppCeBZXm.txt`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/docs/continuation-prompts/session-14-handoff.md`

---
*Agent: `a03b6f97f86e70ee0` | Session: `2dcfb391-7b28-4ca1-99a5-f90bc4b7a0f1` | Rows: 79*
*2026-03-10T19:11:47.630Z -> 2026-03-10T19:13:32.978Z*