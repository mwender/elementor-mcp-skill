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
3. **Check the kit's baseline CSS.** Inspect `settings.custom_css` from the global settings response. If the [kit-baseline rules](#kit-level-css-conventions) aren't present (and there's no opt-out marker), propose adding them to the user before building. This is a one-time per-site check — once the baseline is in place, you skip this step on subsequent sessions.
4. **Narrate the layout before any build call.** Decompose into sections → containers → widgets → settings in prose. Catches design mistakes before they're 20 `add-*` calls deep, and turns the eventual implementation into a transcription rather than a design exercise.
5. **Create the page as draft first.** `elementor-mcp-create-page` with `status: "draft"`. Pick the right page template (see [Page template choice](#page-template-choice)).
6. **Build sections incrementally.** Container → its children → next container. Don't try to one-shot the whole page through `build-page` — small steps make iteration cheap.
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
| Single circular icon badge | image widget styled directly (see [Image-as-badge](#image-as-badge)) |
| Counter/stats display | `counter` widget |
| Testimonial block | `testimonial` widget |

**Decomposition principle.** When in doubt, prefer more widgets over fewer. Three native widgets (icon, title, description) in a column container beat one icon-box widget when the design needs custom styling, and beat one HTML widget almost always. Each widget is its own draggable, content-editable unit in the panel.

## Image-as-badge

A circular gradient icon badge can be the image widget itself — no wrapper container required. Set the wrapper width, padding, gradient background, and 50% border-radius on the image widget directly:

```
update-element(post_id=<id>, element_id="<image-widget-id>", settings={
  "width": {"unit": "px", "size": 52, "sizes": []},
  "_element_width": "initial",
  "_element_custom_width": {"unit": "px", "size": 96, "sizes": []},
  "_padding": {"top": "22", "right": "22", "bottom": "22", "left": "22", "unit": "px", "isLinked": true},
  "_background_background": "gradient",
  "_background_color": "#762ED3",
  "_background_color_b": "#366FFE",
  "_background_gradient_angle": {"unit": "deg", "size": 135, "sizes": []},
  "_border_radius": {"top": "50", "right": "50", "bottom": "50", "left": "50", "unit": "%", "isLinked": true}
})
```

96px wrapper + 22px padding all sides = 52px image area inside a perfect 50%-rounded circle. The image (typically a 1200×1200 SVG) renders centered. One widget, fully editable, no nested containers.

## Container width on flex children — the boxed trap

The single most common layout bug: a 3-column flex row that should sit horizontal collapses to a vertical stack.

**Cause:** child containers default to `content_width: "boxed"` (inherited from common usage), which adds the `e-con-boxed` class. Elementor's CSS forces `e-con-boxed` to 100% width regardless of the container's own `width` setting.

**Fix:** flex children that have explicit widths should be `content_width: "full"`. Their `width` setting (e.g. 340px) is then respected and they lay out as expected.

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

**Worked example — tier card with badge + content column:**

```
# Row flex card container has two children: a badge image and a content column.

# Badge image — fixed size, don't shrink:
update-element(element_id="<badge-img-id>", settings={
  "_element_custom_width": {"unit": "px", "size": 80, "sizes": []},
  "_flex_size": "none",
  ...gradient + radius settings...
})

# Content column — grow into remaining space:
update-container(element_id="<content-col-id>", settings={
  "_flex_size": "grow",
  "_flex_grow": "1"
})
```

Symptom of forgetting the `"none"`: the fixed-size sibling visually disappears or becomes a sliver. If you see that, the growing partner is starving it.

## Column containers default to `flex_align_items: "center"` — set it explicitly

A flex column container created without an explicit `flex_align_items` gets `"center"` injected by Elementor. That centers each child widget *as a block* within the column — which makes any `align: "left"` setting on a heading or text-editor inside it look like it's doing nothing. The text alignment IS being set, but the WIDGET WRAPPER is centered, so left-aligned text inside a narrow centered wrapper still appears centered on the page.

**Always set `flex_align_items: "stretch"` on column containers whose children should fill the column's width.**

```
add-container(parent_id="<parent>", settings={
  "flex_direction": "column",
  "flex_align_items": "stretch",   # ← otherwise Elementor injects "center"
  ...
})
```

Symptom: a heading or text-editor inside a flex-column container appears horizontally centered no matter how many times you set `align: "left"` on it via `update-element`. The fix is on the container, not the widget.

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

## SVG icons via Media Library (safe-svg gate)

WordPress core blocks SVG uploads. The [safe-svg](https://wordpress.org/plugins/safe-svg/) plugin handles this safely, but **version 2.4.0+ has an opt-in role gate**: the plugin is active but its `upload_mimes` filter only registers when at least one role is added to the `safe_svg_upload_roles` option (default: empty array → no uploads allowed).

**If `wp media import` on an SVG fails with "Sorry, you are not allowed to upload SVG files":**

```bash
wp option update safe_svg_upload_roles '["administrator"]' --format=json
```

Then `wp media import path/to/icon.svg --user=<admin-username>` works and returns an attachment ID.

**Always prefer Media Library attachments over inline `<img src="...">` URLs** in image widgets — Media Library items show up in Elementor's image picker for easy swapping later, and they get proper srcset/lazy-loading.

If safe-svg isn't installed at all, recommend installing it (`wp plugin install safe-svg --activate`) before reaching for inline-SVG workarounds.

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

## Kit-level CSS conventions

A small set of CSS rules belongs in Kit → Site Settings → Custom CSS rather than on individual widgets. They're defensive overrides that almost always improve an Elementor site: applied once at the kit level, they cover every widget, every page, every future addition.

**The baseline rules:**

```css
/* Drop trailing margin on the last paragraph of a text-editor widget — keeps
   the widget's bottom edge flush with its container's bottom padding.
   Both selectors handle Elementor's two DOM modes: "Optimized DOM Output"
   enabled (the default in 3.x+, no .elementor-widget-container wrapper) and
   disabled (older sites still have the wrapper). */
.elementor-widget-text-editor > p:last-child,
.elementor-widget-text-editor > .elementor-widget-container > p:last-child {
  margin-bottom: 0;
}

/* Symmetric: drop leading margin on the first paragraph too. */
.elementor-widget-text-editor > p:first-child,
.elementor-widget-text-editor > .elementor-widget-container > p:first-child {
  margin-top: 0;
}

/* Accessibility: visible keyboard-focus outline on links inside Elementor.
   Hello Elementor and many themes hide the default focus ring. */
.elementor a:focus-visible {
  outline: 2px solid currentColor;
  outline-offset: 2px;
}
```

**Why two selectors per margin rule?** Elementor has a setting called "Optimized DOM Output" (Settings → Elementor → Features). When enabled (the default on new installs), text-editor widgets render as `<div class="elementor-widget-text-editor"><p>...</p></div>` — the `<p>` is a direct child. When disabled (older sites), there's an extra wrapper: `<div class="elementor-widget-text-editor"><div class="elementor-widget-container"><p>...</p></div></div>`. The doubled selector handles both. The direct-child combinator (`>`) keeps the rule from accidentally matching paragraphs nested inside lists or blockquotes.

**Verify selectors against real markup before committing them to a kit.** A quick check in agent-browser:

```bash
agent-browser eval "document.querySelectorAll('.elementor-widget-text-editor > p:last-child, .elementor-widget-text-editor > .elementor-widget-container > p:last-child').length"
```

If the count is 0 on a page that clearly has text-editor widgets, the selector is wrong. Don't write a CSS rule to a global kit without confirming it matches at least one element.

### The workflow check (step 3 of [Workflow](#workflow))

On your first interaction with an Elementor site, after `get-global-settings`, look at `settings.custom_css`. Decide whether to propose the baseline:

1. **Opt-out marker present** — if `custom_css` contains the substring `skill-baseline: opt-out` (e.g. as a comment), skip the check entirely. The site owner explicitly declined the baseline.
2. **Baseline marker present** — if `custom_css` contains the substring `skill-baseline: applied`, the rules are already in place. Skip.
3. **Otherwise** — diff the baseline rules against the existing CSS. Surface a short message to the user:
   > "I noticed this site doesn't have the kit-baseline rules ([list which are missing]). They're defensive overrides for trailing paragraph margins and accessibility focus outlines. Want me to add them to Site Settings → Custom CSS?"
   
   On yes: apply via the mechanism below. On no: ask if they want to opt-out permanently (drops a comment so future sessions skip).

**Propose and confirm, don't apply silently.** The kit is a shared resource that affects the whole site.

### Applying the baseline — the ONLY safe path is `wp eval`

> ⚠️ **DO NOT use `elementor-mcp-update-page-settings` on the kit post.** This is the most dangerous trap in this entire skill. The MCP tool does a **full replace** on the kit's `_elementor_page_settings` meta, NOT a merge. If you pass `{"custom_css": "..."}` to the kit's post_id, it will replace the entire settings array with just that one key. Elementor then falls back to its global defaults for everything else — colors revert to `#6EC1E4`/`#54595F`/`#7A7A7A`/`#61CE70`, typography reverts to Roboto, every per-heading color is lost, every `__globals__` reference disappears. The site's brand identity is wiped in a single call.
>
> This is the opposite of `update-element` (which DOES partial-merge — that's why widget styling work is safe). The behavior diverges specifically for kit posts.
>
> The only safe path for kit-level changes is read-modify-write via `wp eval`. **There is no exception. Don't be tempted by the convenience of the MCP path here.**

```bash
KIT_ID=$(wp option get elementor_active_kit)
wp eval '
  $kit_id = '"$KIT_ID"';
  $settings = get_post_meta($kit_id, "_elementor_page_settings", true) ?: [];
  $existing = $settings["custom_css"] ?? "";
  $baseline = "
/* skill-baseline: applied */
.elementor-widget-text-editor > p:last-child,
.elementor-widget-text-editor > .elementor-widget-container > p:last-child { margin-bottom: 0; }
.elementor-widget-text-editor > p:first-child,
.elementor-widget-text-editor > .elementor-widget-container > p:first-child { margin-top: 0; }
.elementor a:focus-visible { outline: 2px solid currentColor; outline-offset: 2px; }
";
  $settings["custom_css"] = trim($existing) . "\n\n" . trim($baseline) . "\n";
  update_post_meta($kit_id, "_elementor_page_settings", $settings);
'
wp elementor flush_css
```

The `skill-baseline: applied` comment is the marker future sessions look for to know the rules are in place. **Always run `wp elementor flush_css` after the write** — Elementor caches kit CSS to disk; without flushing, the rules won't render until something else triggers a regen.

**Why this pattern is safe:** it reads the FULL current settings array, modifies only the `custom_css` key in-memory, and writes the WHOLE array back. Every other key (`system_colors`, `system_typography`, `h1_color`, `body_color`, `__globals__`, etc.) is preserved by identity.

### If the kit is already corrupted

If you discover the kit has been wiped (symptoms: headlines render `rgb(122, 122, 122)`, computed `--e-global-color-text` is `#7A7A7A`, `system_colors` shows Elementor defaults like `#6EC1E4`), WordPress post revisions are your safety net. Elementor preserves kit settings in revisions:

```bash
# List revisions of the kit (active kit is usually post 4)
wp post list --post_type=revision --post_parent=$(wp option get elementor_active_kit) \
  --fields=ID,post_modified,post_title

# Inspect a candidate revision's settings to find the last known-good one
wp eval '
  $rev_id = <REVISION_ID>;
  $ps = get_post_meta($rev_id, "_elementor_page_settings", true);
  echo "Primary: " . $ps["system_colors"][0]["color"] . PHP_EOL;
  echo "h1_color: " . ($ps["h1_color"] ?? "MISSING") . PHP_EOL;
'

# Restore from the chosen revision (preserves the revision; non-destructive)
wp eval '
  $good = get_post_meta(<REVISION_ID>, "_elementor_page_settings", true);
  update_post_meta(<KIT_ID>, "_elementor_page_settings", $good);
'
wp elementor flush_css
```

Always inspect a few revisions to find the most recent pre-corruption snapshot. Reapply any legitimate post-corruption changes (like the baseline CSS) on top of the restored settings.

### Opt-out

If the user says no to the baseline (and "no" should mean "don't ask again on this site"), append the opt-out comment to the kit's custom_css. Use either Path A (MCP) or Path B (wp eval) from above — the only change is the appended string is `/* skill-baseline: opt-out */` instead of the rules block.

Subsequent sessions will detect the marker and skip the check.

### Per-rule opt-out

If the user wants the baseline overall but objects to one specific rule (e.g. they like trailing paragraph margins for prose pages), don't apply that rule. Add only the rules they accept, and the marker comment. There's no need for a finer-grained marker — if the user wants to add a missing rule later, they can re-run the check.

## Element ID tracking

Every `add-*` call returns an element_id (7-character random string). You'll need these for `update-element`, `add-custom-css`, `remove-element`, `find-element`. They're easy to lose track of — keep a running map. A todo list or a scratch table works:

```
Hero container         2e6d951
  Logomark image       78dfdf2
  Wordmark heading     29cd8b9
Tagline (h6)           24de3db
Accent H1 (h2)         64bfc7a
Subhead text-editor    f3eda9f
```

If you lose an ID, recover it with `elementor-mcp-get-page-structure` or `elementor-mcp-find-element`.

## Refactoring HTML widgets to native widgets

If a section is built with HTML widgets and the user wants it refactored to native:

1. Read the structure: `elementor-mcp-get-page-structure` for the layout, `elementor-mcp-get-element-settings` on the HTML widget to capture the content.
2. Decompose the HTML widget's content into native widgets you'll place in the SAME parent container.
3. `elementor-mcp-remove-element` the HTML widget.
4. `add-*` each new native widget into the parent, capturing IDs.
5. `update-element` each new widget for native styling.
6. Visual-review the section; iterate.

This preserves the parent container's styling (background, padding, layout) and only swaps the content.

## Anti-patterns to avoid

| Anti-pattern | What to do instead |
|---|---|
| Falling back to `add-html` after one `add-*` styling rejection | Go straight to `update-element` with the styling keys |
| Skipping `get-global-settings` | Read the kit first; reference it via `__globals__` |
| Defining brand colors per-element | Use `__globals__` references: `"title_color__globals__": "globals/colors?id=accent"` |
| Re-uploading images already in Media Library | Check `wp media list` first |
| Wrapping every icon in its own badge container | Style the image widget itself — see Image-as-badge |
| Building a whole section in one `build-page` call | Build incrementally; iterate per-section |
| Forgetting `agent-browser set viewport` | Always set to 1280×800 (or wider) before screenshotting desktop layouts |
| Putting layout-affecting CSS in custom-css blocks | If there's a native control (padding, margin, flex), use it via update-element |
| Treating `add-custom-css` as primary, `update-element` as fallback | Inverse: native first, CSS only when no native control exists |
| Giving a flex child `_flex_grow: "1"` without setting `_flex_size: "none"` on its fixed sibling | Pair them — see "Flex sizing" above. Symptom: the fixed sibling visually disappears. |
| Creating a flex-column container without explicit `flex_align_items` | Set `flex_align_items: "stretch"` explicitly — Elementor injects `"center"` otherwise, which silently breaks any `align: "left"` on child widgets. |
| Calling `elementor-mcp-update-page-settings` on a kit post | NEVER. It does a full replace on kit meta and wipes the entire brand kit (colors, typography, globals). Use `wp eval` read-modify-write instead — see [Applying the baseline](#applying-the-baseline--the-only-safe-path-is-wp-eval). The MCP tool is safe for regular `page`/`post` post types only. |
| Setting custom typography/colors but skipping the system slots | Set both layers. System slots (Primary/Secondary/Text/Accent) still appear in every widget's picker — if left on Elementor defaults they show Roboto/`#6EC1E4`. See [`references/site-settings.md`](references/site-settings.md). |

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
