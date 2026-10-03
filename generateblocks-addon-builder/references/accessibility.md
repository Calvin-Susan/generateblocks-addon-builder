# Accessibility

The user styles the parts, so you can't control what they look like. You
*can* make sure the semantics, keyboard behaviour and state are always
right, because PHP adds them at render time.

## Pick the APG pattern first

Before building, name the WAI-ARIA Authoring Practices pattern and follow its
keyboard table exactly:

| Component | Pattern | Key points |
|---|---|---|
| On/off switch | Switch | `role="switch"`, `aria-checked`, Space (and Enter) toggles |
| Two-option buttons (Monthly / Annual) | Toggle buttons in a group | `aria-pressed` on each `<button>`, `role="group"` + `aria-label` on the container |
| Show/hide panel, dropdown | Disclosure | `<button aria-expanded aria-controls>`; Escape closes and returns focus to the button for dropdown-style popups; close on outside click |
| Tabs | Tabs | roving `tabindex`, arrow keys, `aria-selected` |
| Live timer | Timer | **not** a live region that ticks; announce only meaningful changes |
| Scrolling strip | Carousel-ish | pause button, pause on hover/focus, copies hidden from AT |
| Breadcrumbs | Breadcrumb | `<nav aria-label>`, list, `aria-current="page"` on the last item |
| Table of contents | Navigation | `<nav aria-label>`, `aria-current` (`true`/`location`) on the current link |

## Make parts operable in PHP

When decorating a part with the tag processor:

- `<button>` → add `type="button"` (stops a form submit) and the state
  attribute.
- Not a button (a GB Element `<div>` the user turned into a control) → add
  `role`, `tabindex="0"`, and the view script handles Enter **and** Space
  (and `preventDefault()` on Space so the page doesn't scroll). Prefer to
  ship templates with real `<button>`s.
- Give the control a name. Use the user's visible text if present.
  Otherwise use an `aria-label` from a block setting, applied only when the
  part doesn't already have one. Icon-only buttons need one (e.g. "Share on
  LinkedIn").
- `aria-controls` only with plain ids. Generate an id for the target if it
  has none (`wp_unique_id`).
- Next to a real control, decorative labels ("Monthly" text beside a switch)
  are clickable for mouse users but not focusable: don't create duplicate
  tab stops.

## Hidden and off-screen content

- Hide panels with the `hidden` attribute (or `display:none`) so their
  content isn't focusable or announced. Never just `opacity:0`.
- Copies made by a script (a marquee's duplicated row) get
  `aria-hidden="true"`, `inert`, ids removed, and `tabindex="-1"` on any
  focusable inside.
- Focus must never land on hidden or off-screen content. If a script hides
  the element that has focus, move focus to its control first.
- Content hidden until an entrance animation plays must still be reachable:
  reveal on focus (`focusin`), and have a CSS fail-safe that shows it if
  the script never loads.

## Motion

- Anything that moves for more than 5 seconds needs a pause control
  (WCAG 2.2.2). Pause on hover and on focus within too. If the user deletes
  the pause part from the layout, warn in the editor.
- `prefers-reduced-motion: reduce`: no movement. Animations show their end
  state, marquees stop (or become scrollable), reveal animations become
  instant. Listen for changes to the media query, not just the initial value.
- Gate any "hidden until animated" CSS on
  `@media screen and (prefers-reduced-motion: no-preference)`.
- Moving content shouldn't steal attention from screen readers. A ticking
  countdown inside `aria-live` is a classic mistake. Keep the live text off
  the live region and provide a static, screen-reader-only summary (e.g.
  "Sale ends Friday 5 pm") instead.

## Announcements

- One polite status element per block (`role="status"`), visually hidden,
  rendered by PHP so it exists before content changes (live regions created
  late are often ignored).
- To announce: clear it, then set the text after ~100 ms, so repeating the
  same message is announced again. Example: "Link copied".
- Don't be chatty. Announce the result of an action the user took, not
  ambient changes.

## Colour and contrast in default layouts

- Text ≥ 4.5:1, large text ≥ 3:1, controls and their states (a switch's
  track vs the page, the knob vs the track) ≥ 3:1. Check **both light and
  dark mode**: theme colour variables often flip.
- Use fixed colours where flipping would break contrast (e.g. a white knob
  on a mid-grey track that's ≥3:1 against both a white and a black page).
- State must not rely on colour alone where it matters: position (a knob
  that moves), weight, an underline, an outline.
- Windows High Contrast / forced colours removes backgrounds. Add
  `outline: 2px solid transparent` to controls and the current-state element;
  forced-colours mode draws it, so tracks and selected segments stay visible.
- Visible focus: `&:focus-visible` with a 2px outline and offset in the
  defaults.
- RTL: move things with logical properties (`margin-inline-start`) rather
  than `transform: translateX`, so the switch knob slides the right way.

## Test it

- Keyboard-only run through every layout: Tab order, Enter/Space/Escape,
  focus visible, focus never lost.
- Assert roles and state in Playwright from the server-rendered HTML
  (before JS) and after interaction.
- axe-core scan of a demo page containing every block and layout
  (`@axe-core/playwright`, WCAG 2.x A/AA tags).
