# Session 4 Continuation Prompt — Claude Capabilities Awareness Skill

## Project Overview

We are building the **Claude Capabilities Awareness Skill** — a skill for skills.sh that gives Claude accurate, post-training knowledge about its own capabilities, API features, extension patterns, and model specifics. The skill dramatically improves Claude's ability to answer questions about what it can do, recommend architectures, and guide users to the right extension patterns.

## Project Location

All files live under:
```
.planning/builds/claude-capabilities-skill/
```

### Skill Files (the publishable artifact)
```
claude-capabilities/
├── SKILL.md                          # 305 lines, v1.2.1, ~4,028 tokens — the always-loaded skill
└── references/
    ├── agent-capabilities.md         # 451 lines, ~2,942 tokens
    ├── api-features.md               # 411 lines, ~2,869 tokens
    ├── claude-code-specifics.md      # 386 lines, ~3,020 tokens
    ├── model-specifics.md            # 312 lines, ~2,719 tokens
    └── tool-types.md                 # 409 lines, ~2,674 tokens
```

### Eval Framework
```
evals/
├── eval_runner.py                    # 851 lines, v1.2.1 — multi-run, LLM judge, negative tests
├── capabilities-skill-eval.py        # Original v1.0 eval runner (legacy, superseded)
├── README.md                         # Eval documentation
├── v121-analysis.md                  # Comprehensive v1.2.1 analysis vs v1.2.0 baseline
├── eval-results-20260210-234658.json # v1.2.1 full results (3-run, with judge)
├── eval-report-20260210-234658.md    # v1.2.1 full report
├── eval-results-20260210-231715.json # v1.2.1 quick validation (1-run, no judge)
├── eval-report-20260210-231715.md    # v1.2.1 quick report
├── eval-results-20260210-222404.json # v1.2.0 results (1-run, keyword only)
├── eval-report-20260210-222404.md    # v1.2.0 report (our primary baseline)
├── eval-results-20260210-221143.json # v1.1.0 results
├── eval-report-20260210-221143.md    # v1.1.0 report
├── eval-results-20260210-220750.json # v1.0.0 results
└── eval-report-20260210-220750.md    # v1.0.0 report
```

### Other Files
```
session-3-analysis.md                 # Session 3 analysis document
```

## Git History

```
53138c2 feat: Phase 0 scaffolding — monorepo, frontend, backend, Docker, CI
fcbb4cb feat(capabilities-skill): v1.2.1 — trim SKILL.md, upgrade eval framework
```

The latest commit (`fcbb4cb`) contains SKILL.md v1.2.1 and the upgraded eval_runner.py, but does NOT include the v1.2.1 eval results (which were generated after the commit). These should be committed at the start of Session 4.

## Version History

| Version | Lines | Tokens | Key Changes |
|---------|-------|--------|-------------|
| v1.0.0  | ~280  | ~3,800 | Initial skill with basic capabilities coverage |
| v1.1.0  | ~320  | ~4,200 | Added extension patterns, model selection guidance |
| v1.2.0  | 344   | ~4,630 | Added 5 reference files, expanded test suite to 15 tests |
| v1.2.1  | 305   | ~4,028 | Trimmed duplications (-13%), upgraded eval framework, fixed find-skills |

## Eval Results Summary (v1.2.1 — 3-run average on Haiku 4.5)

### Keyword Accuracy Lift (Control → Treatment)

| Category | Control | Treatment | Lift |
|----------|---------|-----------|------|
| Architecture Decisions | 1.11 | 2.67 | +141% |
| Can Claude Do X | 1.11 | 5.0 | +350% |
| Implementation Guidance | 2.0 | 5.89 | +195% |
| Model Selection | 1.0 | 6.67 | +567% |
| Extension Awareness | 1.8 | 4.4 | +144% |
| **Overall (positive tests)** | **1.52** | **4.93** | **+224%** |

### Completeness Lift

| Category | Control | Treatment | Lift |
|----------|---------|-----------|------|
| Architecture Decisions | 0.67 | 1.11 | +66% |
| Can Claude Do X | 0.78 | 3.0 | +285% |
| Implementation Guidance | 1.22 | 3.33 | +173% |
| Model Selection | 2.0 | 5.0 | +150% |
| Extension Awareness | 0.8 | 3.33 | +316% |

### Negative Tests (Regression Check)
- Accuracy delta: -0.11 (negligible — skill doesn't degrade unrelated responses)
- Completeness delta: +0.11 (negligible)
- All 3 negative tests maintain >4.0/5.0 scores in both conditions

### Multi-Run Stability
- Accuracy σ/mean ≈ 0.04 (very stable)
- Completeness σ/mean ≈ 0.30 (moderate variance — keyword synonym issue)

### LLM Judge (Known Issue)
- Judge model (Haiku) lacks SKILL.md knowledge, so it cannot properly evaluate capabilities-specific correctness
- Judge systematically scores control higher on positive tests (inversion pattern)
- Judge scores perfectly neutral on negative tests (3.0±0.0), confirming it's unbiased per se
- **Root cause: judge needs SKILL.md as rubric** — this is Priority 1 to fix

## Session 4 Priorities

### Part A: Efficacy Improvements (in priority order)

**1. Fix LLM Judge (High Priority)**
The judge_score() function in eval_runner.py needs to include SKILL.md content in the judge's system prompt so it has a rubric for what constitutes a correct capabilities answer. Currently the judge model has no way to know that treatment responses mentioning specific API headers, model names, or extension patterns are more correct than generic responses.

**2. Improve Architecture Decisions Category (Weakest Performer)**
Category 1 only lifts from 1.11→2.67 on accuracy (+141%), far below other categories. The skill's content focuses on specific capabilities rather than architectural patterns. Consider adding a concise section (10-15 lines) covering: when to use subagents vs tool chaining, when to use MCP vs direct API, batch vs streaming tradeoffs, multi-agent coordination patterns.

**3. Fix Completeness Keyword Synonyms (Reduce Noise)**
Completeness scoring has high variance (σ/mean ≈ 0.30) because the model often mentions concepts without using exact keywords. Expand completeness keyword lists with synonym groups (e.g., "skills.sh" OR "skill registry" OR "skill marketplace") to reduce false negatives.

**4. Add Hallucination Detection Eval Category**
We test for presence of correct info but don't test whether the skill prevents confident-but-wrong answers. Add a category of questions where baseline Claude tends to hallucinate capabilities (e.g., "Can Claude Code access my browser cookies?", "Can Claude remember things between conversations?", "Does Claude have access to the internet by default?").

**5. Run Sonnet Validation**
All evals are on Haiku. Run a single `--model claude-sonnet-4-5-20250929 --runs 1 --no-judge` to validate the lift holds on Sonnet (what most users actually run).

**6. Test with Competing System Prompt Content**
Current eval injects SKILL.md as a clean system prompt. In practice, it loads alongside CLAUDE.md and other context. Test with a realistic system prompt that includes SKILL.md + typical CLAUDE.md content to check for signal dilution.

### Part B: Publication on skills.sh

**Required for publication:**
1. ✅ Valid SKILL.md with `name` and `description` frontmatter (already done)
2. ✅ Skill content and references (already done)
3. ⬜ Add `license` field to SKILL.md frontmatter (e.g., `license: MIT`)
4. ⬜ Create a public GitHub repository with structure:
   ```
   claude-capabilities/
   ├── SKILL.md
   └── references/
       ├── agent-capabilities.md
       ├── api-features.md
       ├── claude-code-specifics.md
       ├── model-specifics.md
       └── tool-types.md
   ```
5. ⬜ Add a README.md to the repo root with skill description and eval results
6. ⬜ Run `npx skills-ref validate ./claude-capabilities` to confirm structure
7. ⬜ Test installation with `npx skills add owner/repo`
8. ⬜ Push to public GitHub — skills.sh automatically indexes it

**Not published (dev artifacts, keep separate):**
- `evals/` directory (eval framework, results, analysis)
- `session-*-analysis.md` files
- This continuation prompt

## Environment Notes

- **API key:** User (Liam) will provide at session start — do NOT hardcode
- **Package install:** `pip install anthropic "httpx[socks]"` (or with `--break-system-packages` depending on environment)
- **Eval run command:** From `evals/` directory:
  ```bash
  ANTHROPIC_API_KEY="..." python3 eval_runner.py --runs 3
  ```
- **Quick validation:** `python3 eval_runner.py --no-judge --runs 1`
- **Sonnet run:** `python3 eval_runner.py --model claude-sonnet-4-5-20250929 --runs 1 --no-judge`
- **Git config:** Sandbox may need inline git config:
  ```bash
  git -c user.name="Liam" -c user.email="liamjons@hotmail.com" commit -m "..."
  ```
- **Git lock files:** If `index.lock` appears, use `mv` not `rm` (mounted filesystem quirk)

## Suggested Session 4 Workflow

1. Commit the v1.2.1 eval results that were generated after the last commit
2. Fix the LLM judge (add SKILL.md rubric to judge system prompt)
3. Expand completeness keyword synonyms
4. Add hallucination detection test category (Category 7)
5. Re-run evals (`--runs 3`) to get v1.2.2 baseline with fixed judge
6. If time permits: improve Architecture Decisions content, run Sonnet validation
7. Prepare publication: add license, create repo structure, write README, validate
8. Commit everything as v1.3.0 (publication-ready)
