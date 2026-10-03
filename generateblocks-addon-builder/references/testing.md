# Testing

Test in a real WordPress with real GenerateBlocks, in a real browser. Unit
tests of helpers are nice; they won't catch kses stripping an attribute,
GB changing a class name, or a flash of unstyled content.

## Local site: WordPress Playground

`@wp-playground/cli` runs WordPress in Node (PHP compiled to WebAssembly),
so you need no PHP, MySQL or Docker.

```json
// package.json scripts
"playground": "wp-playground-cli start --no-auto-mount --mount=./:/wordpress/wp-content/plugins/acme-blocks --blueprint=playground/blueprint.json --port=9400",
"playground:reset": "npm run playground -- --reset"
```

`playground/blueprint.json`:

```json
{
	"$schema": "https://playground.wordpress.net/blueprint-schema.json",
	"landingPage": "/wp-admin/",
	"preferredVersions": { "php": "8.3", "wp": "latest" },
	"features": { "networking": true },
	"login": true,
	"steps": [
		{ "step": "installPlugin", "pluginData": { "resource": "url", "url": "https://downloads.wordpress.org/plugin/generateblocks.latest-stable.zip" }, "ifAlreadyInstalled": "skip", "options": { "activate": true } },
		{ "step": "installTheme", "themeData": { "resource": "wordpress.org/themes", "slug": "generatepress" }, "ifAlreadyInstalled": "skip", "options": { "activate": true } },
		{ "step": "activatePlugin", "pluginPath": "acme-blocks/acme-blocks.php" },
		{ "step": "runPHP", "code": "<?php require '/wordpress/wp-load.php'; /* set pretty permalinks, then wp_insert_post() a demo page built from playground/demo/*.php */" }
	]
}
```

- **Build the demo page with `wp_insert_post()` in the blueprint, before any
  user is logged in.** kses then filters it exactly as it would for an
  Author or Contributor, so editor-validity tests catch stripped
  attributes. (Content you *know* will be stripped, like GB button icons,
  can be appended with `kses_remove_filters()`; document why.)
- Demo sections live in `playground/demo/*.php` as block markup exported
  from the editor. They're test fixtures, not plugin content, and stay out
  of the release zip.
- GB Pro isn't available in Playground. If layouts rely on Global Styles,
  define a test-only constant in a blueprint mu-plugin that makes your
  plugin print the default classes itself.
- `--reset` wipes the site. Stop any running instance first (an old
  instance holding the port causes `EADDRINUSE`).

## Playwright setup

```js
// playwright.config.js
module.exports = defineConfig( {
	testDir: './tests/e2e',
	globalSetup: require.resolve( './tests/e2e/global-setup.js' ),
	workers: 1,              // one shared WordPress: run serially
	fullyParallel: false,
	timeout: 90_000,
	retries: process.env.CI ? 1 : 0,
	use: { baseURL: 'http://127.0.0.1:9400', trace: 'retain-on-failure' },
	webServer: {
		command: 'npx wp-playground-cli start --no-auto-mount --mount=./:/wordpress/wp-content/plugins/acme-blocks --blueprint=playground/blueprint.json --port=9400 --skip-browser',
		port: 9400,            // a port check: every URL first 302s to Playground's auto-login
		reuseExistingServer: ! process.env.CI,
		timeout: 300_000,
	},
} );
```

- `global-setup.js`: the port opens **before** the blueprint finishes, so
  poll the demo page until your block is on it (up to ~5 minutes on a cold
  start).
- GeneratePress lists every page in its primary menu until a menu is set,
  and test pages pile up there and push content down. Create an empty menu
  in global setup.
- Locally you can point `launchOptions.executablePath` at the installed
  Chrome to skip downloading browsers. In CI, `npx playwright install
  --with-deps chromium`.

### Helpers worth writing first

- `rest( page, path, { method, data, params } )`: get a REST nonce from
  `/wp-admin/admin-ajax.php?action=rest-nonce` (after visiting `/wp-admin/`
  to log in), then call `/?rest_route=…` with `X-WP-Nonce`.
- `testPage( page, slug, markup )`: create **or update** a published page
  by slug, so reruns don't pile up pages. Returns its path.
- `pluginAssets( page, url )`: record requests to `/plugins/acme-blocks/`
  and inline `<style>`/`<script>` ids starting with your prefix.
- `openEditor( page, postId )`: open the editor and wait for
  `wp.data.select( 'core/block-editor' ).getBlocks().length > 0`.
- `invalidBlocks( page, markup? )`: walk `getBlocks()` (or
  `wp.blocks.parse( markup )`) and return names where `isValid` is false.

Raw block markup in tests must look like GB's own: include `uniqueId` and
the `gb-{type}-{uniqueId}` class, e.g.
`<!-- wp:generateblocks/element {"uniqueId":"a1a1a1a1","tagName":"div","htmlAttributes":{"data-reveal-part":"panel"}} --><div class="gb-element-a1a1a1a1" data-reveal-part="panel">…</div><!-- /wp:generateblocks/element -->`

## What to test for every block or feature

| Spec | Asserts |
|---|---|
| Assets | A page without the block loads **no** plugin files and prints no plugin inline styles/scripts (check `/`, a page, a plain test page). A page with it loads exactly its view files, never editor files. |
| Server markup | Before JS runs (`page.route` to block the view script, or read `response.text()`): roles, `aria-*`, `hidden`, ids, `type="button"` are right. |
| Behaviour | Click / interact, assert state changes in DOM and visually (`toBeVisible`, `toBeHidden`). Nested blocks don't interfere. Several blocks on one page. Persistence across reload if any. |
| Keyboard | Tab to each control, Enter and Space operate it, Escape where the pattern says, focus never lost or on hidden content. |
| Reduced motion | `test.use( { reducedMotion: 'reduce' } )`: nothing moves, content is visible. |
| Forged settings | Inject a wrapper with hostile config (`javascript:` URLs, `__proto__` enums, huge numbers, broken JSON) and call `window.acmeBlock.init()`. No navigation, no page errors, real blocks still work. |
| Editor validity | The demo page opens with zero invalid blocks and no `pageerror`. Each block's default variation, created with `createBlocksFromInnerBlocksTemplate` and serialised, re-parses valid. Every block is in the inserter. |
| Editor UX | The part panel appears only inside the wrapper; choosing a part writes `htmlAttributes`; placeholder variations insert. |
| Accessibility | axe-core on the demo page's content area (`.entry-content`), in `colorScheme` light **and** dark, tags `wcag2a, wcag2aa, wcag21a, wcag21aa, wcag22aa, best-practice`, expecting `[]`. |

Add every new block and layout to the demo page, so the editor-validity and
axe tests cover it automatically.

### Testing habits

- While building, run only the spec you're changing
  (`npx playwright test tests/e2e/reveal.spec.js`). Run the full suite once
  before committing.
- Rendering-dependent behaviour (IntersectionObserver, animations,
  `requestAnimationFrame`) needs a visible page. A headless browser is
  fine; a hidden app panel is not.
- Prefer waiting for a condition (`expect.poll`, `waitForFunction`) over
  fixed timeouts.

## CI (GitHub Actions)

Jobs on every push:

1. **JS**: `npm ci`, `npm run lint` (`wp-scripts lint-js`, `lint-style`),
   `npm run build`, build the zip, upload it as an artifact.
2. **PHP**: `php -l` on every PHP file with the minimum and latest PHP
   versions. PHPCS with `WordPress-Extra`, `WordPress-Docs` and
   `PHPCompatibilityWP` (`testVersion` = your minimum, prefix and text
   domain rules set to your prefix).
3. **Plugin Check**: `wordpress/plugin-check-action` on the unzipped zip.
4. **Browser tests**: build, install Chromium, `npm run test:e2e`
   (Playwright starts Playground itself), upload traces on failure.

No PHP on the developer's machine? Local syntax checks can run through
`@php-wasm/node` (ships with the Playground CLI); leave PHPCS to CI.

## Release zip

A script that copies only runtime files into `{slug}/`: the main file,
`readme.txt`, `uninstall.php`, `includes/`, `build/`, `LICENSE`. Zip that
folder. Use an allowlist, never "everything except…". Attach it to a GitHub
Release; tell users not to use GitHub's "Download ZIP" (no `build/`, wrong
folder name).
