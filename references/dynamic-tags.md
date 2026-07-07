# Dynamic tags — recipes

Use `emcp-tools-set-dynamic-tag` whenever a widget field should output a computed or context-aware value rather than hardcoded text. This keeps content editable from the panel and avoids manual updates when data changes.

> Both `set-dynamic-tag` and `list-dynamic-tags` are **disabled by default in v3.1** — enable them first via the tool gate (see SKILL.md "The v3 tool gate"). Once enabled, the signature and behavior are unchanged from earlier plugin versions (verified against v3.1.0 — the stored form is the standard `__dynamic__` elementor-tag shortcode).

```
set-dynamic-tag(post_id=<id>, element_id="<widget>",
  setting_key="<field>",   # e.g. "title", "url", "image"
  tag_name="<tag>",        # e.g. "current-date-time", "post-title", "site-title", "acf-text"
  tag_settings={...}       # tag-specific options
)
```

Use `list-dynamic-tags` to discover the tag names available on the site (returns name/title/group/categories per tag) — tag availability varies with Elementor Pro and ACF versions.

## Copyright year — sitewide footer or any page footer

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

Output: **© 2026 [Site Name]. All rights reserved.**

Replace `[Site Name]` with the actual site name. The `before`/`after` fields wrap the dynamic year output with plain text; HTML entities like `&copy;` also work.

Never use an HTML widget for the copyright line — the heading-plus-tag approach auto-updates and stays panel-editable.

## ACF field output

Use `tag_name: "acf-text"` (not `"acf"`). The `key` setting must be `"field_key:field_name"` — colon-separated, e.g. `"field_page_eyebrow:page_eyebrow"`.

- Passing only the field key: PHP warning, but the field still renders.
- Passing only the field name: **silently fails.**
- Unsure of tag names for the active ACF version? `list-dynamic-tags` is ground truth.
