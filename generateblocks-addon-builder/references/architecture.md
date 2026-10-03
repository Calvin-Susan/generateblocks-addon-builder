# Architecture

## Plugin layout

```
acme-blocks/
├── acme-blocks.php            # header (Requires Plugins: generateblocks), requires, register blocks
├── includes/
│   ├── class-acme-sanitize.php        # shared sanitisers (selectors, class names…)
│   ├── class-acme-{block}-render.php  # one render class per block
│   ├── kses.php                       # only if a saved ARIA attribute must survive kses
│   ├── class-acme-global-styles.php   # GB Pro Global Styles installer
│   └── global-styles/{block}.php      # default classes per block
├── src/
│   ├── shared/                        # GB helpers, canvas styles, variation placeholder
│   ├── {block}/
│   │   ├── block.json
│   │   ├── index.js       # registerBlockType
│   │   ├── edit.js        # settings panel + placeholder
│   │   ├── parts.js       # "… part" panel on GB blocks (editor.BlockEdit filter)
│   │   ├── templates.js   # starting layouts (block variations)
│   │   ├── view.js        # front end, vanilla JS
│   │   ├── style.scss     # tiny front-end CSS (structure only, never looks)
│   │   └── editor.scss
│   └── {extension}/editor.js, view.js # extensions of GB blocks (no block.json)
├── webpack.config.js          # adds entries for extensions without block.json
├── playground/blueprint.json  # local test site
└── tests/e2e/                 # Playwright
```

Build with `@wordpress/scripts` (`wp-scripts build`). It finds every
`src/*/block.json`. Extensions that aren't blocks need extra entries:

```js
const defaultConfig = require( '@wordpress/scripts/config/webpack.config' );

module.exports = {
	...defaultConfig,
	entry: () => ( {
		...( 'function' === typeof defaultConfig.entry ? defaultConfig.entry() : defaultConfig.entry ),
		'animations/editor': './src/animations/editor.js',
		'animations/view': './src/animations/view.js',
	} ),
};
```

Main plugin file essentials:

```php
/**
 * Plugin Name:       Acme Blocks
 * Requires at least: 6.6
 * Requires PHP:      7.4
 * Requires Plugins:  generateblocks
 */
add_action( 'init', function () {
	foreach ( array( 'reveal' => array( 'Acme_Reveal_Render', 'render' ) ) as $folder => $render ) {
		if ( file_exists( ACME_DIR . "build/{$folder}/block.json" ) ) {
			register_block_type( ACME_DIR . "build/{$folder}", array( 'render_callback' => $render ) );
		}
	}
} );
```

Also: an admin notice when `GENERATEBLOCKS_VERSION` is missing or < 2.0
(`Requires Plugins` only checks at activation), and a `block_categories_all`
filter that adds the `generateblocks` category if GB is inactive so the block
is never uncategorised.

## The wrapper block pattern

```
acme/reveal                      ← your block: settings only, rendered in PHP
└── generateblocks/element       ← user-styled GB blocks
    ├── generateblocks/text  htmlAttributes: { "data-reveal-part": "button" }
    └── generateblocks/element htmlAttributes: { "data-reveal-part": "panel" }
```

- `save: () => <InnerBlocks.Content />`. Only the inner GB blocks are saved.
  The wrapper is rendered by PHP, so you can change its markup in a later
  version without block validation errors.
- `supports: { html: false, className: true, anchor: true, spacing: { margin: true } }`.
  Keep supports minimal; layout and looks belong on the inner GB blocks.
- Parts can be nested anywhere inside the wrapper (e.g. inside a GB Element
  row). Walk the whole subtree, but **stop at a nested wrapper of the same
  block**, because its parts belong to it. In PHP the inner block has already
  rendered when the outer one runs, so the inner renderer marks its parts as
  claimed (e.g. `data-acme-owned`) and the outer skips them.
- If the block's parts are optional (a pause button, a label), the PHP and
  view script must work when the user deletes them. The editor can warn
  (a `Notice` in the settings panel) when a required part is missing.

### The "… part" panel (`parts.js`)

Add it with the `editor.BlockEdit` filter. Show it only when the selected
block is a GB block **inside** your wrapper:

```js
const PART_BLOCKS = [ 'generateblocks/element', 'generateblocks/text', 'generateblocks/shape', 'generateblocks/media' ];

const withPartPanel = createHigherOrderComponent( ( BlockEdit ) => ( props ) => {
	const { name, clientId, attributes, setAttributes, isSelected } = props;
	const isPartBlock = PART_BLOCKS.includes( name );

	const parentId = useSelect( ( select ) => {
		if ( ! isPartBlock || ! isSelected ) {
			return null; // cheap: no store work for unselected blocks
		}
		return select( blockEditorStore ).getBlockParentsByBlockName( clientId, 'acme/reveal', true )[ 0 ] || null;
	}, [ clientId, isPartBlock, isSelected ] );

	if ( ! parentId ) {
		return <BlockEdit { ...props } />;
	}

	const htmlAttributes = attributes.htmlAttributes || {};
	const onChange = ( next ) => {
		const updated = { ...htmlAttributes };
		if ( next ) {
			updated[ 'data-reveal-part' ] = next;
		} else {
			delete updated[ 'data-reveal-part' ];
		}
		setAttributes( { htmlAttributes: updated } );
	};

	return (
		<>
			<BlockEdit { ...props } />
			<InspectorControls>
				<PanelBody title={ __( 'Reveal part', 'acme' ) }>
					<SelectControl /* options: none, button, panel; help text says how to style each state */ />
				</PanelBody>
			</InspectorControls>
		</>
	);
}, 'withPartPanel' );

addFilter( 'editor.BlockEdit', 'acme/reveal-part', withPartPanel );
```

Always copy `htmlAttributes` before changing it. When a block stops being a
part, also delete any state attributes the block manages on it.

The help text in the panel is your documentation: tell the user the nested
selector that styles each state, e.g. "Style the open state with
`&[aria-expanded="true"]`".

### Settings panel and placeholder (`edit.js`)

- `useInnerBlocksProps( useBlockProps() )`. Don't lock templates; users
  rearrange parts freely.
- When the block has no inner blocks, render a variation picker
  (`__experimentalBlockVariationPicker`, stable in practice and used across
  core) and on select `replaceInnerBlocks( clientId,
  createBlocksFromInnerBlocksTemplate( variation.innerBlocks ), false )`.
  Pass `false` so the wrapper stays selected and its settings are what the
  user sees next.
- Settings use core components with `__next40pxDefaultSize` and
  `__nextHasNoMarginBottom`. Mark experimental imports with an eslint
  disable comment explaining why.
- If the editor should *preview* a state (e.g. "Starts open"), set the
  state attributes on parts with `updateBlockAttributes` in an effect. Only
  do this for attributes kses keeps (see security.md), or allow them.
- Name parts in List View: set `metadata.name` on template blocks (e.g.
  "Button", "Panel"). `__experimentalLabel` on the wrapper can show its
  mode ("Reveal · starts open").
- Block icon: wrap your SVG with the class `gblocks-block-icon` so it takes
  GB's icon colour in the inserter.
- `example: { innerBlocks: … }` from the default variation gives the
  inserter a real preview.

### Starting layouts (`templates.js`)

Block variations whose `innerBlocks` are GB block templates:

```js
const classes = ( part, modifier ) =>
	modifier ? [ `acme-reveal__${ part }`, `acme-reveal__${ part }--${ modifier }` ] : [ `acme-reveal__${ part }` ];

export const variations = [
	{
		name: 'simple',
		title: __( 'Simple', 'acme' ),
		isDefault: true,
		scope: [ 'block' ],
		innerBlocks: [
			[ 'generateblocks/text', {
				tagName: 'button',
				content: __( 'Show details', 'acme' ),
				htmlAttributes: { 'data-reveal-part': 'button', type: 'button' },
				globalClasses: classes( 'button' ),
				metadata: { name: __( 'Button', 'acme' ) },
			} ],
			[ 'generateblocks/element', {
				tagName: 'div',
				htmlAttributes: { 'data-reveal-part': 'panel' },
				globalClasses: classes( 'panel' ),
				metadata: { name: __( 'Panel', 'acme' ) },
			}, [
				[ 'generateblocks/text', { tagName: 'p', content: __( 'Hidden content.', 'acme' ) } ],
			] ],
		],
	},
];
```

- Don't set `uniqueId`; GB assigns it when the block mounts.
- Looks come from `globalClasses`. Only keep a style local (`styles`) when
  the block breaks without it (screen-reader-only text, a `display:flex`
  the script depends on), since users can remove or restyle classes.
- No block patterns. Variations are the starting point; the user owns
  the result.

## Rendering in PHP

Pattern for every render class:

```php
public static function render( $attributes, $content, $block = null ) {
	$config  = self::config( $attributes );            // whitelist + sanitise every attribute
	$content = self::decorate_parts( $content, $config ); // WP_HTML_Tag_Processor

	$wrapper = array(
		'class'          => 'acme-reveal',
		'data-acme-reveal' => wp_json_encode( $config ), // the view script's config
	);
	if ( ! empty( $attributes['anchor'] ) ) {
		$wrapper['id'] = $attributes['anchor'];         // dynamic blocks must print the anchor
	}

	return sprintf( '<div %s>%s</div>', get_block_wrapper_attributes( $wrapper ), $content );
}
```

- `config()` is the single source of truth: `in_array( $value, ENUM, true )`
  with a fallback, `sanitize_text_field`, numeric clamps, your selector /
  class-name sanitisers. The view script receives only this.
- `WP_HTML_Tag_Processor` (WP 6.2+): `next_tag()`, `get_attribute()`,
  `set_attribute()`, `get_tag()`, `get_updated_html()`. It escapes attribute
  values for you. Don't use regex for HTML unless you need a tag *and its
  contents* and you control the shape (and then keep it narrow).
- Use `wp_unique_id( 'acme-reveal-' )` for ids you generate (e.g. for
  `aria-controls`), and keep an id the user already gave a part.
- Data from the post (title, URL, permalink): use `usesContext: [ "postId" ]`
  and `$block->context['postId']`, so the block works inside a Query Loop,
  falling back to the current post.
- Rendered outside the page (REST, feeds, an SEO plugin building a
  description during `wp_head`)? Skip enqueues and side effects:
  `wp_is_serving_rest_request()`, `is_feed()`, `doing_action( 'wp_head' )`.

## Extending existing GB blocks

**Behaviour** (e.g. entrance animations):

1. `editor.BlockEdit` panel on every block where
   `/^generateblocks(-pro)?\//.test( name ) && getBlockType( name )?.attributes?.htmlAttributes`
   (GB 2 blocks; v1 blocks don't have `htmlAttributes`).
2. Store only non-default values (`data-acme-animate="fade-up"`, optional
   `data-acme-speed`…), so markup stays tidy. Delete all your keys before
   writing the new set.
3. PHP `render_block` filter (or `render_block_{name}`) checks for the
   attribute with a cheap `strpos` first, then `wp_enqueue_script()` the
   registered (not enqueued) view script. Pages without it load nothing.
4. Anything that must be in place before the script (e.g. "hidden until
   animated" CSS) goes in `<head>` when you can know the page uses it
   (block themes render templates before `<head>`; on classic themes check
   the queried post's content), otherwise just before the first such block.
   Gate hiding CSS behind a JS-added class and `prefers-reduced-motion:
   no-preference`, with a CSS fail-safe that reveals content if the script
   never arrives.

**Pure CSS** (e.g. masks): write CSS properties into the block's GB
`styles` object at the breakpoint being previewed. GB compiles and saves the
CSS. No front-end asset at all, and users see/edit the values in GB's Styles
panel. See gb-integration.md for the styles format and breakpoints.

**Extra markup** on a GB block (e.g. a video layer inside an Element): use
`render_block_generateblocks/element` in PHP, and only act when your data
attribute is present. Editor preview via `editor.BlockListBlock` adding
`wrapperProps` (e.g. a poster as background), not by changing GB's edit.

**Server-derived data** (e.g. FAQ schema from GB Pro's Accordion): a flag in
`htmlAttributes`, then build the data from the **rendered** HTML in
`render_block_{name}`, so it always matches what visitors see (including
dynamic tags and block conditions).

## Settings page (optional)

If the plugin has several features, a Settings → {Plugin} page with an
on/off per feature works well: "off" hides the block from the inserter and
its panels, while existing content keeps rendering. Create the options it
reads on the front end on `admin_init` with `add_option( …, array(), '', true )`
so a missing option doesn't cost a query per page view. Site-wide output
(anything printed on every page) must only be changeable by users with a
strong capability (`manage_options` / `edit_theme_options`).
