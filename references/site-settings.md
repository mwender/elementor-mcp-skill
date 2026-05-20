# Configuring Elementor Site Settings for a new design

Use this reference when setting up the global kit design tokens for a new site. These steps are one-time setup — subsequent sessions only need to READ the kit via `get-global-settings`.

## The two-layer problem: system slots AND custom slots

Elementor's Site Settings expose **two parallel naming systems** for both typography and colors. You must configure **both layers**. Leaving either at Elementor's defaults (Roboto, `#6EC1E4`, etc.) means those stale values appear as selectable options in the widget typography/color pickers.

### Typography layers

| Layer | Slot names | Picker label |
|---|---|---|
| **System typography** | Primary, Secondary, Text, Accent | Always visible at the top of the typography dropdown |
| **Custom typography** | Your named presets (Heading, Subheading, Body, Label, etc.) | Appear below system slots in the dropdown |

**If you only define custom presets and skip the system slots**, the system slots still appear in every widget's typography dropdown — but they still point to Elementor's Roboto defaults. A user clicking "Primary" in the panel gets Roboto instead of your brand font. Set both.

### Color layers

| Layer | Slot names | Picker label |
|---|---|---|
| **System colors** | Primary, Secondary, Text, Accent | First group in the color picker dropdown |
| **Custom colors** | Your brand names (Deep Navy, Gold, Slate, etc.) | Second group in the color picker dropdown |

Same issue: system color slots remain at Elementor's `#6EC1E4`/`#54595F`/`#7A7A7A`/`#61CE70` if you only add custom colors.

---

## Setting system typography

Use the `update-global-typography` MCP tool — it targets the system slots (Primary / Secondary / Text / Accent) directly:

```
update-global-typography(typography=[
  {
    "_id": "primary",
    "title": "Primary",
    "typography_font_family": "Cormorant Garamond",
    "typography_font_weight": "400",
    "typography_font_size": {"unit": "px", "size": 48, "sizes": []},
    "typography_line_height": {"unit": "em", "size": 1.1, "sizes": []},
    "typography_letter_spacing": {"unit": "px", "size": -0.5, "sizes": []}
  },
  {
    "_id": "secondary",
    "title": "Secondary",
    "typography_font_family": "Cormorant Garamond",
    "typography_font_weight": "300",
    "typography_font_size": {"unit": "px", "size": 32, "sizes": []},
    "typography_line_height": {"unit": "em", "size": 1.2, "sizes": []}
  },
  {
    "_id": "text",
    "title": "Text",
    "typography_font_family": "Inter",
    "typography_font_weight": "400",
    "typography_font_size": {"unit": "px", "size": 16, "sizes": []},
    "typography_line_height": {"unit": "em", "size": 1.7, "sizes": []}
  },
  {
    "_id": "accent",
    "title": "Accent",
    "typography_font_family": "Inter",
    "typography_font_weight": "600",
    "typography_font_size": {"unit": "px", "size": 12, "sizes": []},
    "typography_letter_spacing": {"unit": "em", "size": 0.15, "sizes": []},
    "typography_text_transform": "uppercase"
  }
])
```

Align these with the design's heading/body fonts. A sensible mapping for a serif-heading + sans-body design:

- **Primary** → serif heading font (e.g. Cormorant Garamond, Playfair Display)
- **Secondary** → serif subheading variant (lighter weight, smaller)
- **Text** → sans body font (e.g. Inter, DM Sans)
- **Accent** → sans label/caption font (uppercase, tight tracking)

## Setting custom typography

Custom presets (Heading, Subheading, Body, Label) live in `_elementor_page_settings.custom_typography`. These cannot be safely written via the MCP `update-page-settings` tool on the kit post (full-replace danger — see [SKILL.md §Kit-level CSS conventions](../SKILL.md#applying-the-baseline--the-only-safe-path-is-wp-eval)). Use `wp eval` read-modify-write:

```bash
KIT_ID=$(wp option get elementor_active_kit)
wp eval '
  $kit_id = '"$KIT_ID"';
  $settings = get_post_meta($kit_id, "_elementor_page_settings", true) ?: [];
  $settings["custom_typography"] = [
    [
      "_id" => "primary",
      "title" => "Heading",
      "typography_font_family" => "Cormorant Garamond",
      "typography_font_weight" => "400",
      "typography_font_size" => ["unit" => "px", "size" => 48, "sizes" => []],
      "typography_line_height" => ["unit" => "em", "size" => 1.1, "sizes" => []],
      "typography_letter_spacing" => ["unit" => "px", "size" => -0.5, "sizes" => []],
      "typography_typography" => "custom",
    ],
    [
      "_id" => "secondary",
      "title" => "Subheading",
      "typography_font_family" => "Cormorant Garamond",
      "typography_font_weight" => "300",
      "typography_font_size" => ["unit" => "px", "size" => 32, "sizes" => []],
      "typography_line_height" => ["unit" => "em", "size" => 1.2, "sizes" => []],
      "typography_typography" => "custom",
    ],
    [
      "_id" => "text",
      "title" => "Body",
      "typography_font_family" => "Inter",
      "typography_font_weight" => "400",
      "typography_font_size" => ["unit" => "px", "size" => 16, "sizes" => []],
      "typography_line_height" => ["unit" => "em", "size" => 1.7, "sizes" => []],
      "typography_typography" => "custom",
    ],
    [
      "_id" => "accent",
      "title" => "Label",
      "typography_font_family" => "Inter",
      "typography_font_weight" => "600",
      "typography_font_size" => ["unit" => "px", "size" => 12, "sizes" => []],
      "typography_text_transform" => "uppercase",
      "typography_letter_spacing" => ["unit" => "em", "size" => 0.15, "sizes" => []],
      "typography_typography" => "custom",
    ],
  ];
  update_post_meta($kit_id, "_elementor_page_settings", $settings);
'
wp elementor flush_css
```

## Setting system colors

System colors (Primary / Secondary / Text / Accent) are stored in `_elementor_page_settings.system_colors`. Update them via `wp eval` — there is no dedicated MCP tool that safely targets system colors in isolation:

```bash
KIT_ID=$(wp option get elementor_active_kit)
wp eval '
  $kit_id = '"$KIT_ID"';
  $settings = get_post_meta($kit_id, "_elementor_page_settings", true) ?: [];
  $settings["system_colors"] = [
    ["_id" => "primary",   "title" => "Primary",   "color" => "#1A2744"],
    ["_id" => "secondary", "title" => "Secondary",  "color" => "#C9A84C"],
    ["_id" => "text",      "title" => "Text",       "color" => "#6B7385"],
    ["_id" => "accent",    "title" => "Accent",     "color" => "#F3EFE6"],
  ];
  update_post_meta($kit_id, "_elementor_page_settings", $settings);
'
wp elementor flush_css
```

## Setting custom colors

Use the `update-global-colors` MCP tool — it targets custom color slots:

```
update-global-colors(colors=[
  {"_id": "primary",   "title": "Deep Navy",   "color": "#1A2744"},
  {"_id": "secondary", "title": "Gold",        "color": "#C9A84C"},
  {"_id": "text",      "title": "Slate",       "color": "#6B7385"},
  {"_id": "accent",    "title": "Warm Cream",  "color": "#F3EFE6"}
])
```

## Other kit-level settings that require `wp eval`

These settings live in `_elementor_page_settings` but have no dedicated MCP tool. All use the same read-modify-write pattern:

| Setting | Key(s) |
|---|---|
| Body font + color | `body_color`, `body_typography_font_family`, `body_typography_font_weight`, `body_typography_font_size`, `body_typography_line_height` |
| Heading colors | `h1_color` … `h6_color` |
| Heading typography | `h1_typography_font_family`, `h1_typography_font_size`, `h1_typography_line_height`, etc. |
| Button defaults | `button_background_color`, `button_text_color`, `button_hover_*`, `button_border_*`, `button_typography_*`, `button_padding` |
| Container max width | `container_width` → `{"unit": "px", "size": 1200, "sizes": []}` |
| Link colors | `link_normal_color`, `link_hover_color` |

Always run `wp elementor flush_css` after any `wp eval` kit write.

## Verification checklist

After configuring all four layers, use `get-global-settings` to confirm:

```
# System typography — should show your brand fonts, not Roboto/Roboto Slab
response.typography[0].typography_font_family  # Primary
response.typography[2].typography_font_family  # Text

# Custom typography — should show your named presets
response.settings.custom_typography[0].title   # "Heading" (or whatever you named it)

# System colors — should show your brand hex values
response.colors[0].color  # Primary system color

# Custom colors — should show your brand names and hex values
response.settings.custom_colors[0].title  # "Deep Navy" (or your name)

# Body + headings
response.settings.body_typography_font_family
response.settings.h1_typography_font_family
```

If `response.typography[0].typography_font_family` is still `"Roboto"` after calling `update-global-typography`, check that the MCP session round-tripped correctly and flush CSS again.
