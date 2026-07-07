# Kit-level CSS conventions

A small set of CSS rules belongs in Kit → Site Settings → Custom CSS rather than on individual widgets. They're defensive overrides that almost always improve an Elementor site: applied once at the kit level, they cover every widget, every page, every future addition.

## The baseline rules

```css
/* skill-baseline: applied */

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

**Verify selectors against real markup before committing them to a kit:**

```bash
agent-browser eval "document.querySelectorAll('.elementor-widget-text-editor > p:last-child, .elementor-widget-text-editor > .elementor-widget-container > p:last-child').length"
```

If the count is 0 on a page that clearly has text-editor widgets, the selector is wrong. Don't write a CSS rule to a global kit without confirming it matches at least one element.

## The workflow step-3 check

On your first interaction with an Elementor site, after `get-global-settings`, look at `settings.custom_css`. **The key may be absent entirely on a fresh kit — treat a missing key exactly like an empty string** (i.e., proceed to propose the baseline). Decide whether to propose:

1. **Opt-out marker present** — if `custom_css` contains the substring `skill-baseline: opt-out`, skip. The site owner explicitly declined.
2. **Baseline marker present** — if `custom_css` contains the substring `skill-baseline: applied`, the rules are in place. Skip.
3. **Otherwise** — propose adding them:
   > "I noticed this site doesn't have the kit-baseline rules ([list which are missing]). They're defensive overrides for trailing paragraph margins and accessibility focus outlines. Want me to add them to Site Settings → Custom CSS?"

**Propose and confirm, don't apply silently.** The kit is a shared resource that affects the whole site.

## Applying the baseline — the ONLY safe path is `wp eval`

> ⚠️ **DO NOT use `emcp-tools-update-page-settings` on the kit post.** This is the most dangerous trap in the entire skill. The MCP tool does a **full replace** on the kit's `_elementor_page_settings` meta, NOT a merge. If you pass `{"custom_css": "..."}` to the kit's post_id, it replaces the entire settings array with just that one key. Elementor falls back to its global defaults for everything else — colors revert to `#6EC1E4`/`#54595F`/`#7A7A7A`/`#61CE70`, typography reverts to Roboto, every per-heading color is lost, every `__globals__` reference disappears. The site's brand identity is wiped in a single call.
>
> This is the opposite of `update-element` (which DOES partial-merge). The behavior diverges specifically for kit posts.
>
> **The only safe path for kit-level CSS (and any other generic kit write) is `wp eval` read-modify-write.** The single exception is custom palette entries — see below.

### Custom palette entries — the one safe native path (v3)

Verified against v3.1.0: `emcp-tools-update-global-colors` and `emcp-tools-update-global-typography` genuinely **merge**:

- A new item is **appended** to `custom_colors` / `custom_typography`; an existing custom `_id` is **updated in place** (no duplicate). Every other kit key — system slots, per-tag typography, `__globals__` references, `custom_css` — is untouched.
- `update-global-typography` auto-injects `typography_typography: "custom"` into the saved item (required for Elementor to honor the fonts) even when you don't pass it.

**They cannot write system slots.** Passing a system `_id` (e.g. `"primary"`) does NOT update `system_colors` — it appends a duplicate "Primary" entry into `custom_colors`, leaving two entries with colliding `_id`s and ambiguous `__globals__` resolution. If that happens, remove the duplicate via `wp eval` read-modify-write. System slots (Primary/Secondary/Text/Accent, colors AND typography) always go through `wp eval`.

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

The `skill-baseline: applied` comment is the marker future sessions look for. **Always run `wp elementor flush_css` after any kit write** — Elementor caches kit CSS to disk; without flushing, the rules won't render.

**Why this pattern is safe:** it reads the FULL current settings array, modifies only the `custom_css` key in-memory, and writes the WHOLE array back. Every other key is preserved by identity.

## If the kit is already corrupted

Symptoms: headlines render `rgb(122, 122, 122)`, computed `--e-global-color-text` is `#7A7A7A`, `system_colors` shows Elementor defaults like `#6EC1E4`. WordPress post revisions are your safety net:

```bash
# List revisions of the kit
wp post list --post_type=revision --post_parent=$(wp option get elementor_active_kit) \
  --fields=ID,post_modified,post_title

# Inspect a candidate to find the last known-good state
wp eval '
  $rev_id = <REVISION_ID>;
  $ps = get_post_meta($rev_id, "_elementor_page_settings", true);
  echo "Primary: " . $ps["system_colors"][0]["color"] . PHP_EOL;
  echo "h1_color: " . ($ps["h1_color"] ?? "MISSING") . PHP_EOL;
'

# Restore (non-destructive — revision is preserved)
wp eval '
  $good = get_post_meta(<REVISION_ID>, "_elementor_page_settings", true);
  update_post_meta(<KIT_ID>, "_elementor_page_settings", $good);
'
wp elementor flush_css
```

Reapply any legitimate post-corruption changes (like the baseline CSS) on top of the restored settings.

## Opt-out

If the user declines the baseline, append the opt-out marker to the kit's `custom_css` via `wp eval` read-modify-write (same pattern as applying the baseline, but the appended string is `/* skill-baseline: opt-out */`). Future sessions detect the marker and skip the check entirely.

For a per-rule opt-out (user wants some rules but not others), add only the accepted rules plus the `skill-baseline: applied` marker. No finer-grained marker needed.
