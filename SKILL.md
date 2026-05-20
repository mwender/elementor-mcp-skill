---
name: elementor-mcp
description: >-
  Use this skill whenever you are working with an Elementor or Elementor Pro
  WordPress site via the Elementor MCP — recognizable by `mcp__*__elementor-mcp-*`
  tools being available in the session, or by the user asking to create, edit,
  refactor, or style a page on a WordPress site that uses Elementor (often with
  the Hello Elementor theme). Teaches the fluent patterns for building pages
  with native Elementor widgets (image, heading, text-editor, icon-box, container,
  etc.) instead of falling back to HTML widgets; the two-step `add-*` then
  `update-element` pattern for getting styling into Elementor's native data
  model so it remains editable from the panel UI; the global kit / `__globals__`
  strategy for brand consistency; common traps like `content_width: "boxed"`
  collapsing flex layouts; the safe-svg upload-roles gate; and the agent-browser
  visual-review loop. Trigger even when the user doesn't say "Elementor MCP"
  explicitly — if Elementor MCP tools are available in the session, this skill
  applies to any page-building, page-editing, or page-refactoring task on that
  site. Also trigger when the user mentions building a landing page, sales
  page, one-sheet, or pitch-deck-style page on a WordPress site that uses
  Elementor.
---

# Elementor MCP — fluent page building

Most pain when building Elementor pages via MCP comes from one mistake: dropping into HTML widgets the moment a styling argument is rejected by an `add-*` tool. The widgets accept much more than their strict `add-*` validators expose. The patterns below let you build pages that are visually faithful AND remain fully editable from the Elementor panel UI — which is the whole point of using Elementor.

## Onboarding a new Elementor site

If `mcp__*__elementor-mcp-*` tools are already available in this session, the site is wired up — skip to [Workflow](#workflow).

If they're not, the user is starting fresh and you need to get the MCP plumbing in place. **See [`references/onboarding.md`](references/onboarding.md)** for the full flow:

- **Detect framework** (Standard WP vs Bedrock) — install paths differ
- **Install both MCP plugins** — `wordpress/mcp-adapter` foundation + `msrbuilds/elementor-mcp` for page-building tools. Standard WP path (wp-cli + zip from GitHub Releases) and Bedrock path (Composer + VCS repo) both documented
- **Verify servers registered** with `wp mcp-adapter list`
- **Write `.mcp.json`** — STDIO template (preferred for local sites) and HTTP template (for cloud/remote)
- **First-session checklist** — plugins active, safe-svg role-gate open, kit ID identified, agent-browser viewport sized
- **Memory templates** — three starter files to give every future session on this site a fast cold-start (project context, transport rationale, MEMORY.md index), plus guidance on project-level vs user-level memory placement

One-line framework detection you can run at the project root:

```bash
test -f composer.json && grep -q '"roots/wordpress"' composer.json && echo BEDROCK || echo STANDARD
```

## Workflow

1. **Read the source artifact first.** PDF, Figma, sketch, brief, or reference URL — whatever's authoritative for content and design. Don't guess. **If a reference URL is provided**, open it in `agent-browser` and use `snapshot` + `eval` to extract content and computed design tokens before touching Elementor. See [`references/reference-site-analysis.md`](references/reference-site-analysis.md).
2. **Get the site's global kit.** Call `elementor-mcp-get-global-settings` before placing the first container. It returns Primary/Secondary/Accent/Text colors and H1/H2/body typography — the source of truth for brand. Use these via `__globals__` references rather than redefining colors per element, so kit edits cascade. **If this is a fresh design setup**, the system slots (Primary/Secondary/Text/Accent for both colors and typography) will still be on Elementor defaults (Roboto, `#6EC1E4`, etc.) — configure those alongside any custom presets before building. See **[`references/site-settings.md`](references/site-settings.md)** for the full two-layer setup pattern.
3. **Check the kit's baseline CSS.** Inspect `settings.custom_css` from the global settings response. If the string `skill-baseline: applied` is absent (and `skill-baseline: opt-out` is also absent), propose adding the baseline rules before building. Full rules, application recipe, and opt-out flow in **[`references/kit-css.md`](references/kit-css.md)**.
4. **Narrate the layout before any build call.** Decompose into sections → containers → widgets → settings in prose. Catches design mistakes before they're 20 `add-*` calls deep, and turns the eventual implementation into a transcription rather than a design exercise.
5. **Create the page as draft first.** `elementor-mcp-create-page` with `status: "draft"`. Pick the right page template (see [Page template choice](#page-template-choice)). **After creating any page you intend to build with Elementor, immediately set `_elementor_edit_mode` to `builder`** — otherwise the page renders with the block editor and your Elementor content is invisible: `wp post meta update <id> _elementor_edit_mode builder`.
6. **Build sections incrementally.** Container → its children → next container. Don't try to one-shot the whole page through `build-page` — small steps make iteration cheap. **Set `_title` on every top-level section container** at creation time (e.g. `"_title": "Hero — Section"`). It costs nothing — just another key in the `add-container` call — and turns the Elementor Navigator from an unlabeled "Container / Container / Container" list into a readable page map.
7. **Publish, then visual review.** Use `agent-browser` (see [Visual review](#visual-review-with-agent-browser)). Compare against the source for both rendering glitches AND content fidelity. Iterate per-section using `update-element`, `update-container`, or `add-custom-css`.

## The two-step widget pattern

This is the single most important pattern in the skill. Internalize it.

**`add-<widget>` for content. `update-element` for styling.**

The `add-*` tool validators are strict — they accept only a widget's core content keys and reject most styling args. For example, `add-heading` rejects `title_color`; `add-image` rejects `width`. The tool descriptions list those keys because the widget really does accept them, but they live in conditional sub-control groups that the `add-*` schema validator doesn't expose.

`update-element` does a partial merge into the element's settings without the strict validator and accepts the full long-tail control surface.

**Example — accent-blue centered H2 heading:**

```
# Step 1: add the widget with content only
add-heading(post_id=1537, parent_id="abc123", title="The Next Evolution", header_size="h2")
→ returns element_id "64bfc7a"

# Step 2: apply the styling via update-element
update-element(post_id=1537, element_id="64bfc7a", settings={
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
```

Don't waste calls trying styling args inside `add-*` and recovering from rejections. Go straight to `update-element` after the add. This applies to every widget type.

**Why this matters:** styling stored in element settings shows up in the Elementor Style tab and is editable from the panel UI. Styling stored in Custom CSS blocks is not. The user shouldn't have to dig into code to change a color.

## Choosing widgets — favor native over HTML

HTML widgets defeat Elementor's value proposition. Once content is in `add-html`, the editor user can't drag, recolor, or reorder pieces from the panel — they're stuck with raw markup.

Use the table below to pick the right widget. Reach for `add-html` only when the design truly needs markup with no native equivalent (embeds, specific semantic HTML, etc.).

| Design pattern | Native expression |
|---|---|
| Heading + paragraph | `heading` + `text-editor` widgets in a container |
| Icon + title + description card | column container with image (or icon) + heading + text-editor |
| Feature list with bullets/icons | `icon-list` widget |
| Call-to-action button | `button` widget |
| Horizontal divider | `divider` widget |
| Gradient or solid panel with content | container with `_background_background: "gradient"` or `"classic"` |
| Single circular icon badge | image widget styled directly (see [`references/layout-patterns.md`](references/layout-patterns.md#image-as-badge)) |
| Counter/stats display | `counter` widget |
| Testimonial block | `testimonial` widget |

**Decomposition principle.** When in doubt, prefer more widgets over fewer. Three native widgets (icon, title, description) in a column container beat one icon-box widget when the design needs custom styling, and beat one HTML widget almost always. Each widget is its own draggable, content-editable unit in the panel.

## Container gap — use `flex_gap`, not `gap`

When setting the spacing between a container's flex children, always use `flex_gap` in `update-element` calls — not `gap`. Elementor's CSS generator reads `flex_gap` to emit `--widgets-spacing-row` / `--widgets-spacing-column`. The `gap` key is accepted and stored in the element data, but it does not drive CSS output and will appear to have no effect on the frontend.

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

Also add the matching anti-pattern row (below, in the Anti-patterns table).

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
const g = document.querySelector('.elementor-element-<element_id>');
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

`add-custom-css` is the escape hatch, not the workhorse. Use it only when the styling has no native control:

- `<strong>`, `<em>`, `<a>` styling inside heading/text content
- `white-space: nowrap`
- Pseudo-elements (`::before`, `::after`)
- Complex selectors (`& + &`, `:nth-child`, etc.)
- Mix-blend modes on non-image elements
- CSS Grid declarations (Elementor's grid container support is limited)

If a styling decision can plausibly be expressed via `title_color`, `text_color`, `typography_*`, `_background_*`, `_border_*`, `_padding`, `_margin`, `_element_width`, `_element_custom_width`, `_z_index`, etc., it goes in `update-element` instead.

**`replace: true`** on `add-custom-css` overwrites the element's existing CSS block (defaults to append). Use it when shrinking a CSS block after moving styles to native controls.

## Kit-level CSS — safety rule and baseline check

> ⚠️ **NEVER call `elementor-mcp-update-page-settings` on the kit post.** It does a **full replace** on the kit's `_elementor_page_settings` meta — colors, typography, globals, all wiped. The only safe path for any kit write is `wp eval` read-modify-write. **No exception.**

The kit ships with a small baseline of defensive CSS overrides (paragraph margin resets, focus-ring accessibility fix). Check for the `skill-baseline: applied` marker in `settings.custom_css` at workflow step 3. Full baseline rules, application recipe, corruption recovery, and opt-out flow: **[`references/kit-css.md`](references/kit-css.md)**.

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

## Page template choice

| Use case | Setting |
|---|---|
| Standalone landing page (no theme header/footer) | `template: "elementor_canvas"` |
| Full-width within site chrome | `template: "elementor_header_footer"` |
| Content page integrated with site nav | omit or `template: "default"` |

Set via `elementor-mcp-update-page-settings` after page creation. Use canvas for one-sheets, pitch decks, sales pages; default for content/blog pages.

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

If you lose an ID, recover it with `elementor-mcp-get-page-structure` or `elementor-mcp-find-element`.

## Anti-patterns to avoid

| Anti-pattern | What to do instead |
|---|---|
| Falling back to `add-html` after one `add-*` styling rejection | Go straight to `update-element` with the styling keys |
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
| Calling `elementor-mcp-update-page-settings` on a kit post | NEVER. Full replace wipes entire brand kit. Use `wp eval` read-modify-write — see [`references/kit-css.md`](references/kit-css.md). |
| Setting custom typography/colors but skipping the system slots | Set both layers. System slots (Primary/Secondary/Text/Accent) still appear in every widget's picker — if left on Elementor defaults they show Roboto/`#6EC1E4`. See [`references/site-settings.md`](references/site-settings.md). |
| Setting `grid_columns_grid` without `grid_rows_grid` | Always set both axes. Omitting `grid_rows_grid` defaults to 2 rows, leaving an empty second row. Run the post-build eval to confirm — screenshots cannot catch this. See [Grid containers](#grid-containers--always-set-both-axes-explicitly). |
| Omitting `_title` on top-level section containers | Set `_title` at creation time (e.g. `"_title": "Hero — Section"`). Costs nothing; makes the Elementor Navigator a readable page map instead of "Container / Container / Container." Inner containers and widgets don't need it. |
| Wrong tag name or key format for ACF dynamic tags | Use `tag_name: "acf-text"` (not `"acf"`). The `key` setting must be `"field_key:field_name"` (colon-separated, e.g. `"field_page_eyebrow:page_eyebrow"`). Passing only the field key causes a PHP warning and the field still renders, but passing only the field name silently fails. Use `list-dynamic-tags` to confirm available tag names for the active ACF version. |
| Assuming a user-edited element's structure from memory or conversation summaries | Call `get-page-structure` + `get-element-settings` on the reference element *before* building anything meant to match it. User hands-on edits introduce nesting depth, sub-container order, and setting details that no summary fully captures. The API is ground truth; summaries are hints. |
| Setting container row/column gap via `gap` key | Use `flex_gap` instead — Elementor's CSS generator reads `flex_gap` to emit `--widgets-spacing-row/column`. The `gap` key is stored but silently ignored for CSS output. Symptom: spacing stays at the kit default (20px) no matter what value you write. |

## Key tool reference

| Purpose | Tool |
|---|---|
| Read brand kit | `elementor-mcp-get-global-settings` |
| Create page | `elementor-mcp-create-page` |
| Set page template / page-level options | `elementor-mcp-update-page-settings` |
| Add container | `elementor-mcp-add-container` |
| Update container | `elementor-mcp-update-container` |
| Get widget's accepted schema | `elementor-mcp-get-widget-schema` |
| Get element's current settings | `elementor-mcp-get-element-settings` |
| See page tree | `elementor-mcp-get-page-structure` |
| Find elements by criteria | `elementor-mcp-find-element` |
| Add widget (heading/image/text-editor/button/icon-box/icon-list/divider/html/etc.) | `elementor-mcp-add-<widget>` |
| Apply styling to any element | `elementor-mcp-update-element` ⭐ |
| Remove element | `elementor-mcp-remove-element` |
| Per-element or page-level custom CSS | `elementor-mcp-add-custom-css` (use `replace: true` to overwrite) |
| Upload SVG icon for icon widgets | `elementor-mcp-upload-svg-icon` |
| Build complete page from declarative spec | `elementor-mcp-build-page` (powerful but hard to iterate) |

⭐ = the workhorse tool. If you're not calling `update-element` more than `add-custom-css`, you're probably still in HTML-widget muscle memory.
