# Elementor Pro Form Widget — Styling & Patterns

## Native style controls first, CSS last

The form widget has a rich set of native style controls accessible via `update-element`. Use these before reaching for `add-custom-css`.

### Label controls

```
label_color                        # text color (hex/rgba)
label_spacing                      # gap between label and field: {size, unit}
label_typography_typography        # "custom" to unlock
label_typography_font_family       # e.g. "Inter"
label_typography_font_size         # {size, unit}
label_typography_font_weight       # "400", "600", etc.
label_typography_text_transform    # "uppercase", "none", etc.
label_typography_letter_spacing    # {size, unit} — use px for precision (e.g. 1.2px)
label_typography_line_height       # {size, unit}
```

### Field (input/select/textarea) controls

```
field_text_color                   # input text color
field_background_color             # input background
field_border_border                # "solid" — MUST set this first to enable border styling
field_border_color                 # border color (e.g. "#DADEE7" for a soft gray)
field_border_width                 # {top, right, bottom, left, unit, isLinked}
field_border_radius                # {top, right, bottom, left, unit, isLinked}
field_typography_typography        # "custom"
field_typography_font_family       # e.g. "Inter"
field_typography_font_size         # {size, unit}
input_size                         # "xs"|"sm"|"md"|"lg"|"xl" — controls field height/padding
```

### Spacing controls

```
row_gap                            # vertical spacing between field rows: {size, unit}
column_gap                         # horizontal spacing between fields in the same row: {size, unit}
```

### Button controls (in addition to content-tab settings)

```
button_text_color                  # normal state text color
button_hover_color                 # hover state text color
button_hover_border_color          # hover state border color
button_hover_animation             # animation name string
button_typography_font_family
button_typography_font_size
button_typography_font_weight
button_typography_letter_spacing
button_typography_text_transform   # "uppercase", "none", etc.
button_border_radius               # {top, right, bottom, left, unit, isLinked}
```

### Widget Advanced (card) settings

The form widget itself supports Elementor's standard Advanced-tab controls. Use these to give the form a card treatment — background color, border, and inner padding — without wrapping it in a separate container.

```
_background_background             # "classic" to enable solid background
_background_color                  # card background color (hex/rgba)
_border_border                     # "solid" — MUST set to enable border styling
_border_color                      # card border color
_border_width                      # {top, right, bottom, left, unit, isLinked}
_border_radius                     # {top, right, bottom, left, unit, isLinked}
_padding                           # inner padding: {top, right, bottom, left, unit, isLinked}
```

> ⚠️ `_border_border: "solid"` must be set alongside `_border_color`, same rule as field borders. And note that `_padding` here (on the widget's Advanced tab) IS correctly rendered — this is distinct from container `padding` vs `_padding`: on widgets, `_padding` is the right key for the Advanced padding control.

> ⚠️ `field_border_border: "solid"` must be set alongside `field_border_color`. Setting color alone without the border type has no effect — Elementor skips the CSS output entirely.

## Reference-site form inspection workflow

Before styling a form to match a reference, extract computed values from the live reference DOM. **Use these exact eval recipes — do not write a hand-crafted subset.** Missing a single property (e.g. `textTransform`) means missing a styling detail you'll have to hand-correct later.

Run all three `agent-browser eval` calls:

```js
// 1. Field, label, and wrapper computed styles
(function() {
  var input = document.querySelector('input[type=text]');
  var label = document.querySelector('label');
  var s = window.getComputedStyle(input);
  var ls = window.getComputedStyle(label);
  return JSON.stringify({
    field: {
      borderColor: s.borderColor, borderWidth: s.borderWidth,
      borderRadius: s.borderRadius, background: s.backgroundColor,
      paddingTop: s.paddingTop, paddingLeft: s.paddingLeft,
      fontSize: s.fontSize, color: s.color, height: s.height
    },
    label: {
      fontSize: ls.fontSize, fontWeight: ls.fontWeight, color: ls.color,
      textTransform: ls.textTransform, letterSpacing: ls.letterSpacing,
      marginBottom: ls.marginBottom
    }
  });
})()

// 2. Select, textarea, submit button — full property set
(function() {
  var sel = document.querySelector('select');
  var ta  = document.querySelector('textarea');
  var btn = document.querySelector('button[type=submit], input[type=submit]');
  var ss = sel ? window.getComputedStyle(sel) : {};
  var ts = ta  ? window.getComputedStyle(ta)  : {};
  var bs = btn ? window.getComputedStyle(btn) : {};
  return JSON.stringify({
    select:   { borderColor: ss.borderColor, height: ss.height, paddingLeft: ss.paddingLeft },
    textarea: { borderColor: ts.borderColor, paddingTop: ts.paddingTop },
    button:   {
      background: bs.backgroundColor, color: bs.color,
      fontSize: bs.fontSize, fontWeight: bs.fontWeight,
      letterSpacing: bs.letterSpacing, textTransform: bs.textTransform,
      borderRadius: bs.borderRadius, padding: bs.padding
    }
  });
})()

// 3. Form wrapper / card — walk up from <form> to find the first ancestor with a background
// The <form> element itself is often transparent; the card is on a parent wrapper.
(function() {
  var form = document.querySelector('form');
  var el = form ? form.parentElement : null;
  while (el && el !== document.body) {
    var s = window.getComputedStyle(el);
    if (s.backgroundColor !== 'rgba(0, 0, 0, 0)' && s.backgroundColor !== 'transparent') {
      return JSON.stringify({
        tag: el.tagName,
        className: el.className.slice(0, 80),
        background: s.backgroundColor,
        border: s.border,
        borderRadius: s.borderRadius,
        padding: s.padding
      });
    }
    el = el.parentElement;
  }
  return JSON.stringify({ cardFound: false });
})()
```

> ⚠️ **Always run eval #3.** Checking `getComputedStyle(form).backgroundColor` on the `<form>` element itself almost always returns `rgba(0,0,0,0)` — transparent — because the card background is on a parent wrapper, not the form tag. Eval #3 walks up the DOM until it finds the element actually carrying the background. Missing this step means missing card background color, border, and padding entirely.

**Also check field layout widths** — detect whether any fields are paired side-by-side in a 50/50 row:

```js
// 4. Field group layout — detect column arrangements
JSON.stringify(
  [...document.querySelectorAll('form .form-group, form .field-group, form [class*="field"]')].map(el => ({
    label: el.querySelector('label')?.textContent?.trim().slice(0, 30),
    width: window.getComputedStyle(el).width,
    display: window.getComputedStyle(el).display
  }))
)
```

If any two adjacent fields render at roughly half the form width, set `"width": "50"` on those `form_fields` entries in Elementor.

Map results directly to the native controls above. Key values found matching the precision-med brand:

| Element | Property | Value |
|---|---|---|
| Label | color | `#646F87` (rgb 100,111,135) |
| Label | font-size | 12px |
| Label | font-weight | 400 |
| Label | text-transform | uppercase |
| Label | letter-spacing | 1.2px |
| Label | margin-bottom | 8px (`label_spacing`) |
| Field | border | 1px solid `#DADEE7` |
| Field | border-radius | 4px |
| Field | padding | 12px 16px (achieved via `input_size: "lg"`) |
| Field | font | Inter 14px, color `#121721` |
| Button | background | `#182543` (navy) |
| Button | text color | `#F3EACE` (warm cream — NOT white) |
| Button | font-size | 14px, weight 500 |

## Atomic Forms — definitive findings (do not retry without new information)

Elementor Pro ships individual atomic form widgets: `e-form-input`, `e-form-label`, `e-form-textarea`, `e-form-submit-button`, `e-form-checkbox`. **These are purely presentational.** They render styled HTML form elements but have no native submission handling — no email notifications, no submission storage, no form actions of any kind.

Confirmed via investigation:
- All four widget schemas returned **completely empty properties** from `get-widget-schema`. No configurable settings are exposed through the MCP.
- The widgets have no backend action system. A native atomic form cannot send email or store submissions without custom PHP code.

**"Use Atomic Form" UI toggle:** visible in the Elementor panel on the regular `form` widget. Converts the widget's *rendering* to the atomic engine while keeping all backend actions (email, DB, CRM) intact. This toggle is **not exposed in the form widget's MCP schema** — it cannot be enabled via `update-element` or any MCP tool. It does not appear in `get-widget-schema` results.

**What's missing:** there is no `e-form-select` atomic widget. Dropdown fields cannot be replicated with atomic widgets at all.

**Decision rule:** do not attempt to build a standalone atomic form via MCP if the goal includes email notification or submission storage. The only native path to atomic rendering with working submission is the "Use Atomic Form" panel toggle on the regular `form` widget — and that requires a human to click it in the Elementor editor. Use the regular `form` widget (styled with native controls per this document) for all production forms.

## Font Awesome version note

Elementor ships with **Font Awesome 5**. Icons renamed or added in FA6 will silently render as an empty circle.

| FA6 name | FA5 equivalent |
|---|---|
| `fa-location-dot` | `fa-map-marker-alt` |

Always test icon rendering in agent-browser after placement.

## Field side-by-side layout

Set each field's `width` inside `form_fields` to `"50"` (percent string, not integer) to place two fields on the same row. Elementor uses a CSS grid for form field rows — fields with matching widths that sum to 100 flow into the same row automatically.

```json
"form_fields": [
  {"field_type": "text",  "field_label": "Full Name",     "width": "50"},
  {"field_type": "email", "field_label": "Email Address", "width": "50"},
  {"field_type": "text",  "field_label": "Phone Number",  "width": "100"}
]
```

## Phone masking — use `field_type: "text"`, not `"tel"`

If you apply a `(XXX) XXX-XXXX` JS input mask to a phone field, **do not use `field_type: "tel"`**. Elementor Pro validates `tel` fields against the HTML5 tel character set — digits, `+`, `-`, `*`, `#` only. Parentheses and spaces in `(865) 603-0604` fail this check and produce the error: *"The field accepts only numbers and phone characters (#, -, *, etc.)"* — even though the value was correctly formatted by your script.

**Fix:** set `field_type: "text"` on the phone field. Then restore the mobile numeric keyboard (which `type="tel"` normally triggers) by setting `inputmode="tel"` via JS on the input element itself.

The `custom_id`-based DOM ID (`#form-field-phone`) is derived from `custom_id`, not `field_type` — it survives the type change unchanged, so no selector updates are needed.

### Full phone masking implementation

```js
(function() {
  function applyUSMask(input) {
    var digits = input.value.replace(/\D/g, '').slice(0, 10);
    if (!digits) { input.value = ''; return; }
    var out = '(';
    if (digits.length <= 3) {
      out += digits;
    } else if (digits.length <= 6) {
      out += digits.slice(0, 3) + ') ' + digits.slice(3);
    } else {
      out += digits.slice(0, 3) + ') ' + digits.slice(3, 6) + '-' + digits.slice(6);
    }
    input.value = out;
  }

  function initPhoneMask() {
    var phoneInput = document.getElementById('form-field-phone');
    if (!phoneInput) return;
    phoneInput.setAttribute('inputmode', 'tel'); // numeric keyboard on mobile
    phoneInput.addEventListener('input', function() {
      applyUSMask(this);
    });
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initPhoneMask);
  } else {
    initPhoneMask();
  }
})();
```

Deliver via `elementor-mcp-add-custom-js` appended to the root page container (position `-1`). The script is page-scoped, outputs in the footer, and the DOM is already rendered when it runs.

| Setting | Value | Reason |
|---|---|---|
| `field_type` | `"text"` | Bypasses tel character-set validation |
| `inputmode` (via JS) | `"tel"` | Restores numeric keyboard on mobile |
| Selector | `#form-field-{custom_id}` | Set from `custom_id`, survives field type change |

## Elementor form webhook payload format

When an Elementor Pro form action is set to **Webhook**, the outbound request is:

- **Method:** POST
- **Content-Type:** `application/x-www-form-urlencoded`
- **Keys:** field **labels** — not `custom_id` values

PHP's `parse_str()` converts spaces in URL-encoded keys to underscores. A field labelled "Work Email" becomes the key `Work_Email`; "How can we help?" becomes `How_can_we_help?`.

Elementor always appends two extra params regardless of form fields:

| Key | Value |
|---|---|
| `form_id` | Elementor element ID of the form widget (e.g. `3b7845c`) |
| `form_name` | The form's `form_name` setting string (e.g. `TVIQ Select Demo Request`) |

### WP REST API endpoint — use label-derived keys, not custom_id

A WP REST API endpoint receiving this webhook must register its `args` using the label-derived key names — not the `custom_id` values you set in the Elementor editor. The REST API validates required args **before** the callback runs, so a mismatch between the registered arg names and the incoming keys results in a silent 400 (no callback runs, no debug log entries).

```php
// ✓ Correct — matches what Elementor actually sends
register_rest_route('mysite/v1', '/form-endpoint', [
    'methods'             => 'POST',
    'callback'            => 'my_form_handler',
    'permission_callback' => '__return_true',
    'args'                => [
        'First_Name' => ['required' => true,  'sanitize_callback' => 'sanitize_text_field'],
        'Last_Name'  => ['required' => true,  'sanitize_callback' => 'sanitize_text_field'],
        'Work_Email' => ['required' => true,  'sanitize_callback' => 'sanitize_email',
                         'validate_callback' => 'is_email'],
        'Phone'      => ['required' => false, 'sanitize_callback' => 'sanitize_text_field'],
        // Note: question marks are preserved through parse_str
        'How_can_we_help?' => ['required' => false, 'sanitize_callback' => 'sanitize_textarea_field'],
    ],
]);

// ✗ Wrong — these are custom_id values, not label-derived keys
// Elementor never sends 'first_name', 'email', 'message' etc.
'args' => [
    'first_name' => [...],
    'email'      => [...],
    'message'    => [...],
],
```

### Debugging a webhook 400 — the REST validation trap

If the form shows "Webhook error" and the debug log is empty, the 400 is being returned by WP REST validation before your callback runs. Your callback's `error_log()` calls are never reached.

To diagnose, add a `rest_pre_dispatch` filter temporarily and log the raw request body:

```php
add_filter('rest_pre_dispatch', function ($result, $server, $request) {
    if (strpos($request->get_route(), '/your-endpoint') !== false) {
        error_log('[DEBUG] route=' . $request->get_route());
        error_log('[DEBUG] body=' . file_get_contents('php://input'));
        error_log('[DEBUG] params=' . wp_json_encode($request->get_params()));
    }
    return $result;
}, 10, 3);
```

This fires before validation and reveals the exact keys Elementor is sending. Remove after diagnosis.

### Loopback SSL on Valet local dev

Elementor's Webhook action fires via `wp_remote_post()` — a PHP HTTP request from WordPress back to itself. On Valet local development sites, this loopback request fails with a cURL SSL error because Valet's mkcert-generated CA is not in cURL's own cert store (cURL does not use the macOS system trust store).

Fix with an mu-plugin that adds the filters only in development:

```php
<?php
// web/app/mu-plugins/tviq-local-dev.php
if (!defined('WP_ENV') || WP_ENV !== 'development') {
    return;
}
add_filter('https_local_ssl_verify', '__return_false');
add_filter('https_ssl_verify',       '__return_false');
```

Both filters are needed: `https_local_ssl_verify` covers loopback requests to the same host; `https_ssl_verify` covers all outbound WP HTTP requests. Do NOT add these filters in `config/environments/development.php` — that file loads before WordPress bootstrap and `add_filter()` is not yet defined.

## JS widget — Navigator naming and placement

When delivering a JS masking/interaction script via an HTML widget (or `add-custom-js`):

1. **Name the widget in the Elementor Navigator.** Give it a descriptive label like "Phone Mask Script" so future editors know what it does without opening it. Set `_title` in `add-html`/`add-container` at creation time, or update it via `update-element`.

2. **Place the widget adjacent to the element it interacts with.** A phone mask widget belongs next to — or inside the same section as — the form it targets. A script widget floating at the root page level is invisible in the Navigator's hierarchy and easily orphaned when the section is moved or duplicated. Proximity makes the dependency obvious.
