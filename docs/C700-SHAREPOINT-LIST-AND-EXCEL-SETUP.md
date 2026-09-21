# Managing the C700 AI Agent Gallery: SharePoint List + Excel Coverage File

This guide sets up two companion artifacts for `C700-AI-AGENT-GALLERY.html`:

1. **A SharePoint List** — a lightweight, browsable status board the team uses day-to-day to track each agent (who owns it, what state it's in, links to its docs). Good for quick edits, filtering, and approvals without opening a spreadsheet or the HTML source.
2. **An Excel coverage workbook** — the actual authoring source for every field the gallery page needs, including the long text fields (descriptions, system prompts) a SharePoint List handles poorly. This is the file you edit when you're actually writing or updating an agent's content, and it doubles as a completeness ("coverage") tracker — at a glance, which agents are missing a thumbnail, a long description, a desk guide, etc.

**Important architectural fact that shapes everything below:** the HTML file is fully static. It does not read live from SharePoint or Excel — there's no API call, no Power Automate flow, nothing wired up at runtime. Every agent's data lives baked into the HTML in three separate places (see "How an agent's data actually reaches the page" below). The List and the workbook are *management* tools that sit upstream of the HTML; getting an edit from either of them onto the live page is a manual (or separately-scripted) sync step, not automatic. If you want that to change — a real live connection — see "Optional: making this live" at the end.

---

## How an agent's data actually reaches the page

Before setting up tracking tools, it matters to know what they're tracking. Each agent's data is duplicated across **three locations** in the HTML:

| Location | What it holds | Example |
|---|---|---|
| The gallery card `<a class="card">` markup | Thumbnail image, hover gif, card title, card blurb, whether the card is shown (`data-show="yes"/"no"`) | `<a href="#bert" data-show="yes" class="card has-media">...` |
| The `siteTitles` JS object | The short nav/tab title shown when that agent's page is open | `'#bert':'The Orchestrator [BERT]'` |
| The `agentData` JS object | Everything else — the full hero page: eyebrow, tools, status, model, author, version, links, in/out text, tagline, long description, system prompt, related agents, etc. | `'#bert':{displayTitle:'...', eyebrow:'...', ...}` |

This means adding or updating an agent today means editing all three by hand. The workbook below is laid out so that every column maps to exactly one of these three places — the "Target in HTML" column in the field reference table tells you which.

---

## Part 1 — SharePoint List (status board)

Purpose: a place for the team to see every agent's status at a glance, assign ownership, and track approvals — without needing Excel or HTML access. Keep this List *lean*: it's for tracking, not authoring. Long-form content (descriptions, system prompts) lives in the Excel workbook, not here.

### Steps to create it

1. In the target SharePoint site, go to **Site contents → New → List → Blank list**.
2. Name it **C700 Agent Gallery — Status Board** (or your preferred name).
3. Add the columns below (**Settings → List settings → Create column** for each, or add them from the list view's `+` column button).

### Columns

| Column name | Type | Notes |
|---|---|---|
| Agent Name | Single line of text (this can replace the default "Title" column) | e.g. "Bert" — matches the bracketed name in `displayTitle`, e.g. `[BERT]` |
| Hash / Slug | Single line of text | The URL fragment used in the HTML, e.g. `#bert`. Must be unique, lowercase, no spaces. |
| Display Title | Single line of text | Full card/hero title, e.g. "The Orchestrator [BERT]" |
| Status | Choice (Draft / Active / Retired) | Matches the `status` field shown as a chip on the hero page |
| Shown on Gallery | Yes/No | Matches `data-show="yes"/"no"` on the card |
| Owner / Author | Person or Group | Who maintains this agent's prompt/content |
| Model | Choice (populate with the models actually in use, e.g. Gemini 2.5 Pro) | |
| Version | Single line of text | e.g. "v2.1" |
| GenAI.mil Link | Hyperlink | Almost always `https://genai.mil/` — only diverges if an agent uses a different platform |
| Desk Guide Link | Hyperlink | Link to the agent's PDF desk guide, if one exists |
| Has Audio Overview | Yes/No | Whether this agent has the hero-page audio badge |
| Last Updated | Date | |
| Notes | Multiple lines of text | Free-form status notes, blockers, review comments |

### Suggested views

- **Board view** grouped by `Status`, for a Kanban-style overview.
- **Gallery-ready view**, filtered to `Shown on Gallery = Yes`, sorted by `Agent Name` — this is your checklist against what's actually live on the page.
- **Needs Review view**, filtered to `Status = Draft`.

### Permissions

Give edit access to whoever owns agent content day-to-day; read access site-wide is fine since nothing here is sensitive beyond what's already on the gallery page itself.

---

## Part 2 — Excel Coverage Workbook (authoring source)

Purpose: one row per agent, one column per field the HTML actually needs. This is the master content file. Every column below maps 1:1 to a field in `agentData` (or to the card markup / `siteTitles`, per the "Target in HTML" column), so filling this out completely means you have everything needed to build or update that agent's entry in the page.

### Sheet layout

One sheet, **Agents**, one row per agent (matches the 26 rows already in the gallery), header row frozen. Recommended: also add a **Coverage** sheet with a formula-driven completeness check (see below).

### Column reference (matches `agentData` field names exactly, so a future export/import script has a direct mapping)

| Column | Excel data type | HTML field | Target in HTML | Notes |
|---|---|---|---|---|
| Hash / Slug | Text | (object key) | Card, `siteTitles`, `agentData` | e.g. `#bert` — the join key across all three sheets/targets |
| Display Title | Text | `displayTitle` | Card title, `siteTitles`, `agentData.displayTitle` | e.g. "The Orchestrator [BERT]" |
| Eyebrow | Text | `eyebrow` | `agentData.eyebrow` | Small label above the hero title, e.g. "The Orchestrator" |
| Short Title | Text | `title` | `agentData.title` | Big hero name, e.g. "BERT" |
| Status | Text (data validation: Draft/Active/Retired) | `status` | `agentData.status` | |
| Shown on Gallery | Text (data validation: yes/no) | (card `data-show`) | Card markup | |
| Model | Text | `model` | `agentData.model` | |
| Author | Text | `author` | `agentData.author` | |
| Version | Text | `version` | `agentData.version` | |
| GenAI.mil / Platform URL | Hyperlink | `genaiUrl` | `agentData.genaiUrl` | Usually `https://genai.mil/` |
| Desk Guide URL | Hyperlink | `deskGuideUrl` | `agentData.deskGuideUrl` | Leave blank → renders as `null` (no Desk Guide button) |
| Card Thumbnail URL | Hyperlink | (card `default-img`) | Card markup | Still image shown at rest |
| Card Hover GIF URL | Hyperlink | (card `hover-img`) | Card markup | Animated image shown on hover/focus |
| Hero Media URL | Hyperlink | `heroMedia` | `agentData.heroMedia` | Background image/gif on the agent's own hero page (can be the same asset as the hover gif) |
| Tools (comma-separated) | Text | `tools`, `toolboxItems` | `agentData.tools` / `.toolboxItems` | e.g. `DOCX, PDF, Markdown, HTML, Copy` — same list used for both fields in practice |
| Tagline / Card Blurb | Long text | `tagline` | Card desc, `agentData.tagline` | The 1–3 sentence hook shown on card hover and at the top of the hero page |
| In Text | Long text | `inText` | `agentData.inText` | Short "what goes in" line |
| Out Text | Long text | `outText` | `agentData.outText` | Short "what comes out" line |
| Long Description (para 1, 2, 3, 4...) | Long text, one column per paragraph OR one cell with a clear paragraph delimiter (e.g. `¶` or blank line) | `longDesc[]` | `agentData.longDesc` | Full narrative description, rendered as separate `<p>` blocks |
| Footer Note | Long text | `footerNote` | `agentData.footerNote` | Small disclaimer/context line at the bottom of the hero page; leave blank → `null` |
| Has Audio Overview | Text (yes/no) | `hasAudio` | `agentData.hasAudio` | |
| Audio Title | Text | `audioTitle` | `agentData.audioTitle` | Only relevant if Has Audio = yes |
| Audio File URL | Hyperlink | (JS `AUDIO_SRC`, only used for the one audio-enabled agent today) | Router script | Flag if more than one agent needs audio — currently the code assumes a single shared audio element |
| Works With (slug list) | Text, comma-separated slugs | `worksWith[].slug` | `agentData.worksWith` | e.g. `lane, harry, meredith, faye` — pulls each related agent's own title/image/desc automatically from their own row, no need to re-enter those |
| Sample Outputs (label / URL pairs) | Long text or a separate **Sample Outputs** sheet, one row per sample, joined by Hash/Slug | `sampleOutputs[]` | `agentData.sampleOutputs` | Optional — only renders if populated |
| **Agent System Prompt (agentMarkdown)** | **Not this workbook** — see below | `agentMarkdown` | `agentData.agentMarkdown` | See note |
| Agent Markdown Note | Long text | `agentMarkdownNote` | `agentData.agentMarkdownNote` | Optional flag/caveat text shown near the system-prompt tab; leave blank → `null` |

**Why the system prompt isn't a workbook column:** `agentMarkdown` holds the entire deployable system prompt for that agent — often several thousand words, with its own Markdown formatting, code fences, and classification markings. Excel cells handle this badly (32,767-character cell limit, no real Markdown editing, painful diffing). Store each agent's system prompt as its own file instead — e.g. one `.md` file per agent in a `System Prompts/` document library or folder, named by slug (`bert.md`, `lane.md`, …) — and just put a **link** to that file in the workbook (or in the SharePoint List) rather than the text itself. This also gives you version history on the prompt text for free via SharePoint's built-in file versioning.

### Coverage sheet (completeness tracker)

Add a second sheet, **Coverage**, with one row per agent and one column per *required* field above, each cell a formula like:

```
=IF(TRIM(Agents!D2)="", "MISSING", "OK")
```

...or, more usefully, a single rollup per agent:

```
=COUNTBLANK(Agents!C2:Z2)   ' lower = more complete
```

Conditionally format that column (red/yellow/green) so it's immediately visible which agents are gallery-ready vs. still missing content — this is the direct descendant of what the original project called its "workbook," and is genuinely useful for a 26-agent catalog you're actively growing.

---

## Keeping the HTML in sync

Today, this is a manual step: whenever the workbook changes, copy the relevant fields into the three HTML locations (card markup, `siteTitles`, `agentData`) by hand, or hand me the updated workbook and I'll do the sync as an editing pass. For a handful of changes at a time this is fine; if you're adding/updating agents frequently, it's worth asking me to write a small build script (Node or Python) that reads the workbook directly and regenerates those three sections of the HTML — similar to what earlier notes in this file's own changelog describe the original project doing with a Python-based sync toolkit. Say the word if you want that scripted.

---

## Hyperlinks / addresses I still need from you

To finish wiring `C700-AI-AGENT-GALLERY.html` for a real SharePoint embed, I need:

1. **The real SharePoint site URL** — every asset link in the file currently reads the placeholder `[C700-SHAREPOINT-SITE-URL]` (this replaced the old `https://flankspeed.sharepoint-mil.us/sites/NAVSEA_NNSY_BotFlix` base). Once you've stood up the site and asset library (see Part 1/2 above for suggested structure — `SiteAssets/Visual Assets/AgentGifs`, `.../AgentStills`, `.../Icons`, `.../StaticVisuals`; `Agent Assets/Agent Desk Guides`, `.../Agent Media`), send me the site's base URL and I'll do a single find-and-replace pass to wire every asset link to it.
2. Confirm whether you want the **26 existing agents' real media assets** (gifs, photos, desk guides, the audio file) re-uploaded to that new library under the *same* filenames as before, or renamed/reorganized — that changes whether the placeholder swap is a simple base-URL replace or needs per-file updates too.

Everything else needed to embed the page (the `genai.mil` link, footer text, badge label) is already filled in per your earlier direction.
