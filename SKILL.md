---
name: elementor-mcp
description: >-
  Use this skill whenever you are working with an Elementor or Elementor Pro
  WordPress site via EMCP Tools (formerly "Elementor MCP") — recognizable by
  `mcp__*__emcp-tools-*` tools (plugin v3+) or legacy `mcp__*__elementor-mcp-*`
  tools (pre-3.0) being available in the session, or by the user asking to
  create, edit, refactor, or style a page on a WordPress site that uses
  Elementor (often with the Hello Elementor theme). Teaches the fluent patterns
  for building pages with native Elementor widgets via the catalog flow
  (`list-widgets` → `get-widget-schema` → `add-free-widget`) instead of falling
  back to HTML widgets; schema-first styling via `update-element` and
  `batch-update` so styling lands in Elementor's native data model and remains
  editable from the panel UI; the v3 disabled-by-default tool gate; the global
  kit / `__globals__` strategy for brand consistency; common traps like
  `content_width: "boxed"` collapsing flex layouts; the safe-svg upload-roles
  gate; and the agent-browser visual-review loop. Trigger even when the user
  doesn't say "EMCP" or "Elementor MCP" explicitly — if EMCP tools are
  available in the session, this skill applies to any page-building,
  page-editing, or page-refactoring task on that site. Also trigger when the
  user mentions building a landing page, sales page, one-sheet, or
  pitch-deck-style page on a WordPress site that uses Elementor.
---

# Elementor MCP — fluent page building

Most pain when building Elementor pages via MCP comes from two mistakes: dropping into HTML widgets the moment something doesn't style as expected, and guessing setting keys — Elementor silently stores any key you write and silently ignores the wrong ones. The patterns below let you build pages that are visually faithful AND remain fully editable from the Elementor panel UI — which is the whole point of using Elementor.

## Plugin versions — this document targets EMCP Tools v3+

Tool names below use the v3 prefix (`emcp-tools-<tool>`, shortened to bare tool names in prose). Check which prefix is live in the session:

- **`mcp__*__emcp-tools-*`** → plugin v3.0+; everything below applies as written. Since v3.0 the plugin also works without Elementor, so EMCP tools being present doesn't prove Elementor is active — confirm with `wp plugin list` before applying these patterns.
- **`mcp__*__elementor-mcp-*`** → legacy plugin (pre-3.0). Recommend updating to current (in-dashboard updater since v3.1, GitHub release zip, or `wp plugin update`). After updating, **every AI client must reconnect** — the server route changed from `/wp-json/mcp/elementor-mcp-server` to `/wp-json/mcp/emcp-tools-server` (WP-CLI `--server=emcp-tools-server`); regenerate `.mcp.json` from EMCP Tools → Connection and restart the session. If you must build on a legacy session, three deltas apply:
  1. The tool prefix is `elementor-mcp-<tool>`, and no tools are gated off by default.
  2. Widgets are added with per-widget tools (`add-heading`, `add-image`, …) whose validators **reject styling args** — content goes in the `add-*` call, styling in a follow-up `update-element` (the legacy "two-step pattern"). In v3 this split is unnecessary.
  3. `create-page` does not set `_elementor_edit_mode` — run `wp post meta update <id> _elementor_edit_mode builder` after creating any page you intend to build with Elementor, or the page renders with the block editor and your content is invisible.

The Elementor settings-model knowledge below (setting keys, flex traps, kit safety, dynamic tags) is plugin-version-independent. Where a live tool's behavior diverges from this document, trust the live schema (`get-widget-schema`, `get-container-schema`) — and flag the divergence so the skill can be updated.

## The v3 tool gate — 39 tools ship disabled by default

v3.1 disables 39 abilities out of the box (stored in the `emcp_tools_disabled_tools` wp option) — including six this skill relies on: **`add-custom-css`, `set-dynamic-tag`, `list-dynamic-tags`, `add-pro-widget`, `create-theme-template`, `set-template-conditions`**. A disabled tool is simply absent from the session's tool list — no error, no hint. Before concluding a capability doesn't exist, check the gate:

```bash
wp option get emcp_tools_disabled_tools --format=json
```

Enable tools from the EMCP Tools settings screen in wp-admin, or via read-modify-write on the option (takes effect immediately — reconnect the MCP server if the session's tool list doesn't refresh):

```bash
wp eval '
  $d = get_option("emcp_tools_disabled_tools", []);
  $enable = ["emcp-tools/add-custom-css","emcp-tools/set-dynamic-tag","emcp-tools/list-dynamic-tags",
             "emcp-tools/add-pro-widget","emcp-tools/create-theme-template","emcp-tools/set-template-conditions"];
  update_option("emcp_tools_disabled_tools", array_values(array_diff($d, $enable)));
'
```

Propose-and-confirm before enabling anything beyond those six — the same list gates genuinely destructive tools (file writes, DB writes, plugin/theme/user management) that should stay off unless the user explicitly wants them.

## Onboarding a new Elementor site

If EMCP tools (`mcp__*__emcp-tools-*`, or legacy `mcp__*__elementor-mcp-*`) are already available in this session, the site is wired up — skip to [Workflow](#workflow).

If they're not, the user is starting fresh and you need to get the MCP plumbing in place. **See [`references/onboarding.md`](references/onboarding.md)** for the full flow:

- **Detect framework** (Standard WP vs Bedrock) — install paths differ
- **Install the EMCP Tools plugin** — single plugin; it bundles the WordPress MCP Adapter (since v1.7.4). Standard WP and Bedrock paths both documented
- **Verify servers registered** with `wp mcp-adapter list`
- **Write `.mcp.json`** — STDIO template (preferred for local sites) and HTTP template (for cloud/remote)
- **First-session checklist** — plugins active, safe-svg role-gate open, kit ID identified, agent-browser viewport sized
- **Memory templates** — three starter files to give every future session on this site a fast cold-start (project context, transport rationale, MEMORY.md index), plus guidance on project-level vs user-level memory placement

One-line framework detection you can run at the project root:

```bash
test -f composer.json && grep -q '"roots/wordpress"' composer.json && echo BEDROCK || echo STANDARD
```

## Workflow

1. **Read the source artifact first.** PDF, Figma, sketch, brief, or reference URL — whatever's authoritative for content and design. Don't guess. **If a reference URL is provided, you MUST open it in `agent-browser` and use `snapshot` + `eval` to extract exact content and computed design tokens before touching Elementor.** Screenshots are for visual QA only — `eval` + `getComputedStyle` gives exact values (colors, font sizes, weights, letter-spacing) while screenshots give "looks dark navy-ish." See [`references/reference-site-analysis.md`](references/reference-site-analysis.md).
2. **Get the site's global kit.** Call `emcp-tools-get-global-settings` before placing the first container. It returns Primary/Secondary/Accent/Text colors and H1/H2/body typography — the source of truth for brand. Use these via `__globals__` references rather than redefining colors per element, so kit edits cascade. **If this is a fresh design setup**, the system slots (Primary/Secondary/Text/Accent for both colors and typography) will still be on Elementor defaults (Roboto, `#6EC1E4`, etc.) — configure those alongside any custom presets before building. See **[`references/site-settings.md`](references/site-settings.md)** for the full two-layer setup pattern.
3. **Check the kit's baseline CSS.** Inspect `settings.custom_css` from the global settings response — the key may be absent entirely on a fresh kit; treat absent the same as no marker. If the string `skill-baseline: applied` is absent (and `skill-baseline: opt-out` is also absent), propose adding the baseline rules before building. Full rules, application recipe, and opt-out flow in **[`references/kit-css.md`](references/kit-css.md)**.
4. **Narrate the layout before any build call.** Decompose into sections → containers → widgets → settings in prose. Catches design mistakes before they're 20 `add-*` calls deep, and turns the eventual implementation into a transcription rather than a design exercise.
5. **Create the page as draft first.** `emcp-tools-create-page` with `status: "draft"`. Pick the right page template (see [Page template choice](#page-template-choice)). v3 sets `_elementor_edit_mode: builder` automatically and returns `edit_url`/`preview_url` in the response. (Legacy plugins don't set it — see the legacy deltas above.)
6. **Build sections incrementally.** Container → its children → next container. Don't try to one-shot the whole page through `build-page` — small steps make iteration cheap. **Set `_title` on every top-level section container** at creation time (e.g. `"_title": "Hero — Section"`). It costs nothing — just another key in the `add-container` call — and turns the Elementor Navigator from an unlabeled "Container / Container / Container" list into a readable page map.
7. **Publish, then visual review.** Use `agent-browser` (see [Visual review](#visual-review-with-agent-browser)). Compare against the source for both rendering glitches AND content fidelity. Iterate per-section using `update-element`, `update-container`, or `add-custom-css`.

## Adding widgets — the catalog flow

v3 replaced the old per-widget `add-*` tools with a three-step catalog: **discover → inspect → act.**

1. **`list-widgets`** — the full widget catalog, filterable by `tier` (free/pro), `category`, and `search`.
2. **`get-widget-schema`** — a widget type's curated params, **including the full styling surface** (`title_color`, `align`, the `typography_*` family, etc.). `types: [...]` batches several widgets in one call; `full: true` returns the raw control schema when a key isn't in the curated set.
3. **`add-free-widget`** — or `add-pro-widget` for Pro widgets (disabled by default, see [the tool gate](#the-v3-tool-gate--39-tools-ship-disabled-by-default)) — with `widget_type` plus settings. **Any valid Elementor control passes through at add time**, so content and styling land in one call.

**Example — accent-blue centered H2 heading, one call:**

```
add-free-widget(post_id=1537, parent_id="abc123", widget_type="heading", settings={
  "title": "The Next Evolution",
  "header_size": "h2",
  "align": "center",
  "title_color": "#366FFE",
  "typography_typography": "custom",
  "typography_font_family": "Helvetica",
  "typography_font_weight": "700",
  "typography_font_size": {"unit": "px", "size": 56, "sizes": []},
  "typography_font_size_mobile": {"unit": "px", "size": 34, "sizes": []},
  "typography_line_height": {"unit": "em", "size": 1.15, "sizes": []},
  "_margin": {"top": "24", "right": "0", "bottom": "16", "left": "0", "unit": "px", "isLinked": false}
})
→ returns element_id "64bfc7a"
```

For iteration after the add, `update-element` is the universal workhorse: a partial merge into any element's settings (widgets AND containers), accepting the full long-tail control surface. `update-widget` (new in v3) is identical but widgets-only — it errors helpfully if the ID turns out to be a container; either is fine for widgets.

**The discipline that matters is schema-first, because nothing validates key names.** `add-free-widget` and `update-element` silently store any key you invent — a typo'd or guessed key is accepted and does nothing. Check `get-widget-schema` (or `get-element-settings` on a known-good element) before writing keys you haven't used before.

**Don't confuse the atomic family with classic widgets.** `add-atomic-heading`, the other `add-atomic-*` tools, `add-div-block`, and `add-flexbox` target Elementor v4 *atomic* widgets — a different element system. For standard Elementor / Elementor Pro builds, `add-free-widget` / `add-pro-widget` + `add-container` are the right tools.

**Why native settings matter:** styling stored in element settings shows up in the Elementor Style tab and is editable from the panel UI. Styling stored in Custom CSS blocks is not. The user shouldn't have to dig into code to change a color.

## Choosing widgets — favor native over HTML

HTML widgets defeat Elementor's value proposition. Once content is in an `html` widget, the editor user can't drag, recolor, or reorder pieces from the panel — they're stuck with raw markup.

Use the table below to pick the right widget. Reach for the `html` widget only when the design truly needs markup with no native equivalent (embeds, specific semantic HTML, etc.).

| Design pattern | Native expression |
|---|---|
| Heading + paragraph | `heading` + `text-editor` widgets in a container |
| Icon + title + description card | column container with image (or icon) + heading + text-editor |
| Feature list with bullets/icons | `icon-list` widget |
| Call-to-action button | `button` widget |
| Horizontal divider | `divider` widget |
| Gradient or solid panel with content | container with `background_background: "gradient"` or `"classic"` |
| Single circular icon badge | image widget styled directly (see [`references/layout-patterns.md`](references/layout-patterns.md#image-as-badge)) |
| Counter/stats display | `counter` widget |
| Testimonial block | `testimonial` widget |

**Decomposition principle.** When in doubt, prefer more widgets over fewer. Three native widgets (icon, title, description) in a column container beat one icon-box widget when the design needs custom styling, and beat one HTML widget almost always. Each widget is its own draggable, content-editable unit in the panel.

## Container gap — use `flex_gap`, not `gap`

When setting the spacing between a container's flex children, always use `flex_gap` in `update-element` calls — not `gap`. Elementor's CSS generator reads `flex_gap` to emit the container's gap CSS vars (`--gap` / `--row-gap` on Elementor 4.x; `--widgets-spacing-row/column` on older versions — the kit default flows through `--widgets-spacing`). The `gap` key is accepted and stored in the element data, but it does not drive CSS output and will appear to have no effect on the frontend.

```
# Wrong — stored but ignored by CSS generator:
update-element(element_id="<col-container>", settings={
  "gap": {"column": "0", "row": "32", "unit": "px", "isLinked": false}
})

# Correct — Elementor panel writes this; CSS generator reads this:
update-element(element_id="<col-container>", settings={
  "flex_gap": {"column": "0", "row": "32", "unit": "px", "isLinked": false}
})
```

Symptom of using the wrong key: the page still shows the kit's global default spacing (typically 20px) regardless of the value you wrote.

## Container padding — use `padding`, not `_padding`

The same silent-store problem applies to container padding. Use the `padding` key (no underscore) — it generates the `--padding-top/bottom/left/right` CSS custom properties that the container's stylesheet actually reads. `_padding` is accepted and stored in the element data, but is **silently ignored** for rendered output.

```
# Wrong — stored but never rendered:
update-element(element_id="<container>", settings={
  "_padding": {"top": "56", "right": "88", "bottom": "32", "left": "88", "unit": "px", "isLinked": false}
})

# Correct — generates --padding-top etc. CSS vars:
update-element(element_id="<container>", settings={
  "padding": {"top": "56", "right": "88", "bottom": "32", "left": "88", "unit": "px", "isLinked": false}
})
```

Symptom: container padding appears stuck at ~10px (Elementor's default) regardless of what value you write.

## Container width on flex children — the boxed trap

The single most common layout bug: a 3-column flex row that should sit horizontal collapses to a vertical stack.

**Cause:** child containers default to `content_width: "boxed"` (inherited from common usage), which adds the `e-con-boxed` class. Elementor's CSS forces `e-con-boxed` to 100% width regardless of the container's own `width` setting.

**Fix:** flex children that have explicit widths should be `content_width: "full"`. Their `width` setting (e.g. 340px) is then respected and they lay out as expected.

**Important: fixing children alone is not enough if the outer wrapper is also boxed.** If the outer wrapper container is itself `content_width: "boxed"`, it constrains the entire subtree regardless of what you set on the children. When fixing the children doesn't unstick the layout, check the grandparent wrapper too — or rebuild the section from scratch as a clean top-level flex-row container with `content_width: "full"` direct children. A flat structure (one row container → N full-width children) is always more reliable than a deeply nested boxed wrapper hierarchy.

```
add-container(parent_id="<flex-row-parent>", settings={
  "content_width": "full",            # ← critical for flex children
  "width": {"unit": "px", "size": 340, "sizes": []},
  "flex_direction": "column",
  "flex_align_items": "center",
  ...
})
```

The parent flex row itself can stay `content_width: "boxed"` — that's fine for the row's outer wrapper. Only the flex CHILDREN need `"full"`.

## Flex sizing — pair `_flex_grow` with `_flex_size: "none"`

When a row flex container has one child that should fill remaining space (e.g. a content column next to a fixed-width badge), giving that child `_flex_grow: "1"` is only half the fix. Elementor's default flex behavior on the *other* children is to shrink — so the badge collapses to zero width when its sibling grows.

**Always set both sides:**
- The growing child: `_flex_size: "grow"` + `_flex_grow: "1"`
- The fixed-size sibling(s): `_flex_size: "none"`

```
# Badge image — fixed size, don't shrink:
update-element(element_id="<badge-img-id>", settings={
  "_element_custom_width": {"unit": "px", "size": 80, "sizes": []},
  "_flex_size": "none",
  ...
})

# Content column — grow into remaining space:
update-container(element_id="<content-col-id>", settings={
  "_flex_size": "grow",
  "_flex_grow": "1"
})
```

Symptom of forgetting the `"none"`: the fixed-size sibling visually disappears or becomes a sliver.

## Grid containers — always set both axes explicitly

When creating a `container_type: "grid"` container, always pass **both** `grid_columns_grid` AND `grid_rows_grid` in the same call. If you set only the columns, Elementor infers the row count and defaults to 2 rows — leaving an empty second row that renders as extra whitespace.

```
add-container(parent_id="<parent>", settings={
  "container_type": "grid",
  "grid_columns_grid": "repeat(3, 1fr)",
  "grid_rows_grid": {"unit": "fr", "size": 1, "sizes": []},   # ← always set this explicitly
  ...
})
```

> ⚠️ `grid_rows_grid` is an **object** (like other dimension controls), NOT a string. Its `size` field is the row count; `unit` is the track size unit. Passing a string like `"1fr"` is silently ignored and the default of 2 rows stays. `grid_columns_grid` is the exception — it accepts a full CSS string like `"repeat(3, 1fr)"`. The two controls have different formats.

**Post-build verification eval:** after placing any grid container, confirm both axes:

```bash
agent-browser eval "
const el = document.querySelector('.elementor-element-<element_id>');
const g = el.querySelector(':scope > .e-con-inner') || el;   // boxed containers style the inner node
const s = getComputedStyle(g);
JSON.stringify({ cols: s.gridTemplateColumns, rows: s.gridTemplateRows, children: g.childElementCount })
"
```

If `rows` shows two track sizes (`183px 183px`) instead of one, fix via `update-container` with `{"grid_rows_grid": {"unit": "fr", "size": 1, "sizes": []}}`. Screenshots cannot catch this — the empty row looks identical to section padding.

## Column containers default to `flex_align_items: "center"` — set it explicitly

A flex column container created without an explicit `flex_align_items` gets `"center"` injected by Elementor. That centers each child widget *as a block* within the column — which makes any `align: "left"` setting on a heading or text-editor inside it look like it's doing nothing.

**Always set `flex_align_items: "stretch"` on column containers whose children should fill the column's width.**

```
add-container(parent_id="<parent>", settings={
  "flex_direction": "column",
  "flex_align_items": "stretch",   # ← otherwise Elementor injects "center"
  ...
})
```

Symptom: a heading or text-editor inside a flex-column container appears horizontally centered no matter how many times you set `align: "left"` on it. The fix is on the container, not the widget.

## Custom CSS — fallback only

`add-custom-css` is the escape hatch, not the workhorse. (It's also **disabled by default in v3.1** — see [the tool gate](#the-v3-tool-gate--39-tools-ship-disabled-by-default).) Use it only when the styling has no native control:

- `<strong>`, `<em>`, `<a>` styling inside heading/text content
- `white-space: nowrap`
- Pseudo-elements (`::before`, `::after`)
- Complex selectors (`& + &`, `:nth-child`, etc.)
- Mix-blend modes on non-image elements
- CSS Grid declarations (Elementor's grid container support is limited)

If a styling decision can plausibly be expressed via `title_color`, `text_color`, `typography_*`, `_background_*`, `_border_*`, `_padding`, `_margin`, `_element_width`, `_element_custom_width`, `_z_index`, etc., it goes in `update-element` instead.

**`replace: true`** on `add-custom-css` overwrites the element's existing CSS block (defaults to append). Use it when shrinking a CSS block after moving styles to native controls.

## Dynamic tags — outputting live data

Use `emcp-tools-set-dynamic-tag` whenever a widget field should output a computed or context-aware value rather than hardcoded text. This keeps content editable from the panel and avoids manual updates when data changes. Both `set-dynamic-tag` and `list-dynamic-tags` are **disabled by default in v3.1** — enable them first via [the tool gate](#the-v3-tool-gate--39-tools-ship-disabled-by-default).

```
set-dynamic-tag(post_id=<id>, element_id="<widget>",
  setting_key="<field>",   # e.g. "title", "url", "image"
  tag_name="<tag>",        # e.g. "current-date-time", "post-title", "site-title", "acf-text"
  tag_settings={...}       # tag-specific options
)
```

### Copyright year — sitewide footer or any page footer

Use a `heading` widget with `header_size: "div"` (no semantic heading tag) and the `current-date-time` dynamic tag. The year auto-updates every year with zero maintenance, and all surrounding text stays panel-editable.

```
# Step 1 — add the heading (the styling below could ride along in this call too)
add-free-widget(post_id=<id>, parent_id="<footer-container>", widget_type="heading", settings={
  "title": "placeholder",
  "header_size": "div",
  "align": "center"
})
→ element_id "abc1234"

# Step 2 — style and set dynamic tag (run in parallel)
update-element(post_id=<id>, element_id="abc1234", settings={
  "typography_typography": "custom",
  "typography_font_family": "Inter",
  "typography_font_size": {"unit": "px", "size": 11, "sizes": []},
  "typography_text_transform": "uppercase",
  "typography_letter_spacing": {"unit": "em", "size": 0.14, "sizes": []},
  "title_color": "rgba(255,255,255,0.2)"
})

set-dynamic-tag(post_id=<id>, element_id="abc1234",
  setting_key="title",
  tag_name="current-date-time",
  tag_settings={
    "date_format": "custom",
    "custom_format": "Y",
    "before": "© ",
    "after": " [Site Name]. All rights reserved."
  }
)
```

Output: **© 2025 [Site Name]. All rights reserved.**

Replace `[Site Name]` with the actual site name. The `before`/`after` fields wrap the dynamic year output with plain text; HTML entities like `&copy;` also work.

### ACF field output

See the ACF dynamic tag anti-pattern row below for the correct `tag_name` (`"acf-text"`) and `key` format (`"field_key:field_name"`).

## Kit-level CSS — safety rule and baseline check

> ⚠️ **NEVER call `emcp-tools-update-page-settings` on the kit post.** It does a **full replace** on the kit's `_elementor_page_settings` meta — colors, typography, globals, all wiped. The only safe generic kit write is `wp eval` read-modify-write. **No exception.**

**One safe native path exists in v3 — for CUSTOM palette entries only.** `update-global-colors` and `update-global-typography` genuinely merge: a new item is appended to `custom_colors`/`custom_typography`, an existing custom `_id` is updated in place, and every other kit key is untouched (`update-global-typography` also auto-injects the required `typography_typography: "custom"`). **They cannot write the system slots:** passing `_id: "primary"` does NOT update `system_colors` — it appends a duplicate "Primary" entry into the custom palette, leaving two entries with colliding `_id`s and ambiguous `__globals__` resolution. System slots (Primary/Secondary/Text/Accent) still require the `wp eval` path.

The kit ships with a small baseline of defensive CSS overrides (paragraph margin resets, focus-ring accessibility fix). Check for the `skill-baseline: applied` marker in `settings.custom_css` at workflow step 3 (the key may be absent on a fresh kit). Full baseline rules, application recipe, corruption recovery, and opt-out flow: **[`references/kit-css.md`](references/kit-css.md)**.

## Visual review with agent-browser

```bash
agent-browser set viewport 1280 800              # ALWAYS do this first
agent-browser open https://<site>/<slug>/
agent-browser screenshot /tmp/page.png
```

Then `Read` the screenshot.

**The default agent-browser viewport is narrow.** Without an explicit `set viewport`, screenshots come out at mobile-width and make every flex row look broken. Always size to desktop first unless you're specifically testing the mobile breakpoint.

`agent-browser snapshot` gives a full semantic text outline of the page — every heading, paragraph, link text, and nav item, with `@eN` refs you can target with `agent-browser click @e3` etc. This is more useful than a screenshot for reading content.

`agent-browser eval "<js>"` runs JavaScript in the live browser context and returns the result — use it to read computed styles, query DOM structure, or extract design tokens. This is the primary tool for *understanding* a reference site.

**Screenshots are for visual QA of what you built. eval+snapshot are for understanding what to build.** See **[`references/reference-site-analysis.md`](references/reference-site-analysis.md)** for the full reference-site analysis workflow.

After each visual-review pass, decide:
- **Rendering glitch?** → fix the offending element via `update-element` or `update-container`.
- **Content gap vs source?** → add the missing piece.
- **Both look right?** → mark this section done, move on.

## Diagnostic agent-browser eval recipes

When a screenshot looks wrong, screenshots themselves don't tell you *which CSS rule* is wrong. Six `agent-browser eval` recipes in **[`references/diagnostics.md`](references/diagnostics.md)** turn "why is this broken" investigations from a half-hour exercise into a 30-second answer:

- **Selector match validation** — `document.querySelectorAll('selector').length` — verify every kit-CSS selector matches real markup before committing it
- **Computed color** of an element, with interpretation table (`rgb(122,122,122)` = kit corrupted signal)
- **Elementor global variable resolution** (`--e-global-color-*`)
- **Cascade-winner finder** — which stylesheet rule is setting this property?
- **CSS-variable definition finder** — where across the page is `--e-global-color-text` defined?
- **Full-style sanity blob** — one call, all typography/spacing values for a widget

The **selector-match check** is the most important — never write a CSS rule to a kit's `custom_css` without confirming it matches at least one element on a real page. A `0` return on a page with the target widget type means the selector is wrong.

**Measure the right node:** on `e-con-boxed` containers (Elementor 4.x), flex/grid/gap/align/padding land on the child `.e-con-inner`, NOT the outer `.elementor-element-<id>` node — reading the outer node reports `align-items: normal`, `row-gap: normal`, and wrong grid templates. Query `el.querySelector(':scope > .e-con-inner') || el` in every container eval.

## Page template choice

| Use case | Setting |
|---|---|
| Standalone landing page (no theme header/footer) | `template: "elementor_canvas"` |
| Full-width within site chrome | `template: "elementor_header_footer"` |
| Content page integrated with site nav | omit or `template: "default"` |

Set via `emcp-tools-update-page-settings` after page creation. Use canvas for one-sheets, pitch decks, sales pages; default for content/blog pages.

## Maintenance Mode / Coming Soon page

Elementor Pro's Maintenance Mode intercepts all front-end requests and serves a single template. This is **not** a regular WordPress page — it's an `elementor_library` post.

**Full setup reference:** [`references/maintenance-mode.md`](references/maintenance-mode.md)

### Key facts

**Wrong tool:** `emcp-tools-create-page` → creates a `page` post type; won't appear in Elementor's template picker and causes PHP warnings in the maintenance mode render.

**Right tool:** `emcp-tools-create-theme-template` with `template_type: "page"` (**disabled by default in v3.1** — enable via [the tool gate](#the-v3-tool-gate--39-tools-ship-disabled-by-default)). After creation:
```bash
wp post meta update <id> _wp_page_template elementor_canvas
wp post meta update <id> _elementor_edit_mode builder
wp post update <id> --post_status=publish
```

**Activate maintenance mode** via three WP options:
```bash
wp option update elementor_maintenance_mode_mode        'coming_soon'   # or 'maintenance' for 503
wp option update elementor_maintenance_mode_template_id '<post_id>'
wp option update elementor_maintenance_mode_exclude_mode 'logged_in'    # logged-in users see the real site
```

**Deactivate** (without losing settings): `wp option update elementor_maintenance_mode_mode ''`

For the footer copyright line on the maintenance page, use the `current-date-time` dynamic tag — see [Dynamic tags — outputting live data](#dynamic-tags--outputting-live-data).

## Element ID tracking

Every `add-*` call returns an element_id (7-character random string). You'll need these for `update-element`, `add-custom-css`, `remove-element`, `find-element`. They're easy to lose track of — keep a running map:

```
Hero container         2e6d951
  Logomark image       78dfdf2
  Wordmark heading     29cd8b9
Tagline (h6)           24de3db
Accent H1 (h2)         64bfc7a
Subhead text-editor    f3eda9f
```

If you lose an ID, recover it with `emcp-tools-get-page-structure` or `emcp-tools-find-element`.

## Element ops — batch, duplicate, move (v3)

- **`batch-update`** — an array of `{element_id, settings}` merges applied in ONE save: one revision, one CSS regen, per-op failure reporting. Prefer it over looping `update-element` whenever you're styling more than one element.
- **`duplicate-element`** — deep-copies an element (children included, fresh IDs), placed immediately after the original. Card grids: build ONE card completely, duplicate it N−1 times, then `batch-update` the texts.
- **`move-element`** / **`reorder-elements`** — re-parent an element (`target_parent_id`; `""` for top level; `position: -1` appends) / reorder a container's children by passing the full ordered array of direct-child IDs. Restructure without rebuilding — see the HTML-refactoring recipe in [`references/layout-patterns.md`](references/layout-patterns.md).

## Form widget — native styling

The Pro form widget has extensive native style controls. Use them before reaching for `add-custom-css`. Full reference, DOM-inspection workflow, atomic form limitations, and side-by-side field layout: **[`references/form-widget.md`](references/form-widget.md)**.

Quick control map:
- **Labels**: `label_color`, `label_spacing` (label→field gap), `label_typography_*` (includes `label_typography_text_transform`)
- **Fields**: `field_border_border: "solid"` (required to unlock), `field_border_color`, `field_border_width`, `field_border_radius`, `field_background_color`, `field_text_color`, `field_typography_*`
- **Sizing**: `input_size` (xs/sm/md/lg/xl controls field height/padding), `row_gap`, `column_gap`
- **Button**: `button_text_color`, `button_typography_*` (includes `button_typography_text_transform`, `button_border_radius`), hover variants
- **Widget card (Advanced tab)**: `_background_background: "classic"`, `_background_color`, `_border_border: "solid"`, `_border_color`, `_border_width`, `_padding` — apply directly to the form widget to give it a card treatment without an extra container

> ⚠️ `field_border_border: "solid"` must be set alongside `field_border_color`. Omitting the border type causes Elementor to skip CSS output entirely — the color alone has no effect.

## Anti-patterns to avoid

| Anti-pattern | What to do instead |
|---|---|
| Falling back to `add-html` because a styling key was rejected or seemed unsupported | In v3, `add-free-widget` passes any valid control through at add time; iterate with `update-element`. On legacy plugins, styling goes in a follow-up `update-element` after the `add-*` call. Never HTML. |
| Treating a missing `emcp-tools-*` tool as a missing capability | Six skill-critical tools ship disabled by default in v3.1 (`add-custom-css`, `set-dynamic-tag`, `list-dynamic-tags`, `add-pro-widget`, `create-theme-template`, `set-template-conditions`). Check the `emcp_tools_disabled_tools` option — see [the tool gate](#the-v3-tool-gate--39-tools-ship-disabled-by-default). |
| Using `add-atomic-*` / `add-div-block` / `add-flexbox` for a classic widget build | Those target Elementor v4 *atomic* widgets — a different element system. Classic builds use `add-free-widget` / `add-pro-widget` + `add-container`. |
| Looping `update-element` to style multiple elements | Use `batch-update` — all merges in one save: one revision, less CSS churn, per-op failure reporting. |
| Writing a system color slot via `update-global-colors` | Passing a system `_id` (e.g. `"primary"`) appends a duplicate entry into `custom_colors` instead of updating `system_colors` — colliding `_id`s, ambiguous `__globals__` resolution. Custom palette entries: safe (it merges). System slots: `wp eval` read-modify-write only. |
| Reading computed flex/grid/gap/padding from the outer `.elementor-element` node of a boxed container | Elementor 4.x applies them to the child `.e-con-inner`. Query `:scope > .e-con-inner` (falling back to the element itself for full-width containers), or every eval reports defaults. |
| Skipping `get-global-settings` | Read the kit first; reference it via `__globals__` |
| Defining brand colors per-element | Use `__globals__` references: `"title_color__globals__": "globals/colors?id=accent"` |
| Re-uploading images already in Media Library | Check `wp media list` first |
| Wrapping every icon in its own badge container | Style the image widget itself — see [`references/layout-patterns.md`](references/layout-patterns.md#image-as-badge) |
| Building a whole section in one `build-page` call | Build incrementally; iterate per-section |
| Forgetting `agent-browser set viewport` | Always set to 1280×800 (or wider) before screenshotting desktop layouts |
| Putting layout-affecting CSS in custom-css blocks | If there's a native control (padding, margin, flex), use it via update-element |
| Treating `add-custom-css` as primary, `update-element` as fallback | Inverse: native first, CSS only when no native control exists |
| Giving a flex child `_flex_grow: "1"` without setting `_flex_size: "none"` on its fixed sibling | Pair them — see "Flex sizing" above. Symptom: the fixed sibling visually disappears. |
| Creating a flex-column container without explicit `flex_align_items` | Set `flex_align_items: "stretch"` explicitly — Elementor injects `"center"` otherwise, which silently breaks any `align: "left"` on child widgets. |
| Calling `emcp-tools-update-page-settings` on a kit post | NEVER. Full replace wipes entire brand kit. Custom palette entries: `update-global-colors`/`update-global-typography` (they merge). Everything else: `wp eval` read-modify-write — see [`references/kit-css.md`](references/kit-css.md). |
| Setting custom typography/colors but skipping the system slots | Set both layers. System slots (Primary/Secondary/Text/Accent) still appear in every widget's picker — if left on Elementor defaults they show Roboto/`#6EC1E4`. See [`references/site-settings.md`](references/site-settings.md). |
| Setting `_element_width` to `"custom"` when specifying custom widget widths | Always set `_element_width` to **`"initial"`** (the actual stored value for the "Custom" option in Elementor's data model) and define the size under `_element_custom_width`. Setting it to `"custom"` is written out literally and breaks the layout. |
| Setting `grid_columns_grid` without `grid_rows_grid` | Always set both axes. Omitting `grid_rows_grid` defaults to 2 rows, leaving an empty second row. Run the post-build eval to confirm — screenshots cannot catch this. See [Grid containers](#grid-containers--always-set-both-axes-explicitly). |
| Omitting `_title` on top-level section containers | Set `_title` at creation time (e.g. `"_title": "Hero — Section"`). Costs nothing; makes the Elementor Navigator a readable page map instead of "Container / Container / Container." Inner containers and widgets don't need it. |
| Wrong tag name or key format for ACF dynamic tags | Use `tag_name: "acf-text"` (not `"acf"`). The `key` setting must be `"field_key:field_name"` (colon-separated, e.g. `"field_page_eyebrow:page_eyebrow"`). Passing only the field key causes a PHP warning and the field still renders, but passing only the field name silently fails. Use `list-dynamic-tags` to confirm available tag names for the active ACF version. |
| Assuming a user-edited element's structure from memory or conversation summaries | Call `get-page-structure` + `get-element-settings` on the reference element *before* building anything meant to match it. User hands-on edits introduce nesting depth, sub-container order, and setting details that no summary fully captures. The API is ground truth; summaries are hints. |
| Assuming a common UI control name maps directly to its Elementor setting key | Elementor uses namespaced/prefixed keys that differ from the panel label. `width` (a panel concept) is `_element_width` + `_element_custom_width`; flex grow is `_flex_size: "grow"` + `_flex_grow: "1"`; background type is `background_background` (value: `"classic"` or `"gradient"`). The `_` prefix applies to specific families (widget sizing, flex) — it is NOT a universal prefix. Never guess key names from the UI label. Always use `get-element-settings` on a known-good element or `get-widget-schema` to discover the real key names before writing. Guessing common names is silently accepted and silently does nothing. |
| Setting a heading font-weight of 300 while expecting `<strong>` children to appear bold | Hello Elementor theme sets `strong { font-weight: bolder }`. With a parent at weight 300, `bolder` computes to 400, not 700. Any heading with `typography_font_weight: "300"` that contains bold `<strong>` child text needs: `add-custom-css(element_id=<heading>, css="selector strong { font-weight: 700; }")`. |
| Changing `content_width` or `flex_direction` on a container without checking the current value first | These are high-impact structural settings — a wrong change breaks the entire section layout for every child. Always call `get-element-settings` on the container before updating either. Changing `content_width` from `"boxed"` to `"full"` or flipping `flex_direction` from `"row"` to `"column"` fundamentally alters how every child renders. |
| Adding Custom CSS for a property without first checking whether a native Elementor control already covers it | Duplicate styling creates specificity conflicts — the CSS and native control fight and the result is unpredictable. Before writing any `add-custom-css`, call `get-element-settings` on the target element. If a native key already exists for the property (padding, border, background, typography, etc.), update it via `update-element` instead. |
| Setting container row/column gap via `gap` key | Use `flex_gap` instead — Elementor's CSS generator reads `flex_gap` to emit `--widgets-spacing-row/column`. The `gap` key is stored but silently ignored for CSS output. Symptom: spacing stays at the kit default (20px) no matter what value you write. |
| Using `_padding` to set container padding | Use `padding` (no underscore). `_padding` is stored but silently ignored for rendered output — the container needs `--padding-top/bottom/left/right` CSS vars which only `padding` generates. See [Container padding](#container-padding--use-padding-not-_padding). |
| Using an HTML widget for the copyright year line | Use a `heading` widget (tag `div`) with the `current-date-time` dynamic tag. Year auto-updates; text stays panel-editable. See [Dynamic tags](#dynamic-tags--outputting-live-data). |
| Creating a Maintenance Mode template with `emcp-tools-create-page` | Use `emcp-tools-create-theme-template` (disabled by default in v3.1 — enable first) — the template must be `post_type: elementor_library`. A regular `page` won't appear in Elementor's Maintenance Mode picker. See [`references/maintenance-mode.md`](references/maintenance-mode.md). |
| Setting `field_border_color` on a form widget without setting `field_border_border` | Set `field_border_border: "solid"` in the same call. Elementor skips the border CSS entirely if the border type is unset — the color has no visible effect. |
| Attempting to build a working atomic form (email/storage) via standalone atomic widgets | Atomic form widgets (`e-form-input`, `e-form-label`, etc.) are **purely presentational** — no submission handling, no actions. Their MCP schemas are completely empty. The only native path to atomic rendering with working submission is the "Use Atomic Form" panel toggle on the regular `form` widget, which is NOT accessible via MCP. Use the regular `form` widget for all production forms. See [`references/form-widget.md`](references/form-widget.md#atomic-forms--definitive-findings-do-not-retry-without-new-information). |
| A flex grow-child wrapping to the next row instead of sharing the row | Elementor's default gives flex children `flex-shrink: 0; flex-basis: auto`, so `_flex_size: "grow"` alone doesn't help. Add element-level custom CSS: `selector { flex: 1 1 0% !important; min-width: 0; }` to force correct shrink/basis behavior. |
| Using a FA6 icon name (e.g. `fa-location-dot`) with Elementor | Elementor ships with Font Awesome 5. FA6-only icons silently render as an empty circle. Use the FA5 equivalent (e.g. `fa-map-marker-alt`). See [`references/form-widget.md`](references/form-widget.md#font-awesome-version-note). |
| Writing a hand-crafted form reference eval instead of using the documented recipe | The eval recipes in [`references/form-widget.md`](references/form-widget.md#reference-site-form-inspection-workflow) are the complete checklist — labels (including `textTransform`), fields, button (including `textTransform`, `borderRadius`, `fontSize`), card wrapper, and field layout. A hand-crafted eval that omits even one property means a styling detail missed and a hand-correction session later. Always run all four documented evals before writing a single `update-element` call on a form. |
| Querying `<form>` element background to find card styling | `getComputedStyle(form).backgroundColor` almost always returns transparent — the card background is on a parent wrapper, not the `<form>` tag. Always walk up the DOM from the form element until you find the first ancestor with a non-transparent background. See eval #3 in [`references/form-widget.md`](references/form-widget.md#reference-site-form-inspection-workflow). |
| Using `field_type: "tel"` on a phone field that has a JS input mask | Elementor Pro validates `tel` fields against the HTML5 tel character set — parentheses and spaces in `(555) 123-4567` fail and show a validation error. Use `field_type: "text"` instead, then restore the mobile numeric keyboard via JS: `phoneInput.setAttribute('inputmode', 'tel')`. The field's DOM ID (from `custom_id`) is unaffected by the type change. See [`references/form-widget.md`](references/form-widget.md#phone-masking--use-field_type-text-not-tel). |
| Defining a WP REST endpoint with `custom_id`-based arg names for an Elementor webhook | Elementor sends field **labels** as keys (spaces → underscores via PHP `parse_str`), NOT `custom_id` values. A field labelled "Work Email" arrives as `Work_Email`. WP REST validates required args *before* the callback runs — a key mismatch returns a silent 400 and your debug `error_log()` calls are never reached. Register args using label-derived keys and use `rest_pre_dispatch` to inspect raw params during debugging. See [`references/form-widget.md`](references/form-widget.md#elementor-form-webhook-payload-format). |
| Placing a JS HTML widget at the page root level when it targets a specific section | JS interaction scripts (masking, show/hide, etc.) should be positioned **adjacent to the element they target** in the Elementor Navigator, not floating at the root. Root-level placement makes the dependency invisible to future editors and the widget becomes orphaned if the section is moved. Also name the widget in the Navigator (`_title: "Phone Mask Script"`) so its purpose is immediately obvious. See [`references/form-widget.md`](references/form-widget.md#js-widget--navigator-naming-and-placement). |

## Key tool reference

| Purpose | Tool |
|---|---|
| Read brand kit | `emcp-tools-get-global-settings` |
| Add/update CUSTOM palette entries (merges; NOT system slots) | `emcp-tools-update-global-colors` / `emcp-tools-update-global-typography` |
| Create standard page (sets `_elementor_edit_mode` itself) | `emcp-tools-create-page` |
| Create Elementor library template (maintenance mode, theme parts) | `emcp-tools-create-theme-template` † |
| Set page template / page-level options | `emcp-tools-update-page-settings` |
| Assign a dynamic tag to a widget field | `emcp-tools-set-dynamic-tag` † (discover tags with `emcp-tools-list-dynamic-tags` †) |
| Add container | `emcp-tools-add-container` |
| Update container | `emcp-tools-update-container` |
| Browse the widget catalog (tier/category/search filters) | `emcp-tools-list-widgets` |
| Get widget's accepted schema (curated; `types: []` batch; `full: true`) | `emcp-tools-get-widget-schema` |
| Get element's current settings | `emcp-tools-get-element-settings` |
| See page tree | `emcp-tools-get-page-structure` |
| Find elements by criteria | `emcp-tools-find-element` |
| Add any free widget — content + styling in one call | `emcp-tools-add-free-widget` (`widget_type` + settings) |
| Add a Pro widget (form, posts, …) | `emcp-tools-add-pro-widget` † |
| Apply styling to any element | `emcp-tools-update-element` ⭐ (widgets-only alias: `emcp-tools-update-widget`) |
| Style many elements in one save | `emcp-tools-batch-update` |
| Duplicate an element incl. children | `emcp-tools-duplicate-element` |
| Re-parent / reorder elements | `emcp-tools-move-element` / `emcp-tools-reorder-elements` |
| Remove element | `emcp-tools-remove-element` |
| Per-element or page-level custom CSS | `emcp-tools-add-custom-css` † (use `replace: true` to overwrite) |
| Upload SVG icon for icon widgets | `emcp-tools-upload-svg-icon` |
| Build complete page from declarative spec | `emcp-tools-build-page` (powerful but hard to iterate) |
| Style a Pro form widget (labels, fields, borders, button) | `emcp-tools-update-element` with native form controls — see [`references/form-widget.md`](references/form-widget.md) |

† = disabled by default in v3.1 — see [the tool gate](#the-v3-tool-gate--39-tools-ship-disabled-by-default).
⭐ = the workhorse tool. If you're not calling `update-element` more than `add-custom-css`, you're probably still in HTML-widget muscle memory.
