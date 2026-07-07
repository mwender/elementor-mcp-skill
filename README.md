# elementor-mcp-skill

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) for fluent page-building on WordPress sites that use Elementor / Elementor Pro via [EMCP Tools](https://github.com/msrbuilds/elementor-mcp) (the MCP plugin formerly named "Elementor MCP"). The skill teaches Claude to build and refactor pages with **native Elementor widgets** — image, heading, text-editor, icon-box, container, button, divider, icon-list — instead of falling back to HTML widgets, so the resulting page remains fully editable from the Elementor panel UI.

The skill targets EMCP Tools v3+ (`mcp__*__emcp-tools-*` tools) and auto-triggers when those — or the legacy `mcp__*__elementor-mcp-*` tools (pre-3.0, with a documented detect-and-upgrade path) — are available in a session, or when the user mentions building or editing pages on a WordPress site that uses Elementor.

## Who it's for

- Developers, agencies, and operators building WordPress sites with Elementor or Elementor Pro
- Anyone running the [WordPress MCP Adapter](https://github.com/WordPress/mcp-adapter) plus the [Elementor MCP plugin](https://github.com/msrbuilds/elementor-mcp) alongside Claude Code
- Sites with the Hello Elementor theme (or similar lean theme stack) — though the patterns apply broadly

If your WordPress stack is Gutenberg/blocks or a fully custom theme without Elementor, this skill isn't for you.

## What it covers

Pulled from the skill's main sections — full detail in [`SKILL.md`](SKILL.md):

- **The catalog widget flow (v3)** — `list-widgets` → `get-widget-schema` → `add-free-widget` with content and styling in one call, then `update-element`/`batch-update` for iteration, so all styling decisions land in Elementor's native data model and stay editable from the Style tab (legacy pre-3.0 plugins use the two-step `add-*` + `update-element` pattern, also documented)
- **The v3 tool gate** — 39 abilities ship disabled by default in v3.1, including six the skill relies on (`add-custom-css`, `set-dynamic-tag`, `add-pro-widget`, …); the skill documents the check and the enable flow
- **Native widget mapping** — which built-in widget corresponds to which design pattern, and when HTML widgets are actually appropriate (rarely)
- **Common flex traps** — `content_width: "boxed"` collapsing flex layouts, `_flex_grow` siblings starving the badge, column containers defaulting to `flex_align_items: "center"` and silently centering text
- **Image-as-badge** — turning the image widget itself into a circular gradient icon, no wrapper container required
- **SVG icon onboarding** — the safe-svg plugin's role-gate (active but quiet until configured) and the one-liner fix
- **Visual-review workflow** — `agent-browser` for desktop-viewport screenshots, plus six diagnostic `eval` recipes for when "it looks wrong but I don't know why"
- **Kit-level CSS conventions** — defensive baseline rules (paragraph margin trim, accessibility focus rings) with a propose-and-confirm check on new sites and an opt-out marker for sites that decline
- **Recovery from kit corruption** — what to do when `update-page-settings` on a kit post wipes the brand kit (it shouldn't, but it does — the skill documents the trap, the symptoms, and the revision-based recovery)
- **Onboarding new sites** — plugin installation for both standard WordPress and Bedrock/Composer stacks, `.mcp.json` templates for STDIO and HTTP transports, a first-session checklist, and starter memory-file templates

## Installation

Clone the repo directly into your Claude Code skills directory:

```bash
git clone https://github.com/mwender/elementor-mcp-skill.git ~/.claude/skills/elementor-mcp
```

Or clone anywhere and symlink:

```bash
git clone https://github.com/mwender/elementor-mcp-skill.git ~/code/elementor-mcp-skill
ln -s ~/code/elementor-mcp-skill ~/.claude/skills/elementor-mcp
```

Verify the install: start Claude Code in any project and confirm `elementor-mcp` appears in your available skills list. The skill is now active on every Elementor MCP session you start.

## How it works under the hood

Claude Code skills use a three-tier loading model. This skill is structured to match:

```
elementor-mcp/
├── SKILL.md              # ~475 lines — core patterns, loaded into context when the skill triggers
└── references/           # loaded on-demand, one Read call away
    ├── onboarding.md     # new-site setup (install, .mcp.json, tool gate, memory templates)
    ├── diagnostics.md    # visual-debugging eval recipes
    ├── kit-css.md        # kit baseline CSS + safe kit-write paths + corruption recovery
    ├── form-widget.md    # Pro form widget styling, webhooks, phone masking
    ├── dynamic-tags.md   # copyright-year + ACF dynamic-tag recipes
    ├── layout-patterns.md, maintenance-mode.md, reference-site-analysis.md, site-settings.md
```

`SKILL.md` stays lean by extracting deep-dive content into `references/`. Claude reads those reference files only when `SKILL.md` explicitly links to them, so per-session token cost stays small while the full detail is one Read tool call away when needed.

## Background

This skill was built while battle-testing the Elementor MCP during work on a real production website. Every pattern, anti-pattern, and recovery procedure documented here came from something that actually happened — a styling argument got rejected by a strict validator, a flex layout silently collapsed when child widths met a `content_width: "boxed"` default, the kit's brand colors were wiped by a tool call that should have been a merge but was actually a replace.

The v3 (EMCP Tools) adaptation followed the same standard: every v3-specific claim was verified in a dedicated battle-test session against a live 3.1.0 site — the raw findings are preserved in [`docs/v3-test-findings.md`](docs/v3-test-findings.md).

Capturing those lessons in a skill means future Claude sessions on Elementor sites don't have to re-learn them. The skill is deliberately opinionated — it pushes hard toward native Elementor widgets and against HTML-widget shortcuts, because the latter undermines the entire reason for using Elementor in the first place: leaving the resulting page editable from the panel UI by humans who don't (and shouldn't have to) read code.

## Contributing

Issues and PRs welcome. The most valuable contributions are real-world traps and patterns not yet documented — particularly:

- Verified plugin install flow on Bedrock (documented but not yet battle-tested in a Claude session)
- Patterns for Elementor Theme Builder templates (header / footer / single / archive / loop-item) — the skill currently covers page builds only
- Additional kit-baseline CSS rules with broad applicability across Elementor sites
- Edge cases in the catalog flow (`add-free-widget` / `update-element`) for widget types not yet exercised

If you hit something the skill should have caught and didn't, that's a high-priority issue. The skill's working principle: each documented trap is one a previous session walked into so future sessions don't have to.

## License

MIT
