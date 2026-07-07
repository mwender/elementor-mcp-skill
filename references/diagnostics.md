# Diagnostic agent-browser eval recipes

When a screenshot suggests something's wrong, screenshots themselves are a poor diagnostic tool — they tell you "this looks gray" but not *which CSS rule* is making it gray. The recipes below use `agent-browser eval` to ask the browser what it's actually computing. Use them aggressively. They turn a half-hour "why is this broken" investigation into a 30-second answer.

## Rule zero: measure the right node on containers

On `e-con-boxed` containers (Elementor 4.x), the flex/grid/gap/align/padding styles land on the child `.e-con-inner` node, NOT the outer `.elementor-element-<id>` node. Reading `getComputedStyle` on the outer element reports `align-items: normal`, `row-gap: normal`, and wrong grid templates — a false "the setting didn't take" signal. Full-width containers have no `.e-con-inner`, so always use the fallback pattern:

```js
const el = document.querySelector('.elementor-element-<id>');
const target = el.querySelector(':scope > .e-con-inner') || el;
getComputedStyle(target)...
```

Related version note: Elementor 4.x emits container gap as `--gap` / `--row-gap` (kit default flows through `--widgets-spacing`); older versions used `--widgets-spacing-row/column`. The `flex_gap`-not-`gap` rule in SKILL.md is unaffected — only the variable names differ when you're inspecting.

## Does my selector match anything?

```bash
agent-browser eval "document.querySelectorAll('SELECTOR').length"
```

The single most useful one-liner in this skill. Validate every CSS selector you intend to write to a kit's `custom_css` against real markup BEFORE you commit it. A return of `0` on a page with the target widget type means your selector is wrong — fix it before anything hits the kit. This is the check that catches descendant vs direct-child errors, missing `.elementor-widget-container` wrappers, typos, etc.

## What color is this element actually rendering?

```bash
agent-browser eval "getComputedStyle(document.querySelector('h1.elementor-heading-title')).color"
```

Returns the computed RGB string. Interpretation:
- `rgb(255, 255, 255)` = solid white ✓
- `rgb(122, 122, 122)` = `#7A7A7A` (Elementor's default text fallback — kit colors have been wiped, see [If the kit is already corrupted](../SKILL.md#if-the-kit-is-already-corrupted))
- `rgba(255, 255, 255, 0.5)` = transparency issue

## What does an Elementor global CSS variable resolve to?

```bash
agent-browser eval "getComputedStyle(document.body).getPropertyValue('--e-global-color-text').trim()"
```

Elementor's global colors and typography are exposed as CSS custom properties on `.elementor-kit-<id>` (inherited by `body`). If the resolved value disagrees with `get-global-settings`, the generated kit CSS file is out of sync with kit data — usually means a flush is needed (`wp elementor flush_css`).

Common variables worth checking:

| Variable | Should match kit's |
|---|---|
| `--e-global-color-primary` | Primary color |
| `--e-global-color-secondary` | Secondary color |
| `--e-global-color-text` | Text color |
| `--e-global-color-accent` | Accent color |
| `--e-global-typography-primary-font-family` | Primary typography font family |

## Which stylesheet rule is winning the cascade for a property?

```bash
agent-browser eval "(() => {
  const el = document.querySelector('SELECTOR');
  const rules = [];
  for (const sheet of document.styleSheets) {
    try {
      for (const rule of sheet.cssRules || []) {
        if (rule.style && rule.style.color && el.matches(rule.selectorText)) {
          rules.push({selector: rule.selectorText, color: rule.style.color, source: sheet.href || 'inline'});
        }
      }
    } catch(e) {}
  }
  return JSON.stringify(rules, null, 2);
})()"
```

Walks every loaded stylesheet, filters to rules whose selector matches the target element AND that set `color`, returns the list. Swap `color` for `marginBottom`, `fontFamily`, etc. to debug other properties. This is what surfaced the kit-corruption smoking gun (the generated `post-4.css` setting `--e-global-color-text: #7A7A7A`).

## Where is a CSS variable being set across all stylesheets?

```bash
agent-browser eval "(() => {
  const matches = [];
  for (const sheet of document.styleSheets) {
    try {
      for (const rule of sheet.cssRules || []) {
        if (rule.cssText?.includes('--e-global-color-text')) {
          matches.push({selector: rule.selectorText, source: sheet.href || 'inline', preview: rule.cssText.substring(0, 200)});
        }
      }
    } catch(e) {}
  }
  return JSON.stringify(matches, null, 2);
})()"
```

Pinpoints definitions of a CSS variable across the page's loaded stylesheets. Essential when diagnosing whether a variable's wrong value is coming from the kit, a widget, or a third-party stylesheet.

## Full computed style for a widget (one JSON blob)

```bash
agent-browser eval "(() => {
  const cs = getComputedStyle(document.querySelector('SELECTOR'));
  return JSON.stringify({
    color: cs.color, marginBottom: cs.marginBottom, marginTop: cs.marginTop,
    fontFamily: cs.fontFamily, fontSize: cs.fontSize, fontWeight: cs.fontWeight,
    lineHeight: cs.lineHeight, textAlign: cs.textAlign,
  });
})()"
```

One call returns the typography/spacing values that matter most. Useful as the first move when "something looks off" and you're not sure what specifically.
