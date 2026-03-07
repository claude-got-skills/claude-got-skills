# Browser Eval Spec — 4-Eye Review Findings

## Feasibility Issues

1. **`find role textbox fill` syntax may not work** — needs `--name` flag or fallback to snapshot + `@ref`
2. **`wait --url` glob pattern** (`**/claude.ai/new**`) — double-leading `**` may not be supported, test exact syntax
3. **Auth state expiration** — script needs post-load validation (navigate to `/new`, check not redirected to `/login`)
4. **Session init sequence** — spec should explicitly show `state load` then `open` flow with `--session`

## Gaps

1. **No error handling / retry** — if prompt 4 fails, entire batch is lost. Need per-prompt try/catch with skip-and-log
2. **No rate limiting delay** — 5 prompts back-to-back may trigger throttling. Add 10-15s configurable delay
3. **Thinking panel extraction under-specified** — no concrete locator for toggle, expand, extract
4. **Copy button locator undefined** — primary completion signal has no concrete DOM strategy
5. **`web_search_used` detection too broad** — "searching" matches non-web-search contexts. Use DOM indicator
6. **No "Continue generating" button handling** — truncated responses captured as complete
7. **JSON assembly strategy unspecified** — how are fragments merged into combined results?

## Risks

1. **DOM instability** — Claude.ai deploys mid-eval could break everything. Run control+treatment back-to-back
2. **Manual skill toggle = human error** — add verification screenshot of settings page after toggle
3. **Time gap between conditions** — changes between control/treatment windows are confounding. Minimize gap
4. **n=5 is small** — frame as qualitative validation, not statistical significance

## Simplification Opportunities

1. **Python report generator may be overkill for v1** — simple `jq` markdown table suffices for 5 prompts
2. **Screenshots optional** — make `--screenshots` flag, not default. Text is the real artifact
3. **Drop `--control`/`--treatment` partial run flags** — just support full run + `--report-only` for v1

## Top 7 Action Items for Implementation

1. Fix `find role textbox fill` syntax and add snapshot+@ref fallback
2. Add post-auth-load validation to detect expired sessions
3. Add per-prompt error handling with skip-and-continue
4. Add inter-prompt delays (10-15s configurable)
5. Specify concrete locators for Copy button, thinking toggle, Continue button
6. Add verification step after manual skill toggle
7. Tighten `web_search_used` detection
