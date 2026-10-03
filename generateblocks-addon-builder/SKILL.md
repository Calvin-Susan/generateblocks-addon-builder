---
name: generateblocks-addon-builder
description: Build WordPress add-ons for GenerateBlocks 2.x / GenerateBlocks Pro that feel native - new blocks made of real GB blocks, extra panels on existing GB blocks, GB Pro Global Styles layouts - with security, accessibility and performance designed in from the start. Use whenever the user wants to create, extend or review a block, block plugin or feature that works with GenerateBlocks, GeneratePress or GB Pro (e.g. "make a GenerateBlocks toggle block", "add a panel to the GB Element block", "build a GB add-on plugin").
---

# Building GenerateBlocks add-ons that feel native

A GenerateBlocks user already knows how to style anything with GB's Styles
panel, Global Styles and design tokens. A good add-on **doesn't give them a
second, worse styling system**. It adds *behaviour* and lets GenerateBlocks do
the *looks*. Everything in this skill follows from that.

This skill is the distilled experience of building a multi-block GB add-on
plugin end to end (eight blocks, five extensions of GB's own blocks, full
browser test suite, audits). Read the reference that matches the step you are
on; don't load them all at once.

| Reference | Read it when |
|---|---|
| `references/architecture.md` | Choosing the shape of a feature; building the editor side |
| `references/gb-integration.md` | Touching GB's attributes, styles, Global Styles, icons, breakpoints, design tokens, GeneratePress |
| `references/security.md` | Anything a contributor types that reaches the page |
| `references/accessibility.md` | Any interactive part, motion, announcements, colours |
| `references/performance.md` | Loading assets, avoiding flashes and layout shift, front-end scripts |
| `references/testing.md` | Setting up Playground + Playwright, writing the tests, CI |
| `references/starter-code.md` | You want a complete, working example to adapt (a "Reveal" block) |
| `references/review-checklist.md` | Before committing or releasing; when asked to review a GB add-on |

## The rules

1. **Don't rebuild GenerateBlocks.** Everything visible is a real GB block
   (`generateblocks/element`, `text`, `media`, `shape`) styled with GB's own
   Styles panel. Never write your own colour/spacing/typography controls for
   things GB can already style. Don't build on `window.gb.*` internals
   beyond small, guarded reads; they're undocumented.
2. **New blocks are settings-only wrappers** (the way GB Pro's Accordion and
   Tabs work). The wrapper holds behaviour settings; its inner GB blocks are
   the visible *parts*. A part is marked with a data attribute in the GB
   block's own `htmlAttributes` (e.g. `data-reveal-part="button"`), set from
   a small "… part" panel you add with the `editor.BlockEdit` filter. GB
   saves and renders `htmlAttributes` for you, and the user can see it in
   GB's HTML Attributes panel.
3. **Extending GB's own blocks?** Store settings in their `htmlAttributes`
   (front-end behaviour) or their `styles` (pure CSS, GB compiles it), never
   in new attributes you register on GB's blocks. Content keeps working, and
   stays valid, if your plugin is switched off.
4. **Render on the server.** Blocks are dynamic: `save()` returns
   `<InnerBlocks.Content />`; a PHP render callback outputs the wrapper and
   adds roles, ARIA state, ids and `tabindex` to the parts with
   `WP_HTML_Tag_Processor`. Markup is correct before any JavaScript runs, and
   an editor can't delete the accessibility attributes.
5. **Starting layouts, not patterns.** Each block offers block variations
   picked from a placeholder when it's empty. Parts get GB Pro **Global
   Style classes** (`{prefix}-{block}__{part}` plus `--{modifier}`) via
   `globalClasses`, created once on an admin page load and never
   overwritten. Without GB Pro the blocks still work, just unstyled.
6. **Contributors are untrusted.** Anything they type that reaches HTML,
   `<style>`, a URL or a script config is sanitised in PHP, and *re-validated*
   in the view script, because data attributes can be forged.
7. **Accessible by construction**: WAI-ARIA APG patterns, keyboard support,
   focus never on hidden content, pause for anything moving more than 5 s,
   `prefers-reduced-motion`, ≥4.5:1 text and ≥3:1 control contrast in the
   default layouts.
8. **Nothing loads on pages without the feature.** Use `viewScript` /
   `viewStyle` in block.json or enqueue during render. Avoid site-wide
   options lookups and `wp_head` output on every page. No flash or layout
   shift: the server renders the initial state.
9. **Test in a real WordPress**: Playground with free GenerateBlocks +
   GeneratePress, demo content saved *through kses* (as an Author would),
   Playwright tests for assets, behaviour, keyboard, forged settings, editor
   block validity and an axe scan.
10. **One feature at a time, taken to done**: built, tested, reviewed,
    documented, committed. Only then start the next.

## Workflow for a new block or feature

### 1. Pick the shape

| The user wants… | Shape | Example |
|---|---|---|
| A new interactive component | Wrapper block + parts | toggle, tabs-like switcher, countdown, marquee, dropdown, disclosure |
| A new data-driven component | Wrapper block whose PHP fills GB parts with data | breadcrumbs, share links, table of contents |
| A new ability on every GB block | `editor.BlockEdit` panel → `htmlAttributes` data attribute → small view script loaded only when the attribute renders | entrance animations |
| A visual effect on a GB block | Panel that writes into the block's GB `styles` (GB compiles the CSS) | image masks: zero front-end JS |
| A behaviour on one GB block type | Panel → `htmlAttributes`; PHP `render_block_generateblocks/element` filter adds markup | video backgrounds |
| Something derived from a GB Pro block | Flag in `htmlAttributes`, PHP reads the *rendered* HTML | FAQ schema from the Accordion |

### 2. Agree three things with the user before writing code

Write these down and get a yes:

1. **Security**: list every value a contributor can type that ends up in
   the page (HTML attributes, `<style>`, URLs, selectors, JSON for scripts).
   For each: how PHP sanitises it, how the view script re-validates it, and
   whether saved attributes survive kses for Authors/Contributors.
2. **Accessibility**: role/name/state for each interactive part, the APG
   keyboard pattern, where focus goes, contrast of the default layout in
   light and dark, motion and pausing, screen-reader announcements.
3. **Performance**: which assets load, when, and on which pages; how the
   first paint is already correct (no flash, no shift); what work is
   skipped for off-screen or hidden content.

### 3. Build in this order

1. `block.json` (attributes with `enum`s and defaults, `category:
   "generateblocks"`, `supports.html: false`, `viewScript`/`viewStyle`).
2. PHP render class: a `config()` that whitelists/sanitises every attribute,
   then tag-processor decoration of parts. See `references/starter-code.md`.
3. Editor: `edit.js` (settings panel + variation placeholder), `parts.js`
   (the "… part" panel), `templates.js` (starting layouts).
4. Default Global Styles for the layouts (`references/gb-integration.md`).
5. `view.js`: small vanilla script, no dependencies, exposes
   `window.{prefix}{Block}.init()` so tests and late-loaded content can call it.
6. Tests + a demo section on the test site's demo page.
7. A read-only review pass (`references/review-checklist.md`), fix, re-test.
8. Docs (README, readme.txt, CHANGELOG), then commit.

### 4. Explain as you go

GenerateBlocks users are often agency builders, not block developers. When
you make a WordPress/JS choice (dynamic block, tag processor, kses, defer,
Web Animations), say *why* in plain words, in the code comment and to the user.

## Naming

Pick one short prefix with the user and use it everywhere: blocks
`{prefix}/{block}`, PHP `{prefix}_` functions / `{Prefix}_` classes,
front-end classes, events, storage keys and data attributes `{prefix}-`,
globals `window.{prefix}{Block}`, Global Style classes
`{prefix}-{block}__{part}--{modifier}`. Never reuse a class name your PHP
renderer adds itself as a Global Style name. The examples in this skill use
`acme`.

## Things that will bite you (summary — details in the references)

- WordPress's kses filter strips ARIA attributes beyond a short list (e.g.
  `aria-checked`, `aria-pressed`) and inline SVG for users without
  `unfiltered_html`. Saved markup no longer matches → "This block contains
  unexpected or invalid content". Prefer adding state at render time;
  allow a specific attribute via `wp_kses_allowed_html` only when it must
  be saved.
- `get_block_wrapper_attributes()` runs styles through `safecss_filter_attr`
  and drops properties like `mask-image`. Add such styles afterwards with the
  tag processor. End every style string with `;` (older WP joins with a space).
- Dynamic blocks don't print the HTML anchor themselves. Declare an `anchor`
  attribute and add `id` in PHP.
- The editor canvas is an **iframe**. `document` in editor scripts is the
  outer page. Portal any dynamic `<style>` into the canvas `<head>`.
- GB nested selectors must start with `&` or be a descendant. There are no
  ancestor selectors. Put state on the part itself (`&[aria-expanded="true"]`),
  and if one state needs to style two parts, give each part a class.
- On classic themes (GeneratePress), block `viewStyle` gets printed in the
  footer, which causes a flash. Enqueue it in `wp_enqueue_scripts` when
  `has_block()` is true for the current post.
- "Remove unused CSS" optimisers delete keyframes and classes added only by
  JS. Animate with the Web Animations API.
- A Global Style default you change later only reaches *new* sites (you
  never overwrite the site's copy). Add a new class instead of editing one.
