# 1. Research browser eval selectors

Here is the full analysis of the browser eval script's current state and what needs updating.

---

## Files Examined

- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/browser_eval.sh` (742 lines, main script)
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/browser_eval_report.py` (356 lines, report generator)
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/browser-eval-run-20260310-123147/P1_control.json` (single failed run result)

No other helper scripts or config files exist in the evals directory beyond the eval runner, the browser eval report script, and result files.

---

## 1. Response Detection Selectors (Lines 184-196 and 228-261)

### Lines 184-196: `wait_for_response` text-stability fallback

The current JavaScript uses these CSS selectors to measure response length:

```javascript
var el = document.querySelector('[data-testid="chat-message-text"]');
// fallback:
var els = document.querySelectorAll('div.font-claude-message, div.prose');
```

These are speculative/generic CSS selectors. There is no evidence these match Claude.ai's current DOM structure. The `data-testid="chat-message-text"` attribute and `div.font-claude-message` class may not exist in the live site.

### Lines 228-261: `extract_response` function

Strategy 1 (line 231-244) tries these CSS selectors in order:
```javascript
'[data-testid="chat-message-text"]'
'div.font-claude-message'
'div.prose'
'div[class*="response"]'
'div[class*="message"][class*="assistant"]'
```

Strategy 2 (line 252-261) falls back to broader selectors:
```javascript
'div[class*="message"], div[class*="chat"], article'
```

All of these are guesses. None are validated against the current Claude.ai DOM.

### Lines 268-313: `extract_thinking` function

Tries these CSS selectors:
```javascript
'[data-testid="thinking-content"]'
'div[class*="thinking"]'
'div[class*="reasoning"]'
'details[class*="think"] div'
'div[class*="thought"]'
```

Also uses snapshot grep for a thinking toggle button: `grep -i 'think\|reasoning'` looking for a `@e[N]` ref to click.

---

## 2. Input Field Discovery (Lines 353-361 in `run_prompt`)

The current approach on line 356 greps the snapshot for:
```
'textbox|contenteditable|paragraph|ProseMirror|editor|"Send a message"|"Reply"|"How can"|placeholder'
```

The known working selector from the memory notes is: `textbox "Write your prompt to Claude"`. The current grep pattern does NOT include this string. The grep would match `textbox` generically, but the broader pattern could match wrong elements.

The fallback on line 360 greps for:
```
'textarea|input|[Mm]essage|[Cc]hat|[Pp]rompt'
```

**The P1_control.json result confirms this is broken** -- the only browser eval run attempt failed with `"Input field not found"`, meaning the snapshot grep on line 356 returned no matches and the fallback on line 360 also failed.

---

## 3. Headed vs Headless Mode (`ab` vs `ab_headed`)

- **`ab`** (line 68-71): Runs headless with `agent-browser --session "$SESSION" --executable-path "$CHROME_PATH" --args "$CHROME_ARGS"`. No `--headed` flag.
- **`ab_headed`** (line 73-76): Same but adds `--headed`. Only used for auth flow (`do_auth`, line 90 and 100).

**Issue:** All actual eval runs (in `run_condition` and `run_prompt`) use `ab` (headless). For debugging, there is no way to run the eval in headed mode. The memory notes say "headed mode for eval runs" is needed. You would need to either add a `--headed` CLI flag to the script or change `run_prompt` to optionally use `ab_headed`.

---

## 4. The `--yes` Flag (Missing)

There is **no `--yes` flag** implemented. The full run mode (lines 669-729) has two interactive prompts that block automation:

- **Line 680-683**: `echo -n "Confirm: Is the claude-capabilities skill DISABLED on Claude.ai? [y/N] "` followed by `read -r confirm` with a check that aborts if not "y"/"Y".
- **Line 704**: `echo -n "Have you ENABLED the skill? Press Enter to continue..."` followed by `read -r`.

A `--yes` flag would need to skip both of these interactive confirmations to allow fully unattended runs.

---

## 5. Summary of What Needs Changing

### Critical (script is non-functional without these):

1. **Input field discovery (line 356)**: Add `"Write your prompt to Claude"` to the grep pattern. The known working selector is `textbox "Write your prompt to Claude" [ref=eNN]`. The existing grep for `textbox` alone may match it, but given it failed in practice, the snapshot format may not match the grep pattern. Need to verify against a real snapshot.

2. **Response detection selectors (lines 185-196, 229-246)**: The CSS selectors (`data-testid="chat-message-text"`, `div.font-claude-message`, `div.prose`) are likely stale. These need to be updated to match Claude.ai's current DOM. This requires inspecting the live DOM or capturing a snapshot during a headed debug run to identify the correct selectors.

3. **Snapshot-based completion detection (line 173)**: The grep for `'Copy\|"Retry"\|"Try again"'` may not match the current UI buttons. These button labels need to be verified against the live site.

### Important (functional improvements):

4. **Headed mode for eval runs**: Add a `--headed` or `--debug` flag that uses `ab_headed` instead of `ab` in `run_prompt`, so you can visually observe what is happening during eval runs.

5. **`--yes` flag**: Add a flag to skip interactive confirmations on lines 680-683 and 704 for unattended runs.

6. **Model button selector**: The known working selector is `button "Sonnet 4.6 Extended" [ref=eNN]`. The script currently does NOT verify or select a specific model -- it just uses whatever model is defaulted on Claude.ai. If you want model-specific testing, you would need to add model selection logic.

### Minor issues:

7. **`get_prompt_text` uses `eval echo`** (line 57): The `eval echo` pattern will expand shell metacharacters in prompt text (e.g., `$`, backticks). This could corrupt prompts containing special characters. Using `eval printf '%s'` or a different approach would be safer.

8. **Trailing newlines in results**: The P1_control.json shows category and prompt values with trailing `\n` (e.g., `"Vision / PDF\n"`). This comes from the `eval echo` on lines 55-57 which appends a newline, then the heredoc in `write_result` (line 459) preserves it. The `json_escape` function on line 441 should handle this, but the data still has the artifact.

9. **`wait_for_response` primary detection** (line 169-181): The `ab snapshot -i` approach greps for `Copy` case-insensitively, but this could false-positive on response content containing the word "Copy". The `copy_count >= 1` check on line 177 doesn't help since one match is enough to trigger.

---

**Files referenced:**
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/browser_eval.sh`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/browser_eval_report.py`
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/browser-eval-run-20260310-123147/P1_control.json`

---
*Agent: `a823fb350270c24d7` | Session: `2dcfb391-7b28-4ca1-99a5-f90bc4b7a0f1` | Rows: 24*
*2026-03-10T19:11:41.388Z -> 2026-03-10T19:12:44.067Z*