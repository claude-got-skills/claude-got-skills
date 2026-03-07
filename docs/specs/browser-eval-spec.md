# Browser Eval Spec: Automated Claude.ai A/B Testing

**Status:** Spec complete, ready for implementation
**Tool:** agent-browser (v0.13.0)
**Output:** `evals/browser_eval.sh` + `evals/browser_eval_report.py`

## Architecture

| File | Role |
|------|------|
| `evals/browser_eval.sh` | Bash orchestration — calls agent-browser to run prompts |
| `evals/browser_eval_report.py` | Python report generator — reads JSON, produces markdown |
| `evals/claude-ai-auth-state.json` | Saved browser auth state (gitignored) |

## Authentication

One-time setup with `--auth` flag. Uses `--headed` mode for manual 2FA completion, then `state save` for reuse.

```bash
agent-browser --session claude-eval --headed open "https://claude.ai/login"
agent-browser --session claude-eval wait --url "**/claude.ai/new**" --timeout 120000
agent-browser --session claude-eval state save "$AUTH_STATE"
```

## Skill Toggle

Phase-based with manual checkpoint (v1). Script pauses between control/treatment runs and prompts user to toggle skill on Claude.ai settings page. Automated toggle deferred to v2.

## Per-Prompt Execution Flow

1. Navigate to `https://claude.ai/new`
2. `snapshot -i` to find input field
3. `find role textbox fill "$prompt_text"`
4. `press Enter` to submit
5. Poll for completion: check for Copy button presence, fallback to text-stability
6. Extract response text via `get text` or `eval` JavaScript
7. Extract thinking panel content (expand if collapsed)
8. `screenshot` the result
9. Write JSON fragment with prompt, response, thinking, screenshot path

## Response Completion Detection

Layered approach:
1. **Primary**: Poll for Copy button in last response element (appears only after completion)
2. **Fallback**: Text-stability check (response length stops growing)
3. **Last resort**: Hard timeout at 120s with warning

## Test Prompts (5, from browser-test-analysis.md)

| # | Category | Prompt |
|---|----------|--------|
| P1 | Architecture/API | Document processing pipeline in Python, 50-page docs |
| P2 | API Features | Difference between extended and adaptive thinking |
| P3 | Model Selection | Right model for customer support chatbot |
| P4 | Extension Patterns | Code review checklist repeated in every prompt |
| P5 | Product Capabilities | Can Claude remember last week's conversation? |

## Output Format

Per-prompt JSON fragments assembled into `browser-eval-results.json`:
```json
{
  "metadata": { "eval_type": "browser", "platform": "claude.ai", "timestamp": "..." },
  "results": [{
    "prompt_id": "1",
    "category": "Architecture/API",
    "control": { "response": "...", "thinking": "...", "web_search_used": true, "skill_triggered": false },
    "treatment": { "response": "...", "thinking": "...", "web_search_used": true, "skill_triggered": true }
  }]
}
```

## Thinking Panel Analysis

Automated detection from thinking text:
- **Skill triggered**: "check product self-knowledge", "check assistant capabilities", "capabilities skill"
- **Web search used**: "searching", "web search", "search results", "anthropic.com"

## Usage

```bash
./evals/browser_eval.sh --auth          # One-time auth setup
./evals/browser_eval.sh                 # Full run (control + treatment)
./evals/browser_eval.sh --control       # Control only
./evals/browser_eval.sh --treatment     # Treatment only
./evals/browser_eval.sh --report-only   # Regenerate report from existing results
```

## Complementary to API Eval

| Dimension | eval_runner.py (API) | browser_eval.sh (Browser) |
|-----------|---------------------|---------------------------|
| Platform | Anthropic Messages API | Claude.ai web |
| Web search | Not available | Available |
| Thinking panel | Not available | Visible |
| Scoring | Keyword + LLM judge | Trigger detection + qualitative |
| Speed | ~3 minutes | ~15-20 minutes |
| Use case | Regression testing | Real-world validation |

## Known Challenges

- DOM selectors change → use semantic locators, fallback to `eval` JS
- Auth expiration → `--auth` re-authentication path
- contenteditable input → `find role textbox` primary, `@ref` fallback
- Manual skill toggle → acceptable for v1, automate in v2

## Implementation Dependencies

- `agent-browser` CLI (installed, v0.13.0)
- Python 3 (for report generator)
- Claude.ai Pro/Team account
- Skill installed on Claude.ai capabilities page
