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

## 4. Status: `rowFormatter` blocked in the target tenant

**Tested 2026-09-21: `agent-gallery-view-formatting.json` was stripped on
save/render.** The tenant's security policy (the same "Firepit" layer
documented throughout `BOTFLIX-SINGLEFILE-SHELL-v1.09.html`'s changelog)
does not distinguish JSON view formatting from injected script — both get
treated as customization and removed. `rowFormatter` specifically is the
most script-adjacent piece of the JSON formatting surface (it replaces the
entire row template), so this isn't surprising in hindsight; a narrower
per-column formatter might survive where the full row template doesn't, but
it wasn't tested since it wouldn't get close to a hero-card layout anyway.

`agent-gallery-view-formatting.json` is kept in this repo as a record of
what was tried, not as something to paste into this tenant again.

## 5. Next test: native Gallery view (not JSON, not formatting)

SharePoint's built-in **Gallery** view type is a different mechanism
entirely — it's a core view-rendering mode picked from the UI, with no JSON
and no formatter to strip. Worth testing before falling back to the
static-build pattern, since it may survive where both live script and JSON
formatting failed:

1. Open the list → **+ Add view** (or the view switcher dropdown) → name it
   → set **View format** to **Gallery** → **Create**.
2. SharePoint auto-picks a card layout from the list's columns. Click
   **Edit tile** to control it directly:
   - Set the image field to `ThumbnailURL` (or attach real images to the
     list items and use that field instead of a URL column).
   - Set which fields show under the title (`Tagline`, `Category`, etc.).
3. Filter the view to `Status = Live` (view filter, not formatting) so
   draft rows stay hidden.

Expected ceiling: this renders as SharePoint's own fixed card chrome — no
dark theme, no red badge, no gradient overlay, no hover reveal. It will
look like a native SharePoint gallery, not BotFlix. The question this test
answers is narrower than "does it look right" — it's "does *any* card
rendering of this list survive the tenant's policy," which determines
whether a live-bound gallery is possible here at all, in any visual form.

If Gallery view also gets stripped or disabled, that's a strong signal the
tenant blocks all non-default list rendering, and the static-build fallback
(§3) becomes the only live-adjacent option left.

## 6. Native Gallery view worked well enough to expose a third option

Gallery view rendered. Its tile editor accepts a field that stores actual
HTML — a **Multiple lines of text** column set to **Enhanced rich text
(rich text with pictures, tables, and hyperlinks)**. Shown as a tile field,
its stored HTML renders inside the card. This is a different mechanism from
both prior attempts: it's stored *content*, not injected *formatting/script*,
so it isn't necessarily subject to the same policy that stripped `rowFormatter`
— worth testing, not yet confirmed.

**The tradeoff to understand going in:** a rich-text field holds static HTML
per item — it has no `[$Column]` token syntax, so it can't reference the
item's other columns at render time the way JSON formatting could. "Pulls
information from the list" therefore has to mean something *writes* the
composed HTML into this field whenever the source columns change, not that
the HTML itself is a live template. That something is a Power Automate flow.

### 6.1 Add the column

On the list, add **CardHTML** — Multiple lines of text → **Enhanced rich
text**. Add it as a field on the Gallery view's tile (Edit tile → add field
→ CardHTML).

### 6.2 Two template variants

- `hero-card-template.html` — BotFlix-style image-bleed with gradient
  overlay and corner badge. Uses `position:absolute`, which rich-text
  sanitizers commonly strip. **Test this one first.**
- `hero-card-template-fallback.html` — same content, stacked block layout,
  no absolute positioning anywhere. Use this if the primary template's
  layout collapses on save (a sure sign `position` got stripped).

Both use the same `{{PLACEHOLDER}}` tokens:

| Placeholder | Source column |
|---|---|
| `{{ThumbnailURL}}` | `ThumbnailURL` |
| `{{AgentURL}}` | `AgentURL` |
| `{{Category}}` | `Category` |
| `{{Title}}` | `Title` |
| `{{Tagline}}` | `Tagline` |
| `{{ApprovedDisplay}}` | `'block'` if `NNSYApproved` is Yes, else `'none'` |

### 6.3 Power Automate flow to populate CardHTML

1. **Create a flow**: Automated cloud flow, trigger = **When an item is
   created or modified** (SharePoint) → Site = target site, List = the
   agent gallery list.
2. **Compose** action ("GeneratedCardHTML"): paste the contents of
   `hero-card-template.html` (or the fallback) into the input box as plain
   text, then click into each `{{...}}` position and use the dynamic-content
   picker to insert the matching field from the trigger — it inserts
   `@{triggerBody()?['ColumnName']}` inline. For `{{ApprovedDisplay}}`, type
   the expression directly instead of picking a field:
   `@{if(equals(triggerBody()?['NNSYApproved'], true), 'block', 'none')}`
3. **Condition**: `equals(outputs('GeneratedCardHTML'), triggerBody()?['CardHTML'])`
   - **Yes** → Terminate (nothing changed, stop here).
   - **No** → **Update item**, set `CardHTML` = `outputs('GeneratedCardHTML')`.

The condition is the loop-prevention: updating `CardHTML` re-fires the
trigger, but on that second pass the generated HTML now equals the stored
value, so the flow terminates immediately instead of looping.

### 6.4 What to test and report back

1. Does the flow run and populate `CardHTML` without erroring?
2. Does the primary template's `position:absolute` survive, or does the
   card collapse (switch to the fallback if so)?
3. Does any inline style get silently dropped (gradient, border-radius,
   box-shadow) even if the layout itself holds?

Any one of these failing narrows the option further; all three holding
means this is the live gallery.
