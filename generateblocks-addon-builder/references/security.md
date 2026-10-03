# Security

## Threat model

- **Contributors and Authors can save any block attributes**, including
  ones your UI never offers (via the code editor or REST). Treat every
  attribute as attacker-controlled.
- **Anyone who can write HTML can forge your data attributes and config
  JSON.** That includes contributors via a Custom HTML block (kses limits
  it, but `data-*` survives), and other plugins. The view script must be
  safe with any input.
- Site-wide output (a `<head>` script, a setting printed on every page) must
  only be changeable by a high-capability user (`manage_options` /
  `edit_theme_options`), via a settings page with nonces and
  `register_setting` sanitize callbacks.

## For each value, decide where it ends up

| Destination | PHP | View script |
|---|---|---|
| Enum (mode, direction, animation) | `in_array( $v, ALLOWED, true ) ? $v : DEFAULT` | Same whitelist again; look up in objects with `Object.hasOwn( MAP, key )` (a forged `"__proto__"` or `"constructor"` must not hit the prototype) |
| Number | cast + clamp (`max( 0, min( 3600, (int) $v ) )`) | `Number()` + `Number.isFinite` + clamp |
| Plain text | `sanitize_text_field`; output via tag processor `set_attribute` (escapes) or `esc_html` | `textContent`, never `innerHTML` |
| Class names | split on whitespace, `sanitize_html_class` each | `classList.add` with a regex check |
| HTML ids / CSS selectors | strict validator (below) | wrap `querySelectorAll` in try/catch; never build CSS strings from it |
| URL | build it **in PHP from a fixed pattern** + `rawurlencode`; `esc_url` for user URLs, with a protocol whitelist | `new URL( value, location.href )` in try/catch, then require `/^https?:$/` protocol. Never assign an unchecked URL to `location`, `href` or `src` (a `javascript:` redirect is the classic forged-config attack). |
| JSON for the script | `wp_json_encode( $config )` of the *sanitised* config, into an attribute (escaped by the tag processor / `get_block_wrapper_attributes`) | `JSON.parse` in try/catch; check it's an object; read each field defensively |
| Inline `<script>` data | `wp_json_encode( $data, JSON_HEX_TAG \| JSON_HEX_AMP \| JSON_UNESCAPED_SLASHES )` printed with `wp_get_inline_script_tag` / `wp_add_inline_script` | n/a |
| Inside a `<style>` | only through a validator; `<style>` content is raw text, so `esc_html` would break selectors and doesn't help | n/a |

## Selectors in `<style>`

If users can name targets ("hide `#monthly-prices` when off") and you print
no-flash CSS for them, the selector reaches a `<style>` element. A safe
selector validator:

- rejects, **even inside quotes**: `<` (could close `</style>`), `\`
  escapes, control characters, `{`, `}`, `;`, `@`, `/*`, `url(`;
- outside quotes allows only selector punctuation, letters, digits,
  non-ASCII letters: `[A-Za-z0-9_\-#.\[\]="'~^$*|:(), >+]`;
- requires brackets, parentheses and quotes to balance (an open `(` could
  swallow the next rule);
- caps the length (e.g. 200), validates UTF-8, deduplicates;
- prints **one rule per selector**, so one invalid selector only voids its
  own rule.

A bare word is treated as an id (`pricing` → `#pricing`), except common tag
names (`html`, `body`, `main`, `header`, `footer`, `nav`, `aside`,
`article`, `section`), which the script resolves at runtime.

## Markup via the tag processor

- `WP_HTML_Tag_Processor::set_attribute()` escapes values. Never
  concatenate attribute strings yourself.
- Remove attributes you own from forged input: if a part could carry a
  forged `href` or `onclick`-like attribute you'd otherwise trust, overwrite
  it from your own data. For example, a share button's `href` is always
  rebuilt from a fixed network list, never taken from saved markup.
- `get_block_wrapper_attributes()` filters inline `style` through
  `safecss_filter_attr`, which drops some legitimate properties. If you
  need one (e.g. `mask-image`), validate it yourself and add it with the tag
  processor after.

## Kses: saved attributes that disappear

For users without `unfiltered_html`, kses removes attributes outside its
allowed list from saved content. `data-*`, `role`, `aria-label`,
`aria-labelledby`, `aria-describedby`, `aria-controls`, `aria-expanded`,
`aria-hidden`, `aria-live`, `aria-current` survive. `aria-checked`,
`aria-pressed` and inline `<svg>` don't. When an attribute is stripped, saved
markup no longer matches `save()` output, and the editor flags the block as
invalid.

Preferred: **don't save state**; add it at render time in PHP. If the
editor must preview the state in saved markup (so the user can style it),
allow exactly those attributes for the `post` context:

```php
add_filter( 'wp_kses_allowed_html', function ( $tags, $context ) {
	if ( 'post' !== $context || ! is_array( $tags ) ) {
		return $tags;
	}
	foreach ( $tags as $tag => $attributes ) {
		if ( is_array( $attributes ) ) {
			$tags[ $tag ]['aria-checked'] = true; // plain ARIA state: can't run code
		}
	}
	return $tags;
}, 10, 2 );
```

Never allow event handlers, `style` beyond core's handling, or `href`/`src`
this way.

## Other checks

- `if ( ! defined( 'ABSPATH' ) ) { exit; }` at the top of every PHP file.
- Settings pages: `current_user_can`, `settings_fields` / nonces,
  `register_setting( …, array( 'sanitize_callback' => … ) )`.
- Admin-side writes (like the Global Styles installer) check capability
  and run only on normal admin loads.
- Direct DB queries only when needed (atomic lock), always `$wpdb->prepare`,
  with a phpcs:ignore comment saying why.
- Forged-settings tests (see testing.md): inject a wrapper with hostile
  config into a real page, call `window.acmeBlock.init()`, assert nothing
  ran, nothing threw, and the real blocks still work.
