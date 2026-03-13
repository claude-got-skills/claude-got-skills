---
allowed-tools:
  - Agent
  - Bash
  - Glob
  - Grep
  - Read
  - Write
description: Run a comprehensive codebase quality review with parallel agents
model: opus
---

# Codebase Review

Run a full quality review of the codebase using parallel analysis agents with
adversarial verification. Adapts agent count to codebase size.

## Process

Follow these waves precisely. Each wave must complete before the next begins.

---

### Wave 1: Reconnaissance

This wave runs in the orchestrator (you). Do NOT spawn agents for this wave.

**1a. Create output directory**

```bash
REVIEW_DATE=$(date +%Y-%m-%d)
mkdir -p .planning/reviews/$REVIEW_DATE
```

Store this path as REVIEW_DIR for all subsequent steps.

**1b. Measure the codebase**

Run these commands to understand the codebase size and shape. Adapt file
extensions to the project (the examples below cover TypeScript/JavaScript/Python
projects — add or remove extensions as appropriate):

```bash
# Detect source directories (skip node_modules, .next, dist, build, vendor, .git)
find . -type d -maxdepth 1 \
  -not -name node_modules -not -name .next -not -name dist \
  -not -name build -not -name vendor -not -name .git \
  -not -name __pycache__ -not -name .planning -not -name '.*' \
  | sort
```

For each source directory found, count files, lines, and estimate tokens:

```bash
# Per-directory measurement
for dir in {detected source directories}; do
  files=$(find "$dir" -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.css" -o -name "*.rs" -o -name "*.go" \) 2>/dev/null | grep -v node_modules | wc -l)
  chars=$(find "$dir" -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.css" -o -name "*.rs" -o -name "*.go" \) 2>/dev/null | grep -v node_modules | xargs cat 2>/dev/null | wc -c)
  tokens=$((chars / 4))
  echo "$dir: $files files, $tokens estimated tokens"
done
```

```bash
# Large files (>500 lines) — these need focused attention
find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.rs" -o -name "*.go" \) \
  -not -path "*/node_modules/*" -not -path "*/.next/*" -not -path "*/dist/*" \
  | xargs wc -l 2>/dev/null | sort -rn | grep -v 'total$' | awk '$1 > 500'
```

**Note:** Avoid complex awk expressions — macOS BSD awk has different syntax
from GNU awk. Prefer `grep` + simple `awk`, or pipe through `while read` loops.

```bash
# Recent churn — files changed in last 30 days (highest bug probability)
git log --since="30 days ago" --name-only --pretty=format: 2>/dev/null \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -30
```

```bash
# Recent bug-fix commits — files with recent fixes have higher probability of related bugs
git log --since="30 days" --oneline --all 2>/dev/null \
  | grep -i "fix\|bug\|patch\|hotfix" | head -20
```

Pass both the churn data and recent bug-fix commits to review agents as context
for prioritising which files to read deeply.

```bash
# Markers indicating known issues
grep -rn "TODO\|FIXME\|HACK\|XXX\|BUG\|WARN" \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" \
  --include="*.py" --include="*.rs" --include="*.go" \
  . 2>/dev/null | grep -v node_modules | wc -l
```

**1c. Run deterministic tools**

Run whichever of these are available in the project. Capture output for review
agents. These run in parallel.

**If any tool fails with permission/sandbox errors**, retry with
`dangerouslyDisableSandbox: true`. ESLint, tsc, and ast-grep need write access
to temp directories which the sandbox may block.

```bash
# ESLint (JS/TS projects)
npx eslint . 2>&1 | tail -100
# or: bun lint 2>&1 | tail -100
```

```bash
# TypeScript type check
npx tsc --noEmit 2>&1 | tail -100
# or: bunx tsc --noEmit 2>&1 | tail -100
```

```bash
# ast-grep (if sgconfig.yml or rules/ directory exists)
if [ -f sgconfig.yml ] || [ -d rules ]; then
  ast-grep scan 2>/dev/null | head -200
elif command -v ast-grep &>/dev/null; then
  # Use the plugin's bundled starter rules if no project rules exist.
  # The plugin ships rules in references/ast-grep-starter-rules/.
  # Copy them to a temp location and run ast-grep with that config.
  PLUGIN_RULES_DIR=$(find ~/.claude/plugins -path "*/codebase-review/references/ast-grep-starter-rules" -type d 2>/dev/null | head -1)
  if [ -n "$PLUGIN_RULES_DIR" ] && [ -d "$PLUGIN_RULES_DIR" ]; then
    TEMP_AST_DIR=$(mktemp -d)
    cp -r "$PLUGIN_RULES_DIR/rules" "$TEMP_AST_DIR/"
    cp "$PLUGIN_RULES_DIR/sgconfig.yml" "$TEMP_AST_DIR/"
    (cd "$TEMP_AST_DIR" && ast-grep scan --config sgconfig.yml "$(pwd -P)" 2>/dev/null | head -200)
    rm -rf "$TEMP_AST_DIR"
  else
    # Fallback: run inline rules for critical patterns
    ast-grep scan --inline-rules "id: empty-catch
language: typescript
rule:
  kind: catch_clause
  has:
    kind: statement_block
    not:
      has:
        kind: expression_statement
    stopBy: end" . 2>/dev/null | head -50
  fi
fi
```

```bash
# Dead code detection (if available)
npx knip --no-progress 2>&1 | tail -50
```

**1d. Check for project review standards**

```bash
# Project-specific check files
ls .claude/checks/*.md 2>/dev/null
```

```bash
# REVIEW.md (Anthropic convention)
if [ -f REVIEW.md ]; then cat REVIEW.md; fi
```

```bash
# CLAUDE.md (for project context)
if [ -f CLAUDE.md ]; then cat CLAUDE.md; fi
```

Read any check files found. These become supplementary context for review agents.

**Extract CLAUDE.md gotchas:** If CLAUDE.md exists and contains a "Gotchas",
"Known Issues", "Pitfalls", or "Footguns" section (case-insensitive), extract
it in full. This will be passed to every review agent as "Known Risk Patterns"
so they can mechanically verify these documented issues are not present in their
scope. This converts probabilistic exploration into deterministic checking.

**1e. Calculate partitions**

Using the measurements from 1b, partition the codebase into N scopes. Rules:

- **Maximum: 300K estimated tokens per agent. This is a hard limit, not a
  guideline.** If any scope exceeds 300K tokens, split it further. A scope
  of 427K tokens will result in the agent reviewing only ~11% of its files.
- Keep directory groups together where possible
- Split directories that exceed 300K tokens on their own (e.g. split
  `components/` into domain vs UI subgroups, split `lib/` into cohesive
  subdomain groups)
- Ensure every file >500 lines is explicitly listed in its agent's scope
  so it gets deep attention
- Agent count guide:
  - <30K lines: 2-3 agents
  - 30-80K lines: 4-5 agents
  - 80-150K lines: 6-7 agents
  - 150K+ lines: 8-12 agents
- Include test directories ONLY if the user has explicitly requested test
  review. By default, focus on production code.

**1f. Write reconnaissance output**

Write two files:

1. `$REVIEW_DIR/partitions.md` — the partition plan showing each agent's scope,
   estimated tokens, large files, and description
2. `$REVIEW_DIR/deterministic-findings.md` — all ESLint, tsc, ast-grep, knip
   output (so review agents can avoid re-flagging)

**IMPORTANT: Wave 1 must fully complete before proceeding.** Ensure ALL
reconnaissance commands have finished successfully (including any retries for
failed deterministic tools). Do NOT launch Wave 2 agents in the same parallel
batch as any Wave 1 Bash commands — a failed Bash command in a parallel batch
will cancel all other tool calls in the same message, including correctly-running
agents.

---

### Wave 2: Parallel Review

Spawn N `codebase-reviewer` agents **plus 1 `pattern-checker` agent** using the
Agent tool. ALL agents MUST be launched in a SINGLE message with
`run_in_background: true` to maximise parallelism.

Each scope-partitioned review agent receives this prompt:

```
You are review agent {N} of {total}.

## Your Scope
{list of directories and/or specific files from the partition plan}

## Large Files Requiring Deep Analysis
{files >500 lines in this scope — read these in full}

## Recent Churn (last 30 days)
{files with recent changes in this scope — higher bug probability}

## Recent Bug-Fix Commits
{output from git log bug-fix grep — files with recent fixes have higher probability of related bugs}

## Known Issues — DO NOT re-flag these
{relevant deterministic findings for files in this scope}

## Known Risk Patterns (from CLAUDE.md gotchas)
{extracted gotchas/pitfalls/footguns section from CLAUDE.md, or "None found" if no CLAUDE.md or no gotchas section.
Agents MUST verify these patterns are NOT present in their scope. Treat each gotcha as a mandatory check.}

## Project Standards
{actual content of .claude/checks/*.md files and REVIEW.md — read these files and paste their full content here,
or "None found" if no project standards exist. Do NOT leave this as a placeholder.}

## Output
Write your findings to: {REVIEW_DIR}/scope-{N}-findings.md
Use the finding format defined in your agent instructions.
```

Use `subagent_type: "codebase-reviewer"` for each scope agent.

**Pattern checker agent:** In addition to the scope-partitioned agents, spawn 1
`pattern-checker` agent with this prompt:

```
## Task: Cross-cutting pattern analysis

Run targeted grep/search patterns across the ENTIRE codebase to find systemic
issues that scope-partitioned agents miss (because each agent only sees a subset
of instances).

## Known Issues — DO NOT re-flag these
{deterministic findings from Wave 1}

## Known Risk Patterns (from CLAUDE.md gotchas)
{extracted gotchas — use these to generate additional search patterns}

## Output
Write your findings to: {REVIEW_DIR}/pattern-checker-findings.md
Use the same finding format as scope review agents.
```

Use `subagent_type: "pattern-checker"` for this agent.

After launching all agents, report progress to the user as each agent completes
rather than waiting silently. Example: "Agent 3/6 complete (lib/ scope — found
8 issues)." This gives the user visibility during what can be a 15-20 minute
wait.

Once ALL agents have completed, verify:

```bash
ls -la $REVIEW_DIR/scope-*-findings.md $REVIEW_DIR/pattern-checker-findings.md 2>/dev/null | wc -l
# Should match the number of scope agents + 1 (pattern checker)
```

Read the header of each findings file to get the counts. If any agent failed
to write output, note it as a gap.

---

### Wave 3: Triage

Spawn 1 `review-synthesizer` agent with this prompt:

```
## Task: Triage review findings

## Input Files
{list all scope-N-findings.md AND pattern-checker-findings.md paths that were successfully written}

## Deterministic Findings
{path to deterministic-findings.md}

## Codebase Context
- Total source files: {N}
- Total source lines: {N}
- Review agents used: {N}
- Scopes: {list of scope descriptions}

## Output
Write deduplicated, ranked findings to: {REVIEW_DIR}/triage-findings.md

## Instructions
1. Read ALL scope findings files
2. Deduplicate — merge findings about the same issue across scopes
3. Rank by severity then confidence
4. Remove any findings that duplicate deterministic tool output
5. Separate findings into two groups:
   - VERIFY: all Critical and High findings (these go to verification).
     If the user requested `--verify-all`, also include Medium findings.
   - ACCEPTED: remaining findings (these go directly to the report)
6. Write the triage output with both groups clearly separated
```

Use `subagent_type: "review-synthesizer"` for this agent.

---

### Wave 4: Verification

Read `$REVIEW_DIR/triage-findings.md` and extract the VERIFY group (Critical
and High findings).

Group the findings by file proximity — findings in the same or nearby files
should go to the same verification agent. Target 3-6 findings per verification
agent.

Spawn M `review-verifier` agents in parallel (all in a SINGLE message with
`run_in_background: true`). Each receives:

```
## Task: Adversarially verify these findings

## Findings to Verify
{3-6 findings from the triage output, with full detail including file paths and evidence}

## Instructions
For EACH finding:
1. Read the actual code cited — not just the snippet, but the full file for surrounding context
2. Read callers and callees — trace how this code is actually used
3. Look for guards, defensive code, or framework guarantees that make the finding moot
4. Check if test coverage exists for this specific path
5. Actively try to DISPROVE the finding — look for reasons it's wrong

For each finding, render a verdict:
- **CONFIRMED** — the issue is real, reachable, and impactful. Strengthen the evidence.
- **DOWNGRADED** — real issue but less severe than claimed. Explain why.
- **DISMISSED** — false positive. Explain what defensive code or guarantee disproves it.

## Output
Write verdicts to: {REVIEW_DIR}/verification-{N}.md
```

Use `subagent_type: "review-verifier"` for each agent.

Wait for all verification agents to complete.

---

### Wave 5: Final Report

Spawn 1 `review-synthesizer` agent (reuse the same agent type) with this prompt:

```
## Task: Produce the final review report

## Verified Findings
{list all verification-N.md paths}

## Accepted Findings (Medium/Low — already accepted, no verification needed)
{path to triage-findings.md — use the ACCEPTED group}

## Deterministic Findings Summary
{path to deterministic-findings.md}

## Codebase Context
- Total source files: {N}
- Total source lines: {N}
- Review agents: {N}
- Verification agents: {M}
- Date: {YYYY-MM-DD}

## Previous Report (for delta analysis)
{If a previous REVIEW-REPORT.md exists in an earlier date directory under .planning/reviews/,
provide its path here. Otherwise "None — first run."}

## Instructions
1. Read all verification verdict files
2. Include only CONFIRMED and DOWNGRADED findings (discard DISMISSED)
3. Combine with ACCEPTED findings from triage
4. Produce the final REVIEW-REPORT.md using the report template in your agent instructions
5. If a previous report was provided, add a "Delta from Previous Review" section
   highlighting NEW findings not present in the previous report and RESOLVED
   findings that appeared before but not in this run
6. Write to: {REVIEW_DIR}/REVIEW-REPORT.md
7. Also write a machine-readable JSON file to: {REVIEW_DIR}/findings.json
   containing all findings as an array of objects with fields: id, title,
   severity, category, confidence, file, lines, issue, impact, suggested_fix,
   verification_verdict (if applicable)
```

---

### Present Results

Read `$REVIEW_DIR/REVIEW-REPORT.md` and present to the user:

1. **Executive summary** — one paragraph on overall codebase health
2. **Finding counts** by severity and category
3. **Critical findings** — show full detail for any verified Critical issues
4. **Top 5 High findings** — brief summary of each
5. **Path to full report** for Medium/Low findings

Do NOT dump the entire report into the conversation. The user can read the
file for full detail.

If zero findings survived verification, congratulate the user — that's a
genuinely healthy codebase.

---

## Optional Flags

Check the user's invocation message for these optional flags:

### `--verify-all`

Send Medium findings (not just Critical/High) to adversarial verification in
Wave 4. Increases token cost by ~30% but improves Medium finding quality.
When active, the triage step puts Medium findings in the VERIFY group instead
of ACCEPTED.

### `--thorough`

Runs a two-pass review for maximum coverage (~80% finding capture vs ~65%
single-pass). Process:

1. Complete Waves 1-5 as normal (Pass 1)
2. Re-partition the codebase with a DIFFERENT strategy:
   - If Pass 1 used directory-based partitioning, Pass 2 uses alphabetical
     file distribution (files sorted by path, dealt round-robin to N agents)
   - Different partition boundaries mean different agents read different files
3. Run Waves 2-3 again with the new partitions (reuse Wave 1 reconnaissance)
4. In the final report (Wave 5), merge Pass 1 and Pass 2 findings:
   - Findings from both passes get a confidence boost (+15)
   - Findings from only one pass keep their original confidence
   - Deduplicate before ranking

This roughly doubles the token cost and review time but significantly increases
the probability of finding issues in files that a single-pass agent might skip.
