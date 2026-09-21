# AI Agent Gallery — SharePoint list → hero cards (JSON view formatting)

This is the no-script path from the two options discussed for the BotFlix-style
agent gallery: a SharePoint list drives the cards natively through JSON view
formatting, so it stays live and isn't subject to the script-blocking that
forced `BOTFLIX-SINGLEFILE-SHELL-v1.09.html` into a static-build pattern.

## 1. List schema

Create a list (e.g. `AI Agent Gallery`) with these columns:

| Column | Internal name | Type | Notes |
|---|---|---|---|
| Title | `Title` | Single line of text | Built-in. Agent name, e.g. "Leiter" |
| Tagline | `Tagline` | Single line of text | One-line hook shown on the card |
| ThumbnailURL | `ThumbnailURL` | Single line of text | Direct URL to the card image (upload to a Site Assets library first, copy its URL) |
| AgentURL | `AgentURL` | Single line of text | Where the card links — the agent's hero/detail page or deploy prompt |
| Category | `Category` | Choice | e.g. "Productivity", "Research" — shown as the small label above the title |
| NNSYApproved | `NNSYApproved` | Yes/No | Drives the red "APPROVED" badge |
| Status | `Status` | Choice: `Live`, `Draft` | Draft rows are hidden from the card view without deleting them |

Keeping `ThumbnailURL`/`AgentURL` as plain text (not the "Hyperlink or Picture"
column type) is deliberate — that column type stores an object
(`{desc, url}`) and the formatting has to reference `[$Column.url]` instead
of `[$Column]`. Plain text is one less thing to get wrong on a first pass.

## 2. Apply the formatting

1. Open the list, switch to (or create) the view you want to use as the gallery.
2. Click the view options (⚙ next to the column headers, or **All Items ▾** →
   **Format current view**).
3. Switch the formatting pane to **Advanced mode**.
4. Paste the contents of `agent-gallery-view-formatting.json` into the box.
5. **Preview**, then **Save**.

Cards render as `inline-block` tiles, so they wrap into a grid automatically
as the view's width changes — no separate CSS Grid/Flexbox container needed.

## 3. What this gets you, and what it doesn't

**Live and native:**
- Adding/editing/removing a list item updates the gallery immediately —
  no rebuild step, no workbook, no script blocking (this is SharePoint's own
  renderer, not injected script — the same restriction that forced BotFlix
  into `STATIC MODE` shouldn't apply here).
- Permissions follow the list's own permissions — no separate app registration.

**Not achievable with JSON formatting alone** (these need actual script,
i.e. SPFx, which brings back the App Catalog / tenant-admin deployment
question):
- BotFlix's hover-triggered description reveal and 1.35x scale — JSON
  formatting has no `:hover` state; what you see is what's always rendered.
  This version shows the title/tagline/badge at rest instead.
- The horizontal Netflix-style scrolling carousel rows split by author —
  this is a static wrapping grid, not a carousel.
- The deploy walkthrough drawer, hover-video previews, and in-shell iframe
  routing — all script-driven, out of scope for this approach entirely.

If those specific interactions turn out to be non-negotiable, the fallback
is the static-build pattern BotFlix already uses (a scheduled job reads the
list instead of the Excel workbook and regenerates the shell) — worth
knowing going in, but only worth building if this native version proves
insufficient or also gets blocked.

## 4. Next step

Apply `agent-gallery-view-formatting.json` to a test list in the target
SharePoint site and confirm two things: (a) it renders as expected, and
(b) nothing in that tenant's security policy strips the `rowFormatter`
the way it stripped live script elsewhere in this project. Report back
either way — a block here would be useful to know before investing further
in this path.
