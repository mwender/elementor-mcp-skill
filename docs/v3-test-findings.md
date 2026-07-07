# EMCP Tools v3.1.0 — skill battle-test findings

Testing `~/.claude/skills/elementor-mcp` (written against plugin v1.x–2.x) against EMCP Tools v3.1.0 on orra.org.
Format: `- [CONFIRMED|DIVERGED|NEW] <tool or claim> — expected: <what the skill says> — actual: <what happened>`

## 1. Trigger + tool inventory

- [DIVERGED] Skill auto-trigger — expected: skill description says to trigger whenever `mcp__*__emcp-tools-*` tools are available in the session — actual: the skill did NOT auto-trigger at session start despite ~90 `emcp-tools-*` tools being present; it had to be loaded manually via the Skill tool. (Caveat: this session's opening prompt was a meta-task *about* the skill, not a page-building request, which may have suppressed the trigger.)
- [NEW] Session environment — both stdio MCP servers (`orra-elementor`, `orra-wp`) disconnected early in the session and could not be re-loaded via ToolSearch. All testing below was done through an equivalent one-shot stdio JSON-RPC harness (`initialize` → `notifications/initialized` → `tools/call` piped into `wp mcp-adapter serve --server=emcp-tools-server --user=mwender`), which returns the exact same 94 tools the live session had. Tool behavior is identical; only the transport differs. This one-shot pattern is worth documenting in onboarding as a diagnostic/recovery tool.
- [NEW] Plugin bug (benign): every WP load emits two PHP notices on **stderr** — `Ability "emcp-tools/create-theme-template" is already registered` and `Ability "emcp-tools/set-template-conditions" is already registered` (WP 6.9 `WP_Abilities_Registry::register` called incorrectly). Notices go to stderr, not stdout, so they do NOT corrupt the stdio JSON-RPC stream.

### Tool inventory (94 tools on `emcp-tools-server` v3.1.0)

Full list captured via `tools/list`: 3 `core-*` info tools + 91 `emcp-tools-*` tools.

**Disabled-by-default mechanism (NEW, not in skill):** v3 has an `emcp_tools_disabled_tools` wp option (and an `emcp_tools_active_modules` option, default `["prompts","brand-kits","templates","themer"]`). Out of the box the disabled list contains 39 abilities: all destructive file/DB/plugin/theme/user tools (`write-file`, `edit-file`, `delete-file`, `insert-row`, `update-rows`, `delete-rows`, `install-plugin`, `activate/deactivate/update/delete-plugin`, `install/switch/update/delete-theme`, `create-user`, `update-user`, `delete-media`), PHP/code-snippet tools, popup tools — **and, critically for the skill: `add-pro-widget`, `create-theme-template`, `set-template-conditions`, `list-dynamic-tags`, `set-dynamic-tag`, `add-custom-css`.**

Diff vs the skill's "Key tool reference" table (old `elementor-mcp-*` → new `emcp-tools-*`):

- [CONFIRMED] Present unchanged (prefix swap only): `get-global-settings`, `create-page`, `update-page-settings`, `add-container`, `update-container`, `get-widget-schema`, `get-element-settings`, `get-page-structure`, `find-element`, `update-element`, `remove-element`, `upload-svg-icon`, `build-page`, `get-container-schema`.
- [CONFIRMED] Per-widget `add-*` tools removed — expected: skill's v3 note says `add-heading`/`add-image`/etc. no longer exist, replaced by `list-widgets` → `get-widget-schema` → `add-free-widget`/`add-pro-widget` — actual: confirmed; no per-classic-widget add tools exist. (`add-atomic-heading` etc. exist but are for Elementor v4 *atomic* widgets, a different thing.)
- [DIVERGED] `add-custom-css` — expected: skill's escape-hatch workhorse, listed in Key tool reference — actual: exists as an ability but is **disabled by default** in v3.1 (`emcp_tools_disabled_tools`). Not callable until enabled in EMCP settings. The skill must document this gate.
- [DIVERGED] `set-dynamic-tag` + `list-dynamic-tags` — expected: documented as the standard dynamic-tag flow — actual: both **disabled by default** in v3.1. The copyright-year recipe fails out of the box until the tools are enabled.
- [DIVERGED] `create-theme-template` — expected: the documented tool for Maintenance Mode / library templates — actual: **disabled by default** (and its registration is the source of one of the duplicate-ability notices).
- [DIVERGED] `add-pro-widget` — expected: skill's v3 note says "use `add-free-widget` or `add-pro-widget`" — actual: `add-pro-widget` is **disabled by default**; only `add-free-widget` is callable out of the box (even on this site with Elementor Pro 4.1.0 active).
- [NEW] Tools that postdate the skill (present + enabled): element ops `move-element`, `reorder-elements`, `duplicate-element`, `batch-update`, `delete-page-content`, `export-page`; catalog `list-widgets`, `add-free-widget`, `update-widget`; atomic-widget family (`add-atomic-heading/-paragraph/-button/-image/-svg/-video/-youtube/-divider/-widget`, `update-atomic-widget`, `add-div-block`, `add-flexbox`); Gutenberg block family (`add-block`, `update-block`, `remove-block`, `move-block`, `duplicate-block`, `get-block-schema`, `get-post-blocks`, `list-blocks`, `insert-pattern`, `list-patterns`); templates (`apply-template`, `import-template`, `save-as-template`, `resolve-template`, `list-templates`, `list-theme-templates`, `get/update/delete-theme-template`, `list-condition-targets`); kit writes (`update-global-colors`, `update-global-typography`, `list-global-classes`); media (`get/list/update-media`, `search-images`, `add-stock-image`, `sideload-image`); WP admin surface (`create/get/update/delete-post`, `list-posts`, `list-pages`, `list-post-types`, `list-taxonomies`, `set-post-terms`, `get-user`, `list-users`, `list-plugins`, `search-plugins`, `list-themes`, `search-themes`, `get-settings`, `update-settings`); read-only FS/DB (`read-file`, `list-directory`, `search-files`, `query`, `describe-table`, `list-tables`); diagnostics (`detect-elementor-version`, `analyze-performance`, `scan-security`); `add-custom-js`; `core-get-environment-info`, `core-get-site-info`, `core-get-user-info`.
- [NEW] Site plugin state during test: elementor 4.1.1, elementor-pro 4.1.0, emcp-tools 3.1.0, WP with Abilities API (6.9).

## 2. Two-step widget pattern mechanics

Test page: post 4465 (draft → published for frontend checks, deleted at cleanup). Elementor 4.1.1 + Pro 4.1.0.

- [DIVERGED] `add-free-widget` styling-arg rejection — expected: skill says `add-*` validators are strict and "reject most styling args" (e.g. `add-heading` rejects `title_color`), forcing the two-step add-then-update pattern — actual: `add-free-widget` with `widget_type:"heading"` **accepted and stored** `title_color`, `align`, `typography_typography`, `typography_font_size`, and `_margin` in the add call, verbatim (verified via `get-element-settings`). Its schema says "Any valid Elementor control passes through." The two-step pattern is now **optional** in v3 — styling at add time works. (Two-step may still be good practice for readability, but the skill's core justification — validator rejection — is gone.)
- [DIVERGED] `get-widget-schema` curated mode — expected: (by the skill's strict-add model) curated params would be content keys only — actual: curated `heading` schema includes the full styling surface: `title_color`, `title_hover_color`, `align`, `blend_mode`, complete `typography_*` family, `text_stroke_*`, `title_text_shadow_text_shadow`. Content-only assumption is wrong. `full:true` returns the raw auto-generated control schema (~everything, incl. `_element_width`, `_margin` implied via Advanced controls, `motion_fx_*`, `_css_classes`, etc.), with useful per-control notes ("Toggle switch. Use \"yes\" to enable"). New in v3: `types:[...]` batch lookup param.
- [CONFIRMED] `update-element` — expected: partial merge, full long-tail styling surface, no strict validation — actual: confirmed. Merged `align`, `title_color`, `typography_*`, `_margin` into existing settings without touching other keys; also silently stored a deliberately bogus key (`totally_made_up_key: "zzz"`) — no validation, exactly the "guessing is silently accepted and silently does nothing" behavior the skill warns about. Works on widgets and containers (`element_type` echoed in response).
- [NEW] `update-widget` vs `update-element` — `update-widget` (new in v3) has identical merge semantics ("Settings are merged (partial update)") and identical non-validation (stored a bogus key too). Only observed difference: it is **type-guarded** — calling it on a container returns error "Target element is not a widget." `update-element` remains the universal tool; `update-widget` is a widgets-only alias with a type check. Same styling payload produced identical stored results on the same widget.
- [DIVERGED] `create-page` + `_elementor_edit_mode` — expected: skill step 5 says you must manually run `wp post meta update <id> _elementor_edit_mode builder` after create-page or the page renders with the block editor — actual: v3 `create-page` already sets `_elementor_edit_mode: builder` (verified via post meta immediately after creation). Manual step obsolete. Response now returns `edit_url` + `preview_url`.

## 3. Silent-store trap spot-checks (Elementor-core behaviors)

All verified on the live frontend via agent-browser eval. **Measurement caveat (NEW, affects the skill's diagnostics recipes):** on `e-con-boxed` containers under Elementor 4.1, flex/grid/align/gap/block-padding apply to the child `.e-con-inner` node, NOT the outer `.elementor-element-<id>` node. Evals that read `getComputedStyle` on the outer element report `align-items: normal`, `row-gap: normal`, wrong grid templates — query `:scope > .e-con-inner` (fall back to the element itself for full-width containers).

- [CONFIRMED] `flex_gap` works / `gap` is stored-but-ignored — container with `flex_gap` row 32 rendered `row-gap: 32px`; container with `gap` row 40 stayed at the 20px kit default. Detail update: in Elementor 4.1 `flex_gap` emits `--gap`/`--row-gap` on the container (kit default flows through `--widgets-spacing`); the skill's claim that it emits `--widgets-spacing-row/column` reflects older Elementor — symptom and rule unchanged, variable names in the diagnostics text are stale.
- [CONFIRMED] `padding` works / `_padding` is stored-but-ignored on containers — `padding: 77px` rendered 77px block padding (`--padding-top/bottom` + logical `--padding-block-*` vars all 77px); `_padding: 77px` container stayed at the 10px default. Symptom as documented.
- [CONFIRMED] `_element_width: "initial"` + `_element_custom_width` — heading rendered exactly 222px wide. `_element_width: "custom"` with the same custom width rendered full-width (1120px), i.e. broken as documented.
- [CONFIRMED] `grid_rows_grid` omission defaults to 2 rows — grid with only `grid_columns_grid: "repeat(3, 1fr)"` got `--e-con-grid-template-rows: repeat(2, 1fr)` (rendered two row tracks). Passing the object `{"unit":"fr","size":1,"sizes":[]}` produced `repeat(1, 1fr)` (one track). Also [CONFIRMED]: `grid_columns_grid` accepts the CSS string form — 3 columns rendered.
- [CONFIRMED] flex-column container without explicit `flex_align_items` gets `"center"` injected — visible both in stored settings immediately after `add-container` (injection happens at add time, so it's plugin behavior that survived the v3 rewrite) and as rendered `align-items: center` on the inner node.

## 4. Kit safety

Kit = post 6 (`elementor_active_kit`). Full `_elementor_page_settings` backed up via `wp eval` before any write; restored byte-identical after testing. `update-page-settings` was NOT called on the kit (per the skill's destructive-full-replace warning — not re-verified).

- [NEW] `emcp-tools-update-global-colors` **MERGES — safe for custom colors.** Sending a single new color object left all 4 system colors, all 8 existing custom colors, and every other kit key (typography, `__globals__`, Hello footer text, etc.) intact; the new color was appended to `custom_colors`. Sending an existing custom `_id` updates that entry **in place** (no duplicate). This can replace the skill's "wp eval read-modify-write only" rule for *custom-palette* writes.
- [DIVERGED/TRAP] `update-global-colors` cannot write SYSTEM slots — sending `_id: "primary"` did NOT update `system_colors[primary]`; it **appended a duplicate entry** `{_id:"primary", title:"Primary"}` into `custom_colors`. The palette then contains two "Primary" entries with colliding `_id`s (potential `__globals__` resolution ambiguity). System slots (Primary/Secondary/Text/Accent) still require the `wp eval` read-modify-write path.
- [NEW] `emcp-tools-update-global-typography` **MERGES** the same way: one new item appended to `custom_typography`; `system_typography`, per-tag kit settings (`h2_typography_*` etc.) and everything else untouched. Bonus: it auto-injects `typography_typography: "custom"` into the saved item (required for Elementor to honor the fonts) even when not passed. Not tested: whether a system `_id` (e.g. "primary") duplicates into custom_typography like colors do — assume it does, same code path.
- [CONFIRMED] `get-global-settings` — response shape matches the skill: top-level convenience `colors`/`typography` (system slots) plus full `settings` (system_colors, custom_colors, system_typography, custom_typography, kit page settings). Note: on this site the kit has NO `custom_css` key at all — the skill's step-3 baseline check must handle the key being absent, not just missing the marker.

## 5. Onboarding facts (verbatim)

- `wp mcp-adapter list` output (plus the two stderr PHP notices noted in §1):

  ```
  ID	Name	Version	Tools	Resources	Prompts
  mcp-adapter-default-server	MCP Adapter Default Server	v1.0.0	3	0	0
  emcp-tools-server	MCP Tools for Elementor Server	v3.1.0	94	0	0
  ```

- [DIVERGED] vs skill onboarding doc — expected: two-plugin setup (`wordpress/mcp-adapter` + `msrbuilds/elementor-mcp`) registering an `elementor-mcp-server` — actual: single plugin `emcp-tools` 3.1.0 (adapter bundled since v1.7.4); server ID is `emcp-tools-server`; a second `mcp-adapter-default-server` (3 tools: `mcp-adapter-discover-abilities` / `execute-ability` / `get-ability-info`) is also registered by the bundled adapter. Composer package on this site: installed as `emcp-tools` plugin (no separate mcp-adapter plugin in `wp plugin list`).
- Working `.mcp.json` for this session (no credentials involved — stdio runs as a WP user via WP-CLI):

  ```json
  {
    "mcpServers": {
      "orra-wp": {
        "type": "stdio",
        "command": "wp",
        "args": ["mcp-adapter", "serve", "--server=mcp-adapter-default-server", "--user=mwender",
                 "--path=/Users/mwender/webdev/laravel-valet/bedrock/orra.org/web/wp"]
      },
      "orra-elementor": {
        "type": "stdio",
        "command": "wp",
        "args": ["mcp-adapter", "serve", "--server=emcp-tools-server", "--user=mwender",
                 "--path=/Users/mwender/webdev/laravel-valet/bedrock/orra.org/web/wp"]
      }
    }
  }
  ```

- [CONFIRMED] WP-CLI form is `--server=emcp-tools-server` (used for every call in this test run).
- [CONFIRMED] HTTP route — `https://orra.test/wp-json/mcp/emcp-tools-server` exists (returns 401 `rest_forbidden` unauthenticated, i.e. route registered + auth-gated). Legacy `/wp-json/mcp/elementor-mcp-server` returns 404, confirming the skill's "every AI client must reconnect after updating" note.

## 6. New-surface notes

- [NEW] `move-element` — moves an element to a new parent and/or position (`post_id`, `element_id`, `target_parent_id` — empty string for top-level, `position` — `-1` appends). Directly useful for the skill's HTML-refactoring recipe (re-parenting without rebuild).
- [NEW] `reorder-elements` — reorders a container's children by passing the full ordered array of child IDs (all must be direct children).
- [NEW] `duplicate-element` — duplicates an element incl. all children with fresh IDs, placed immediately after the original. (Card-grid recipe: build one card, duplicate N times, update each.)
- [NEW] `batch-update` — array of `{element_id, settings}` merge operations in ONE save. Tested live: 2 widgets updated in one call, response `{"success":true,"updated":2,"failed":[]}` (per-op failure reporting). Should replace loops of `update-element` in every skill recipe — one save = one revision, less CSS churn.
- [CONFIRMED] `set-dynamic-tag` — unchanged from the skill's documentation once enabled. Same signature (`post_id`, `element_id`, `setting_key`, `tag_name`, `tag_settings`). The copyright-year recipe (`current-date-time`, `date_format:"custom"`, `custom_format:"Y"`, `before`/`after`) rendered "© 2026 ORRA. All rights reserved." on the frontend on the first try. Stored form is the documented `__dynamic__` elementor-tag shortcode. CAVEAT (see §1): tool is disabled by default in v3.1 — must be enabled in EMCP settings (or by removing `emcp-tools/set-dynamic-tag` from the `emcp_tools_disabled_tools` option; the tool list updates immediately, no cache). `list-dynamic-tags` (also disabled by default) works and returns name/title/group/categories per tag.
- [NEW] Gutenberg block tools present: `add-block`, `update-block`, `remove-block`, `move-block`, `duplicate-block`, `get-block-schema`, `get-post-blocks`, `list-blocks`, `insert-pattern`, `list-patterns`. They did NOT cause confusion during the Elementor build (naming is clearly parallel: `*-block` vs `*-element`/`*-widget`). The likelier v3 confusion source is the `add-atomic-*` family (Elementor v4 atomic widgets — `add-atomic-heading`, `add-div-block`, `add-flexbox`, etc.), which sits right next to `add-free-widget` in the tool list; the skill rewrite should say explicitly when atomic tools apply (Elementor v4 experiments) vs classic widgets.
- [NEW] `delete-post` soft-deletes (`"deleted":"trashed"`) — cleanup still needs `wp post delete --force` for a hard delete.

## Cleanup

- Throwaway page 4465 hard-deleted; kit 6 `_elementor_page_settings` restored from backup (verified: 8 custom colors, 1 custom typography, system primary #CD010D); `emcp_tools_disabled_tools` restored to original 39 entries; Elementor CSS cache cleared.
