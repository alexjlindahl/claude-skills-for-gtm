# Clay Org Chart Builder

A Claude skill that builds an interactive org chart of a company's current C-Suite and VP leaders from Clay data.

Ask Claude something like *"org chart for Snowflake"* or *"map the leaders at Datadog"* and you get a single self-contained page with:

- **Org chart view** — a reporting tree with the CEO's reports grouped into function lanes (Revenue, Engineering, Finance, and so on). Drag anyone onto another tile to change who they report to.
- **By-level view** — C-Suite and VP Leadership groups.
- **"New in role" badges** for leaders who started their current title recently.
- **A drawer per person** with their title, location, LinkedIn link, the reason for their reporting line, and buttons to look up their work email and mobile number with Clay.
- **"In the news"** — recent, dated, linked articles that mention people on the chart.
- **Optional branding** — style the page in your own company's colors and fonts.

## Requirements

- **A Clay account with the Clay connector enabled in Claude.** Every person, title and location on the chart comes from Clay search. The skill stops if Clay isn't connected; it never fills gaps from the web or from memory.
- **Python 3** in Claude's environment (available by default in claude.ai and Claude Code).
- **For the email and mobile lookup buttons:** the page must be published as an artifact in claude.ai. The buttons call Clay through the *viewer's own* Clay connector, so the page never holds credentials. If you open the page as a plain HTML file, the chart works but the lookup buttons explain that lookups need claude.ai.

## Things to know

- **Clay credits.** Leader searches can use credits, so the skill confirms the scope with you before it searches. Email and mobile lookups run only when the viewer clicks a button and confirms, and they spend the viewer's credits.
- **Reporting lines are inferred, not sourced.** Clay search doesn't return manager relationships. Lines are inferred from titles and functions (for example, VP Engineering → CTO), drawn dashed, and labeled with their basis and confidence. People with no evidence for a manager go to a **Needs review** tray instead of being guessed. Lines you set by dragging are drawn solid.
- **Scope.** C-Suite and VP level only. Directors, advisors, board-only seats and former employees are filtered out.

## Install

**claude.ai:** download this `clay-org-chart-builder` folder, zip it, and upload the zip as a custom skill in your Claude settings.

**Claude Code:** copy this folder into `~/.claude/skills/` (or your project's `.claude/skills/`).

## Files

| Path | What it is |
|---|---|
| `SKILL.md` | The instructions Claude follows |
| `scripts/orgatlas.py` | Deterministic helper: normalizes and dedupes Clay records, infers reporting lines, renders the page, and extracts data back out of a built page |
| `assets/orgchart_template.html` | The page template the renderer fills in |

## Testing without Clay credits

Write a small `raw.json` fixture and run `normalize` with `--demo`. Every record is labeled Demo and the page shows a Demo banner. See the "Testing without Clay credits" section of `SKILL.md`.
