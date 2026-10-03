# Review checklist

Use after each feature (one read-only review pass, then fix and re-test)
and before a release (run security, performance, accessibility and
WordPress-best-practice reviews in parallel, each read-only, then fix).
Report findings with file:line, a concrete failure scenario, and severity.

## Native GenerateBlocks feel

- [ ] Everything visible is a GB block the user can select and style in
      GB's Styles panel. No custom colour/spacing/typography controls.
- [ ] Parts marked via `htmlAttributes` `data-*`; the "… part" panel shows
      only inside the wrapper and explains the nested selector for each state.
- [ ] Extensions of GB blocks store settings in `htmlAttributes` or `styles`,
      not new registered attributes. Content stays valid with the plugin off.
- [ ] Block in the `generateblocks` category; icon has `gblocks-block-icon`.
- [ ] Starting layouts via variations + placeholder; no block patterns;
      List View names on parts.
- [ ] Layout looks are GB Pro Global Style classes
      (`{prefix}-{block}__{part}--{modifier}`), base before modifier, never
      reusing a class the PHP renderer adds. Local `styles` only where the
      block breaks without them.
- [ ] New default classes were *added*, not edited (edits never reach
      existing sites).
- [ ] Works (unstyled) on free GenerateBlocks.

## Security

- [ ] Every attribute passes through `config()` in PHP: enums whitelisted,
      numbers clamped, text sanitised, class names `sanitize_html_class`.
- [ ] Nothing user-typed reaches `<style>` without the selector validator;
      one rule per selector.
- [ ] URLs built from fixed patterns in PHP; view script re-checks protocol
      before navigating or assigning `href`/`src`.
- [ ] View script: `JSON.parse` in try/catch, type checks, `Object.hasOwn`
      lookups, `textContent` not `innerHTML`, `querySelectorAll` of user
      selectors in try/catch.
- [ ] Inline script data uses `wp_json_encode` with `JSON_HEX_TAG`.
- [ ] Saved markup survives kses for Authors/Contributors (or the needed
      attribute is allowed in `wp_kses_allowed_html`, ARIA states only).
- [ ] Site-wide settings gated by capability, nonces, sanitize callbacks.
- [ ] `ABSPATH` guard in every PHP file; `$wpdb->prepare` on every query.
- [ ] A forged-settings test exists for the block.

## Accessibility

- [ ] Follows a named APG pattern: role, name, state, keyboard table.
- [ ] Roles/state/ids/`type="button"`/`tabindex` added in PHP, correct before JS.
- [ ] Non-`<button>` controls handle Enter and Space (not Enter on links).
- [ ] Hidden content is `hidden`/`display:none` (GB `display` overridden
      with `!important` where needed); clones are `inert` + `aria-hidden`.
- [ ] Focus never lost or left inside hidden content.
- [ ] Motion > 5 s pausable; hover/focus pause; reduced motion respected
      and live-updated.
- [ ] Live regions: one polite status per block, rendered by PHP, clear then
      set; no ticking announcements.
- [ ] Default layouts: text ≥ 4.5:1, controls ≥ 3:1, in light and dark;
      `:focus-visible` outline; transparent outline for forced colours;
      logical properties for RTL.
- [ ] axe scan of the demo page passes in light and dark.

## Performance

- [ ] A page without the feature loads no plugin files and prints no plugin
      inline CSS/JS (test proves it).
- [ ] Assets via block.json `viewScript`/`viewStyle` or enqueued on render;
      optional sub-features in separate small scripts.
- [ ] `viewStyle` enqueued in `<head>` for posts that use the block
      (classic themes).
- [ ] No flash/shift: server renders state; remembered state restored by an
      inline script right after the block; hide-until-ready CSS removed by
      the script; fail-safe if the script never loads.
- [ ] Front-end JS: vanilla, no `@wordpress/*` deps, Web Animations,
      IntersectionObserver/visibility for off-screen work, `init()` idempotent
      and exposed on `window`.
- [ ] `editor.BlockEdit` filters bail early for other blocks and unselected
      blocks.
- [ ] Front-end options exist (no per-page query for a missing option).
- [ ] Global Styles defaults kept lean.

## WordPress best practices

- [ ] `Requires Plugins: generateblocks`, version notice for GB < 2.0,
      `Requires at least` / `Requires PHP` accurate.
- [ ] Prefix on every global function, class, option, handle, hook, CSS
      class, event and storage key. Text domain on every string;
      `wp_set_script_translations` for editor scripts.
- [ ] Dynamic block prints its anchor; style strings end with `;`; styles
      `get_block_wrapper_attributes` would strip are added after with the
      tag processor.
- [ ] `usesContext: ["postId"]` for anything about "the current post".
- [ ] No side effects when rendered in REST, feeds, or during `wp_head`.
- [ ] `uninstall.php` removes the plugin's options (not the user's Global
      Styles or content).
- [ ] PHPCS (WordPress-Extra, WordPress-Docs, PHPCompatibilityWP) and
      Plugin Check pass; `npm run lint` passes.
- [ ] README / readme.txt / CHANGELOG updated: parts, settings, classes,
      state selectors, accessibility behaviour, known limitations.
- [ ] Release zip built from an allowlist of runtime files.
