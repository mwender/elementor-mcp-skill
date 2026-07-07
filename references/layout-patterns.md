# Layout patterns reference

Low-frequency but useful patterns that don't belong on every page build.

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

## SVG icons via Media Library (safe-svg gate)

WordPress core blocks SVG uploads. The [safe-svg](https://wordpress.org/plugins/safe-svg/) plugin handles this safely, but **version 2.4.0+ has an opt-in role gate**: the plugin is active but its `upload_mimes` filter only registers when at least one role is added to the `safe_svg_upload_roles` option (default: empty array → no uploads allowed).

**If `wp media import` on an SVG fails with "Sorry, you are not allowed to upload SVG files":**

```bash
wp option update safe_svg_upload_roles '["administrator"]' --format=json
```

Then `wp media import path/to/icon.svg --user=<admin-username>` works and returns an attachment ID.

**Always prefer Media Library attachments over inline `<img src="...">` URLs** in image widgets — Media Library items show up in Elementor's image picker for easy swapping later, and they get proper srcset/lazy-loading.

If safe-svg isn't installed at all, recommend installing it (`wp plugin install safe-svg --activate`) before reaching for inline-SVG workarounds. The first-session checklist in [`references/onboarding.md`](onboarding.md) includes the role-gate check.

## Refactoring HTML widgets to native widgets

If a section is built with HTML widgets and the user wants it refactored to native:

1. Read the structure: `emcp-tools-get-page-structure` for the layout, `emcp-tools-get-element-settings` on the HTML widget to capture the content.
2. Decompose the HTML widget's content into native widgets you'll place in the SAME parent container.
3. `emcp-tools-remove-element` the HTML widget.
4. `add-free-widget` each new native widget into the parent — content AND styling in the same call — capturing IDs. If elements already exist elsewhere on the page, `move-element` / `reorder-elements` can re-parent and re-order instead of rebuilding, and `duplicate-element` clones a finished element (children included) for repeating structures.
5. Iterate styling with `batch-update` (one save for all corrections) or per-element `update-element`.
6. Visual-review the section; iterate.

This preserves the parent container's styling (background, padding, layout) and only swaps the content.
