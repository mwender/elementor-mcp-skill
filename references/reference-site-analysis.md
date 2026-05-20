# Analyzing a reference site with agent-browser

When given a reference site to replicate, **do not rely solely on screenshots**. Screenshots show what a page looks like but hide what it actually is — font names become "looks serif," colors become "dark navy-ish," spacing becomes a guess. `agent-browser` exposes the live DOM and computed styles, giving exact values.

Use this workflow before building any section.

## Step 1 — Open the reference site

```bash
agent-browser open https://example-reference.com/
```

## Step 2 — `snapshot` for complete content

`snapshot` returns a full semantic text outline of the page: every heading, paragraph, link text, nav item, and image alt, with `@eN` refs for interaction. Use this to capture all copy verbatim — headings, body text, CTA labels — so nothing gets paraphrased during the build.

```bash
agent-browser snapshot
```

Example output excerpt:
```
- main:
  - paragraph: A New Standard in Healthcare
  - heading "Medicine Designed For You Alone" [ref=e8] [level=1]
  - paragraph: Our precision medicine program goes far beyond routine care...
  - link "Explore the Program" [ref=e9]
  - heading "Six Pillars of Comprehensive Care" [ref=e12] [level=2]
  - heading "Comprehensive Health Evaluation" [ref=e13] [level=3]
  - paragraph: An in-depth review of your past, present...
```

This gives you the full content map without opening DevTools.

## Step 3 — `eval` for computed design tokens

`eval` runs arbitrary JavaScript in the browser context and returns the result. Use it to pull computed styles directly off elements — the browser has already resolved all CSS variables, inheritance, and cascade for you.

### Extract global design tokens (run once per site)

```bash
agent-browser eval "
JSON.stringify({
  body: {
    bg: getComputedStyle(document.body).backgroundColor,
    color: getComputedStyle(document.body).color,
    font: getComputedStyle(document.body).fontFamily,
    size: getComputedStyle(document.body).fontSize,
    lineHeight: getComputedStyle(document.body).lineHeight
  },
  h1: (() => { const el = document.querySelector('h1'); if (!el) return null; const s = getComputedStyle(el); return { font: s.fontFamily, size: s.fontSize, weight: s.fontWeight, color: s.color, lineHeight: s.lineHeight }; })(),
  h2: (() => { const el = document.querySelector('h2'); if (!el) return null; const s = getComputedStyle(el); return { font: s.fontFamily, size: s.fontSize, weight: s.fontWeight, color: s.color }; })(),
  h3: (() => { const el = document.querySelector('h3'); if (!el) return null; const s = getComputedStyle(el); return { font: s.fontFamily, size: s.fontSize, weight: s.fontWeight, color: s.color }; })()
}, null, 2)
"
```

### Extract page section structure

```bash
agent-browser eval "
JSON.stringify(
  [...document.querySelectorAll('section, main > div')].slice(0, 10).map(el => {
    const s = getComputedStyle(el);
    const h = el.querySelector('h1,h2,h3');
    return {
      class: el.className.slice(0, 80),
      bg: s.backgroundColor,
      padding: s.padding,
      heading: h ? h.textContent.trim().slice(0, 60) : null
    };
  }), null, 2)
"
```

### Extract the color palette

```bash
agent-browser eval "
const colors = new Set();
document.querySelectorAll('*').forEach(el => {
  const s = getComputedStyle(el);
  ['backgroundColor','color','borderColor'].forEach(p => {
    const v = s[p];
    if (v && v !== 'rgba(0, 0, 0, 0)' && v !== 'rgb(0, 0, 0)') colors.add(v);
  });
});
JSON.stringify([...colors].slice(0, 24))
"
```

### Inspect a specific section

For any section in the `snapshot` output, target it by ref or selector:

```bash
# By ref from snapshot output
agent-browser eval "
const el = document.querySelector('[data-ref=\"e12\"]') || document.querySelectorAll('section')[2];
const s = getComputedStyle(el);
JSON.stringify({ bg: s.backgroundColor, padding: s.padding, font: s.fontFamily, color: s.color })
"

# By semantic selector
agent-browser eval "
const cta = document.querySelector('section:last-of-type');
const s = getComputedStyle(cta);
const btn = cta.querySelector('a, button');
const bs = btn ? getComputedStyle(btn) : null;
JSON.stringify({
  sectionBg: s.backgroundColor,
  sectionPadding: s.padding,
  btn: bs ? { bg: bs.backgroundColor, color: bs.color, font: bs.fontFamily, size: bs.fontSize, weight: bs.fontWeight, padding: bs.padding, borderRadius: bs.borderRadius } : null
}, null, 2)
"
```

## When to use screenshots vs eval+snapshot

| Task | Best tool |
|---|---|
| Understanding reference site content (headings, copy, CTA text) | `snapshot` |
| Extracting exact colors, fonts, sizes from reference | `eval` |
| QA of what you've just built — visual layout check | `screenshot` |
| Checking responsive behavior at a breakpoint | `set viewport` + `screenshot` |
| Finding a specific interactive element to click | `snapshot` (get `@eN` ref), then `click @eN` |
| Verifying a CSS rule fired (which rule is winning) | `eval` — `getComputedStyle` + cascade inspection |

**Rule of thumb:** use eval+snapshot to *understand what to build*; use screenshots to *verify what you built*.

## Converting RGB to hex

`getComputedStyle` returns `rgb()` values, not hex. Convert inline:

```bash
agent-browser eval "
function toHex(rgb) {
  const [r,g,b] = rgb.match(/\d+/g).map(Number);
  return '#' + [r,g,b].map(n => n.toString(16).padStart(2,'0')).join('').toUpperCase();
}
const bg = getComputedStyle(document.querySelector('header')).backgroundColor;
toHex(bg)
"
```

Or convert a batch of colors collected from the palette query above using the same `toHex` function.

## Scroll to reveal lazy-loaded sections

Some sites lazy-load sections below the fold. If `snapshot` shows fewer sections than expected, scroll first:

```bash
agent-browser scroll down
agent-browser eval "document.querySelectorAll('section').length"
agent-browser snapshot  # re-run after scroll to capture revealed content
```
