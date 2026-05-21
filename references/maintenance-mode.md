# Maintenance Mode / Coming Soon — setup pattern

Elementor Pro's Maintenance Mode feature intercepts all front-end requests and serves a single template instead. This reference documents the exact post type, WP options, and dynamic tag patterns established on the precision-med.com site (template post 566).

---

## Post type: `elementor_library` — not `page`

The template is an **Elementor library item**, not a standard WordPress page. Creating it as a regular page (`elementor-mcp-create-page`) will fail to appear in Elementor's Maintenance Mode template picker and may cause PHP warnings during the maintenance mode render.

**Correct tool:** `elementor-mcp-create-theme-template` with `template_type: "page"`

After creation, set the page template to canvas via WP-CLI:
```bash
wp post meta update <post_id> _wp_page_template elementor_canvas --path=/path/to/wp
```

Confirm the meta looks like this:
| Meta key | Value |
|---|---|
| `_elementor_template_type` | `page` |
| `_elementor_edit_mode` | `builder` |
| `_wp_page_template` | `elementor_canvas` |

The post must be **published** for Elementor to assign it.

---

## Three WP options that activate Maintenance Mode

Set these via `wp option add` (first time) or `wp option update` (subsequent changes):

```bash
wp option update elementor_maintenance_mode_mode        'coming_soon'
wp option update elementor_maintenance_mode_template_id '566'
wp option update elementor_maintenance_mode_exclude_mode 'logged_in'
```

| Option | Values | Notes |
|---|---|---|
| `elementor_maintenance_mode_mode` | `coming_soon` \| `maintenance` | `coming_soon` = 200 OK (pre-launch); `maintenance` = 503 (downtime) |
| `elementor_maintenance_mode_template_id` | post ID string | Must be a published `elementor_library` post |
| `elementor_maintenance_mode_exclude_mode` | `logged_in` \| `super_admin` | `logged_in` lets all authenticated users see the real site |

To **deactivate** maintenance mode without deleting settings:
```bash
wp option update elementor_maintenance_mode_mode ''
```

---

## Dynamic tag copyright year — the right way to build the footer

**Never** use an HTML widget for the copyright line in a coming-soon/maintenance footer. The native `heading` widget with the `current-date-time` dynamic tag auto-updates the year and keeps the text editable from the panel.

### Two-step recipe

**Step 1 — add the heading widget:**
```
add-heading(post_id=<id>, parent_id="<footer-container>",
  title="placeholder",
  header_size="div",     ← div tag, no semantic heading
  align="center"
)
→ returns element_id "abc1234"
```

**Step 2 — style and set the dynamic tag in parallel:**
```
update-element(element_id="abc1234", settings={
  "typography_typography": "custom",
  "typography_font_family": "Inter",
  "typography_font_size": {"unit": "px", "size": 11, "sizes": []},
  "typography_text_transform": "uppercase",
  "typography_letter_spacing": {"unit": "em", "size": 0.14, "sizes": []},
  "title_color": "rgba(255,255,255,0.2)"
})

set-dynamic-tag(element_id="abc1234",
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

The resulting output: **© 2025 [Site Name]. All rights reserved.**

Replace `[Site Name]` with the actual site name (e.g. `Precision Medicine`). The `before` and `after` fields are plain text — HTML entities like `&copy;` work but `©` (the literal character) also works fine.

### What the stored data looks like

After `set-dynamic-tag` the element's `__dynamic__` setting contains:
```json
{
  "title": "[elementor-tag id=\"{uid}\" name=\"current-date-time\" settings=\"{url_encoded_json}\"]"
}
```
Where the decoded settings are:
```json
{"date_format":"custom","custom_format":"Y","before":"© ","after":" [Site Name]. All rights reserved."}
```
You don't need to construct this string manually — `set-dynamic-tag` handles encoding.

---

## Complete Maintenance Mode page build checklist

1. `elementor-mcp-create-theme-template` → note the post ID
2. `wp post meta update <id> _wp_page_template elementor_canvas`
3. `wp post meta update <id> _elementor_edit_mode builder`
4. Build the page content (canvas template, full-viewport layout)
5. Publish the template: `wp post update <id> --post_status=publish`
6. Set the three maintenance mode options via `wp option update`
7. Verify: visit the site root in agent-browser without being logged in (or check the body class for `elementor-maintenance-mode`)

---

## Page structure pattern (precision-med reference)

```
Page Wrapper (flex column, min-height: 100vh, navy background, radial gradient via custom CSS)
├── [Optional] Header (flex row, top-left logo mark)
│     └── Logo column: two heading widgets — site name + tagline
│           "PRECISION" — Inter 13px 600, uppercase, ls 0.2em, white
│           "MEDICINE"  — Inter 10px 400, uppercase, ls 0.25em, white 45%
├── Main Stage (flex column, center, flex-grow: 1)
│     ├── Badge HTML widget: "—— COMING SOON ——" (gold, flanking hairlines)
│     ├── Eyebrow heading: tagline (Inter 12px, white 38%, uppercase)
│     ├── H1 heading: main headline with gold italic <em> line
│     │     └── Custom CSS: selector h1 em { display:block; font-style:italic; color:#E0BD52; }
│     ├── Separator HTML widget: 1px × 48px vertical gold gradient
│     └── Text-editor: body copy (Inter 17px, white 60%, max-width 560px)
└── Footer (flex row, centered)
      └── Heading widget (div tag) with current-date-time dynamic tag → © YEAR Site Name. All rights reserved.
```

**Container padding key:** use `padding` (no underscore) for Elementor containers — this generates the `--padding-top/bottom/left/right` CSS custom properties. `_padding` is stored but silently ignored for rendered output.
