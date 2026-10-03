# GenerateBlocks integration reference

Verified against GenerateBlocks 2.x and GenerateBlocks Pro 2.x (late 2026).
Items marked **(internal)** are not documented public API. Read them
defensively (optional chaining, `method_exists`, fallbacks), and re-check them
against the installed GB version before relying on them.

## GB 2 blocks and their attributes

| Block | Use it for |
|---|---|
| `generateblocks/element` | Containers, rows, panels, tracks. Any tag (`tagName`). |
| `generateblocks/text` | Headings, paragraphs, labels, **buttons and links** (`tagName: 'button' \| 'a' \| 'span' \| 'p' \| 'h2'…`), with an optional icon. |
| `generateblocks/media` | Images. |
| `generateblocks/shape` | Inline SVG shapes/icons. |
| `generateblocks-pro/*` | Pro blocks (accordion, tabs…): extend them, don't wrap them. |

Attributes shared by GB 2 blocks that add-ons care about:

- `uniqueId`: GB's id for the block; class `gb-{type}-{uniqueId}` carries its
  local styles. GB assigns it in the editor. Leave it out of templates. In
  raw block markup you write by hand (tests, demo content) include it,
  plus `css`.
- `tagName`: the element GB renders.
- `htmlAttributes`: object of extra HTML attributes. GB saves and renders it.
  **This is where your data attributes go.**
- `styles`: the block's local styles (see format below). GB compiles them
  into `css`.
- `css`: compiled CSS, written by GB's editor.
- `globalClasses`: array of GB Pro Global Style class names.
- `content` (text): rich text.
- `icon`, `iconLocation`, `iconOnly` (text): an inline SVG icon.
- `metadata.name`: the List View name (core).

## The `styles` format

GB keeps styles as an object, not CSS:

```js
{
	display: 'flex',
	columnGap: '1rem',
	paddingTop: '1rem', paddingRight: '1rem', paddingBottom: '1rem', paddingLeft: '1rem', // longhands only
	borderTopLeftRadius: '999px', /* …all four corners… */
	'&:hover': { backgroundColor: 'var(--accent)' },        // nested: & = this block
	'&[aria-expanded="true"]': { color: '#fff' },          // state on the part itself
	'svg': { width: '1em' },                               // no &: descendant
	'@media (max-width:1024px)': { flexDirection: 'column' },
}
```

- camelCase properties, **longhands** (`paddingTop`…, not `padding`). Write
  small helpers (`padding( y, x )`, `radius( v )`, `border( w, style, colour )`).
- Nested keys must start with `&` or be a descendant selector. **No ancestor
  selectors** (`.is-open &` doesn't work). So:
  - state lives on the part: `&[aria-checked="true"]`, `&[data-active="true"]`,
    `&[aria-expanded="true"]`, `&[aria-current]`;
  - to style a child by its parent's state, nest from the parent:
    `&[aria-checked="true"] > *`;
  - a look that needs two parts' states gets a class on each part.
- Breakpoints: GB's defaults are `@media (max-width:1024px)` (tablet) and
  `@media (max-width:767px)` (mobile). Read the site's actual keys
  **(internal)**: `window.gb?.stylesBuilder?.defaultAtRules`, entries with
  `id` `mediumSmallWidth` and `smallWidth`, `value` = the at-rule.
  Fall back to the defaults.
- When your panel writes styles, write at the breakpoint the editor is
  previewing, keep values the user typed by hand, and only touch your own
  properties.

## GB Pro Global Styles (layout classes)

Global Styles are GB Pro's reusable classes. Give template parts classes via
`globalClasses` and create the classes on the site.

**Storage (internal)**: post type `gblocks_styles`, one post per class:

| Field | Value |
|---|---|
| `post_title` | `.acme-reveal__button` |
| meta `gb_style_selector` | `.acme-reveal__button` |
| meta `gb_style_data` | the styles array (same format as above, PHP arrays) |
| meta `gb_style_css` | compiled CSS |
| meta `gb_style_category` | your plugin name (GB Pro 2.8+ groups by category) |
| `menu_order` | position in GB Pro's global stylesheet: **base classes before modifiers** |

Useful GB Pro methods **(internal)**, guard each with `class_exists` /
`method_exists`:

- `GenerateBlocks_Pro_Styles::get_class_by_name( '.selector' )`: does it exist?
- `GenerateBlocks_Pro_Styles::can_manage_styles()`: may this user create them?
- `GenerateBlocks_Pro_Enqueue_Styles::get_instance()->build_css()`: rebuild
  the stylesheet. GB Pro rebuilds it on every class save via
  `wp_after_insert_post` → `build_css_file_on_save` (priority 100). Remove
  that action while inserting many classes, then call `build_css()` once.

**Installer rules** (learned the hard way):

1. Defaults live in `includes/global-styles/{block}.php`, each returning
   `static function () { return array( 'acme-reveal__button' => array( … ) ); }`.
   Returning a closure keeps helper variables out of the global scope.
2. Run on `admin_init`, **only on an ordinary page load**: bail on
   `wp_doing_ajax()`, `wp_doing_cron()`, `REST_REQUEST`. Only when GB Pro is
   available and the user can manage styles.
3. Take an atomic lock so two admin tabs can't both create classes:
   `INSERT IGNORE INTO {$wpdb->options}` a `{prefix}_global_styles_lock` row
   (`add_option()` isn't atomic). Treat a lock older than a minute as stale
   (compare-and-delete).
4. Keep an option listing class names already handled. Save it **after each
   class**, so a timed-out run resumes. A class the site deleted stays
   deleted, and an existing class (same selector) is marked handled, not
   overwritten.
5. **Never overwrite a class.** Once created it belongs to the site. Changing
   a default only reaches new sites, so ship a new class instead of editing one.
6. Keep the total lean. Every class is in GB Pro's site-wide stylesheet on
   every page.
7. Without GB Pro the blocks must still work (unstyled layouts). For a local
   test site without GB Pro, a constant (set by the test site's mu-plugin,
   never on real sites) can print the compiled defaults as a stylesheet on
   `enqueue_block_assets`, so tests and demos look right.

A compiler for the defaults only needs: declarations (camelCase →
kebab-case), nested `&` / descendant keys (split on top-level commas), and
`@media` keys.

## Other GB hooks and globals

| What | How |
|---|---|
| Add icons to GB's icon picker | JS filter `generateblocks.editor.iconSVGSets`: `( sets ) => ( { ...sets, acmeIcons: { group: 'Acme', svgs: { key: { label, icon: <svg …/> } } } } )` |
| GB's shape library (incl. GB Pro Asset Library shapes) | **(internal)** `window.generateBlocksInfo?.svgShapes` |
| GB Pro design tokens (colours) | **(internal)** `window.gbDesignTokens?.tokens`: items `{ name: '--token', type: 'color', label, category, scope: [ 'backgroundColor', … ] }`. Offer tokens whose `scope` includes the property, store as `var(--token)`. Fall back to the theme palette from `getSettings().__experimentalFeatures.color.palette.theme` or `getSettings().colors`. |
| Block category | `"category": "generateblocks"` in block.json |
| Block icon colour | class `gblocks-block-icon` on your icon's SVG |
| Version check | `defined( 'GENERATEBLOCKS_VERSION' ) && version_compare( GENERATEBLOCKS_VERSION, '2.0.0', '>=' )` |

Don't recreate GB's Styles Builder or reach deeper into `window.gb.*`. If a
feature seems to need it, write into the block's `styles` and let GB's own
panel show it.

## GeneratePress

- Global colour CSS variables, with fallbacks for other themes:
  `var(--accent, #1e73be)`, `var(--contrast, #222222)`,
  `var(--contrast-2, #575760)`, `var(--contrast-3, #b2b2be)`,
  `var(--base, #f0f0f0)`, `var(--base-2, #f7f8f9)`, `var(--base-3, #ffffff)`.
  Many users redefine these (or use `light-dark()`), so they may flip in
  dark mode. Check that default-layout contrast survives that, or use fixed
  colours where a control must keep its contrast (e.g. a switch knob).
- GeneratePress is a classic theme. Blocks in the post content render
  *after* `<head>`, so block `viewStyle` lands in the footer (see
  performance.md). GP Elements (hooks, block elements) render blocks outside
  the post, so never assume a block is only in `post_content`.
- For local tests, install free GenerateBlocks + GeneratePress in
  Playground. GB Pro can't be installed there (not on wordpress.org). Test
  Pro-only paths on a real site.

## Kses and GB content

- Users without `unfiltered_html` (Authors, Contributors, everyone but
  Super Admins on multisite) save through kses. It strips inline `<svg>`
  (so GB icons on buttons) and ARIA attributes outside WP's short list.
  The block then shows as invalid in the editor. The SVG case is a GB
  limitation; document it rather than fight it.
- `data-*` attributes survive kses, so mark parts with `data-*`.
