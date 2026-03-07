# Session 7 Handoff: Repo Cleanup, Structure Fix + Browser Testing

## Project Context

We are building Claude Code skills under the **claude-got-skills** GitHub username. This is a GitHub user account (not an org), so that published skills appear as `claude-got-skills/claude-capabilities` on skills.sh — giving more marketplace credibility than `liam-jons/claude-capabilities`. Each skill gets its own repo under this username. Future skills, plugins, commands, and agents will each get separate repos.

The first skill — **Claude Capabilities Awareness** — is functionally complete (v1.4.0), with eval results across Haiku, Sonnet, and Opus confirming strong treatment uplift. The critical remaining work is **browser testing** to determine whether this should ship as a standalone skill or be packaged as a plugin with auto-invoke.

**GitHub repo**: https://github.com/claude-got-skills/claude-capabilities
**Skill status**: v1.3.0 is LIVE on Claude.ai capabilities page. v1.4.0 is committed locally but the repo is in a broken state (see below).
**Local folder**: `claude-got-skills/claude-capabilities/` — this is the mounted development folder.

---

## Session 6 Completed Work

### Track 1: Skill Improvement ✅ DONE

All five steps completed across Sessions 6 and 6-continued:

1. **SKILL.md condensed** — verbose sections replaced with summary + ref pointers (~850 token reduction)
2. **Architecture decision content added** — feature combinations, chunking, cost optimization, integration patterns
3. **References deduplicated** — Memory Tool and Computer Use overlap removed (~300-400 token savings)
4. **15 new eval test cases added** to `capabilities-skill-eval.py` (Agent SDK runner, 25 total tests)
5. **Evals completed** with `eval_runner.py` (Messages API runner, 22 tests × 2 conditions):
   - **Haiku 4.5**: Strong uplift across all positive categories, zero regression on negative tests
   - **Sonnet 4.5**: Comparable uplift, slightly better completeness scores
   - Combined report: `evals/v1.4.0-eval-report.md`

### Track 2: Browser Testing ❌ NOT DONE

This is the critical remaining work and should be the primary focus of Session 7.

### Git Push ⚠️ BLOCKED

The local git repo is in a broken state — files were moved out of git-tracked locations during reorganization.

---

## Structural Issues (Must Fix Before Anything Else)

### Issue 1: Accidental outer .git

```
claude-got-skills/.git                       ← ACCIDENTAL — zero commits, wrong remote
claude-got-skills/claude-capabilities/.git   ← REAL REPO — 2 commits, GitHub remote
```

The outer `claude-got-skills/.git` was created accidentally. It has `git@github.com:claude-got-skills/claude-capabilities.git` as remote but zero commits. `claude-got-skills/` is just a local folder on Liam's machine that holds multiple skill repos — it is NOT itself a repo.

**Fix**: Delete `claude-got-skills/.git/` entirely.

### Issue 2: Published skill files deleted from git working tree

During reorganization, the 8 git-tracked files were moved from the repo root into `skill-published/` and `skill-unpublished/` subfolders. Git sees them all as deleted:

```
D LICENSE
D README.md
D SKILL.md
D references/agent-capabilities.md
D references/api-features.md
D references/claude-code-specifics.md
D references/model-specifics.md
D references/tool-types.md
```

The `skill-published/` and `skill-unpublished/` directories are untracked and currently contain identical copies.

**Context on Liam's intent**: The published/unpublished split was a safety mechanism — test changes in unpublished, then "promote" to published, so the live skill (that users downloaded) doesn't get accidentally broken by in-progress edits. This is a valid concern, but maintaining two folder copies is error-prone and redundant when git provides better tools for this.

**Fix**: Use git branches instead of folder copies:
- **`main` branch** = always the published version (what users get from skills.sh/GitHub)
- **`dev` branch** (or feature branches) = working copy for testing changes
- **Git tags** (v1.3.0, v1.4.0, etc.) = immutable snapshots of each published version
- Workflow: edit on `dev` → run evals → browser test → merge to `main` → tag → push

This gives the same safety (published version is protected on `main`, never directly edited) plus full history, easy rollback, and no file duplication.

**Concrete steps**:
1. Restore tracked files to repo root: `git checkout .` (from inside `claude-capabilities/`)
2. Delete `skill-published/` and `skill-unpublished/` (redundant — git has the content)
3. Move development artifacts into a clean structure (see proposed layout below)
4. Commit the restructure
5. Create `dev` branch for future work

### Issue 3: eval_runner.py default path is wrong

```python
DEFAULT_SKILL_PATH = SCRIPT_DIR.parent / "claude-capabilities" / "SKILL.md"
# Resolves to: claude-capabilities/claude-capabilities/SKILL.md → DOES NOT EXIST
```

This was copied from bid-manager where the path hierarchy was different. The evals only worked because we passed `--skill-path ./SKILL.md` explicitly.

**Fix**: Change to `SCRIPT_DIR.parent / "SKILL.md"` (since evals/ is directly inside the repo root, and SKILL.md is at the repo root).

### Issue 4: .gitignore is minimal

Currently only contains `.env`. Missing: `.DS_Store`, `__pycache__/`, `*.pyc`, and potentially large eval result files.

**Fix**: Expand .gitignore properly.

### Issue 5: Stale git lock files

```
.git/HEAD.lock  (from Feb 11 13:38)
.git/index.lock (from Feb 11 13:21)
```

These block all git operations.

**Fix**: `rm .git/HEAD.lock .git/index.lock`

---

## Proposed Repo Structure

This is for the `claude-capabilities` repo specifically. Other skills/plugins will get their own repos.

```
claude-capabilities/                  ← git repo root
├── .gitignore                        ← comprehensive
├── .env                              ← API keys (gitignored)
├── LICENSE                           ← MIT
├── README.md                         ← installation, eval summary, structure
│
├── SKILL.md                          ← THE publishable skill file
├── references/                       ← publishable reference files
│   ├── agent-capabilities.md
│   ├── api-features.md
│   ├── claude-code-specifics.md
│   ├── model-specifics.md
│   └── tool-types.md
│
├── evals/                            ← evaluation infrastructure
│   ├── eval_runner.py                ← canonical runner (Messages API, LLM judge)
│   ├── capabilities-skill-eval.py    ← supplementary runner (Agent SDK)
│   ├── v1.4.0-eval-report.md         ← combined eval report
│   ├── eval-results-*.json           ← raw results (consider gitignoring)
│   └── eval-report-*.md              ← per-run reports
│
├── docs/                             ← project documentation
│   ├── design/                       ← design docs
│   ├── analysis/                     ← eval analysis
│   ├── planning/                     ← planning docs
│   └── continuation-prompts/         ← session handoffs
│
├── knowledge-base/                   ← source material used to build the skill
│   └── *.md
│
└── monitoring/                       ← monitoring/distill setup
    └── distill-monitoring-setup.md
```

**Key principle**: The publishable skill is at the repo root (`SKILL.md` + `references/`). This is what GitHub shows, what skills.sh indexes, and what users install. Everything else (`evals/`, `docs/`, `knowledge-base/`, `monitoring/`) is development infrastructure that ships with the repo but isn't part of the skill itself.

**Branching model**:
- `main` — always the current published version. Only merge here when ready to publish.
- `dev` — working branch for changes, evals, testing. Safe to break things here.
- Tags: `v1.3.0`, `v1.4.0`, etc. for published releases.

---

## Eval Results Summary (v1.4.0)

### Keyword Scoring — Treatment Accuracy (mean per category)

| Category | Haiku | Sonnet | Uplift Pattern |
|----------|-------|--------|----------------|
| Architecture Decisions | 3.67 | 4.00 | +2.3-2.7 from control |
| Can Claude Do X | 4.33 | 5.00 | +2.3-3.0 from control |
| Extension Awareness | 4.80 | 4.60 | +2.8-3.0 from control |
| Hallucination Detection | 3.25 | 3.25 | +1.5-1.75 from control |
| Implementation Guidance | 5.67 | 6.00 | +3.3 from control |
| Model Selection | 6.00 | 6.00 | +4.0-5.0 from control |
| Negative (No Regression) | 4.67 | 4.67 | ≤0.33 delta (safe) |

**Token overhead**: ~97k tokens (full skill + references as system prompt)

### Eval File Inventory

| File | Role |
|------|------|
| `evals/eval_runner.py` | **Canonical** runner — Messages API, 22 tests, control/treatment, LLM judge + keyword scoring. Requires ANTHROPIC_API_KEY. |
| `evals/capabilities-skill-eval.py` | Supplementary — Agent SDK, 25 tests, keyword-only. Works with sandbox OAuth. Less reliable (truncated responses). |
| `evals/eval-results-20260211-184157.json` | Haiku v1.4.0 raw results |
| `evals/eval-results-20260211-190645.json` | Sonnet v1.4.0 raw results |
| `evals/v1.4.0-eval-report.md` | Combined analysis report |

There are also ~12 earlier eval result files from v1.1.0–v1.3.0 development (Sessions 2-5). These have historical value but could be archived or gitignored to reduce clutter.

---

## Session 7 Priority Order

### 1. Fix repo structure (30 mins)

In order:
1. Remove stale lock files: `rm .git/HEAD.lock .git/index.lock`
2. Delete accidental outer git: `rm -rf ../claude-got-skills/.git`
3. Restore git-tracked files to root: `git checkout .` (restores SKILL.md, references/, LICENSE, README.md)
4. Delete redundant copies: `rm -rf skill-published/ skill-unpublished/`
5. Update `.gitignore` (add .DS_Store, __pycache__/, *.pyc)
6. Fix `eval_runner.py` DEFAULT_SKILL_PATH to `SCRIPT_DIR.parent / "SKILL.md"`
7. Verify eval runner works: quick `--no-judge` test run
8. Commit the restructure
9. Create `dev` branch for future work

### 2. Browser testing (THE critical task — 45 mins)

This was Track 2 from Session 6 and is the most important functional work remaining.

**Prerequisites**: Chrome browser with Claude in Chrome extension connected.

**Control test** (skill disabled):
1. Temporarily remove the skill from Claude.ai capabilities page
2. Open a new conversation
3. Run these 5 natural prompts:
   - "I'm building a document processing pipeline in Python. Each document is about 50 pages. What's the best way to set this up with your API?"
   - "What's the difference between extended thinking and adaptive thinking? When should I use each one?"
   - "I need to pick the right Claude model for a customer support chatbot. It needs to be fast and cheap but still accurate. What do you recommend?"
   - "I've been writing the same code review checklist into every prompt. There must be a better way to do this in Claude Code?"
   - "Can Claude remember what we talked about last week in a different conversation?"
4. Record responses

**Treatment test** (skill enabled):
1. Re-add the skill to Claude.ai capabilities page
2. Open a new conversation
3. Run the same 5 prompts
4. Record and compare

**What to measure**:
- **Trigger reliability**: Does the skill fire on all 5 prompts, or only explicit ones?
- **Response accuracy**: Does treatment contain correct specifics (API headers, model IDs, parameter names)?
- **Response completeness**: Does it mention relevant features the control misses?
- **Hallucination prevention**: Does prompt 5 get a more accurate answer?
- **Naturalness**: Does the skill make responses feel like a knowledge dump, or is the information integrated naturally?

### 3. Skill vs Plugin decision

**If the skill triggers reliably on all 5 prompts** (including implicit ones like prompts 1 and 4):
→ Standalone skill is sufficient. Current on-demand approach works. No packaging needed.

**If it only triggers on explicit capability questions** (prompts 2, 3, 5 but not 1, 4):
→ Needs plugin packaging with `autoInvoke` so the skill loads at session start for ALL conversations
→ This means the repo structure needs to support plugin format (plugin.json, etc.)
→ The ~4,100 token session-start cost becomes a consideration

**If it doesn't trigger at all or triggers inconsistently**:
→ Description needs more implicit trigger phrases
→ May need both: improved description + plugin packaging

### 4. Push to GitHub

After structure is fixed and browser testing is done:
- If staying as skill: commit cleanup + v1.4.0 content, push to main, tag v1.4.0
- If converting to plugin: restructure for plugin format first, then push

### 5. Update README.md

After browser testing, update README with:
- v1.4.0 eval results (Haiku + Sonnet + Opus summary)
- Browser testing results and any trigger analysis findings
- Installation instructions (may change if plugin)

---

## Key Technical Context

### .env for API eval access

The eval runner needs an Anthropic API key. The Cowork sandbox sets `ANTHROPIC_API_KEY` to an empty string by default, so the `.env` loader at the top of `eval_runner.py` explicitly checks for empty strings:

```python
if v and (k not in os.environ or not os.environ[k]):
    os.environ[k] = v
```

The `.env` file lives at the repo root and contains the key. It is gitignored.

### Frontmatter constraints (Claude.ai capabilities page)

These were hard-won through trial and error in Sessions 3-4:

- No YAML block scalars (`>`, `|`)
- No colons in description (use `-` instead)
- "Claude" is reserved in `name` field → use `assistant-capabilities`
- Only allowed top-level keys: `name`, `description`, `license`, `allowed-tools`, `compatibility`, `metadata`
- No nested objects under `metadata`
- Despite `name: assistant-capabilities`, the platform loads it as `claude-capabilities` (uses repo/folder name)

### Two eval runners — use eval_runner.py

1. **`eval_runner.py`** (CANONICAL) — Anthropic Messages API directly. 22 tests, control vs treatment via system prompt injection, LLM-as-judge scoring (accuracy/completeness/actionability 0-3), keyword scoring with synonym groups, deprecated pattern detection. Generates JSON results + markdown reports. Requires `ANTHROPIC_API_KEY` in `.env`.

2. **`capabilities-skill-eval.py`** (SUPPLEMENTARY) — Claude Agent SDK (`claude_agent_sdk.query()`). 25 tests, keyword-only scoring, no judge. Works with Cowork sandbox OAuth tokens but produces truncated responses due to `allowed_tools=[]`.

Always use `eval_runner.py` unless specifically testing Agent SDK compatibility.

### Session compaction warning

Sessions 6 and 6-continued suffered from multiple context compactions that caused:
- Confusion between the two eval runners (re-creating a new one instead of using eval_runner.py)
- Lost context about the .env setup and auth workarounds
- Lost context about the repo structure and file locations

This handoff is written to be self-contained to avoid repeating those issues.

---

## File Inventory (Current State)

### Git-tracked files (currently showing as "deleted" — need `git checkout .`)

```
SKILL.md                     ← v1.4.0 in git, deleted from filesystem
references/                  ← v1.4.0 in git, deleted from filesystem
LICENSE                      ← in git, deleted from filesystem
README.md                    ← in git, deleted from filesystem
```

### Untracked development files

```
.env                         ← API key (gitignored)
.gitignore                   ← minimal, needs expanding
evals/                       ← eval runner + 14 result sets + reports
docs/                        ← design docs, analysis, continuation prompts
knowledge-base/              ← 28 source material files
monitoring/                  ← distill monitoring setup
skill-published/             ← REDUNDANT copy (delete)
skill-unpublished/           ← REDUNDANT copy (delete)
```

### Git state

```
Branch: main
Last commit: 2921d11 "v1.4.0: Condense SKILL.md, add architecture patterns, deduplicate refs"
Previous: 3e16b7f "v1.3.0: Claude capabilities awareness skill"
Remote: https://github.com/claude-got-skills/claude-capabilities.git
Stale locks: .git/HEAD.lock, .git/index.lock (must delete before any git ops)
```
