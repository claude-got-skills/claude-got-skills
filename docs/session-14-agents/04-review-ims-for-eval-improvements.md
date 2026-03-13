# 4. Review IMS for eval improvements

Now I have a thorough understanding of both the IMS content and the existing eval suite. Let me compile my findings.

---

## IMS Search Results and Eval Question Recommendations

### Summary of IMS Content

The Knowledge Hub contains approximately 2,200+ content items focused on **Phew Design Limited**, a UK-based SME that builds safeguarding, audit, and LMS products for the education, healthcare, and public sector. The IMS is primarily a **bid response library** — Q&A pairs, company profiles, compliance documentation, and case studies used for procurement/tender responses.

The IMS contains **zero content directly about Claude, AI capabilities, ChatGPT, Gemini, Copilot, or competing AI tools**. Searches for "AI", "machine learning", "ChatGPT", "Gemini", "Claude", "generative", and "competitor comparison" all returned either zero results or low-relevance matches.

However, the IMS is valuable precisely because it represents the **real working context of an SME that is adopting AI tools** (the staff attended AI training sessions, per item `08726af7`). The content reveals genuine patterns of how a small UK public-sector software company works — and therefore what kinds of questions their staff would actually ask Claude.

### Useful Findings and Suggested Eval Questions

While no IMS content directly tests Claude capabilities, the IMS reveals **real-world task patterns and confusion points** from the perspective of a non-developer SME user base. Below are eval questions derived from these patterns.

---

#### Finding 1: Bid/Tender Response Automation (IMS-derived pattern)

**Source items:** The entire Q&A library structure — hundreds of bid response Q&A pairs with standard/advanced answers, covering compliance, security, product features, and corporate info.

**Insight:** This company literally spends its time answering repetitive bid questions (GDPR compliance, ISO certifications, system features). This is a prime Claude use case that users often don't realize exists.

**Suggested eval question:**
```
"We respond to 30+ procurement tenders per year. Each one asks similar 
questions — data protection policies, ISO certifications, business 
continuity plans. We have a library of standard answers. Can Claude help 
us speed up bid writing by matching questions to our existing answers 
and drafting responses?"
```

**Why this tests genuine confusion:** Users with existing answer libraries don't realize Claude can process their entire Q&A library via Projects or long context, match new questions semantically, and draft tailored responses. They think of Claude as "chat" not "knowledge base + drafting engine."

**Category:** Conversational Platform Users (Category 9) — maps to the existing SMB task pattern focus.

**Key things the skill should help Claude mention:** Projects for persistent Q&A library, PDF upload for tender documents, 1M context window for processing long documents, skills for repeatable bid-writing workflows.

---

#### Finding 2: Multi-Format Document Processing (IMS pattern: PDF/CSV exports)

**Source items:** `17e9fa69` (Audit Data Export — PDF/CSV), `e99f7249` (Reporting Suite — PDF/CSV export), `ee67e0a2` (Print and PDF controls).

**Insight:** The company works heavily with PDF reports, CSV data exports, and structured data. Users often don't know Claude can both *read* these formats AND *generate* them.

**Suggested eval question:**
```
"I have 15 audit reports in PDF format and a CSV export of all our 
audit responses. I need to cross-reference them to find gaps — audits 
where certain questions weren't answered or where responses conflict 
with our policies. Can Claude handle this kind of analysis across 
multiple files?"
```

**Why this tests genuine confusion:** Users assume Claude can only handle one document at a time, don't know about multi-file upload, and don't realize the 1M context window can hold multiple reports simultaneously. The "do I need to split it up" confusion is real (already partially tested in 9.1 but only for single files).

**Category:** Conversational Platform Users (Category 9) or Can Claude Do X (Category 2).

---

#### Finding 3: Compliance Documentation Generation (IMS pattern: ISO, GDPR, Cyber Essentials)

**Source items:** `8bea7d5d` (ISO certifications), `ef3363b6` (security audit compliance), `5f008806` (GDPR DPIAs), many compliance Q&A pairs.

**Insight:** SMEs spend enormous time maintaining compliance documentation — DPIAs, security policies, incident response plans. Users don't know Claude can help draft, review, and update these.

**Suggested eval question:**
```
"We need to update our Data Protection Impact Assessment for a new 
product feature. The current DPIA is 40 pages. Can Claude review it, 
identify sections that need updating based on the new feature description, 
and draft the updated sections while maintaining the existing format and 
regulatory language?"
```

**Why this tests genuine confusion:** Users in compliance roles think of AI as "creative writing" not "regulatory document review." They don't know about PDF upload, structured output for maintaining formats, or Projects for keeping policy templates persistent.

**Category:** Conversational Platform Users (Category 9).

---

#### Finding 4: "Can Claude Replace Our [Specific Tool]?" (IMS pattern: self-contained platform)

**Source items:** `20bd1a0f` (Phew Audit System), `29886907` (full product portfolio), the general pattern of a company with bespoke tools.

**Insight:** Users from companies with bespoke internal tools wonder if Claude can replace or augment specific tool functions — not "can Claude do X in general" but "can Claude do what [our tool] does."

**Suggested eval question:**
```
"We currently use a dedicated platform for collecting survey responses 
from 50+ organizations, generating action plans, and producing trend 
reports. The license costs us thousands per year. Could we use Claude 
to do the same thing — send out questions, collect responses, and 
generate reports? Or is that not what Claude is for?"
```

**Why this tests genuine confusion:** Users conflate "AI assistant" with "SaaS application." Claude *can* help with data analysis, report generation, and even survey design, but it *cannot* replace a multi-user web application with authentication, persistent storage, and workflow automation. The skill should help Claude give a nuanced answer — "here's what I can help with, here's what you still need a dedicated tool for."

**Category:** Hallucination Detection (Category 7) — tests whether Claude over-promises OR under-promises.

---

#### Finding 5: Team/Enterprise Access and Data Isolation (IMS pattern: multi-department access)

**Source items:** `a8caec7d` (multi-department platform use), access control Q&As, data isolation questions.

**Insight:** SME users worry about team-wide AI adoption: Can multiple people use the same Claude setup? Is our data isolated from other users? Can we control who sees what?

**Suggested eval question:**
```
"I want to roll out Claude across our 10-person team. Some staff handle 
sensitive safeguarding data, others just need it for general admin. 
What are my options for team access? Can I control what data different 
team members can see in Claude?"
```

**Why this tests genuine confusion:** Users don't know about Claude Team plans, enterprise features, or the fact that Projects provide team-level context isolation. They assume "Claude" is a single-user tool with no team management.

**Category:** Conversational Platform Users (Category 9).

---

#### Finding 6: Website Accessibility Compliance (IMS pattern: WCAG, public sector)

**Source items:** `2c33541b` (WCAG 2.2 AA), blog posts about digital accessibility (multiple items from Phase 0b), `0ad3eb98` (responsive design).

**Insight:** Public sector and education organizations must meet WCAG accessibility standards. Users don't know Claude can review HTML/CSS for accessibility issues.

**Suggested eval question:**
```
"We're a public sector organization and our website needs to meet WCAG 
2.2 Level AA standards. Can Claude review our HTML pages and identify 
accessibility issues? What about checking our PDF documents for 
accessibility?"
```

**Why this tests genuine confusion:** Users in the public sector think accessibility auditing requires specialized (expensive) tools. Claude can actually analyze HTML for WCAG issues, review alt text, check color contrast ratios, etc. — but users don't know this is within Claude's capabilities.

**Category:** Can Claude Do X (Category 2).

---

#### Finding 7: Training Content Creation for Regulated Sectors (IMS pattern: LMS, safeguarding training)

**Source items:** `29886907` (LMS product with SCORM, training events), case studies showing training programs.

**Insight:** Organizations in regulated sectors need to create training materials (courses, quizzes, assessments) that meet specific compliance frameworks.

**Suggested eval question:**
```
"I need to create a safeguarding training course for our staff — it 
needs a presentation, quiz questions, and a knowledge assessment at 
the end. Can Claude help create all these materials? Can it generate 
content that could be loaded into our Learning Management System?"
```

**Why this tests genuine confusion:** Users don't realize Claude can generate SCORM-compatible content structures, create quiz questions with answer keys, and produce training materials in various formats. They think of Claude as "writing help" not "curriculum design tool."

**Category:** Can Claude Do X (Category 2) or Extension Awareness (Category 5) — skills could help here.

---

#### Finding 8: Incident Response and Security Documentation (IMS pattern: security breach monitoring)

**Source items:** `8042b457` (security breach monitoring), `b1297d98` (malicious software prevention), various penetration testing Q&As.

**Insight:** Small companies with lean IT teams need to respond to security incidents, write post-incident reports, and update security policies — all tasks Claude can assist with.

**Suggested eval question:**
```
"We had a potential security incident and need to write an incident 
report for our ISO 27001 auditor. I have server logs, email 
notifications, and our incident response procedure document. Can 
Claude help me analyze the logs, determine what happened, and draft 
the formal incident report?"
```

**Why this tests genuine confusion:** Users don't realize Claude can analyze log files, cross-reference with procedures, and draft formal documentation. They also may not know about data privacy implications of sharing security logs.

**Category:** Conversational Platform Users (Category 9) — overlaps with data privacy (9.2).

---

#### Finding 9: ChatGPT/Competitor Feature Confusion (IMS gap: no competitor content)

**Source item:** `86bcac23` (Research Notes) explicitly notes: "Competitor positioning — Phew does not publicly name competitors or provide comparison content."

**Insight:** The IMS's *lack* of competitor comparison content mirrors a real user problem. Users coming from ChatGPT, Gemini, or Copilot carry assumptions about what "AI" can and cannot do that may be wrong for Claude specifically.

**Suggested eval question (a):**
```
"I've been using ChatGPT with its Code Interpreter to analyze Excel 
files. I'm thinking of switching to Claude. Does Claude have the same 
kind of code execution where I can upload a spreadsheet and run Python 
analysis on it?"
```

**Why this tests genuine confusion:** Users transferring from ChatGPT assume either (a) Claude has identical features, or (b) Claude lacks features ChatGPT has. The skill should help Claude explain its code execution sandbox and how it compares — without making competitive claims.

**Suggested eval question (b):**
```
"My colleague says Copilot in Microsoft 365 can access all our company 
documents in SharePoint and answer questions about them. Can Claude do 
the same thing — connect to our company's document storage and search 
across everything?"
```

**Why this tests genuine confusion:** Users compare Claude to embedded enterprise AI (Copilot, Gemini in Workspace). Claude's MCP connectors, Projects, and file upload capabilities are different from but partially overlap with these products. Users need accurate guidance on what's possible and what's not.

**Category:** New category recommended: "Competitor Migration / Feature Comparison" or fold into Can Claude Do X (Category 2).

---

#### Finding 10: Small Team AI Adoption — "Where Do We Start?" (IMS pattern: 10-person company)

**Source item:** `0190b303` (Company Profile — 10 employees), the general SME context.

**Insight:** A 10-person company adopting AI faces different questions than an enterprise or a developer. They need practical, immediate value.

**Suggested eval question:**
```
"We're a 10-person company and our MD wants us all to start using AI. 
We don't have developers. What Claude features would give us the 
quickest wins? We mainly write proposals, manage compliance docs, 
and communicate with clients."
```

**Why this tests genuine confusion:** Non-technical users at small companies need a practical starting guide, not a feature matrix. The skill should help Claude recommend Projects, PDF upload, web search, and document generation as immediate-value features — without overwhelming with API/developer features.

**Category:** Conversational Platform Users (Category 9) — similar to 9.7 (platform choice for non-developer) but broader in scope.

---

### Mapping to Existing Categories

| Suggested Question | Best Category | Existing Test Overlap |
|---|---|---|
| Bid/tender response automation | Cat 9 (Conversational) | Partial overlap with 9.1 (PDF upload) but different use case |
| Multi-file cross-reference analysis | Cat 2 or 9 | Extends 9.1 (single file) to multi-file |
| DPIA/compliance doc review | Cat 9 | New — compliance-specific |
| "Can Claude replace our tool?" | Cat 7 (Hallucination) | New — tests over/under-promising |
| Team access and data isolation | Cat 9 | New — team/enterprise features |
| WCAG accessibility review | Cat 2 | New — specific capability discovery |
| Training content creation | Cat 2 or 5 | New — regulated content creation |
| Security incident analysis | Cat 9 | Related to 9.2 (data privacy) |
| ChatGPT code interpreter comparison | Cat 2 | New — competitor migration |
| Copilot/enterprise AI comparison | Cat 2 | New — competitor migration |
| Small team AI adoption guide | Cat 9 | Extends 9.7 and 9.11 |

### Recommendations

1. **Highest priority additions** (test real, common confusion points):
   - Bid/tender automation (Finding 1) — directly reflects the IMS's own use case
   - ChatGPT feature comparison (Finding 9a) — extremely common user migration path
   - "Can Claude replace our tool?" (Finding 4) — tests nuanced over/under-promising
   - Multi-file analysis (Finding 2) — extends existing 9.1 to a harder scenario

2. **Consider a new category**: "Competitor Migration" for questions where users carry assumptions from ChatGPT, Gemini, or Copilot. Currently no tests cover this.

3. **The IMS confirmed the SMB task pattern thesis**: The entire knowledge base represents exactly the kind of organization (10-person UK public-sector SME) that Category 9 was designed for. The task patterns (bid writing, compliance documentation, training materials, report generation) are well-represented in the IMS content.

---

**Files referenced:**
- `/Users/liamj/Documents/development/claude-got-skills/claude-capabilities/evals/eval_runner.py`
- `/Users/liamj/.claude/projects/-Users-liamj-Documents-development-claude-got-skills/2dcfb391-7b28-4ca1-99a5-f90bc4b7a0f1/tool-results/toolu_01VzzqPjTksJfyUV2EUsQZwc.txt`

---
*Agent: `ae58cc9e79a6f2782` | Session: `2dcfb391-7b28-4ca1-99a5-f90bc4b7a0f1` | Rows: 104*
*2026-03-13T16:07:43.217Z -> 2026-03-13T16:10:52.923Z*