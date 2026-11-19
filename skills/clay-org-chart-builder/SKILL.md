---
name: "clay-org-chart-builder"
description: "Build an interactive C-Suite and VP org chart for any company from Clay, with inferred reporting lines, focus mode, new-in-role badges, recent news, on-click email and mobile lookups, and optional branding. Use for \"org chart for X\", \"map the leaders at X\", \"add more people to the X org chart\"."
---

# Clay Org Chart Builder (OrgAtlas)

Builds an interactive, editable **org chart** of a target account's current C-Suite and VP leaders using Clay, optionally styled in the user's own company brand. The output is a single self-contained page with two views:
- **Org chart:** a reporting tree with the CEO's reports grouped into function lanes. Lines are inferred from titles and functions. People with no evidence for a manager go to a **Needs review** tray instead of being guessed. The user can drag anyone onto someone else's tile to change who they report to.
- **By level:** C-Suite and VP Leadership groups.

Each tile opens a drawer where the user can:
- **Find an email or mobile number:** a button runs the lookup through the viewer's own Clay connector, on click, after a credits confirmation.
- **Edit the tile:** change the manager, buying role or level.

These edits and any contact details found are saved in the viewer's browser.

The chart itself is built **agent-side** through the Clay MCP tools, and its data is embedded in the page. The page's only contact with Clay is the on-click contact lookup, which goes through the viewer's own claude.ai Clay connector. No Clay credentials ever appear in the page, logs or URLs.

## Non-negotiables

- Every fact about **people and the org** comes from Clay. Never fill gaps with web search, memory or guesses. That covers people, titles, emails, departments, locations, employment history and reporting lines.
- **The one web-sourced section is "In the news"** (Step 7a). It lists dated, linked articles that name people already on the chart. It is labeled as public web sources, not Clay, and nothing in it is ever used to change a person, title or line.
- **Reporting lines are inferred, and always labeled that way.** The Clay search tools do not return manager relationships. The normalizer infers them deterministically from titles and functions, and records a plain-language basis and a confidence (`medium`, `low` or `unplaced`) for each one. People with no evidence are `unplaced` and sit in Needs review; they are never parked under the CEO to fill the chart. The page draws inferred lines dashed and lines set by the user solid, and each drawer shows the reason for its line. Never describe an inferred line as Clay data or as confirmed. Never "improve" the inference with web search or memory.
- **Contact lookups only on request.** Do **not** run email, phone or work-history enrichment while building the chart. Emails and mobile numbers are looked up only by the page's "Find email / Find mobile with Clay" buttons, which the viewer clicks and confirms (Clay credits). The lookup runs through the viewer's own Clay connector via the Artifact `mcp` capability, so the page never holds credentials. Keep a build-time email only if the search result already contains one.
- Clay searches can use credits. Search only after the user has confirmed the scope (Step 2). Never re-run a search just to "check".
- Demo data exists only for testing (`--demo`). Never present it as Clay results, and never fall back to it when Clay fails.
- Use only Clay DSL fields and operators that appear in the tool schema you can see this session. Do not invent field names or response properties.

## Workflow

### Step 0 — Set up the helper files and confirm Clay

1. Write the two files from the **Bundled files** section at the end of this skill into a working folder, e.g. `orgatlas/scripts/orgatlas.py` and `orgatlas/assets/orgchart_template.html`. Keep that relative layout, because the script finds the template at `../assets/`.
2. Load the Clay tools if they are deferred: `search-companies`, `search-contacts`, `load-more-search-results`, `get-current-workspace`. Read their full descriptions, especially the Search DSL grammar and the company and people field lists.
3. Call `get-current-workspace` and tell the user which Clay workspace the search will run in, in one line.
   - If the Clay tools are missing or this call fails, stop. Say Clay isn't connected and that they should connect the Clay connector, then try again. Do not continue with other data.

### Step 1 — Ask which company

If the user hasn't already named the target company, ask: "Which company do you want to build the org chart for? A website domain helps me find the right one." Wait for the answer.

### Step 2 — Set expectations and ask about branding

Send this notice, filled in, **together with** the branding question (use AskUserQuestion when available):

> I'll search Clay for current C-Suite and VP leaders at **{Company}**: CEOs and other chiefs, presidents, EVPs, SVPs and VPs. Former employees, directors, advisors, consultants and board-only seats are left out. **The first build includes up to 30 people**, and you can add more later. This search uses Clay credits and runs as soon as you answer below.

Question: **"Do you want the org chart styled with your company's brand?"**
- **Yes, use my brand**: next, ask for their company website (e.g. `acme.com`)
- **No, use the default style**: a clean off-white workspace with purple accents

Answering this question is the user's go-ahead for the search. If they answer "Yes", ask for the website and do Step 3 before or alongside Step 4.

### Step 3 — Extract the user's brand (only if they said yes)

The brand is the **user's** company (the seller), not the target account.

1. Open the homepage in the built-in browser and read computed styles with its JavaScript tool. Start with the CTA buttons' `backgroundColor`, the `fontFamily` of `h1` and `body`, the hex custom properties on `:root`, Google Fonts `<link>`s and the header logo. WebFetch turns pages into markdown and drops the CSS, so use it only when no browser is available. Pull out only what is actually present:
   - **Brand name:** `og:site_name`, the `<title>` prefix, or the logo `alt` text
   - **Primary color:** in order of preference:
     1. The main CTA button background, when it has a hue
     2. Custom properties that name the brand (`--primary`, `--brand`, a named brand swatch such as `--blueberry-400`)
     3. `<meta name="theme-color">`

     Ignore black, white, greys, and pale tints that would need heavy darkening (many sites use black CTAs). If the brand's only colored CTA is a pale tint, pick its strongest saturated brand swatch instead.
   - **Fonts:** the `family=` values of Google Fonts `<link>` tags, or else the first family in the heading and body `font-family` declarations
   - **Logo:** only when the logo is inline `<svg>` markup, or a small SVG file the fetch tool returns as text. Base64-encode it as `data:image/svg+xml;base64,...`. Otherwise leave `logoDataUri` empty; the page then shows the brand name as a wordmark. Do not use outside tools to download binary images.
2. Fonts must come from Google Fonts. Published pages may load stylesheets only from `fonts.googleapis.com`. If the brand font isn't there (e.g. a commercial font), pick the closest Google font, such as a geometric sans for a geometric sans, and say you substituted it.
3. Write `theme.json`:
   ```json
   {"brandName":"Acme","primary":"#0A66FF","fontHeading":"Space Grotesk","fontBody":"Inter",
    "googleFontsUrl":"https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@600;700&family=Inter:wght@400;500;600;700&display=swap",
    "logoDataUri":null,"source":"acme.com"}
   ```
   The renderer adjusts the primary color automatically so text meets WCAG AA contrast in both light and dark mode. Pale brand colors get darkened on light backgrounds. Mention this if the adjustment is visible.
4. If the site can't be fetched, or yields no usable color, tell the user in one line and continue with the default style. Don't block the build on branding.

### Step 4 — Resolve the target company

1. **If a domain is known**, run `search-companies` with a `select from companies where domain = "<domain>"` query.
2. **If only a name is known**, use the company-name field that the `search-companies` schema lists. Use an exact match first, then a looser operator from the schema if nothing matches. Use a small limit, such as `limit 5`.
3. **If one company matches clearly**, use it.
4. **If there are several**, show up to 5 choices with name, domain, industry and location (whatever Clay returned). Ask the user to pick one with AskUserQuestion. Never auto-pick between plausible matches.
5. **If there are none**, say so and ask for the company's domain. Do not guess a domain.
   - A domain can return the parent plus an acquired unit that shares it, e.g. "Manifold (acq. by Snyk)". When the extra record is clearly labeled as acquired or a subsidiary, use the parent and mention it in one line. Otherwise ask.
6. Save `company.json` with only values Clay returned: `{"name":..., "domain":..., "clayCompanyId":..., "industry":..., "location":...}`. From here on, pass the resolved company's **LinkedIn company URL** as the identifier, never the bare name. A shared domain would pull in people from the subsidiary. Use the domain only if Clay returned no LinkedIn URL.

### Step 5 — Search current leaders

Call `search-contacts` with `companyIdentifiers: ["<resolved LinkedIn company URL>"]`. This filters by current employer. Start from this query, then adjust names and operators so they match the people schema exactly:

```
select from people where experiences.any(is_current = true and (
  job_title contains ("chief", "president", "ceo", "cfo", "coo", "cto", "cio", "cmo", "cro", "cpo", "chro", "clo")
  or job_title is_similar_to ("Vice President", "Executive Vice President", "Senior Vice President")
)) limit 30
```

- If the schema offers a seniority field or enum for current experiences, add it as an OR branch for VP and C-level seniority. Also record each person's seniority value so that "Head of" titles can qualify as VP.
- **First build:** `limit 30`. Each page returns at most 20 contacts whatever the limit, so a first build normally needs one `load-more-search-results` call with the returned `taskId` (while `hasMore: true`) to reach 30 records. Never fetch more than 50 records total on a first build.
- Keep every search page's `taskId`, and record which one each contact came from. The page's email lookup uses it (Step 6), and so does Step 8.
- **Mobile lookup setup:** call `list_subroutines` (read-only, no credits) and look for a workspace function that finds a mobile or phone number.
  - **Pick:** prefer one named like "Mobile Phone Number", then "Get Phone Number". It must have required inputs the page can fill.
  - **Available fields:** the person's `linkedinUrl`, `name`, `companyName`, `companyDomain` and `email` (the sourced email, if any); the company's `companyLinkedinUrl`; or `none` for an empty value.
  - **If one fits,** write `enrich.json`: `{"server":"Clay","email":true,"phone":{"id":"<function id>","name":"<function name>","inputs":{"<input name>":"<field>"}}}`.
  - **If none fits,** write `{"server":"Clay","email":true}`, so the page shows only the email lookup. Tell the user in one line.
  - **Never** use a function whose description says it writes to a CRM, sequence or outbound tool.
- **Errors:** report Clay's message in plain words, without any raw payload or credential. Use these meanings:
  - Auth or connection problem: reconnect the Clay connector.
  - Quota or credits exhausted: the workspace is out of search credits.
  - Rate limited: wait a minute and ask the user again.
  - Anything else: a general Clay error.

  Never retry in a loop, and never substitute other data.

### Step 6 — Normalize, classify and dedupe

1. Convert each returned contact into this intermediate shape, **copying only fields Clay actually returned**. Leave out anything that's missing.

   Clay's current search response maps like this (confirm against the live response):
   - `profile_id` → `clayId`
   - `name` → `name`
   - `latest_experience_title` → `title`
   - `url` → `linkedinUrl`
   - `location_name` → `location`
   - `latest_experience_start_date` (`YYYY-MM`) → `roleStartDate`. This is when the person started their **current title**, so it can be a promotion rather than a new hire. It powers the page's "New in role" badges. Keep it only when it is present; never estimate it.
   - `entityId` → `clayEntityId`
   - the search page's `taskId` → `clayTaskId`

   Also add the resolved company's LinkedIn URL to `company.json` as `linkedinUrl`.

   **Ignore `audienceFields` entirely.** That block is the workspace's own CRM record. Its titles, emails, phones and "No Longer at Company" flags often describe a previous employer, so it is not search data about this person.
   ```json
   {"clayId":"...", "name":"...", "title":"<current title at this company>", "linkedinUrl":"...",
    "location":"...", "roleStartDate":"YYYY-MM", "email":"<only if already in the result>", "seniority":"<only if returned>",
    "isCurrent": true}
   ```
   - **Current employment:** set `isCurrent: true` when `latest_experience_company` matches the resolved company. The search already filtered on a current experience. Set `false` when it names a different company, e.g. a Snyk search that returns the CEO of FossID. Such a person can't be shown with a verified Snyk title.
   - Use the title of the current experience **at the resolved company**, not the headline.
   - Save the list as `raw.json`.
2. Run the normalizer:
   `python3 scripts/orgatlas.py normalize --raw raw.json --company company.json --out people.json --limit 30`

   It does all of the following:
   - **Excludes:**
     - Former employees
     - Directors and below
     - Advisors, consultants, assistants and board-only seats
     - "Head of" titles without VP seniority from Clay
     - Customer-facing or deputy C-titles such as "Field CTO" or "Forward Deployed CTO", which aren't the company's C-Suite seat
   - **Classifies level:** "Chief … Officer", C-level acronyms, and President are C-Suite. EVP, SVP, VP and Global VP are VP.
   - **Maps function** from the title: Executive, Revenue, Sales, Marketing, Finance, Product, Engineering, People, Legal, Operations, Strategy, Customer or Other
   - **Dedupes** by Clay ID, then normalized LinkedIn URL, then normalized name plus company domain
   - **Ranks** CEO, then President, other chiefs, EVP, SVP and VP
   - **Caps** the list at the limit
   - **Infers reporting lines** over the whole list (`reportsTo: {id, basis, confidence}`). It is re-run on every merge.
     - **Top of the chart:** the CEO, or else the President.
     - **Sub-executives:** a Chief Accounting Officer reports to the CFO, and a CISO to the CIO, or else the CTO.
     - **Other C-Suite:** report to the CEO.
     - **VPs:** report to the closest more-senior VP in the same function (EVP, then SVP, then VP). Failing that, they report to that function's C-level head, e.g. VP Engineering to the CTO.
     - **No same-function leader:** an EVP goes under the top executive with `low` confidence. Anyone else gets `reportsTo: {id: null, confidence: "unplaced"}` and is shown in the page's Needs review tray.

     Every line points to someone more senior, so the chart can't contain cycles.

   It prints a report of counts, exclusion reasons and inferred-line confidence. Keep it for the summary.

### Step 7a — Recent news about these leaders

After normalizing, and before rendering, find recent news that names the charted leaders. This uses web search only, not Clay, so it costs no Clay credits.

1. **Search** with WebSearch, in the same turn where possible:
   - the company plus "news" and the current and previous two months (e.g. "Snyk announces August 2026");
   - the top executive by name, and each other C-Suite leader by name with the company;
   - any VP whose name is distinctive enough to search reliably.
   Use `extended` mode for these, since the results must be recent.
2. **Verify every item** with WebFetch before using it: read its publication date and which charted leaders it actually names or quotes. Keep an item only when:
   - its date is within the last **3 months** of today (drop undated items);
   - it names at least one person on the chart, as the same person (same company and role);
   - it is a news article, a company press release or newsroom post, or a byline the leader wrote. Skip people-search and profile sites (RocketReach, SignalHire, theorg, LinkedIn profiles) and job posts.
   When a press release is syndicated, keep the original and list the copies under `alsoAt`.
3. **Write `news.json`**. Summaries are one or two sentences in your own words; never copy article text beyond a few words, and never put words in a leader's mouth.
   ```json
   {"checkedAt":"YYYY-MM-DD","windowMonths":3,"items":[
     {"date":"YYYY-MM-DD","title":"<headline>","url":"https://...","source":"<outlet>",
      "kind":"News | Press release | Byline","summary":"<your words>",
      "people":["<people.json id>"],"alsoAt":[{"source":"<outlet>","url":"https://..."}]}]}
   ```
   `people` uses the ids from `people.json`. The renderer drops items with no date, no https link, no title, or no one on the chart.
4. If nothing qualifies, still write `news.json` with an empty `items` list. The page then says no news named these leaders.
5. Tell the user in the Step 7 reply how many items were found and who they mention.

### Step 7 — Build and publish the page

1. Run the renderer:
   `python3 scripts/orgatlas.py render --people people.json --company company.json [--theme theme.json] --enrich enrich.json --news news.json --out "<Company> Org Chart.html"`
   - The default output is an Artifact page body with no doctype, html, head or body tags.
   - If the page will be delivered as a plain file instead of published with the Artifact tool, add `--full-document`.
2. If the Artifact tool is available, publish the file with:
   - `icon: "chart"` and a one-sentence description
   - `capabilities: {"mcp": {"servers": [{"server": "Clay", "tools": ["add-contact-data-points", "get-task-context", "search-contacts-by-name", "run_subroutine_direct"]}]}}`

   Leave out `run_subroutine_direct` when `enrich.json` has no phone function. On a republish, omit `capabilities` unless the tool list changes.

   If the Artifact tool isn't available, save the file to the outputs folder and send it to the user, or write it into a connected folder. The contact buttons then explain that lookups need the page open in claude.ai.
3. Reply briefly, covering:
   - How many leaders were found (C-Suite and VP counts)
   - What was excluded, from the report
   - The workspace searched
   - How many people are in Needs review because Clay's data had no same-function leader for them (`inferredLines.unplaced` in the report), and how many lines are low confidence
   - That dragging a person onto someone's tile changes their manager, and that manager, buying-role and level changes are saved locally in their browser and can be reset
   - That reporting lines are inferred from titles, not supplied by Clay
   - That the side panel's "Find email / Find mobile with Clay" buttons use Clay credits from the viewer's workspace, and that results are saved in the viewer's browser
   - That they can say "add 20 more to the {Company} org chart" to expand it

### Step 8 — Adding more people later

When the user asks for more people for a company that has already been charted:

1. Recover the existing dataset:
   - **Same session:** reuse `people.json`, `company.json` and `theme.json`.
   - **Otherwise:** find the published page (Artifact `list`, then `read`), save its HTML, and run `python3 scripts/orgatlas.py extract --html <file> --outdir .`. The page embeds all three files.
2. Get the next batch:
   - If a `taskId` from this session is still available, call `load-more-search-results` (20 per page).
   - Otherwise, re-run the Step 5 query against the same resolved LinkedIn company URL with a limit of **current total + requested amount**. Deduping removes the people already on the chart.
   - Default to 20 more if the user doesn't give a number. Tell them it uses Clay credits.
3. Merge: `python3 scripts/orgatlas.py normalize --raw raw_more.json --company company.json --existing people.json --out people.json --limit <N more>`
4. Re-run Step 7a so the news covers the new people and the window moves forward, then re-render with the same theme, and republish to the **same artifact URL** so the link stays the same. Local edits carry over because they're stored per company in the viewer's browser. That includes manual manager changes: a newly added person can't overwrite a line the user set.
5. Report how many new people were added, and say plainly if Clay had no more qualifying leaders.

## Page behavior (built into the template)

- **Header:** the user's logo or wordmark, "OrgAtlas", and a Clay status pill. In Demo mode the pill turns amber and a banner appears.
- **Account header:** company, domain, industry, location, source, retrieval time, and leader, C-Suite and VP counts
- **Coverage strip** (account header):
  - **Counts:** leaders (C-Suite / VP), Champions, Exec buyers, Influencers, Blockers, Not yet classified, contacts found, Need a manager and Lines you set.
  - **Buying-role counts:** clicking one highlights those people and dims the rest.
  - **Need a manager:** jumps to the tray.
- **Toolbar:**
  - **Controls:** the view switch, search, and a Function dropdown. "Clear filters" appears while any filter is on.
  - **··· menu:** "Expand all teams" and "Reset local edits". The reset is two-step, with no browser dialogs.
- **View switch:** "Org chart" (the default) or "By level". The choice is remembered per company.
- **Org chart** (desktop):
  - **Layout:** the top executive is centered at the top. Their direct reports sit in function lanes: Engineering & Security, Product, Go-to-market, People, Finance & Legal, Operations & Strategy, and Other. Each lane has a colored header and a count, with teams indented beneath.
  - **Lines:** dashed for inferred, dotted amber for low confidence, solid in the primary color for lines the user set.
  - **Tiles:** kept slim, with initials, name and title. A colored left edge and pill appear only once a buying role is set. Email and mobile icons show when found, and an amber dot marks a local edit.
  - **Collapse:** each manager's tile has a collapse toggle ("▾ 3" / "▸ 5 hidden"). The state is remembered.
  - **Zoom and scroll:** zoom out, in, and Fit, remembered per viewer. Edge fades and a "Scroll →" hint appear when the chart is wider than the view. The chart opens centered on the top executive.
  - **Moving people:** drag a tile onto another tile to change who that person reports to. Drops onto yourself or anyone in your own team are refused, so no cycles.
  - **Drop zones:** during a drag, a "top of the chart" zone appears. The tray is also a drop target, for sending someone back to review.
  - **Filters:** search, function and role filters dim non-matching tiles instead of hiding them, so the tree stays intact.
  - **Focus mode:** clicking a tile opens its drawer and focuses the chart on that person. Their chain up to the top executive and their whole team stay bright, with those lines drawn in the primary color; everyone else dims. A "Focused on …" chip in the chart bar clears it, as do Esc, closing the drawer, or clicking empty chart space. On desktop the drawer is non-modal (no scrim), the page makes room for it, and clicking another tile switches focus. On phones the drawer stays modal.
  - **Motion:** on first load, lines draw in and tiles rise in one after another, top executive first, then lane by lane. When someone changes manager (drag, tray, or dropdown), their tile glides from its old spot to its new one and pulses once where it lands. Newly shown tiles fade in. All motion is turned off for viewers who ask for reduced motion.
- **New in role:** when any record has `roleStartDate`, the toolbar shows a "New in role" switch: Off, 3 mo, 6 mo (the default) or 12 mo, remembered per company. Leaders whose current title started within the window get an orange "New · N mo" badge and an orange ring, the coverage strip shows the count, and the drawer's Clay details show "In role since <Mon YYYY>". Months are counted from today. The footer says a recent promotion also counts. With no start dates the switch is hidden.
- **In the news** (bottom of the page, both views): dated cards with the outlet, a type tag (News, Press release, Byline), the headline linking to the article, a short summary, "Also at" links for syndicated copies, and chips for the leaders it names. Chips open that person's drawer. A row of person filters with counts sits above the cards. The header says "from public web sources, not Clay" and when it was checked. Tiles of people in the news show a small newspaper icon, and their drawer gets an "In the news" section with a link to their filtered list.
- **Needs review tray:** a sticky right-hand panel, which moves below the chart under 1100px. It lists everyone with no manager yet, grouped by the same function lanes, together with any teams under them, and explains why no line was guessed.
- **Phone width:** the org chart becomes an indented outline, and the coverage strip scrolls sideways. The drawer's "Reports to" dropdown replaces dragging.
- **By level:** "Level 1 — C-Suite" and "Level 2 — VP Leadership". Drag a card between sections to correct its level.
- **Drawer:** collapsible sections, in this order:
  - **Contact:** its badge shows how many contacts are found.
  - **Reporting:**
    - **Reports to:** a dropdown with Needs review, Top of the chart, or a person. It excludes the person and anyone in their team.
    - **Relationship note:** inferred, with its confidence and reason; Set by you, plus the original inference; or Needs review.
    - **Direct reports:** clickable, the first 6 with "Show all".
  - **Classification:** buying role and level.
  - **Clay details:** collapsed by default. Company, function, location, LinkedIn, verification and retrieved time, with "Not returned by Clay" where a field is missing.

  The drawer closes with Esc, its close button, or (on phones) the scrim. Focus is trapped only on phones, where it is modal.
- **Contact details** (top of the drawer):
  - **Showing a value:** an email already in the Clay search result, or one found earlier, shows with Copy.
  - **Looking one up:** otherwise the row offers "Find email with Clay" and, when a phone function is configured, "Find mobile with Clay". Each asks for an inline credits confirmation before it runs.
  - **How the email lookup runs:** `add-contact-data-points` (`Email`) on the build-time `clayTaskId` and `clayEntityId`. If Clay rejects that task, it falls back to `search-contacts-by-name` with the company's LinkedIn URL, then the same call.
  - **How the mobile lookup runs:** `run_subroutine_direct` with the configured function and input mapping, passing the sourced email when one exists.
  - **Waiting:** the page checks `get-task-context` every 4 seconds for up to 150 seconds and shows the real elapsed time.
  - **Timeouts:** a timed-out lookup is remembered, so "Check again" only re-reads the result and never starts a second paid lookup.
  - **Errors:** each connector error code gets its own message, e.g. add or reconnect the Clay connector, access turned off for this page, policy blocks it, or Clay didn't answer. When the connection isn't available, the page says lookups need claude.ai with the Clay connector.
  - **Saving:** results are saved per company in the viewer's browser, stamped "Sourced from Clay <time>", and shown as email and mobile icons on tiles. "Reset local edits" does not clear them, because they cost credits.
- **Local edits** (manager, buying role, level, collapsed teams, zoom, New-in-role window) are saved in `localStorage`, namespaced as `orgatlas:v1:<clayCompanyId | domain | name>`. They never overwrite the embedded Clay data or the embedded inference.
- **Empty state:** "No matching leaders", with nothing filled in
- **Layout:** responsive down to phone width, with light and dark themes and visible focus styles

## Out of scope

Reporting lines sourced from anywhere other than title inference or the user's own edits, director-level searches, bulk or automatic email or phone enrichment (lookups are per person, on click only), CRM sync, podcast or PR research beyond the Step 7a news section, conversation hooks, AI-written emails, seller assignment, CSV export, and multiple themes. If asked for one of these, say it isn't part of this skill and offer it as a separate follow-up.

## Testing without Clay credits

To check the page itself, write a small fixture `raw.json` and run `normalize` with `--demo`. Every record is then labeled Demo and the page shows the Demo banner. Never publish a demo build as if it were a real account map.

## Bundled files

Write these exactly as shown at the start of each run (Step 0).

### `scripts/orgatlas.py`

````python
#!/usr/bin/env python3
"""OrgAtlas helper: classify Clay leader records, merge/dedupe, and render the org chart.

Subcommands
  normalize  raw leader records (from Clay search output) -> people.json + exclusion report
  render     people.json + company.json [+ theme.json] -> orgchart.html
  extract    pull the embedded dataset back out of a previously built orgchart.html

Everything here is deterministic. It never invents people, titles, emails, or reporting lines.
"""
import argparse, colorsys, datetime as dt, json, re, sys
from pathlib import Path

HERE = Path(__file__).resolve().parent
TEMPLATE = HERE.parent / "assets" / "orgchart_template.html"

# ---------------------------------------------------------------- classification
ALWAYS_EXCLUDE = re.compile(
    r"\b(advis[eo]rs?|consultants?|contractors?|freelancer?|interns?|assistant|associate vice president|"
    r"avp|former|retired|emerit(us|a)|chief of staff|office of the|to the|board members?|board of directors|"
    r"non[- ]executive|trustees?|observer|fellow|student)\b", re.I)
DIRECTOR = re.compile(r"\bdirectors?\b", re.I)
FULL_CHIEF = re.compile(r"\bchief\b[\w\s,&/'-]*?\bofficer\b", re.I)
ACRONYM = re.compile(r"\b(ceo|cfo|coo|cto|cio|cmo|cro|cpo|chro|clo|ciso|cso|cdo|cao|cco|cbo)\b", re.I)
PRESIDENT = re.compile(r"(?<!vice )(?<!vice-)\bpresident\b", re.I)
VP = re.compile(
    r"\b(executive vice[- ]president|senior vice[- ]president|group vice[- ]president|vice[- ]president|"
    r"evp|svp|gvp|sr\.? vp|global vp|vp)\b", re.I)
HEAD_OF = re.compile(r"\bhead of\b", re.I)
# Customer-facing or deputy "CTO"-style titles that are not the company's C-Suite seat
NOT_CSUITE = re.compile(r"\b(field|forward[- ]deployed|deputy|regional|divisional|associate|assistant)\b", re.I)

FUNCTIONS = [  # checked in order; first match wins (CEO checked before this list)
    ("Revenue", r"\b(cro|chief revenue|revenue|go[- ]to[- ]market|gtm|commercial)\b"),
    ("Sales", r"\b(sales|business development|partnerships?|alliances|channels?|account management)\b"),
    ("Marketing", r"\b(cmo|marketing|brand|communications|comms|public relations|growth|demand)\b"),
    ("Finance", r"\b(cfo|finance|financial|accounting|controller|treasury|tax|investor relations|fp&a)\b"),
    ("Product", r"\b(product|products|design|ux)\b"),
    ("Engineering", r"\b(cto|cio|ciso|engineering|technology|technical|infrastructure|platform|data|security|it|"
                    r"information|software|ai|machine learning|research|r&d)\b"),
    ("People", r"\b(chro|people|hr|human resources|talent|recruiting|culture|workplace)\b"),
    ("Legal", r"\b(clo|legal|counsel|compliance|privacy|regulatory)\b"),
    ("Operations", r"\b(coo|operations|operating|supply chain|procurement|facilities|logistics|manufacturing)\b"),
    ("Strategy", r"\b(strategy|strategic|corporate development|corp dev|transformation)\b"),
    ("Customer", r"\b(customers?|clients?|success|support|experience|services)\b"),
]
FUNCTIONS = [(n, re.compile(p, re.I)) for n, p in FUNCTIONS]
EXEC = re.compile(r"\b(ceo|chief executive)\b", re.I)
EXEC_FALLBACK = re.compile(r"(?<!vice )(?<!vice-)\b(president|chair(man|woman|person)?|founder|co-?founder)\b", re.I)


def classify_level(title, seniority=None):
    """Return ("C-Suite"|"VP", None) or (None, exclusion_reason)."""
    t = title or ""
    if not t.strip():
        return None, "No current title"
    if ALWAYS_EXCLUDE.search(t):
        return None, "Advisor / board / assistant / consultant / former"
    has_vp = bool(VP.search(t))
    c_level = FULL_CHIEF.search(t) or (ACRONYM.search(t) and not has_vp) or (PRESIDENT.search(t) and not has_vp)
    if c_level and not NOT_CSUITE.search(t):
        return "C-Suite", None
    if c_level:
        return None, "Field / deputy C-title (not the C-Suite seat)"
    if has_vp:
        return "VP", None
    if HEAD_OF.search(t) and seniority and VP.search(str(seniority)):
        return "VP", None
    if DIRECTOR.search(t):
        return None, "Director or below"
    if HEAD_OF.search(t):
        return None, "Head of (no VP seniority from Clay)"
    return None, "Not C-Suite or VP"


def classify_function(title):
    t = title or ""
    if EXEC.search(t):
        return "Executive"
    for name, rx in FUNCTIONS:
        if rx.search(t):
            return name
    if EXEC_FALLBACK.search(t):
        return "Executive"
    return "Other"


def rank(p):
    t = p["title"].lower()
    if p["level"] == "C-Suite":
        if EXEC.search(t):
            return 0
        if PRESIDENT.search(t) and not FULL_CHIEF.search(t):
            return 1
        return 2
    if re.search(r"executive vice|\bevp\b", t):
        return 3
    if re.search(r"senior vice|\bsvp\b|sr\.? vp", t):
        return 4
    return 5


# ---------------------------------------------------------------- reporting-line inference
# Clay's search does not return manager relationships. These are INFERRED from titles and
# functions only, always labeled as inferred, and never presented as Clay data.
SUB_EXEC = [  # (title pattern, ordered parent patterns among the C-Suite)
    (re.compile(r"accounting|\bcao\b", re.I), [r"financial officer|\bcfo\b"]),
    (re.compile(r"security|\bciso\b|\bcso\b", re.I), [r"information officer|\bcio\b", r"technology officer|\bcto\b"]),
]
HEAD_PREF = {  # which C-level title leads a function when several share it
    "Engineering": [r"technology|\bcto\b", r"information officer|\bcio\b"],
    "Finance": [r"financial|\bcfo\b"],
    "People": [r"people|human|\bchro\b"],
}


def tier(p):
    return rank(p)  # 0 CEO, 1 President, 2 other chiefs, 3 EVP, 4 SVP, 5 VP


def infer_reporting(people):
    """Set p['reportsTo'] = {'id','basis','confidence'} or None for the top of the chart."""
    ordered = sorted(people, key=rank)
    root = next((p for p in ordered if p["level"] == "C-Suite" and EXEC.search(p["title"])), None) \
        or next((p for p in ordered if p["level"] == "C-Suite" and PRESIDENT.search(p["title"])), None)
    csuite = [p for p in ordered if p["level"] == "C-Suite" and p is not root]

    def is_sub(p):
        return any(rx.search(p["title"]) for rx, _ in SUB_EXEC)

    def head_for(fn):
        cands = [c for c in csuite if c["function"] == fn and not is_sub(c)]
        for pat in HEAD_PREF.get(fn, []):
            hit = next((c for c in cands if re.search(pat, c["title"], re.I)), None)
            if hit:
                return hit
        return cands[0] if cands else None

    def unplaced(why):  # no evidence for a manager: the page shows these in "Needs review", not under the CEO
        return {"id": None, "basis": why, "confidence": "unplaced"}

    for p in ordered:
        if p is root:
            p["reportsTo"] = None
            continue
        rel = None
        if p["level"] == "C-Suite":
            for rx, parents in SUB_EXEC:
                if rx.search(p["title"]):
                    for pat in parents:
                        mgr = next((c for c in csuite if c is not p and re.search(pat, c["title"], re.I)), None)
                        if mgr:
                            rel = {"id": mgr["id"], "basis": f"{p['title']} usually reports to the {mgr['title']}",
                                   "confidence": "medium"}
                            break
                    break
            if not rel and root:
                rel = {"id": root["id"], "basis": f"C-Suite executives usually report to the {root['title']}",
                       "confidence": "medium"}
            if not rel:
                rel = unplaced("No CEO or President in this dataset")
        else:
            if p["function"] not in ("Other", "Executive"):
                senior = [v for v in ordered if v["level"] == "VP" and v is not p
                          and v["function"] == p["function"] and tier(v) < tier(p)]
                if senior:
                    mgr = max(senior, key=tier)  # closest more-senior tier
                    rel = {"id": mgr["id"], "basis": f"Same function ({p['function']}), more senior VP title",
                           "confidence": "medium"}
                else:
                    head = head_for(p["function"])
                    if head:
                        rel = {"id": head["id"], "basis": f"Same function ({p['function']}) as the {head['title']}",
                               "confidence": "medium"}
            if not rel and root and tier(p) == 3:
                rel = {"id": root["id"], "basis": f"EVPs usually report to the {root['title']}, but no same-function leader was found",
                       "confidence": "low"}
            if not rel:
                rel = unplaced("No same-function leader was found in this dataset")
        p["reportsTo"] = rel
    return people


# ---------------------------------------------------------------- identity / dedupe
def norm_linkedin(url):
    if not url:
        return None
    u = url.strip().lower()
    u = re.sub(r"^https?://", "", u)
    u = re.sub(r"^([a-z]{2,3}\.)?www\.|^[a-z]{2}\.", "", u)
    u = u.split("?")[0].split("#")[0].rstrip("/")
    return u or None


def norm_name(n):
    return re.sub(r"[^a-z0-9]+", " ", (n or "").lower()).strip()


def keys_for(p, domain):
    ks = []
    if p.get("clayId"):
        ks.append("clay:" + str(p["clayId"]))
    li = norm_linkedin(p.get("linkedinUrl"))
    if li:
        ks.append("li:" + li)
    if p.get("name"):
        ks.append("nm:" + norm_name(p["name"]) + "@" + (domain or "").lower())
    return ks


def primary_id(p, domain):
    return keys_for(p, domain)[0]


# ---------------------------------------------------------------- normalize
CARRY = ("clayId", "location", "linkedinUrl", "email", "clayTaskId", "clayEntityId", "roleStartDate")
START_RE = re.compile(r"^(\d{4})-(\d{2})(?:-\d{2})?$")


def clean_start(v):
    """Clay's latest_experience_start_date, kept only as YYYY-MM."""
    m = START_RE.match(str(v or "").strip())
    return f"{m.group(1)}-{m.group(2)}" if m and 1 <= int(m.group(2)) <= 12 else None


def normalize(args):
    raw = json.loads(Path(args.raw).read_text())
    if isinstance(raw, dict):
        raw = raw.get("people") or raw.get("results") or []
    company = json.loads(Path(args.company).read_text())
    domain = company.get("domain")
    now = dt.datetime.now(dt.timezone.utc).replace(microsecond=0).isoformat()

    existing = []
    if args.existing and Path(args.existing).exists():
        existing = json.loads(Path(args.existing).read_text())
    seen = {}
    for p in existing:
        for k in keys_for(p, domain):
            seen[k] = p

    excluded, dupes, fresh = {}, 0, []
    for r in raw:
        name = (r.get("name") or "").strip()
        title = (r.get("title") or "").strip()
        if not name:
            excluded["Missing name"] = excluded.get("Missing name", 0) + 1
            continue
        if r.get("isCurrent") is False:
            excluded["Former employee"] = excluded.get("Former employee", 0) + 1
            continue
        level, why = classify_level(title, r.get("seniority"))
        if not level:
            excluded[why] = excluded.get(why, 0) + 1
            continue
        if r.get("roleStartDate"):
            r = dict(r, roleStartDate=clean_start(r["roleStartDate"]))
        person = {
            "name": name,
            "title": title,
            "level": level,
            "function": classify_function(title),
            "companyName": company.get("name"),
            "currentEmploymentVerified": r.get("isCurrent") is True,
            "buyingRole": "Unclassified",
            "source": "Demo" if args.demo else "Clay",
            "retrievedAt": r.get("retrievedAt") or now,
        }
        for f in CARRY:
            if r.get(f):
                person[f] = r[f]
        if domain:
            person["companyDomain"] = domain
        ks = keys_for(person, domain)
        hit = next((seen[k] for k in ks if k in seen), None)
        if hit:
            dupes += 1
            for f in CARRY:  # fill gaps only
                if f in person and not hit.get(f):
                    hit[f] = person[f]
            for k in keys_for(hit, domain):
                seen[k] = hit
            continue
        person["id"] = ks[0]
        for k in ks:
            seen[k] = person
        fresh.append(person)

    fresh.sort(key=rank)
    cap = args.limit if args.limit is not None else len(fresh)
    added, overflow = fresh[:cap], fresh[cap:]
    people = existing + added
    people.sort(key=rank)
    infer_reporting(people)
    Path(args.out).write_text(json.dumps(people, indent=2))
    report = {
        "rawRecords": len(raw),
        "added": len(added),
        "total": len(people),
        "cSuite": sum(p["level"] == "C-Suite" for p in people),
        "vp": sum(p["level"] == "VP" for p in people),
        "duplicatesMerged": dupes,
        "qualifiedButOverCap": len(overflow),
        "inferredLines": {c: sum(1 for p in people if (p.get("reportsTo") or {}).get("confidence") == c)
                          for c in ("medium", "low", "unplaced")},
        "excluded": excluded,
    }
    print(json.dumps(report, indent=2))


# ---------------------------------------------------------------- theme
def _hex(h):
    h = h.lstrip("#")
    if len(h) == 3:
        h = "".join(c * 2 for c in h)
    return tuple(int(h[i:i + 2], 16) / 255 for i in (0, 2, 4))


def _to_hex(rgb):
    return "#" + "".join(f"{max(0, min(255, round(c * 255))):02x}" for c in rgb)


def _lum(rgb):
    def ch(c):
        return c / 12.92 if c <= 0.03928 else ((c + 0.055) / 1.055) ** 2.4
    r, g, b = (ch(c) for c in rgb)
    return 0.2126 * r + 0.7152 * g + 0.0722 * b


def contrast(a, b):
    la, lb = sorted((_lum(_hex(a)), _lum(_hex(b))), reverse=True)
    return (la + 0.05) / (lb + 0.05)


def mix(a, b, t):
    A, B = _hex(a), _hex(b)
    return _to_hex(tuple(x + (y - x) * t for x, y in zip(A, B)))


def usable_primary(primary, surface, target=4.5):
    """Darken (light surface) or lighten (dark surface) until text/UI contrast is met."""
    toward = "#000000" if _lum(_hex(surface)) > 0.5 else "#ffffff"
    c, t = primary, 0.0
    while contrast(c, surface) < target and t < 1:
        t += 0.05
        c = mix(primary, toward, t)
    return c


DEFAULT_THEME = {
    "brandName": None,
    "primary": "#5B3FD9",
    "fontHeading": "Inter",
    "fontBody": "Inter",
    "googleFontsUrl": "https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap",
    "logoDataUri": None,
    "source": "OrgAtlas default",
}


def theme_css(theme):
    light_bg, dark_bg = "#FAF9F7", "#15141A"
    p = theme["primary"]
    p_light = usable_primary(p, "#FFFFFF")
    p_dark = usable_primary(p, "#1F1E26")
    on_p = "#FFFFFF" if contrast("#FFFFFF", p_light) >= contrast("#111111", p_light) else "#111111"
    on_pd = "#FFFFFF" if contrast("#FFFFFF", p_dark) >= contrast("#111111", p_dark) else "#111111"
    fh = theme.get("fontHeading") or "Inter"
    fb = theme.get("fontBody") or "Inter"
    stack = "system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif"
    light = f"""
  --bg:{light_bg}; --surface:#FFFFFF; --surface-2:#F3F2EF; --line:#E4E2DD; --conn:#BDB8AF; --text:#1B1A18; --muted:#5F5C57;
  --primary:{p_light}; --on-primary:{on_p}; --primary-soft:{mix(p_light, '#FFFFFF', 0.88)}; --primary-line:{mix(p_light, '#FFFFFF', 0.6)};
  --ok:#1E7A46; --ok-soft:#E3F4EA; --warn:#8A5A00; --warn-soft:#FFF4DB; --danger:#B42318; --danger-soft:#FDECEA;
  --shadow:0 1px 2px rgba(20,20,30,.06), 0 4px 14px rgba(20,20,30,.06);
  --r-champion:#1E7A46; --r-buyer:#6D3FD1; --r-influencer:#0B6BB8; --r-blocker:#B42318;
  --g1:#0B6BB8; --g2:#6D3FD1; --g3:#C2410C; --g4:#0F7B6C; --g5:#9A6700; --g6:#B4235F; --g7:#6B6F76;
  --new:#C2410C; --new-2:#EA580C; --on-new:#FFFFFF; --new-soft:#FFEDD5;"""
    dark = f"""
  --bg:{dark_bg}; --surface:#1F1E26; --surface-2:#282733; --line:#373645; --conn:#5A586A; --text:#F2F1F6; --muted:#A9A7B6;
  --primary:{p_dark}; --on-primary:{on_pd}; --primary-soft:{mix(p_dark, '#1F1E26', 0.8)}; --primary-line:{mix(p_dark, '#1F1E26', 0.45)};
  --ok:#6FD39A; --ok-soft:#1C3527; --warn:#F2C46B; --warn-soft:#3A2F16; --danger:#FF8A80; --danger-soft:#3D1F1D;
  --shadow:0 1px 2px rgba(0,0,0,.4), 0 6px 18px rgba(0,0,0,.35); color-scheme:dark;
  --r-champion:#6FD39A; --r-buyer:#B69CFF; --r-influencer:#7CC2FF; --r-blocker:#FF8A80;
  --g1:#7CC2FF; --g2:#B69CFF; --g3:#FFA36B; --g4:#5FD4C0; --g5:#F2C46B; --g6:#FF8FC0; --g7:#A9A7B6;
  --new:#FB923C; --new-2:#FDBA74; --on-new:#1C1006; --new-soft:#3B2211;"""
    fonts = f"--font-heading:'{fh}', {stack}; --font-body:'{fb}', {stack};"
    return (f":root{{{light} {fonts}}}\n"
            f"@media (prefers-color-scheme: dark){{:root:not([data-theme=\"light\"]){{{dark}}}}}\n"
            f":root[data-theme=\"dark\"]{{{dark}}}")


# ---------------------------------------------------------------- news
DATE_RE = re.compile(r"^\d{4}-\d{2}-\d{2}$")


def clean_news(raw, people):
    """Keep only well-formed items that link to a real article and name someone on the chart."""
    ids = {p["id"] for p in people}
    items, dropped = [], 0
    for it in raw.get("items", []):
        url = str(it.get("url") or "")
        who = [i for i in it.get("people", []) if i in ids]
        if not (DATE_RE.match(str(it.get("date") or "")) and url.startswith("https://") and it.get("title") and who):
            dropped += 1
            continue
        also = [a for a in it.get("alsoAt", []) if str(a.get("url", "")).startswith("https://") and a.get("source")]
        items.append({"date": it["date"], "title": str(it["title"]), "url": url, "source": str(it.get("source") or ""),
                      "kind": str(it.get("kind") or "News"), "summary": str(it.get("summary") or ""),
                      "people": who, "alsoAt": also})
    items.sort(key=lambda x: x["date"], reverse=True)
    return {"checkedAt": raw.get("checkedAt"), "windowMonths": raw.get("windowMonths", 3), "items": items}, dropped


# ---------------------------------------------------------------- render
def render(args):
    people = json.loads(Path(args.people).read_text())
    company = json.loads(Path(args.company).read_text())
    theme = dict(DEFAULT_THEME)
    if args.theme and Path(args.theme).exists():
        theme.update({k: v for k, v in json.loads(Path(args.theme).read_text()).items() if v})
    retrieved = max((p["retrievedAt"] for p in people), default=None)
    payload = {
        "company": company,
        "people": people,
        "retrievedAt": retrieved,
        "theme": {k: theme.get(k) for k in ("brandName", "primary", "fontHeading", "fontBody",
                                             "googleFontsUrl", "logoDataUri", "source")},
        "builtAt": dt.datetime.now(dt.timezone.utc).replace(microsecond=0).isoformat(),
        # on-demand contact sourcing through the VIEWER's Clay connector (never credentials)
        "enrich": {"server": "Clay", "email": True},
    }
    if args.enrich and Path(args.enrich).exists():
        payload["enrich"].update(json.loads(Path(args.enrich).read_text()))
    if args.news and Path(args.news).exists():
        payload["news"], dropped = clean_news(json.loads(Path(args.news).read_text()), people)
        if dropped:
            print(f"Dropped {dropped} news item(s): missing date/https link/title, or no one on the chart named")
    data = json.dumps(payload).replace("</", "<\\/")
    font_link = ""
    if theme.get("googleFontsUrl", "").startswith("https://fonts.googleapis.com/"):
        font_link = (f'<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>'
                     f'<link rel="stylesheet" href="{theme["googleFontsUrl"]}">')
    title = f"{company.get('name', 'Account')} Org Chart"
    html = (TEMPLATE.read_text()
            .replace("__TITLE__", title.replace("<", ""))
            .replace("__FONT_LINK__", font_link)
            .replace("/*__THEME_CSS__*/", theme_css(theme))
            .replace("__DATA_JSON__", data))
    if args.full_document:
        html = ('<!doctype html><html lang="en"><head><meta charset="utf-8">'
                '<meta name="viewport" content="width=device-width, initial-scale=1">'
                + html.replace("<!--BODY-->", "</head><body>", 1) + "</body></html>")
    else:
        html = html.replace("<!--BODY-->", "", 1)
    Path(args.out).write_text(html)
    print(f"Wrote {args.out} ({len(people)} people)")


def extract(args):
    html = Path(args.html).read_text()
    m = re.search(r'<script type="application/json" id="orgatlas-data">(.*?)</script>', html, re.S)
    if not m:
        sys.exit("No OrgAtlas dataset found in that file.")
    payload = json.loads(m.group(1).replace("<\\/", "</"))
    out = Path(args.outdir)
    out.mkdir(parents=True, exist_ok=True)
    (out / "people.json").write_text(json.dumps(payload["people"], indent=2))
    (out / "company.json").write_text(json.dumps(payload["company"], indent=2))
    (out / "theme.json").write_text(json.dumps(payload.get("theme") or {}, indent=2))
    print(f"Extracted {len(payload['people'])} people for {payload['company'].get('name')} into {out}")


def main():
    ap = argparse.ArgumentParser()
    sub = ap.add_subparsers(dest="cmd", required=True)
    n = sub.add_parser("normalize")
    n.add_argument("--raw", required=True)
    n.add_argument("--company", required=True)
    n.add_argument("--out", required=True)
    n.add_argument("--existing")
    n.add_argument("--limit", type=int)
    n.add_argument("--demo", action="store_true")
    r = sub.add_parser("render")
    r.add_argument("--people", required=True)
    r.add_argument("--company", required=True)
    r.add_argument("--theme")
    r.add_argument("--enrich")
    r.add_argument("--news")
    r.add_argument("--out", required=True)
    r.add_argument("--full-document", action="store_true")
    e = sub.add_parser("extract")
    e.add_argument("--html", required=True)
    e.add_argument("--outdir", required=True)
    a = ap.parse_args()
    {"normalize": normalize, "render": render, "extract": extract}[a.cmd](a)


if __name__ == "__main__":
    main()
````

### `assets/orgchart_template.html`

````html
<title>__TITLE__</title>
__FONT_LINK__
<style>
/*__THEME_CSS__*/
*{box-sizing:border-box}
html,body{margin:0}
body{background:var(--bg);color:var(--text);font:14px/1.45 var(--font-body);-webkit-font-smoothing:antialiased}
h1,h2,h3{font-family:var(--font-heading);margin:0;letter-spacing:-.01em;text-wrap:balance}
button,input,select{font:inherit;color:inherit}
:focus-visible{outline:3px solid var(--primary);outline-offset:2px;border-radius:8px}
.wrap{max-width:1440px;margin:0 auto;padding:0 24px}
.topbar{border-bottom:1px solid var(--line);background:var(--surface)}
.topbar .wrap{display:flex;align-items:center;gap:12px;min-height:56px}
.brand{display:flex;align-items:center;gap:10px;font-weight:700}
.brand img{max-height:28px;max-width:140px;display:block}
.product{font-size:12px;color:var(--muted);font-weight:600;letter-spacing:.06em;text-transform:uppercase;padding-left:10px;border-left:1px solid var(--line)}
.spacer{flex:1}
.conn{display:inline-flex;align-items:center;gap:6px;font-size:12px;font-weight:600;padding:4px 10px;border-radius:999px;background:var(--ok-soft);color:var(--ok)}
.conn::before{content:"";width:7px;height:7px;border-radius:50%;background:currentColor}
.conn.demo{background:var(--warn-soft);color:var(--warn)}
.banner{background:var(--warn-soft);color:var(--warn);font-weight:600;padding:10px 0;border-bottom:1px solid var(--line)}
/* ---------- account header + coverage strip ---------- */
.account{padding:22px 0 10px}
.account h1{font-size:28px}
.meta{display:flex;flex-wrap:wrap;gap:4px 16px;color:var(--muted);font-size:13px;margin-top:4px}
.meta b{color:var(--text);font-weight:600}
.cover{display:flex;flex-wrap:wrap;gap:8px;margin-top:14px}
.cv{display:flex;align-items:center;gap:8px;border:1px solid var(--line);background:var(--surface);border-radius:12px;padding:7px 12px;text-align:left}
button.cv{cursor:pointer}
button.cv:hover{border-color:var(--primary-line)}
button.cv[aria-pressed="true"]{border-color:var(--primary);box-shadow:0 0 0 2px var(--primary-soft)}
.cv .n{font:700 18px/1 var(--font-heading);font-variant-numeric:tabular-nums}
.cv .l{font-size:11.5px;color:var(--muted);line-height:1.2}
.cv i{width:9px;height:9px;border-radius:3px;flex:none}
.cv.warn{border-color:var(--warn);background:var(--warn-soft)}
.cv.warn .n{color:var(--warn)}
.cv-sep{width:1px;background:var(--line);margin:4px 2px}
/* ---------- toolbar ---------- */
.toolbar{position:sticky;top:env(safe-area-inset-top,0px);z-index:6;background:var(--bg);padding:10px 0;display:flex;flex-wrap:wrap;gap:10px;align-items:center;border-bottom:1px solid var(--line)}
.seg{display:inline-flex;border:1px solid var(--line);border-radius:10px;overflow:hidden;background:var(--surface)}
.seg button{border:0;background:transparent;padding:8px 12px;font-weight:600;cursor:pointer;color:var(--muted)}
.seg button[aria-pressed="true"]{background:var(--primary);color:var(--on-primary)}
.search{flex:1 1 200px;max-width:300px;position:relative}
.search input,.sel{width:100%;padding:8px 12px;border:1px solid var(--line);border-radius:10px;background:var(--surface)}
.search input{padding-left:34px}
.search svg{position:absolute;left:11px;top:50%;transform:translateY(-50%);color:var(--muted)}
.sel{width:auto;min-width:170px;cursor:pointer}
.btn{border:1px solid var(--line);background:var(--surface);border-radius:10px;padding:8px 12px;font-weight:600;cursor:pointer}
.btn:hover{border-color:var(--primary-line)}
.btn:disabled{opacity:.5;cursor:default}
.btn.primary{background:var(--primary);border-color:var(--primary);color:var(--on-primary)}
.btn.sm{padding:6px 10px;font-size:12.5px}
.menu{position:relative}
.menu summary{list-style:none;cursor:pointer;border:1px solid var(--line);background:var(--surface);border-radius:10px;padding:7px 12px;font-weight:700;letter-spacing:.1em}
.menu summary::-webkit-details-marker{display:none}
.menu[open] summary{border-color:var(--primary-line)}
.menu .pop{position:absolute;right:0;top:calc(100% + 6px);background:var(--surface);border:1px solid var(--line);border-radius:12px;box-shadow:var(--shadow);padding:6px;min-width:230px;z-index:8}
.menu .pop button{display:block;width:100%;text-align:left;border:0;background:transparent;padding:9px 10px;border-radius:8px;cursor:pointer;font-weight:600}
.menu .pop button:hover{background:var(--surface-2)}
.menu .pop button:disabled{opacity:.5;cursor:default}
.menu .pop .danger{color:var(--danger)}
.menu .pop .armed{background:var(--danger);color:#fff}
.menu .pop .hint{padding:2px 10px 6px}
/* ---------- chart bar ---------- */
.chartbar{display:flex;flex-wrap:wrap;align-items:center;gap:6px 14px;margin:14px 0 6px;color:var(--muted);font-size:12px}
.chartbar strong{color:var(--text);font:700 13px var(--font-heading);text-transform:uppercase;letter-spacing:.08em}
.legend{display:inline-flex;align-items:center;gap:6px}
.legend i{display:inline-block;width:24px;height:0;border-top:2px dashed var(--conn)}
.legend i.user{border-top:2px solid var(--primary)}
.zoom{display:inline-flex;align-items:center;border:1px solid var(--line);border-radius:10px;background:var(--surface);overflow:hidden}
.zoom button{border:0;background:transparent;padding:5px 10px;cursor:pointer;font-weight:700;color:var(--text)}
.zoom button:hover{background:var(--surface-2)}
.zoom span{min-width:46px;text-align:center;font-size:12px;font-variant-numeric:tabular-nums;color:var(--muted)}
/* ---------- layout: chart + needs-review tray ---------- */
.stage{display:grid;grid-template-columns:minmax(0,1fr) 300px;gap:18px;align-items:start}
.stage.no-tray{grid-template-columns:minmax(0,1fr)}
.topzone{position:absolute;left:8px;right:8px;top:8px;z-index:4;visibility:hidden;border:2px dashed var(--primary-line);border-radius:12px;padding:10px;text-align:center;color:var(--primary);font-weight:600;background:var(--primary-soft)}
body.dragging .topzone{visibility:visible}
.topzone.drop-ok,.tray.drop-ok{border-color:var(--primary);border-style:solid}
.scroller{position:relative;min-width:0}
.scroller::before,.scroller::after{content:"";position:absolute;top:0;bottom:0;width:36px;pointer-events:none;opacity:0;transition:opacity .15s;z-index:2}
.scroller::before{left:0;background:linear-gradient(90deg,var(--bg),transparent)}
.scroller::after{right:0;background:linear-gradient(270deg,var(--bg),transparent)}
.scroller.can-left::before,.scroller.can-right::after{opacity:1}
.scrollhint{position:absolute;right:10px;top:6px;z-index:3;font-size:11.5px;font-weight:700;color:var(--muted);background:var(--surface);border:1px solid var(--line);border-radius:999px;padding:2px 9px;display:none}
.scroller.can-right .scrollhint{display:block}
.orgwrap{overflow:auto;padding:12px 4px 28px;max-height:calc(100vh - 150px);min-height:420px}
.forest{display:flex;flex-direction:column;gap:36px;width:max-content;min-width:100%}
.tree{display:flex;flex-direction:column;align-items:center}
.node-root{position:relative;width:250px}
.node-root.has-kids::after{content:"";position:absolute;left:50%;bottom:-16px;height:16px;border-left:2px solid var(--conn)}
.bus{display:flex;gap:18px;align-items:flex-start;padding-top:32px}
.group{position:relative;width:250px;flex:none}
.group::before{content:"";position:absolute;top:-16px;left:-10px;right:-10px;border-top:2px solid var(--conn)}
.group:first-child::before{left:50%}
.group:last-child::before{right:50%}
.group:only-child::before{display:none}
.group::after{content:"";position:absolute;top:-16px;left:50%;height:16px;border-left:2px solid var(--conn)}
.ghead{position:relative;display:flex;align-items:center;gap:8px;padding:6px 10px;border-radius:10px;background:var(--surface);border:1px solid var(--line);border-top:3px solid var(--gc);font-size:12px;font-weight:700}
.ghead .gn{color:var(--muted);font-weight:600;margin-left:auto;font-variant-numeric:tabular-nums}
.sub{list-style:none;margin:0;padding:0 0 0 14px}
.sub>li{position:relative;padding:8px 0 0 16px}
.sub>li::before{content:"";position:absolute;left:0;top:0;bottom:0;border-left:2px solid var(--conn)}
.sub>li:last-child::before{bottom:auto;height:30px}
.sub>li::after{content:"";position:absolute;left:0;top:29px;width:15px;border-top:2px dashed var(--conn)}
.sub>li.e-user::after{border-top:2px solid var(--primary)}
.sub>li.e-low::after{border-top:2px dotted var(--warn)}
.gsub{padding-left:10px}
/* ---------- tiles ---------- */
.node{position:relative}
.ocard{display:flex;gap:9px;align-items:center;text-align:left;width:100%;border:1px solid var(--line);border-left:4px solid var(--rc, var(--line));background:var(--surface);border-radius:11px;padding:8px 10px;cursor:grab;box-shadow:var(--shadow);transition:border-color .12s, opacity .12s, transform .12s}
.ocard:hover{border-color:var(--primary-line);border-left-color:var(--rc, var(--primary-line))}
.ocard.root{padding:11px 12px}
.ocard.selected{outline:2px solid var(--primary);outline-offset:1px}
.ocard.dim{opacity:.3}
.ocard.hit{box-shadow:0 0 0 3px var(--primary-soft), var(--shadow)}
.ocard.drop-ok{border-color:var(--primary);background:var(--primary-soft);transform:scale(1.02)}
.ocard.drop-no{cursor:no-drop}
.ocard.dragging{opacity:.4}
.role-Champion{--rc:var(--r-champion)} .role-Executive-Buyer{--rc:var(--r-buyer)} .role-Influencer{--rc:var(--r-influencer)} .role-Blocker{--rc:var(--r-blocker)}
.av{flex:none;width:32px;height:32px;border-radius:9px;display:grid;place-items:center;font:700 12px var(--font-heading);background:var(--primary-soft);color:var(--primary)}
.is-c .av{background:var(--primary);color:var(--on-primary)}
.ocard.root .av{width:40px;height:40px;font-size:14px}
.cbody{min-width:0;flex:1}
.nm{display:flex;align-items:center;gap:6px;font-weight:700;min-width:0}
.nm b{white-space:nowrap;overflow:hidden;text-overflow:ellipsis;font-weight:700}
.tt{display:-webkit-box;color:var(--muted);font-size:12px;line-height:1.3;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden;margin-top:1px}
.ic{display:inline-flex;gap:3px;flex:none;color:var(--ok)}
.ic svg{width:13px;height:13px}
.edot{width:7px;height:7px;border-radius:50%;background:var(--warn);flex:none}
.rpill{display:inline-block;margin-top:4px;font-size:10.5px;font-weight:700;padding:1px 7px;border-radius:6px;color:var(--rc);background:var(--surface-2)}
.fold{position:absolute;right:8px;bottom:-9px;z-index:1;border:1px solid var(--line);background:var(--surface);border-radius:999px;padding:0 8px;font-size:11px;font-weight:700;color:var(--muted);cursor:pointer;line-height:17px}
.fold:hover{border-color:var(--primary-line);color:var(--text)}
/* ---------- needs-review tray ---------- */
.tray{position:sticky;top:64px;border:2px dashed var(--warn);border-radius:16px;background:var(--surface);padding:14px;max-height:calc(100vh - 90px);overflow:auto}
.tray h2{font-size:15px;display:flex;align-items:center;gap:8px}
.tray h2 .n{background:var(--warn-soft);color:var(--warn);border-radius:999px;padding:0 8px;font-size:12px}
.tray p{margin:6px 0 10px;color:var(--muted);font-size:12.5px}
.tgroup{margin:12px 0 0}
.tgroup .ghead{margin-bottom:6px}
.tlist{display:flex;flex-direction:column;gap:8px}
.tlist .sub{padding-left:10px}
.tray .empty{border:1px dashed var(--line);border-radius:10px;padding:14px;color:var(--muted);font-size:12.5px;text-align:center}
/* ---------- level view ---------- */
.tier{margin:14px 0 26px;background:var(--surface);border:1px solid var(--line);border-radius:16px;padding:16px;transition:border-color .15s, background .15s}
.tier.drop{border-color:var(--primary);background:var(--primary-soft)}
.tier-head{display:flex;align-items:baseline;gap:10px;margin-bottom:12px}
.tier-head h2{font-size:16px}
.tier-head .count{font-size:12px;color:var(--muted);font-weight:600}
.tier-head .lvl{font-size:11px;font-weight:700;letter-spacing:.08em;text-transform:uppercase;color:var(--primary)}
.row{display:grid;grid-template-columns:repeat(auto-fill,minmax(230px,1fr));gap:10px}
.empty{border:1px dashed var(--line);border-radius:12px;padding:22px;text-align:center;color:var(--muted)}
.bigempty{margin:48px auto;max-width:520px;text-align:center;background:var(--surface);border:1px solid var(--line);border-radius:16px;padding:32px}
.bigempty h2{font-size:20px;margin-bottom:6px}
.foot{color:var(--muted);font-size:12px;padding:10px 0 40px;max-width:90ch}
/* ---------- drawer ---------- */
.scrim{position:fixed;inset:0;background:rgba(10,10,20,.35);opacity:0;pointer-events:none;transition:opacity .15s;z-index:20}
.scrim.open{opacity:1;pointer-events:auto}
.drawer{position:fixed;top:0;right:0;height:100%;width:min(440px,100%);background:var(--surface);border-left:1px solid var(--line);z-index:21;transform:translateX(100%);transition:transform .2s;display:flex;flex-direction:column}
.drawer.open{transform:none}
.dhead{padding:18px 20px;border-bottom:1px solid var(--line);display:flex;gap:12px;align-items:flex-start}
.dhead .av{width:46px;height:46px;font-size:16px}
.dhead h2{font-size:19px}
.dhead p{margin:2px 0 0;color:var(--muted)}
.x{margin-left:auto;border:0;background:transparent;font-size:22px;line-height:1;cursor:pointer;color:var(--muted);padding:4px 8px}
.dbody{padding:6px 20px 20px;overflow:auto;flex:1}
.dsec{border-bottom:1px solid var(--line);padding:4px 0}
.dsec:last-child{border-bottom:0}
.dsec>summary{cursor:pointer;list-style:none;display:flex;align-items:center;gap:8px;padding:10px 0;font-size:12px;font-weight:700;color:var(--muted);text-transform:uppercase;letter-spacing:.05em}
.dsec>summary::-webkit-details-marker{display:none}
.dsec>summary::after{content:"▾";margin-left:auto;transition:transform .15s}
.dsec:not([open])>summary::after{transform:rotate(-90deg)}
.dsec>summary .badge{text-transform:none;letter-spacing:0;font-weight:700;padding:1px 7px;border-radius:6px;background:var(--surface-2);color:var(--muted)}
.dsec .inner{padding:0 0 12px}
dl{display:grid;grid-template-columns:110px 1fr;gap:8px 12px;margin:0}
dt{color:var(--muted);font-size:12.5px}
dd{margin:0;font-weight:600;word-break:break-word;min-width:0}
dd a{color:var(--primary)}
.field{margin:0 0 12px}
.field label{display:block;font-size:12px;font-weight:600;color:var(--muted);margin-bottom:4px}
.field select{width:100%;padding:9px 10px;border:1px solid var(--line);border-radius:10px;background:var(--surface)}
.hint{font-size:12px;color:var(--muted);margin-top:4px;font-weight:400}
.note{background:var(--surface-2);border-radius:10px;padding:10px 12px;font-size:12.5px;color:var(--muted);margin:0 0 12px}
.note b{color:var(--text)}
.note.warn{background:var(--warn-soft)}
.reports{display:flex;flex-direction:column;gap:6px;margin:6px 0 0}
.reports button{display:flex;gap:8px;align-items:center;text-align:left;border:1px solid var(--line);background:var(--surface);border-radius:10px;padding:6px 9px;cursor:pointer}
.reports button:hover{border-color:var(--primary-line)}
.reports .av{width:24px;height:24px;font-size:10px;border-radius:7px}
.more{border:0;background:transparent;color:var(--primary);font-weight:700;cursor:pointer;padding:6px 0}
.crow{display:flex;gap:8px;align-items:center;flex-wrap:wrap;margin:0 0 10px}
.clabel{width:56px;flex:none;color:var(--muted);font-size:12.5px}
.cval{font-weight:700;user-select:all;word-break:break-all;min-width:0}
.cmsg{font-size:12.5px;color:var(--muted);flex:1 1 180px;min-width:0}
.cmsg.err{color:var(--danger)}
.mini{border:1px solid var(--line);background:var(--surface);border-radius:8px;padding:4px 9px;font-size:12px;font-weight:600;cursor:pointer}
.mini:hover{border-color:var(--primary-line)}
.spin{display:inline-block;width:12px;height:12px;border:2px solid var(--primary-line);border-top-color:var(--primary);border-radius:50%;animation:spin 1s linear infinite;vertical-align:-2px;margin-right:6px}
@keyframes spin{to{transform:rotate(360deg)}}
.sr{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0 0 0 0)}
@media (max-width:1100px){
  .stage{grid-template-columns:minmax(0,1fr)}
  .tray{position:static;max-height:none}
}
@media (max-width:720px){
  .wrap{padding:0 16px}
  .account h1{font-size:22px}
  .product{display:none}
  .row{grid-template-columns:1fr}
  .search{max-width:none}
  .orgwrap{max-height:none;min-height:0;overflow:visible;padding:8px 0 20px}
  .forest.outline{width:auto;min-width:0}
  .forest.outline .sub{padding-left:6px}
  .forest.outline .sub>li{padding-left:12px}
  .scrollhint,.scroller::before,.scroller::after,.draghint{display:none!important}
  .cover{flex-wrap:nowrap;overflow-x:auto;margin-right:-16px;padding-right:16px;scrollbar-width:none}
  .cv{flex:none}
  dl{grid-template-columns:96px 1fr}
}
/* ---------- new in role ---------- */
.newpill{display:inline-flex;align-items:center;gap:4px;margin:4px 6px 0 0;font-size:10.5px;font-weight:800;letter-spacing:.02em;padding:1px 7px 1px 6px;border-radius:6px;color:var(--on-new);background:linear-gradient(90deg,var(--new),var(--new-2));box-shadow:0 1px 6px -1px color-mix(in srgb,var(--new) 60%,transparent)}
.newpill::before{content:"";width:6px;height:6px;border-radius:50%;background:currentColor;box-shadow:0 0 0 2px color-mix(in srgb,currentColor 35%,transparent)}
.ocard.is-new{box-shadow:0 0 0 2px var(--new-soft), var(--shadow)}
.ocard.is-new .av{box-shadow:0 0 0 2px var(--surface),0 0 0 4px var(--new)}
.newseg{display:inline-flex;align-items:center;gap:6px;font-size:12px;font-weight:600;color:var(--muted)}
.newseg .seg button{padding:5px 9px}
.newseg .cnt{font-variant-numeric:tabular-nums;color:var(--on-new);background:var(--new);border-radius:999px;padding:1px 8px;font-weight:800}
.newseg .seg button[aria-pressed="true"]{background:var(--new);color:var(--on-new)}
/* ---------- focus mode ---------- */
.stage .ocard,.stage .ghead,.sub>li::before,.sub>li::after,.group::after,.node-root.has-kids::after{transition:opacity .28s ease, filter .28s ease, border-color .28s ease, transform .12s}
.stage.focusing .ocard.unfocus{opacity:.22;filter:saturate(.2)}
.stage.focusing .ocard.unfocus:hover{opacity:.6}
.stage.focusing .sub>li:not(.on-path)::after,.stage.focusing .sub>li:not(.on-path):not(.trunk)::before{opacity:.2}
.stage.focusing .sub>li.trunk::before{border-left-color:var(--primary)}
.stage.focusing .sub>li.on-path:not(.trunk)::before{border-image:linear-gradient(var(--primary) 0 30px, color-mix(in srgb,var(--conn) 22%,transparent) 30px) 1}
.stage.focusing .sub>li.on-path::after{border-top-color:var(--primary);border-top-style:solid}
.stage.focusing .group:not(.on-path) .ghead{opacity:.35}
.stage.focusing .group:not(.on-path)::after,.stage.focusing .group:not(.on-path)::before{opacity:.25}
.stage.focusing .group.on-path::after,.stage.focusing .node-root.on-path.has-kids::after{border-left-color:var(--primary)}
.stage.focusing .ocard.focus-me{box-shadow:0 0 0 4px var(--primary-soft),0 10px 28px -8px color-mix(in srgb,var(--primary) 55%,transparent);transform:translateY(-1px)}
.focuschip{display:inline-flex;align-items:center;gap:8px;padding:3px 4px 3px 10px;border-radius:999px;background:var(--primary-soft);color:var(--text);font-weight:600}
.focuschip button{border:0;background:var(--surface);border-radius:999px;padding:2px 9px;cursor:pointer;font-size:11.5px;font-weight:700}
@media (min-width:721px){ .drawer{box-shadow:-18px 0 40px -18px rgba(10,10,30,.28)} body.drawer-open #app{max-width:none;padding-right:min(470px,45vw)} }
/* ---------- in the news ---------- */
.news{margin:26px 0 8px;border-top:1px solid var(--line);padding-top:18px}
.news-head{display:flex;flex-wrap:wrap;align-items:baseline;gap:6px 12px;margin-bottom:4px}
.news-head h2{font-size:18px}
.news-head .src{font-size:12px;color:var(--muted)}
.nchips{display:flex;flex-wrap:wrap;gap:6px;margin:10px 0 14px}
.nchip{display:inline-flex;align-items:center;gap:6px;border:1px solid var(--line);background:var(--surface);border-radius:999px;padding:4px 10px 4px 4px;font-size:12px;font-weight:600;cursor:pointer;color:var(--text)}
.nchip .av{width:22px;height:22px;font-size:9.5px;border-radius:999px}
.nchip .c{color:var(--muted);font-variant-numeric:tabular-nums}
.nchip[aria-pressed="true"]{border-color:var(--primary);background:var(--primary-soft)}
.nchip.all{padding-left:10px}
.nlist{display:grid;grid-template-columns:repeat(auto-fill,minmax(320px,1fr));gap:12px}
.ncard{border:1px solid var(--line);background:var(--surface);border-radius:14px;padding:14px;box-shadow:var(--shadow);display:flex;flex-direction:column;gap:8px}
.nmeta{display:flex;flex-wrap:wrap;align-items:center;gap:6px 10px;font-size:12px;color:var(--muted)}
.nmeta time{font-weight:700;color:var(--text);font-variant-numeric:tabular-nums}
.nkind{font-size:10.5px;font-weight:700;padding:1px 7px;border-radius:6px;background:var(--surface-2);color:var(--muted)}
.nkind.pr{background:var(--primary-soft);color:var(--primary)}
.ncard h3{font-size:15px;line-height:1.3}
.ncard h3 a{color:var(--text);text-decoration:none}
.ncard h3 a:hover{color:var(--primary);text-decoration:underline}
.ncard p{margin:0;color:var(--muted);font-size:13px}
.npeople{display:flex;flex-wrap:wrap;gap:6px;margin-top:auto}
.npeople button{display:inline-flex;align-items:center;gap:6px;border:1px solid var(--line);background:var(--surface-2);border-radius:999px;padding:2px 9px 2px 3px;font-size:12px;font-weight:600;cursor:pointer}
.npeople button:hover{border-color:var(--primary-line)}
.npeople .av{width:20px;height:20px;font-size:9px;border-radius:999px}
.nalso{font-size:12px;color:var(--muted)}
.nalso a{color:var(--primary)}
.ic .nws{color:var(--muted)}
.dnews{display:flex;flex-direction:column;gap:8px}
.dnews a{color:var(--text);font-weight:600;text-decoration:none}
.dnews a:hover{color:var(--primary);text-decoration:underline}
.dnews .hint{margin:0}
/* ---------- motion ---------- */
@keyframes oa-pop{from{opacity:0;transform:translateY(10px) scale(.96)}to{opacity:1;transform:none}}
@keyframes oa-drawx{from{transform:scaleX(0)}to{transform:scaleX(1)}}
@keyframes oa-drawy{from{transform:scaleY(0)}to{transform:scaleY(1)}}
@keyframes oa-fade{from{opacity:0}to{opacity:1}}
@keyframes oa-land{0%{box-shadow:0 0 0 0 color-mix(in srgb,var(--primary) 55%,transparent),var(--shadow)}100%{box-shadow:0 0 0 14px transparent,var(--shadow)}}
.intro{--step:150ms}
.intro .ocard{animation:oa-pop .5s cubic-bezier(.2,.8,.2,1) backwards;animation-delay:calc(var(--d,0) * var(--step) + var(--i,0) * 35ms + 60ms)}
.intro .ghead{animation:oa-pop .45s cubic-bezier(.2,.8,.2,1) backwards;animation-delay:calc(var(--step) * .7 + var(--i,0) * 35ms)}
.intro .sub>li::after{transform-origin:left;animation:oa-drawx .3s ease-out backwards;animation-delay:calc(var(--d,0) * var(--step) + var(--i,0) * 35ms - 40ms)}
.intro .sub>li::before{transform-origin:top;animation:oa-drawy .35s ease-out backwards;animation-delay:calc(var(--d,0) * var(--step) + var(--i,0) * 35ms - 110ms)}
.intro .group::before{transform-origin:center;animation:oa-drawx .4s ease-out backwards;animation-delay:calc(var(--step) * .35)}
.intro .group::after,.intro .node-root.has-kids::after{transform-origin:top;animation:oa-drawy .25s ease-out backwards;animation-delay:calc(var(--step) * .5)}
.intro .fold{animation:oa-fade .5s ease backwards;animation-delay:.9s}
.ocard.landed{animation:oa-land .9s ease-out .35s 1}
.ocard.appear{animation:oa-pop .35s cubic-bezier(.2,.8,.2,1) backwards}
@media (prefers-reduced-motion: reduce){*,*::before,*::after{transition:none!important;animation:none!important}}
</style>
<!--BODY-->
<script type="application/json" id="orgatlas-data">__DATA_JSON__</script>
<header class="topbar"><div class="wrap">
  <div class="brand" id="brand"></div><span class="product">OrgAtlas</span>
  <span class="spacer"></span><span class="conn" id="conn"></span>
</div></header>
<div id="banner"></div>
<main class="wrap" id="app"></main>
<div class="scrim" id="scrim"></div>
<aside class="drawer" id="drawer" role="dialog" aria-modal="true" aria-labelledby="dname" aria-hidden="true"></aside>
<div class="sr" aria-live="polite" id="live"></div>
<script>
(function(){
const D = JSON.parse(document.getElementById('orgatlas-data').textContent);
const C = D.company || {}, T = D.theme || {};
const ROLES = ["Unclassified","Champion","Executive Buyer","Influencer","Blocker"];
const LEVELS = ["C-Suite","VP"];
const TOP = '__top__', TRAY = '__tray__';
const GROUPS = [ // function lanes under the top executive
  ['Engineering & Security', ['Engineering'], 'var(--g1)'],
  ['Product', ['Product'], 'var(--g2)'],
  ['Go-to-market', ['Revenue', 'Sales', 'Marketing', 'Customer'], 'var(--g3)'],
  ['People', ['People'], 'var(--g4)'],
  ['Finance & Legal', ['Finance', 'Legal'], 'var(--g5)'],
  ['Operations & Strategy', ['Operations', 'Strategy'], 'var(--g6)'],
  ['Other', ['Executive', 'Other'], 'var(--g7)'],
];
const groupOf = fn => GROUPS.find(g => g[1].includes(fn)) || GROUPS[GROUPS.length - 1];
const isDemo = D.people.some(p => p.source === 'Demo');
const KEY = 'orgatlas:v1:' + String(C.clayCompanyId || C.domain || C.name || 'account').toLowerCase();
const byId = Object.fromEntries(D.people.map(p => [p.id, p]));
const $ = s => document.querySelector(s);
const esc = s => String(s ?? '').replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
const initials = n => n.split(/\s+/).filter(w => /^[A-Za-z]/.test(w)).slice(0,2).map(w => w[0]).join('').toUpperCase() || '?';
const fmt = iso => { try { return new Date(iso).toLocaleString(undefined,{dateStyle:'medium',timeStyle:'short'}); } catch(e){ return iso || '—'; } };
const load = (k, d) => { try { return JSON.parse(localStorage.getItem(k) || 'null') ?? d; } catch(e) { return d; } };
const store = (k, v) => { try { localStorage.setItem(k, JSON.stringify(v)); } catch(e) {} };
let edits = load(KEY, {});
const folded = load(KEY + ':folded', {});
const state = { q:'', fn:'All', role:null, sel:null, view: load(KEY + ':view', 'org'), zoom: load(KEY + ':zoom', 1), newWin: load(KEY + ':new', 6) };
const NEW_WINDOWS = [0, 3, 6, 12];
const MON = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];
const reduceMotion = () => window.matchMedia('(prefers-reduced-motion: reduce)').matches;
function monthsIn(s){ // months since the start of the current title (Clay gives YYYY-MM)
  const m = /^(\d{4})-(\d{2})/.exec(s || ''); if (!m) return null;
  const n = new Date(); return (n.getFullYear() * 12 + n.getMonth() + 1) - (+m[1] * 12 + +m[2]);
}
const startLabel = s => { const m = /^(\d{4})-(\d{2})/.exec(s || ''); return m ? `${MON[+m[2] - 1]} ${m[1]}` : ''; };
const isNew = p => { if (!state.newWin) return false; const k = monthsIn(p.roleStartDate); return k !== null && k >= 0 && k < state.newWin; };
const hasStarts = D.people.some(p => p.roleStartDate);
let focusSet = null, firstPaint = true, justMoved = null;
const NEWS = (D.news && D.news.items) || [];
const newsFor = id => NEWS.filter(n => n.people.includes(id));
const fmtDay = d => { try { return new Date(d + 'T12:00:00').toLocaleDateString(undefined, { month: 'short', day: 'numeric', year: 'numeric' }); } catch(e) { return d; } };
state.newsBy = null;
function computeFocus(km){
  if (!state.sel || state.view !== 'org' || !byId[state.sel]) return null;
  const set = new Set([state.sel]);
  let cur = mgrOf(state.sel), hops = 0; while (cur && hops++ < 500) { set.add(cur); cur = mgrOf(cur); }
  const st = [...(km[state.sel] || [])]; while (st.length) { const c = st.pop(); set.add(c); st.push(...(km[c] || [])); }
  return set;
}
const announce = m => { $('#live').textContent = m; };
const narrow = () => window.matchMedia('(max-width: 720px)').matches;
const roleCls = r => r && r !== 'Unclassified' ? 'role-' + r.replace(/\s+/g, '-') : '';

/* ---------- effective structure: Clay data + inference + local edits ---------- */
const inferredMgr = id => { const r = byId[id]?.reportsTo; return r && r.id && byId[r.id] ? r.id : null; };
const rawMgr = id => { const e = edits[id]?.managerId; if (e === TOP || e === TRAY) return null; if (e && byId[e]) return e; return inferredMgr(id); };
function mgrOf(id){ // walk up to break any cycle defensively
  const m = rawMgr(id); if (!m) return null;
  let cur = m, hops = 0; while (cur && hops++ < 500) { if (cur === id) return null; cur = rawMgr(cur); }
  return m;
}
function placement(id){ // 'child' | 'root' (top of the chart) | 'tray' (needs review)
  if (mgrOf(id)) return 'child';
  const e = edits[id]?.managerId;
  if (e === TOP) return 'root';
  if (e === TRAY) return 'tray';
  return byId[id].reportsTo === null ? 'root' : 'tray';
}
const baseMgrValue = id => inferredMgr(id) || (byId[id].reportsTo === null ? TOP : TRAY);
const relSource = id => edits[id]?.managerId ? 'user' : (inferredMgr(id) ? 'inferred' : 'none');
const edgeCls = id => relSource(id) === 'user' ? 'e-user' : (byId[id].reportsTo?.confidence === 'low' ? 'e-low' : 'e-inf');
const eff = p => ({...p, level: edits[p.id]?.level || p.level, buyingRole: edits[p.id]?.buyingRole || p.buyingRole});
const edited = p => !!edits[p.id] && Object.keys(edits[p.id]).length > 0;
function kidsMap(){ const m = {}; D.people.forEach(p => { const k = mgrOf(p.id); if (k) (m[k] = m[k] || []).push(p.id); }); return m; }
function isDescendant(ancestor, id, km){ const stack = [...(km[ancestor] || [])]; while (stack.length){ const c = stack.pop(); if (c === id) return true; stack.push(...(km[c] || [])); } return false; }
const teamSize = (id, km) => { let n = 0; const st = [...(km[id] || [])]; while (st.length) { const c = st.pop(); n++; st.push(...(km[c] || [])); } return n; };
function setEdit(id, field, value){
  const src = byId[id]; edits[id] = edits[id] || {};
  const base = field === 'managerId' ? baseMgrValue(id) : src[field];
  if (value === base) delete edits[id][field]; else edits[id][field] = value;
  if (!Object.keys(edits[id]).length) delete edits[id];
  store(KEY, edits);
}

/* ---------- header ---------- */
$('#brand').innerHTML = T.logoDataUri ? `<img src="${esc(T.logoDataUri)}" alt="${esc(T.brandName || 'Logo')}">` : `<span>${esc(T.brandName || 'OrgAtlas')}</span>`;
const conn = $('#conn');
conn.textContent = isDemo ? 'Demo data — not live Clay' : 'Clay · live search results';
if (isDemo) { conn.classList.add('demo'); $('#banner').innerHTML = '<div class="banner"><div class="wrap">Demo mode: these records are fixtures, not Clay results.</div></div>'; }

function matches(p){
  const q = state.q.trim().toLowerCase();
  return (state.fn === 'All' || p.function === state.fn) && (!state.role || eff(p).buyingRole === state.role)
    && (!q || (p.name + ' ' + p.title + ' ' + p.function + ' ' + (p.location||'')).toLowerCase().includes(q));
}
const filtering = () => !!(state.q.trim() || state.fn !== 'All' || state.role);
const ICON = {
  email: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" aria-hidden="true"><rect x="3" y="5" width="18" height="14" rx="2"/><path d="m3 7 9 6 9-6"/></svg>',
  phone: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" aria-hidden="true"><rect x="7" y="2" width="10" height="20" rx="2"/><path d="M11 18h2"/></svg>',
  news: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" aria-hidden="true"><path d="M4 5h13v14H6a2 2 0 0 1-2-2z"/><path d="M17 9h3v8a2 2 0 0 1-2 2"/><path d="M8 9h5M8 13h5"/></svg>',
};

function coverage(all, km){
  const cnt = r => all.filter(p => p.buyingRole === r).length;
  const contacts = D.people.filter(p => found[p.id]?.email?.value || p.email || found[p.id]?.phone?.value).length;
  const tray = D.people.filter(p => placement(p.id) === 'tray').length;
  const userLines = D.people.filter(p => edits[p.id]?.managerId).length;
  const cs = all.filter(p => p.level === 'C-Suite').length;
  const rb = (r, label, color) => `<button class="cv" data-role="${esc(r)}" aria-pressed="${state.role === r}"><i style="background:${color}"></i><span class="n">${cnt(r)}</span><span class="l">${label}</span></button>`;
  return `<div class="cover" aria-label="Account coverage">
    <div class="cv"><span class="n">${all.length}</span><span class="l">Leaders<br>${cs} C-Suite · ${all.length - cs} VP</span></div>
    <span class="cv-sep" aria-hidden="true"></span>
    ${rb('Champion', 'Champions', 'var(--r-champion)')}${rb('Executive Buyer', 'Exec buyers', 'var(--r-buyer)')}${rb('Influencer', 'Influencers', 'var(--r-influencer)')}${rb('Blocker', 'Blockers', 'var(--r-blocker)')}
    <button class="cv" data-role="Unclassified" aria-pressed="${state.role === 'Unclassified'}"><i style="background:var(--line)"></i><span class="n">${cnt('Unclassified')}</span><span class="l">Not yet<br>classified</span></button>
    <span class="cv-sep" aria-hidden="true"></span>
    <div class="cv"><span class="n">${contacts}</span><span class="l">Contacts<br>found</span></div>
    ${tray ? `<button class="cv warn" id="goto-tray"><span class="n">${tray}</span><span class="l">Need a<br>manager</span></button>` : ''}
    <div class="cv"><span class="n">${userLines}</span><span class="l">Lines you<br>set</span></div>
    ${state.newWin && hasStarts ? `<span class="cv-sep" aria-hidden="true"></span><div class="cv"><i style="background:var(--new)"></i><span class="n">${all.filter(isNew).length}</span><span class="l">New in role<br>last ${state.newWin} mo</span></div>` : ''}
  </div>`;
}

function tile(p, km, cls = ''){
  const n = (km[p.id] || []).length, tsz = teamSize(p.id, km);
  const f = filtering() ? (matches(p) ? 'hit' : 'dim') : '';
  const hasE = !!(found[p.id]?.email?.value || p.email), hasM = !!found[p.id]?.phone?.value;
  const nNews = newsFor(p.id).length;
  const nw = isNew(p), k = nw ? monthsIn(p.roleStartDate) : 0;
  const newPill = nw ? `<span class="newpill" title="In this role since ${esc(startLabel(p.roleStartDate))} (Clay)">New · ${k < 1 ? 'this month' : k + ' mo'}</span>` : '';
  const role = p.buyingRole !== 'Unclassified' ? `<span class="rpill">${esc(p.buyingRole)}</span>` : '';
  const fc = focusSet ? (p.id === state.sel ? 'focus-me' : focusSet.has(p.id) ? 'in-focus' : 'unfocus') : '';
  const lm = justMoved === p.id ? 'landed' : '';
  return `<div class="node"><button class="ocard ${cls} ${f} ${fc} ${lm} ${nw ? 'is-new' : ''} ${roleCls(p.buyingRole)} ${p.level === 'C-Suite' ? 'is-c' : ''} ${state.sel === p.id ? 'selected' : ''}" draggable="true" data-id="${esc(p.id)}" aria-label="${esc(p.name)}, ${esc(p.title)}${nw ? `, new in role since ${esc(startLabel(p.roleStartDate))}` : ''}${p.buyingRole !== 'Unclassified' ? ', ' + esc(p.buyingRole) : ''}${n ? `, ${n} direct report${n === 1 ? '' : 's'}` : ''}${nNews ? `, ${nNews} news item${nNews === 1 ? '' : 's'}` : ''}. Open details">
    <span class="av" aria-hidden="true">${esc(initials(p.name))}</span>
    <span class="cbody"><span class="nm"><b>${esc(p.name)}</b>${hasE || hasM || nNews ? `<span class="ic">${hasE ? `<span title="Email found">${ICON.email}</span>` : ''}${hasM ? `<span title="Mobile found">${ICON.phone}</span>` : ''}${nNews ? `<span class="nws" title="In the news: ${nNews} item${nNews === 1 ? '' : 's'}">${ICON.news}</span>` : ''}</span>` : ''}${edited(p) ? '<span class="edot" title="Edited in this browser"></span>' : ''}</span>
      <span class="tt">${esc(p.title)}</span>${newPill}${role}</span>
  </button>${n ? `<button class="fold" data-fold="${esc(p.id)}" aria-expanded="${!folded[p.id]}" aria-label="${folded[p.id] ? 'Show' : 'Hide'} ${esc(p.name)}'s team of ${tsz}">${folded[p.id] ? `▸ ${tsz} hidden` : `▾ ${n}`}</button>` : ''}</div>`;
}
const onPath = id => focusSet && focusSet.has(id) ? 'on-path' : '';
const pathCls = (ks, j) => !focusSet ? '' : onPath(ks[j]) + (ks.slice(j + 1).some(k => focusSet.has(k)) ? ' trunk' : ''); // trunk: the vertical line runs on to a later sibling on the path
const subTree = (id, km, d = 2, gi = 0) => (km[id] || []).length && !folded[id]
  ? `<ul class="sub">${km[id].map((k, j) => `<li class="${edgeCls(k)} ${pathCls(km[id], j)}" style="--d:${d};--i:${gi + j}">${tile(eff(byId[k]), km)}${subTree(k, km, d + 1, gi + j)}</li>`).join('')}</ul>` : '';

function orgChart(km){
  const roots = D.people.filter(p => placement(p.id) === 'root').map(p => p.id);
  if (narrow()) { // phone: an indented outline instead of a wide tree
    return `<div class="forest outline ${firstPaint ? 'intro' : ''}">${roots.map(r => `<div>${tile(eff(byId[r]), km, 'root')}${subTree(r, km, 1)}</div>`).join('')}</div>`;
  }
  const tree = r => {
    const kids = km[r] || [];
    if (!kids.length || folded[r]) return `<div class="tree"><div class="node-root">${tile(eff(byId[r]), km, 'root')}</div></div>`;
    const lanes = GROUPS.map(g => [g, kids.filter(k => g[1].includes(byId[k].function))]).filter(([, ks]) => ks.length);
    const other = kids.filter(k => !GROUPS.some(g => g[1].includes(byId[k].function)));
    if (other.length) lanes.push([GROUPS[GROUPS.length - 1], other]);
    return `<div class="tree">
      <div class="node-root has-kids ${onPath(r)}">${tile(eff(byId[r]), km, 'root')}</div>
      <div class="bus">${lanes.map(([g, ks], gi) => `<section class="group ${ks.some(k => focusSet && focusSet.has(k)) ? 'on-path' : ''}" style="--gc:${g[2]};--i:${gi}" aria-label="${esc(g[0])}">
        <div class="ghead"><span>${esc(g[0])}</span><span class="gn">${ks.length}</span></div>
        <ul class="sub gsub">${ks.map((k, j) => `<li class="${edgeCls(k)} ${pathCls(ks, j)}" style="--d:1;--i:${gi + j}">${tile(eff(byId[k]), km)}${subTree(k, km, 2, gi + j)}</li>`).join('')}</ul>
      </section>`).join('')}</div>
    </div>`;
  };
  return `<div class="forest ${firstPaint ? 'intro' : ''}" style="zoom:${state.zoom}">${roots.map(tree).join('')}</div>`;
}

function trayPanel(km){
  const ids = D.people.filter(p => placement(p.id) === 'tray').map(p => p.id);
  const lanes = GROUPS.map(g => [g, ids.filter(k => g[1].includes(byId[k].function))]).filter(([, ks]) => ks.length);
  return `<aside class="tray" id="tray" aria-labelledby="tray-h">
    <h2 id="tray-h">Needs review <span class="n">${ids.length}</span></h2>
    <p>Clay's data has no same-function leader for these people, so no reporting line was guessed. Drag each one onto their manager's tile, or pick a manager in their side panel.</p>
    ${ids.length ? lanes.map(([g, ks]) => `<div class="tgroup" style="--gc:${g[2]}"><div class="ghead"><span>${esc(g[0])}</span><span class="gn">${ks.length}</span></div>
      <div class="tlist">${ks.map(k => `<div>${tile(eff(byId[k]), km)}${subTree(k, km)}</div>`).join('')}</div></div>`).join('')
      : '<div class="empty">Everyone has a manager. Drag someone here to take them out of the chart for review.</div>'}
  </aside>`;
}

function newsSection(){
  if (!D.news) return '';
  const counts = {}; NEWS.forEach(n => n.people.forEach(id => { counts[id] = (counts[id] || 0) + 1; }));
  const who = Object.keys(counts).filter(id => byId[id]).sort((a, b) => counts[b] - counts[a] || byId[a].name.localeCompare(byId[b].name));
  if (state.newsBy && !counts[state.newsBy]) state.newsBy = null;
  const list = state.newsBy ? NEWS.filter(n => n.people.includes(state.newsBy)) : NEWS;
  const win = D.news.windowMonths || 3;
  const card = n => `<article class="ncard">
      <div class="nmeta"><time datetime="${esc(n.date)}">${esc(fmtDay(n.date))}</time><span>${esc(n.source)}</span><span class="nkind ${/press/i.test(n.kind) ? 'pr' : ''}">${esc(n.kind)}</span></div>
      <h3><a href="${esc(n.url)}" target="_blank" rel="noopener noreferrer">${esc(n.title)}</a></h3>
      ${n.summary ? `<p>${esc(n.summary)}</p>` : ''}
      ${n.alsoAt.length ? `<div class="nalso">Also at ${n.alsoAt.map(a => `<a href="${esc(a.url)}" target="_blank" rel="noopener noreferrer">${esc(a.source)}</a>`).join(', ')}</div>` : ''}
      <div class="npeople">${n.people.filter(id => byId[id]).map(id => `<button data-open="${esc(id)}" title="${esc(byId[id].title)}"><span class="av" aria-hidden="true">${esc(initials(byId[id].name))}</span>${esc(byId[id].name)}</button>`).join('')}</div>
    </article>`;
  return `<section class="news" id="news" aria-labelledby="news-h">
    <div class="news-head"><h2 id="news-h">In the news</h2><span class="src">Last ${win} months · ${NEWS.length} item${NEWS.length === 1 ? '' : 's'} naming these leaders · from public web sources, not Clay${D.news.checkedAt ? ` · checked ${esc(fmtDay(D.news.checkedAt))}` : ''}</span></div>
    ${NEWS.length ? `<div class="nchips" role="group" aria-label="Filter news by person">
      <button class="nchip all" data-newsby="" aria-pressed="${!state.newsBy}">Everyone <span class="c">${NEWS.length}</span></button>
      ${who.map(id => `<button class="nchip" data-newsby="${esc(id)}" aria-pressed="${state.newsBy === id}"><span class="av" aria-hidden="true">${esc(initials(byId[id].name))}</span>${esc(byId[id].name)} <span class="c">${counts[id]}</span></button>`).join('')}
    </div>
    <div class="nlist">${list.map(card).join('')}</div>`
    : `<div class="empty">No news in the last ${win} months named any of these leaders.</div>`}
  </section>`;
}

function render(){
  const all = D.people.map(eff);
  const app = $('#app');
  const km = kidsMap();
  focusSet = computeFocus(km);
  const before = flipFirst();
  const head = `
    <section class="account">
      <h1>${esc(C.name)}</h1>
      <div class="meta">
        ${C.domain ? `<span><b>${esc(C.domain)}</b></span>` : ''}
        ${C.industry ? `<span>${esc(C.industry)}</span>` : ''}
        ${C.location ? `<span>${esc(C.location)}</span>` : ''}
        <span>Source: <b>${isDemo ? 'Demo fixtures' : 'Clay'}</b></span>
        <span>Retrieved <b>${esc(fmt(D.retrievedAt))}</b></span>
      </div>
      ${all.length ? coverage(all, km) : ''}
    </section>`;
  if (!all.length) {
    app.innerHTML = head + `<div class="bigempty"><h2>No matching leaders</h2><p>Clay returned no current C-Suite or VP leaders for ${esc(C.name)}. Nothing has been filled in with guesses.</p></div>`;
    return;
  }
  const fns = [...new Set(all.map(p => p.function))].sort();
  const nEdits = Object.keys(edits).filter(id => byId[id]).length;
  const nFold = Object.keys(folded).filter(id => folded[id]).length;
  const toolbar = `
    <div class="toolbar" role="search">
      <div class="seg" role="group" aria-label="View">
        <button data-view="org" aria-pressed="${state.view === 'org'}">Org chart</button>
        <button data-view="level" aria-pressed="${state.view === 'level'}">By level</button>
      </div>
      <div class="search"><svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" aria-hidden="true"><circle cx="11" cy="11" r="7"/><path d="m20 20-3.5-3.5"/></svg>
        <label class="sr" for="q">Search leaders</label><input id="q" type="search" placeholder="Search name, title, location" value="${esc(state.q)}"></div>
      <label class="sr" for="fn">Function</label>
      <select id="fn" class="sel"><option value="All">All functions</option>${fns.map(f => `<option ${f === state.fn ? 'selected' : ''}>${esc(f)}</option>`).join('')}</select>
      ${filtering() ? '<button class="btn sm" id="clear">Clear filters</button>' : ''}
      ${hasStarts ? `<div class="newseg"><span id="new-l">New in role</span><div class="seg" role="group" aria-labelledby="new-l">${NEW_WINDOWS.map(w => `<button data-new="${w}" aria-pressed="${state.newWin === w}">${w ? w + ' mo' : 'Off'}</button>`).join('')}</div>${state.newWin ? `<span class="cnt">${all.filter(isNew).length}</span>` : ''}</div>` : ''}
      <span class="spacer"></span>
      <details class="menu"><summary aria-label="More options">···</summary><div class="pop">
        <button id="unfold" ${nFold ? '' : 'disabled'}>Expand all teams</button>
        <button id="reset" class="danger" ${nEdits ? '' : 'disabled'}>Reset local edits${nEdits ? ` (${nEdits})` : ''}</button>
        <div class="hint">Resets managers, buying roles and levels you changed. Contact details you found are kept.</div>
      </div></details>
    </div>`;
  let body;
  if (state.view === 'org') {
    body = `
      <div class="chartbar"><strong>Org chart</strong>
        <span class="legend"><i></i> Inferred from titles</span>
        <span class="legend"><i class="user"></i> Set by you</span>
        ${focusSet ? `<span class="focuschip">Focused on ${esc(byId[state.sel].name)} · ${focusSet.size - 1} connected <button id="unfocus">Clear focus</button></span>` : '<span class="draghint">Click a person to focus on their reporting line. Drag them onto someone\'s tile to change who they report to.</span>'}
        <span class="spacer"></span>
        ${narrow() ? '' : `<span class="zoom" role="group" aria-label="Zoom"><button id="z-out" aria-label="Zoom out">−</button><span id="z-val">${Math.round(state.zoom * 100)}%</span><button id="z-in" aria-label="Zoom in">+</button><button id="z-fit">Fit</button></span>`}
      </div>
      <div class="stage ${focusSet ? 'focusing' : ''}">
        <div class="scroller" id="scroller"><div class="topzone" id="topzone">Drop here to put this person at the top of the chart, with no manager</div><span class="scrollhint" aria-hidden="true">Scroll →</span><div class="orgwrap" id="orgwrap">${orgChart(km)}</div></div>
        ${trayPanel(km)}
      </div>`;
  } else {
    const tier = (lvl, n, label, cls) => {
      const list = all.filter(p => p.level === lvl && matches(p));
      return `<section class="tier ${cls}" data-level="${lvl}" aria-labelledby="h-${cls}">
        <div class="tier-head"><span class="lvl">Level ${n}</span><h2 id="h-${cls}">${label}</h2><span class="count">${list.length} shown</span></div>
        <div class="row">${list.length ? list.map(p => tile(p, {})).join('') : `<div class="empty">${filtering() ? 'No leaders match these filters.' : 'Drag a card here to move someone to this level.'}</div>`}</div>
      </section>`;
    };
    body = `<div class="chartbar"><strong>By level</strong><span>Grouped by title seniority. Drag a card to the other level to correct it.</span></div>
      ${tier('C-Suite', 1, 'C-Suite', 'l1')}${tier('VP', 2, 'VP Leadership', 'l2')}`;
  }
  const prev = $('#orgwrap'), prevScroll = prev ? [prev.scrollLeft, prev.scrollTop] : null;
  app.innerHTML = head + toolbar + body + newsSection() + `<p class="foot">People, titles, locations and LinkedIn links come from Clay. Clay does not supply reporting relationships: lines are inferred from titles and functions (dotted amber lines are low confidence), and people with no evidence wait in Needs review. A coloured left edge marks a buying role you set. An amber dot marks a correction saved in this browser only.${hasStarts ? ' “New” badges use the start date Clay has for each person\'s current title, so a recent promotion also counts.' : ''}</p>`;
  wire(km);
  const ow = $('#orgwrap');
  if (ow) {
    if (prevScroll) { ow.scrollLeft = prevScroll[0]; ow.scrollTop = prevScroll[1]; }
    else { const rc = ow.querySelector('.node-root'); if (rc) ow.scrollLeft = Math.max(0, rc.offsetLeft + rc.offsetWidth / 2 - ow.clientWidth / 2); }
    updateHints();
  }
  if (!firstPaint) flipPlay(before);
  firstPaint = false; justMoved = null;
}
/* FLIP: tiles that change position glide from their old spot instead of jumping */
function flipFirst(){
  const m = {}; if (firstPaint || reduceMotion()) return m;
  document.querySelectorAll('#app .ocard[data-id]').forEach(c => { m[c.dataset.id] = c.getBoundingClientRect(); });
  return m;
}
function flipPlay(before){
  if (reduceMotion() || !Object.keys(before).length) return;
  document.querySelectorAll('#app .ocard[data-id]').forEach(c => {
    const a = before[c.dataset.id];
    if (!a) { c.classList.add('appear'); return; }
    const b = c.getBoundingClientRect(), dx = a.left - b.left, dy = a.top - b.top;
    if (Math.abs(dx) < 1 && Math.abs(dy) < 1) return;
    const z = c.closest('.forest:not(.outline)') ? state.zoom : 1;
    c.animate([{ transform: `translate(${dx / z}px, ${dy / z}px)` }, { transform: 'none' }],
      { duration: 480, easing: 'cubic-bezier(.2,.8,.2,1)' });
  });
}
function updateHints(){
  const ow = $('#orgwrap'), sc = $('#scroller'); if (!ow || !sc) return;
  sc.classList.toggle('can-left', ow.scrollLeft > 4);
  sc.classList.toggle('can-right', ow.scrollLeft + ow.clientWidth < ow.scrollWidth - 4);
}
function setZoom(z){
  state.zoom = Math.max(0.4, Math.min(1.25, Math.round(z * 100) / 100)); store(KEY + ':zoom', state.zoom);
  const f = $('#orgwrap .forest'); if (f) f.style.zoom = state.zoom;
  const v = $('#z-val'); if (v) v.textContent = Math.round(state.zoom * 100) + '%';
  updateHints();
}
function fitZoom(){
  const ow = $('#orgwrap'), f = ow && ow.querySelector('.forest'); if (!f) return;
  f.style.zoom = 1; const w = f.scrollWidth;
  setZoom(Math.min(1, (ow.clientWidth - 8) / w)); ow.scrollLeft = 0;
}

let dragId = null;
function endDrag(){ dragId = null; document.body.classList.remove('dragging'); document.querySelectorAll('.drop-ok,.drop-no').forEach(x => x.classList.remove('drop-ok','drop-no')); }
function wire(km){
  const q = $('#q');
  if (q) q.addEventListener('input', e => { state.q = e.target.value; const pos = e.target.selectionStart; render(); const n = $('#q'); n.focus(); n.setSelectionRange(pos,pos); });
  const fn = $('#fn'); if (fn) fn.onchange = e => { state.fn = e.target.value; render(); $('#fn').focus(); };
  const cl = $('#clear'); if (cl) cl.onclick = () => { state.q = ''; state.fn = 'All'; state.role = null; render(); $('#q').focus(); };
  document.querySelectorAll('[data-role]').forEach(b => b.onclick = () => { state.role = state.role === b.dataset.role ? null : b.dataset.role; render(); document.querySelector(`[data-role="${CSS.escape(b.dataset.role)}"]`)?.focus(); });
  const gt = $('#goto-tray'); if (gt) gt.onclick = () => { state.view = 'org'; render(); $('#tray')?.scrollIntoView({ behavior: 'smooth', block: 'start' }); $('#tray h2')?.setAttribute('tabindex', '-1'); $('#tray h2')?.focus(); };
  document.querySelectorAll('.seg button').forEach(b => b.onclick = () => { state.view = b.dataset.view; store(KEY + ':view', state.view); render(); document.querySelector(`.seg button[data-view="${state.view}"]`)?.focus(); });
  const r = $('#reset');
  if (r) r.onclick = () => {
    if (!r.classList.contains('armed')) { r.classList.add('armed'); r.textContent = 'Click again to reset'; return; }
    edits = {}; try { localStorage.removeItem(KEY); } catch(e) {} announce('Local edits reset'); render();
  };
  const uf = $('#unfold'); if (uf) uf.onclick = () => { Object.keys(folded).forEach(k => delete folded[k]); store(KEY + ':folded', folded); render(); };
  document.querySelectorAll('[data-fold]').forEach(b => b.onclick = e => { e.stopPropagation(); const id = b.dataset.fold; folded[id] = !folded[id]; if (!folded[id]) delete folded[id]; store(KEY + ':folded', folded); render(); document.querySelector(`[data-fold="${CSS.escape(id)}"]`)?.focus(); });
  document.querySelectorAll('[data-new]').forEach(b => b.onclick = () => { state.newWin = +b.dataset.new; store(KEY + ':new', state.newWin); render(); document.querySelector(`[data-new="${state.newWin}"]`)?.focus(); announce(state.newWin ? `Highlighting leaders new in role in the last ${state.newWin} months` : 'New-in-role highlight off'); });
  const uff = $('#unfocus'); if (uff) uff.onclick = closeDrawer;
  document.querySelectorAll('[data-newsby]').forEach(b => b.onclick = () => { state.newsBy = b.dataset.newsby || null; render(); document.querySelector(`[data-newsby="${CSS.escape(state.newsBy || '')}"]`)?.focus(); });
  document.querySelectorAll('#news [data-open]').forEach(b => b.onclick = () => openDrawer(b.dataset.open));
  const owc = $('#orgwrap'); if (owc) owc.addEventListener('click', e => { if (state.sel && !e.target.closest('.ocard,.fold')) closeDrawer(); });
  const zi = $('#z-in'), zo = $('#z-out'), zf = $('#z-fit');
  if (zi) { zi.onclick = () => setZoom(state.zoom + 0.1); zo.onclick = () => setZoom(state.zoom - 0.1); zf.onclick = fitZoom; }
  const ow = $('#orgwrap'); if (ow) ow.addEventListener('scroll', updateHints, { passive: true });
  document.querySelectorAll('.ocard').forEach(c => {
    c.onclick = () => openDrawer(c.dataset.id);
    c.addEventListener('dragstart', e => { dragId = c.dataset.id; e.dataTransfer.setData('text/plain', dragId); e.dataTransfer.effectAllowed = 'move'; c.classList.add('dragging'); document.body.classList.add('dragging'); });
    c.addEventListener('dragend', () => { c.classList.remove('dragging'); endDrag(); });
  });
  if (state.view === 'org') {
    document.querySelectorAll('.ocard').forEach(c => { // drop a person onto someone's tile = report to them
      const target = c.dataset.id;
      const ok = () => dragId && dragId !== target && !isDescendant(dragId, target, km);
      c.addEventListener('dragover', e => { if (ok()) { e.preventDefault(); e.dataTransfer.dropEffect = 'move'; c.classList.add('drop-ok'); } else c.classList.add('drop-no'); });
      c.addEventListener('dragleave', () => c.classList.remove('drop-ok','drop-no'));
      c.addEventListener('drop', e => {
        e.preventDefault(); e.stopPropagation(); if (!ok()) return;
        const id = dragId; setEdit(id, 'managerId', target); justMoved = id; if (folded[target]) { delete folded[target]; store(KEY + ':folded', folded); }
        announce(`${byId[id].name} now reports to ${byId[target].name}`); endDrag(); render();
      });
    });
    const zone = (el, value, msg) => { if (!el) return;
      el.addEventListener('dragover', e => { if (dragId) { e.preventDefault(); el.classList.add('drop-ok'); } });
      el.addEventListener('dragleave', e => { if (!el.contains(e.relatedTarget)) el.classList.remove('drop-ok'); });
      el.addEventListener('drop', e => { e.preventDefault(); if (!dragId) return; const id = dragId; setEdit(id, 'managerId', value); justMoved = id; announce(`${byId[id].name} ${msg}`); endDrag(); render(); });
    };
    zone($('#topzone'), TOP, 'moved to the top of the chart');
    zone($('#tray'), TRAY, 'moved to Needs review');
  } else {
    document.querySelectorAll('.tier').forEach(t => { // drop onto a section = change level
      t.addEventListener('dragover', e => { e.preventDefault(); t.classList.add('drop'); });
      t.addEventListener('dragleave', e => { if (!t.contains(e.relatedTarget)) t.classList.remove('drop'); });
      t.addEventListener('drop', e => {
        e.preventDefault(); t.classList.remove('drop');
        const id = e.dataTransfer.getData('text/plain'); if (!id || !byId[id]) return;
        setEdit(id, 'level', t.dataset.level); announce(`${byId[id].name} moved to ${t.dataset.level}`); render();
      });
    });
  }
}
let rsz = null, wasNarrow = narrow();
window.addEventListener('resize', () => { clearTimeout(rsz); rsz = setTimeout(() => { if (narrow() !== wasNarrow) { wasNarrow = narrow(); if (!$('#drawer').classList.contains('open')) render(); } else updateHints(); }, 150); });
document.addEventListener('click', e => { document.querySelectorAll('details.menu[open]').forEach(m => { if (!m.contains(e.target)) m.removeAttribute('open'); }); });

/* ---------- contact sourcing through the VIEWER's Clay connector ----------
   The page never holds Clay credentials: calls go through the claude.ai mcp capability
   as the person viewing, and only when they click and confirm (lookups use Clay credits). */
const ENR = D.enrich || {};
const SERVER = ENR.server || 'Clay';
const found = load(KEY + ':contacts', {});   // {personId: {email:{value,at}, phone:{value,at}, pending:{kind:{taskId,eids}}}}
const busy = {};                              // `${id}:${kind}` -> {phase, msg}
const KINDS = ENR.phone ? ['email', 'phone'] : ['email'];
const LABEL = { email: 'Email', phone: 'Mobile' };
let mcpP = null;
const getMcp = () => (mcpP = mcpP || (window.claude && window.claude.use ? window.claude.use('mcp').catch(() => null) : Promise.resolve(null)));
async function clay(tool, input){
  const mcp = await getMcp();
  if (!mcp) throw { code: 'unavailable' };
  const r = await mcp.callTool(SERVER, tool, input, { cache: false });
  let pl = r && 'payload' in r ? r.payload : r;
  if (typeof pl === 'string') { try { pl = JSON.parse(pl); } catch(e) {} }
  return pl;
}
function walk(o, fn, d = 0){
  if (d > 14 || o == null) return;
  if (typeof o === 'string') { const s = o.trim(); if (s[0] === '{' || s[0] === '[') { try { walk(JSON.parse(s), fn, d + 1); } catch(e) {} } return; }
  if (typeof o !== 'object') return;
  fn(o); for (const v of Object.values(o)) walk(v, fn, d + 1);
}
const firstKey = (o, k) => { let hit = null; walk(o, x => { if (hit == null && typeof x[k] === 'string' && x[k]) hit = x[k]; }); return hit; };
const strings = (v, out = []) => { if (v == null) return out; if (typeof v !== 'object') out.push(String(v)); else Object.values(v).forEach(x => strings(x, out)); return out; };
function matchValue(kind, text){
  if (kind === 'email') { const m = text.match(/[A-Z0-9._%+'-]+@[A-Z0-9.-]+\.[A-Z]{2,}/i); return m && m[0]; }
  const re = /\+?\(?\d[\d\s().-]{7,}\d/g; let m;
  while ((m = re.exec(text))) { const digits = m[0].replace(/\D/g, ''); if (digits.length >= 10 && digits.length <= 15) return m[0].trim(); }
  return null;
}
// get-task-context documents each enrichment as {name, state: completed | in-progress | error, value}
function readEnrichment(payload, kind){
  const all = []; walk(payload, x => { if (typeof x.state === 'string' && ('value' in x || 'name' in x)) all.push(x); });
  const named = all.filter(e => (kind === 'email' ? /mail/i : /phone|mobile/i).test(e.name || ''));
  const list = named.length ? named : all;
  for (const e of list) if (/complete|success|done/i.test(e.state)) { const v = matchValue(kind, strings(e.value).join(' ')); if (v) return { status: 'done', value: v }; }
  if (!list.length || list.some(e => /progress|pending|queued|running/i.test(e.state))) return { status: 'pending' };
  if (list.every(e => /error|fail/i.test(e.state))) return { status: 'error', msg: firstKey(list, 'error') || firstKey(list, 'message') || 'Clay reported an error for this lookup.' };
  return { status: 'none' };
}
const normLi = u => (u || '').toLowerCase().replace(/^https?:\/\//, '').replace(/^[a-z]{2,3}\.linkedin/, 'linkedin').replace(/^www\./, '').split('?')[0].replace(/\/$/, '');
function pickEntity(payload, p){
  const ents = []; walk(payload, x => { if (x.entityId != null) ents.push(x); });
  const li = normLi(p.linkedinUrl), nm = p.name.toLowerCase();
  const hit = ents.find(e => li && normLi(e.url || e.linkedinUrl || e.linkedin_url) === li)
    || ents.find(e => String(e.profile_id || '') === String(p.clayId || '-'))
    || ents.find(e => (e.name || '').toLowerCase() === nm)
    || (ents.length === 1 ? ents[0] : null);
  return hit ? String(hit.entityId) : null;
}
async function startEmail(p){
  const dp = { dataPoints: [{ type: 'Email' }] };
  if (p.clayTaskId && p.clayEntityId) {
    try { await clay('add-contact-data-points', { taskId: p.clayTaskId, entityIds: [p.clayEntityId], ...dp }); return { taskId: p.clayTaskId, eids: [p.clayEntityId] }; }
    catch (e) { if (e.code !== 'tool_error') throw e; } // the build-time search isn't usable here: find the person again
  }
  const r = await clay('search-contacts-by-name', { contactIdentifiers: [{ contactName: p.name, companyIdentifier: C.linkedinUrl || C.domain }] });
  const taskId = firstKey(r, 'taskId'), eid = pickEntity(r, p);
  if (!taskId || !eid) throw { code: 'no_match' };
  await clay('add-contact-data-points', { taskId, entityIds: [eid], ...dp });
  return { taskId, eids: [eid] };
}
async function startPhone(p){
  const inputs = {};
  for (const [name, field] of Object.entries(ENR.phone.inputs || {})) {
    inputs[name] = field === 'email' ? (found[p.id]?.email?.value || p.email || '')
      : field === 'companyLinkedinUrl' ? (C.linkedinUrl || '') : String(p[field] ?? C[field] ?? '');
  }
  const r = await clay('run_subroutine_direct', { subroutine_id: ENR.phone.id, inputs: [inputs] });
  const taskId = firstKey(r, 'taskId');
  if (!taskId) throw { code: 'no_task' };
  return { taskId, eids: null };
}
async function poll(job, kind, key){
  const t0 = Date.now();
  while (Date.now() - t0 < 150000) {
    await new Promise(r => setTimeout(r, 4000));
    const pl = await clay('get-task-context', job.eids ? { taskId: job.taskId, entityIds: job.eids } : { taskId: job.taskId });
    const r = readEnrichment(pl, kind);
    if (r.status !== 'pending') return r;
    setBusy(key, 'running', `Waiting on Clay (${Math.round((Date.now() - t0) / 1000)}s)`);
  }
  return { status: 'timeout' };
}
function errText(e){
  switch (e && e.code) {
    case 'unavailable': case 'not_granted': case 'capability_disabled': case 'capability_removed':
      return 'Sourcing works when this page is open in claude.ai by someone with the Clay connector.';
    case 'server_not_connected': return 'Add the Clay connector in claude.ai Settings → Connectors, then try again.';
    case 'needs_reauth': return 'Reconnect Clay in claude.ai Settings → Connectors, then try again.';
    case 'selection_required': return 'You have more than one Clay connection. Choose one when claude.ai asks, then try again.';
    case 'not_in_manifest': case 'consent_required': return 'Clay access is turned off for this page. Allow Clay for this page, then try again.';
    case 'blocked_by_policy': case 'approval_required': return "Your organization's policy blocks this Clay action.";
    case 'server_unavailable': case 'upstream_error': case 'rate_limited':
      return "Clay didn't answer in time. The lookup may still have started, so check your Clay credits before trying again.";
    case 'tool_error': return 'Clay: ' + (e.message || 'the lookup failed.');
    case 'no_match': return "Clay couldn't match this person for a lookup.";
    case 'no_task': return 'Clay started the function but returned nothing to check on.';
    default: return (e && e.message) || 'Something went wrong talking to Clay.';
  }
}
function saveFound(){ store(KEY + ':contacts', found); }
function setBusy(key, phase, msg){ if (phase) busy[key] = { phase, msg }; else delete busy[key]; refreshContact(); }
async function source(id, kind){
  const p = byId[id], key = id + ':' + kind;
  setBusy(key, 'running', 'Asking Clay');
  try {
    let job = found[id]?.pending?.[kind];
    if (!job) {
      job = kind === 'email' ? await startEmail(p) : await startPhone(p);
      found[id] = found[id] || {}; found[id].pending = { ...(found[id].pending || {}), [kind]: job }; saveFound();
    }
    const r = await poll(job, kind, key);
    if (r.status !== 'timeout' && found[id]?.pending) { delete found[id].pending[kind]; }
    if (r.status === 'done') { found[id][kind] = { value: r.value, at: new Date().toISOString() }; saveFound(); setBusy(key, null); render(); announce(`${LABEL[kind]} found for ${p.name}`); }
    else if (r.status === 'none') { saveFound(); setBusy(key, 'none', `Clay found no ${kind === 'email' ? 'email' : 'mobile number'} for ${p.name}.`); }
    else if (r.status === 'error') { saveFound(); setBusy(key, 'error', r.msg); }
    else setBusy(key, 'timeout', "Clay is still working on this. Check again in a minute. That won't start a second lookup.");
  } catch (e) { setBusy(key, 'error', errText(e)); }
}
function contactHTML(id){
  const p = byId[id];
  const row = kind => {
    const key = id + ':' + kind, b = busy[key];
    const val = kind === 'email' ? (found[id]?.email?.value || p.email) : found[id]?.phone?.value;
    if (val) {
      const note = found[id]?.[kind]?.at ? `Sourced from Clay ${esc(fmt(found[id][kind].at))} · saved in this browser` : 'Returned in the original Clay search';
      return `<div><div class="crow"><span class="clabel">${LABEL[kind]}</span><span class="cval">${esc(val)}</span><button class="mini" data-copy="${esc(val)}">Copy</button></div><div class="hint" style="margin-left:64px">${note}</div></div>`;
    }
    const noun = kind === 'email' ? 'email' : 'mobile';
    let right;
    if (b?.phase === 'confirm') right = `<span class="cmsg">This lookup uses Clay credits from your workspace.</span><button class="btn primary sm" data-go="${kind}">Find ${noun}</button><button class="mini" data-cancel="${kind}">Cancel</button>`;
    else if (b?.phase === 'running') right = `<span class="cmsg" role="status"><span class="spin" aria-hidden="true"></span>${esc(b.msg)}</span>`;
    else if (b?.phase === 'timeout') right = `<span class="cmsg">${esc(b.msg)}</span><button class="mini" data-go="${kind}">Check again</button>`;
    else if (b?.phase === 'error') right = `<span class="cmsg err">${esc(b.msg)}</span><button class="mini" data-ask="${kind}">Try again</button>`;
    else if (b?.phase === 'none') right = `<span class="cmsg">${esc(b.msg)}</span>`;
    else if (found[id]?.pending?.[kind]) right = `<span class="cmsg">A lookup started earlier.</span><button class="mini" data-go="${kind}">Check for result</button>`;
    else right = `<button class="btn primary sm" data-ask="${kind}">Find ${noun} with Clay</button>`;
    return `<div class="crow"><span class="clabel">${LABEL[kind]}</span>${right}</div>`;
  };
  return KINDS.map(row).join('');
}
function wireContact(id){
  const el = $('#contact-sec'); if (!el) return;
  el.querySelectorAll('[data-ask]').forEach(b => b.onclick = () => { setBusy(id + ':' + b.dataset.ask, 'confirm'); $(`#contact-sec [data-go="${b.dataset.ask}"]`)?.focus(); });
  el.querySelectorAll('[data-cancel]').forEach(b => b.onclick = () => { setBusy(id + ':' + b.dataset.cancel, null); $(`#contact-sec [data-ask="${b.dataset.cancel}"]`)?.focus(); });
  el.querySelectorAll('[data-go]').forEach(b => b.onclick = () => source(id, b.dataset.go));
  el.querySelectorAll('[data-copy]').forEach(b => b.onclick = () => {
    const v = b.dataset.copy;
    const fallback = () => { const s = b.previousElementSibling; const r = document.createRange(); r.selectNodeContents(s); const sel = getSelection(); sel.removeAllRanges(); sel.addRange(r); announce('Selected, press Ctrl or Cmd plus C to copy'); };
    try { navigator.clipboard.writeText(v).then(() => { b.textContent = 'Copied'; announce('Copied'); }, fallback); } catch (e) { fallback(); }
  });
}
function refreshContact(){
  const el = $('#contact-sec'); if (!el || !state.sel) return;
  const had = el.contains(document.activeElement);
  el.innerHTML = contactHTML(state.sel); wireContact(state.sel);
  if (had && !el.contains(document.activeElement)) el.querySelector('button')?.focus();
}

let lastFocus = null;
const showAllReports = {};
function openDrawer(id){
  const src = byId[id]; if (!src) return;
  const p = eff(src); state.sel = id; if (!$('#drawer').classList.contains('open')) lastFocus = document.activeElement;
  const km = kidsMap();
  const mgr = mgrOf(id), inf = inferredMgr(id), srcRel = relSource(id), place = placement(id);
  const kids = km[id] || [];
  const cur = mgr || (place === 'root' ? TOP : TRAY);
  const options = [`<option value="${TRAY}" ${cur === TRAY ? 'selected' : ''}>Needs review (no manager yet)</option>`,
    `<option value="${TOP}" ${cur === TOP ? 'selected' : ''}>Top of the chart (no manager)</option>`]
    .concat(D.people.filter(o => o.id !== id && !isDescendant(id, o.id, km))
      .map(o => `<option value="${esc(o.id)}" ${o.id === mgr ? 'selected' : ''}>${esc(o.name)} · ${esc(o.title)}</option>`)).join('');
  const r = src.reportsTo;
  const relNote = srcRel === 'user'
    ? `<p class="note"><b>Set by you.</b> ${inf ? `Original inference: ${esc(byId[inf].name)}, because ${esc(lc(r.basis))}.` : r && r.confidence === 'unplaced' ? `Originally in Needs review: ${esc(lc(r.basis))}.` : 'No line was inferred originally.'}</p>`
    : srcRel === 'inferred'
      ? `<p class="note ${r.confidence === 'low' ? 'warn' : ''}"><b>Inferred, ${esc(r.confidence)} confidence.</b> ${esc(r.basis)}. Clay does not supply reporting lines, so check this before relying on it.</p>`
      : place === 'tray'
        ? `<p class="note warn"><b>Needs review.</b> ${esc(r?.basis || 'No manager was inferred')}. Pick their manager above, or drag their tile onto the manager's tile.</p>`
        : `<p class="note"><b>Top of the chart.</b> The most senior title in Clay's results.</p>`;
  const shown = showAllReports[id] ? kids : kids.slice(0, 6);
  const nContacts = (found[id]?.email?.value || p.email ? 1 : 0) + (found[id]?.phone?.value ? 1 : 0);
  const dr = $('#drawer');
  dr.innerHTML = `
    <div class="dhead"><span class="av" aria-hidden="true">${esc(initials(p.name))}</span>
      <div><h2 id="dname">${esc(p.name)}</h2><p>${esc(p.title)}</p></div>
      <button class="x" id="close" aria-label="Close details">×</button></div>
    <div class="dbody">
      <details class="dsec" open><summary>Contact <span class="badge">${nContacts ? nContacts + ' found' : 'none yet'}</span></summary>
        <div class="inner" id="contact-sec">${contactHTML(id)}</div></details>
      ${D.news ? (() => { const ns = newsFor(id); return ns.length ? `<details class="dsec" open><summary>In the news <span class="badge">${ns.length}</span></summary>
        <div class="inner dnews">${ns.map(n => `<div><a href="${esc(n.url)}" target="_blank" rel="noopener noreferrer">${esc(n.title)}</a><div class="hint">${esc(fmtDay(n.date))} · ${esc(n.source)}</div></div>`).join('')}
          <button class="more" id="news-all" style="text-align:left">See all in the news section</button></div></details>` : ''; })() : ''}
      <details class="dsec" open><summary>Reporting <span class="badge">${kids.length ? kids.length + ' direct report' + (kids.length === 1 ? '' : 's') : place === 'tray' ? 'needs review' : srcRel === 'user' ? 'set by you' : 'inferred'}</span></summary>
        <div class="inner">
          <div class="field"><label for="f-mgr">Reports to</label><select id="f-mgr">${options}</select></div>
          ${relNote}
          ${kids.length ? `<div class="hint">Direct reports</div><div class="reports">${shown.map(k => `<button data-open="${esc(k)}"><span class="av" aria-hidden="true">${esc(initials(byId[k].name))}</span><span><b>${esc(byId[k].name)}</b><span class="hint" style="display:block;margin:0">${esc(byId[k].title)}</span></span></button>`).join('')}</div>
            ${kids.length > shown.length ? `<button class="more" id="more-rep">Show all ${kids.length}</button>` : ''}` : ''}
        </div></details>
      <details class="dsec" open><summary>Classification</summary>
        <div class="inner">
          <div class="field"><label for="f-role">Buying role</label>
            <select id="f-role">${ROLES.map(x => `<option ${x === p.buyingRole ? 'selected' : ''}>${x}</option>`).join('')}</select>
            <div class="hint">Your classification. Saved in this browser only.</div></div>
          <div class="field"><label for="f-level">Org level</label>
            <select id="f-level">${LEVELS.map(l => `<option ${l === p.level ? 'selected' : ''}>${l}</option>`).join('')}</select>
            <div class="hint">A local correction. The Clay-derived value (${esc(src.level)}) is kept.</div></div>
        </div></details>
      <details class="dsec"><summary>Clay details</summary>
        <div class="inner"><dl>
          <dt>Company</dt><dd>${esc(p.companyName)}${p.companyDomain ? ` · ${esc(p.companyDomain)}` : ''}</dd>
          <dt>Function</dt><dd>${esc(p.function)} <span class="hint">(from title)</span></dd>
          <dt>In role since</dt><dd>${p.roleStartDate ? `${esc(startLabel(p.roleStartDate))} <span class="hint">(${monthsIn(p.roleStartDate)} mo; start of current title, may be a promotion)</span>` : '<span class="hint">Not returned by Clay</span>'}</dd>
          <dt>Location</dt><dd>${p.location ? esc(p.location) : '<span class="hint">Not returned by Clay</span>'}</dd>
          <dt>LinkedIn</dt><dd>${p.linkedinUrl ? `<a href="${esc(/^https?:/.test(p.linkedinUrl) ? p.linkedinUrl : 'https://' + p.linkedinUrl)}" target="_blank" rel="noopener">${esc(p.linkedinUrl.replace(/^https?:\/\/(www\.)?/,''))}</a>` : '<span class="hint">Not returned by Clay</span>'}</dd>
          <dt>Verification</dt><dd>${p.source === 'Demo' ? 'Demo fixture, not Clay data' : p.currentEmploymentVerified ? 'Current employment verified by Clay' : 'Returned by Clay current-employer search'}</dd>
          <dt>Retrieved</dt><dd>${esc(fmt(p.retrievedAt))}</dd>
        </dl></div></details>
    </div>`;
  dr.classList.add('open'); dr.setAttribute('aria-hidden','false');
  const modal = narrow(); dr.setAttribute('aria-modal', String(modal)); $('#scrim').classList.toggle('open', modal);
  document.body.classList.add('drawer-open');
  const refocus = sel => { render(); openDrawer(id); $(sel).focus(); };
  $('#f-mgr').onchange = e => { const v = e.target.value; setEdit(id, 'managerId', v); announce(v === TOP ? `${src.name} moved to the top of the chart` : v === TRAY ? `${src.name} moved to Needs review` : `${src.name} now reports to ${byId[v].name}`); refocus('#f-mgr'); };
  $('#f-role').onchange = e => { setEdit(id, 'buyingRole', e.target.value); announce('Buying role saved'); refocus('#f-role'); };
  $('#f-level').onchange = e => { setEdit(id, 'level', e.target.value); announce('Level saved'); refocus('#f-level'); };
  const more = $('#more-rep'); if (more) more.onclick = () => { showAllReports[id] = true; openDrawer(id); $('#drawer .reports button:nth-child(7)')?.focus(); };
  dr.querySelectorAll('[data-open]').forEach(b => b.onclick = () => { openDrawer(b.dataset.open); $('#close').focus(); });
  wireContact(id);
  const na = $('#news-all'); if (na) na.onclick = () => { state.newsBy = id; closeDrawer(); $('#news')?.scrollIntoView({ behavior: reduceMotion() ? 'auto' : 'smooth', block: 'start' }); };
  $('#close').onclick = closeDrawer;
  if (!dr.contains(document.activeElement)) $('#close').focus();
  render();
  if (!narrow()) { // keep the focused tile visible beside the drawer
    const ow = $('#orgwrap'), c = document.querySelector(`#orgwrap .ocard[data-id="${CSS.escape(id)}"]`);
    if (ow && c) { const o = ow.getBoundingClientRect(), r = c.getBoundingClientRect();
      const visR = Math.min(o.right, window.innerWidth - dr.offsetWidth - 16);
      if (r.left < o.left + 8 || r.right > visR - 8) ow.scrollTo({ left: ow.scrollLeft + (r.left + r.width / 2) - (o.left + (visR - o.left) / 2), behavior: reduceMotion() ? 'auto' : 'smooth' }); }
  }
}
const lc = s => s ? s.charAt(0).toLowerCase() + s.slice(1) : '';
function closeDrawer(){
  const dr = $('#drawer'); document.body.classList.remove('drawer-open'); dr.classList.remove('open'); dr.setAttribute('aria-hidden','true'); $('#scrim').classList.remove('open');
  const id = state.sel; state.sel = null; render();
  (document.querySelector(`.ocard[data-id="${CSS.escape(id || '')}"]`) || lastFocus)?.focus?.();
}
$('#scrim').onclick = closeDrawer;
document.addEventListener('keydown', e => {
  const dr = $('#drawer');
  if (!dr.classList.contains('open')) return;
  if (e.key === 'Escape') closeDrawer();
  if (e.key === 'Tab' && narrow()) { const f = [...dr.querySelectorAll('button,select,a[href],summary')].filter(x => x.offsetParent); if (!f.length) return;
    const a = f[0], z = f[f.length-1];
    if (e.shiftKey && document.activeElement === a) { e.preventDefault(); z.focus(); }
    else if (!e.shiftKey && document.activeElement === z) { e.preventDefault(); a.focus(); } }
});
render();
})();
</script>
````