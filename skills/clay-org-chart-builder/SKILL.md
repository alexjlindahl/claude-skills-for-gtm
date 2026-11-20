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

1. Copy this skill's `scripts/orgatlas.py` and `assets/orgchart_template.html` into a working folder, keeping that relative layout (the script finds the template at `../assets/`), and run the commands below from that folder.
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
