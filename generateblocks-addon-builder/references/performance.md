# Performance

Rule of thumb: **a page that doesn't use the feature must be byte-for-byte
the same as without the plugin.** And a page that does use it must be
correct on first paint.

## Load only where used

- Blocks: declare `viewScript` and `viewStyle` in block.json. WordPress
  enqueues them only when the block renders.
- Extensions of GB blocks (no block.json): `wp_register_script` on `init`
  with `array( 'in_footer' => true, 'strategy' => 'defer' )`, then
  `wp_enqueue_script` from a `render_block` filter when your data attribute
  is in the block's HTML (cheap `strpos` check first).
- Optional sub-features get their own small script, loaded only when that
  option is used (e.g. a "copy link" script only for blocks with a copy
  button). Add extra webpack entries for them.
- Front-end scripts: vanilla JS, no `@wordpress/*` dependencies (wp-i18n
  alone pulls in several scripts). Translatable strings the script needs
  are rendered by PHP into the config or the markup. Keep editor-only
  helpers in separate modules so the front-end bundle doesn't import them.
- Data the script needs site-wide (speeds, allowed hosts) goes in via
  `wp_add_inline_script( $handle, …, 'before' )`, only when the script is
  enqueued and only when it differs from the defaults.
- Options read on the front end must exist (create them on `admin_init` with
  autoload on). A missing option is an extra DB query on every page.
- Global Styles you create are in GB Pro's site-wide stylesheet. Keep the
  defaults small; dozens of classes is fine, hundreds isn't.

## No flash, no layout shift

- The server renders the initial state: roles, `aria-*`, `hidden`, the
  "is-on"/"is-off" class, the countdown's initial numbers. JS only takes
  over.
- **Classic themes** (GeneratePress) print a block's `viewStyle` when the
  block renders, which is after `<head>`, so it's unstyled for a moment.
  Fix:
  ```php
  add_action( 'wp_enqueue_scripts', function () {
  	$post = is_singular() ? get_post() : null;
  	if ( ! $post || false === strpos( $post->post_content, '<!-- wp:acme/' ) ) {
  		return;
  	}
  	foreach ( array( 'reveal' ) as $name ) {
  		$handle = generate_block_asset_handle( 'acme/' . $name, 'viewStyle' );
  		if ( has_block( 'acme/' . $name, $post ) && wp_style_is( $handle, 'registered' ) ) {
  			wp_enqueue_style( $handle );
  		}
  	}
  } );
  ```
- State remembered in `localStorage` (a toggle's last choice, dark mode)
  would flash from the server default to the saved state. Print a tiny
  inline script **right after the block** that applies the saved state
  before first paint (`document.currentScript.previousElementSibling`).
  For page-wide state (dark mode) print it early in `<head>`. The main
  script then takes over and the inline script does nothing if the main
  script already ran.
- Content the script will hide (the "other" pricing plan) is hidden by a
  server-printed `<style>` (sanitised selectors only) that the script
  removes once it's in control.
- Reserve space for anything the script fills in. Never inject content above
  the fold after load.

## Front-end script habits

- Initialise on `DOMContentLoaded` (or immediately if already loaded), and
  expose `window.acmeBlock = { init }` so content added later (AJAX, tests)
  can be set up. `init()` must skip elements already initialised (a flag on
  the element).
- Animate with the **Web Animations API** (`element.animate()`), not CSS
  keyframes or classes only the script adds. "Remove unused CSS" optimisers
  (common in caching plugins) delete CSS they don't see used in the HTML.
  It also gives you `pause()`, `playbackRate` and exact timing.
- `IntersectionObserver` to start work only when the block is near the
  viewport (entrance animations, marquees, countdown timers off-screen).
  Stop timers and animations when hidden (`document.visibilitychange`).
- `ResizeObserver` (debounced with rAF) instead of `resize` listeners.
- Measure in one batch, then write. Avoid layout thrash in loops.
- Images in cloned content stay `loading="lazy"` until the block is near the
  screen, then switch to eager. Only one copy of a video plays.
- Use one shared `matchMedia( '(prefers-reduced-motion: reduce)' )` and
  listen for `change`.
- Third-party embeds (video players): load the provider's script only when
  the block is near the viewport, and only from an allow-listed host.

## Editor performance

- `editor.BlockEdit` filters run for **every block on the page**. Return
  early for blocks that aren't yours, and only query the store for the
  selected block (`isSelected`).
- `useSelect` with precise dependencies; don't select whole block trees on
  every render unless the wrapper needs them.

## Measure it

- A Playwright test that loads a page *without* the blocks and asserts the
  plugin requested no files and printed no inline `<style>`/`<script>`.
- One per block/feature asserting exactly which files load when it's used.
- Report gzipped sizes of view scripts in the PR or commit. A few KB each is
  the target.
