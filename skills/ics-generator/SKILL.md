---
name: ics-generator
description: "Generate Ideal Customer Signals (ICS) analysis for any company — a deep GTM intelligence deliverable that maps observable buying signals to outreach plays. Use this skill whenever the user asks to identify buying signals, ideal customer signals, ICS, purchase intent signals, or trigger-based selling opportunities for a company or product. Also trigger when the user asks for signal-based prospecting frameworks, intent signal mapping, or GTM signal analysis. Even if the user just names a company and says 'run ICS' or 'what are the buying signals for X', use this skill. This skill combines web research, strategic reasoning, and structured output to produce a full ICS playbook."
---

# Ideal Customer Signals (ICS) Generator

## What This Skill Does

Produces a comprehensive Ideal Customer Signals analysis for a target company. The output maps observable external signals (hiring patterns, tech adoption, funding events, org changes, etc.) to buying intent, then connects each signal to actionable GTM plays — messaging angles, campaign ideas, channels, and detection methods.

The analysis is grounded in live web research about the company, not generic frameworks.

## Inputs

The user provides:
- **Company Name** (required) — the company whose ICS to analyze
- **Company URL** (optional) — speeds up research
- **Competitors** (optional) — names of competitors to weave into displacement analysis
- **Output format** (optional) — user chooses: rendered artifact (default), docx, pdf, or markdown

If the user doesn't specify format, default to a rendered HTML artifact. If they ask for a document, use the appropriate skill (docx or pdf).

## Workflow

### Phase 1: Research (always do this first)

Use web search to build a grounded understanding of the company. Run 4-6 searches covering:
- What the company sells, core product lines, and value proposition
- Target market, pricing model, and GTM motion (PLG vs sales-led vs hybrid)
- Recent news, funding, partnerships, product launches (last 6-12 months)
- Key competitors and competitive positioning
- If competitors were provided by the user, research those too

Do NOT skip research or rely solely on training data. The quality of the entire output depends on this phase being thorough and current.

### Phase 2: Analysis (Chain of Thought)

Work through the analysis framework in `references/analysis-framework.md`. Read that file before proceeding.

The framework has 9 sections. Each section builds on the previous ones — don't skip ahead or generate signals disconnected from the company context established in earlier sections.

### Phase 3: Output

**HTML artifact (default):** Create a well-designed, single-page HTML artifact with a table of contents, collapsible sections, and clean typography. Use a professional but not generic design — think consultancy deliverable, not template. Include the full analysis with all 9 sections.

**Document formats:** If the user requests docx or pdf, read the appropriate skill (docx at `/mnt/skills/public/docx/SKILL.md` or pdf at `/mnt/skills/public/pdf/SKILL.md`) and follow its instructions to produce a polished document.

**Markdown:** If the user requests markdown, output directly in chat.

## Quality Standards

The difference between a good ICS analysis and a generic one is specificity. Every signal, messaging angle, and campaign idea should be clearly tied to *this* company's products, personas, and market position — not interchangeable boilerplate that could apply to any B2B SaaS company.

Signals should pass the "swap test": if you could swap in a different company name and the signal still reads the same, it's too generic. Rewrite it with concrete product names, specific persona titles, named technologies, and real competitive dynamics.

The "How to Programmatically Detect" field is especially important — it should describe real, implementable detection logic using actual data sources (job boards, technographic APIs, news feeds, SEC filings, G2, etc.), not vague suggestions like "monitor industry trends."

## Handling Competitors

If the user provides competitor names:
- Research each competitor during Phase 1
- Weave competitive displacement signals throughout the analysis (not just in a separate section)
- In Step 7, provide specific competitor-by-competitor displacement signals with messaging that references real product gaps or switching triggers
- In the outbound templates, include at least one competitor displacement template

If no competitors are provided, still include a general competitive displacement section in Step 7, but note which competitors you're inferring based on research.
