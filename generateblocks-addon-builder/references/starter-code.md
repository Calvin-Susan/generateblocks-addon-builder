# Starter code: a "Reveal" block

A complete, small wrapper block that shows every pattern in this skill: a
button part that shows and hides a panel part (the APG *disclosure*
pattern). Both parts are ordinary GenerateBlocks blocks the user styles in
GB. Adapt it; don't ship it unchanged.

Replace `acme` / `Acme` / `ACME` with the plugin's prefix throughout.

```
acme-blocks.php
includes/class-acme-reveal-render.php
includes/class-acme-global-styles.php
includes/global-styles/reveal.php
src/shared/variation-placeholder.js
src/reveal/{block.json,index.js,icon.js,edit.js,parts.js,templates.js,view.js,style.scss}
```

Why this shape: the server writes `aria-expanded`, `aria-controls`, the panel
`id` and `hidden`, so the markup is correct and accessible before
JavaScript, and none of that is *saved* in post content. kses never sees
it, and there's nothing for an editor to break. The only saved marker is
`data-reveal-part`, which kses allows.

---

## acme-blocks.php

```php
<?php
/**
 * Plugin Name:       Acme Blocks
 * Description:       Extra blocks for GenerateBlocks.
 * Version:           0.1.0
 * Requires at least: 6.6
 * Requires PHP:      7.4
 * Requires Plugins:  generateblocks
 * License:           GPL-2.0-or-later
 * Text Domain:       acme
 *
 * @package Acme
 */

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

define( 'ACME_VERSION', '0.1.0' );
define( 'ACME_DIR', plugin_dir_path( __FILE__ ) );

require_once ACME_DIR . 'includes/class-acme-reveal-render.php';
require_once ACME_DIR . 'includes/class-acme-global-styles.php';

add_action( 'init', 'acme_register_blocks' );
/**
 * Register blocks from their built block.json, with PHP render callbacks.
 */
function acme_register_blocks() {
	$blocks = array(
		'reveal' => array( 'Acme_Reveal_Render', 'render' ),
	);

	foreach ( $blocks as $folder => $render ) {
		if ( file_exists( ACME_DIR . "build/{$folder}/block.json" ) ) {
			register_block_type( ACME_DIR . "build/{$folder}", array( 'render_callback' => $render ) );
		}
	}
}

add_action( 'wp_enqueue_scripts', 'acme_head_block_styles' );
/**
 * Classic themes (GeneratePress) print a block's viewStyle when the block
 * renders, after <head>, so it would show unstyled for a moment. Load it in
 * <head> when the post being viewed uses the block.
 */
function acme_head_block_styles() {
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
}

add_action( 'admin_notices', 'acme_generateblocks_notice' );
/**
 * "Requires Plugins" only checks on activation; GB may be deactivated later
 * or be an old 1.x version.
 */
function acme_generateblocks_notice() {
	if ( ! current_user_can( 'activate_plugins' ) ) {
		return;
	}

	if ( defined( 'GENERATEBLOCKS_VERSION' ) && version_compare( GENERATEBLOCKS_VERSION, '2.0.0', '>=' ) ) {
		return;
	}

	printf(
		'<div class="notice notice-warning"><p>%s</p></div>',
		esc_html__( 'Acme Blocks needs GenerateBlocks 2.0 or newer to be active.', 'acme' )
	);
}
```

## src/reveal/block.json

```json
{
	"$schema": "https://schemas.wp.org/trunk/block.json",
	"apiVersion": 3,
	"name": "acme/reveal",
	"version": "0.1.0",
	"title": "Reveal",
	"category": "generateblocks",
	"description": "A button that shows and hides content, built from GenerateBlocks blocks.",
	"keywords": [ "disclosure", "show more", "toggle", "hide" ],
	"textdomain": "acme",
	"attributes": {
		"startOpen": { "type": "boolean", "default": false },
		"animation": { "type": "string", "default": "fade", "enum": [ "none", "fade" ] },
		"anchor": { "type": "string" }
	},
	"supports": {
		"anchor": true,
		"html": false,
		"className": true,
		"spacing": { "margin": true }
	},
	"editorScript": "file:./index.js",
	"viewScript": "file:./view.js",
	"viewStyle": "file:./style-index.css"
}
```

## src/reveal/index.js

```js
import { registerBlockType } from '@wordpress/blocks';
import { InnerBlocks } from '@wordpress/block-editor';

import metadata from './block.json';
import Edit from './edit';
import { revealIcon } from './icon';
import { variations } from './templates';
import './parts';
import './style.scss';

registerBlockType( metadata.name, {
	icon: revealIcon,
	edit: Edit,
	// The wrapper is rendered in PHP; only the inner GenerateBlocks blocks are saved.
	save: () => <InnerBlocks.Content />,
	variations,
} );
```

## src/reveal/icon.js

```js
import { SVG, Path } from '@wordpress/primitives';

// gblocks-block-icon: GenerateBlocks colours its block icons with this class.
export const revealIcon = (
	<SVG className="gblocks-block-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
		<Path d="M4 5h16v4H4zM4 11h16v8H4z" fillOpacity="0.5" />
		<Path d="M4 5h16v4H4z" />
	</SVG>
);
```

## src/shared/variation-placeholder.js

```js
/**
 * "Choose a starting layout" placeholder for an empty wrapper block.
 */
import {
	useBlockProps,
	store as blockEditorStore,
	// eslint-disable-next-line @wordpress/no-unsafe-wp-apis -- stable in practice; used across core.
	__experimentalBlockVariationPicker as BlockVariationPicker,
} from '@wordpress/block-editor';
import { createBlocksFromInnerBlocksTemplate, store as blocksStore } from '@wordpress/blocks';
import { useDispatch, useSelect } from '@wordpress/data';

export default function VariationPlaceholder( { blockName, clientId, setAttributes, icon, label, instructions } ) {
	const { variations, defaultVariation } = useSelect(
		( select ) => ( {
			variations: select( blocksStore ).getBlockVariations( blockName, 'block' ),
			defaultVariation: select( blocksStore ).getDefaultBlockVariation( blockName, 'block' ),
		} ),
		[ blockName ]
	);
	const { replaceInnerBlocks } = useDispatch( blockEditorStore );

	return (
		<div { ...useBlockProps() }>
			<BlockVariationPicker
				icon={ icon }
				label={ label }
				instructions={ instructions }
				variations={ variations }
				allowSkip
				onSelect={ ( variation = defaultVariation ) => {
					if ( variation.attributes ) {
						setAttributes( variation.attributes );
					}
					if ( variation.innerBlocks ) {
						// false: keep the wrapper selected so its settings show next.
						replaceInnerBlocks( clientId, createBlocksFromInnerBlocksTemplate( variation.innerBlocks ), false );
					}
				} }
			/>
		</div>
	);
}
```

## src/reveal/edit.js

```js
/**
 * A settings-only wrapper: the visible parts are GenerateBlocks blocks inside
 * it, styled in GB's Styles panel. In the editor the panel always shows, so
 * it can be edited.
 */
import { __ } from '@wordpress/i18n';
import {
	InspectorControls,
	useBlockProps,
	useInnerBlocksProps,
	store as blockEditorStore,
} from '@wordpress/block-editor';
import { Notice, PanelBody, SelectControl, ToggleControl } from '@wordpress/components';
import { useSelect } from '@wordpress/data';

import VariationPlaceholder from '../shared/variation-placeholder';
import { revealIcon } from './icon';

/**
 * Which parts the block has, ignoring a nested Reveal's (they're its own).
 *
 * @param {Array}  blocks Inner blocks.
 * @param {Object} found  Accumulator.
 * @return {Object} { button, panel }
 */
function findParts( blocks, found = { button: false, panel: false } ) {
	blocks.forEach( ( block ) => {
		if ( 'acme/reveal' === block.name ) {
			return;
		}

		const part = block.attributes?.htmlAttributes?.[ 'data-reveal-part' ];

		if ( Object.hasOwn( found, part ?? '' ) ) {
			found[ part ] = true;
		}

		findParts( block.innerBlocks, found );
	} );

	return found;
}

function RevealEdit( { attributes, setAttributes, clientId } ) {
	const { startOpen, animation } = attributes;
	const { button, panel } = useSelect(
		( select ) => findParts( select( blockEditorStore ).getBlocks( clientId ) ),
		[ clientId ]
	);
	const innerBlocksProps = useInnerBlocksProps( useBlockProps() );

	return (
		<>
			<InspectorControls>
				<PanelBody title={ __( 'Reveal', 'acme' ) }>
					<ToggleControl
						__nextHasNoMarginBottom
						label={ __( 'Starts open', 'acme' ) }
						checked={ startOpen }
						onChange={ ( value ) => setAttributes( { startOpen: value } ) }
					/>
					<SelectControl
						__next40pxDefaultSize
						__nextHasNoMarginBottom
						label={ __( 'Opening animation', 'acme' ) }
						value={ animation }
						options={ [
							{ value: 'none', label: __( 'None', 'acme' ) },
							{ value: 'fade', label: __( 'Fade', 'acme' ) },
						] }
						help={ __( 'Visitors who prefer reduced motion never see it.', 'acme' ) }
						onChange={ ( value ) => setAttributes( { animation: value } ) }
					/>
					{ ( ! button || ! panel ) && (
						<Notice status="warning" isDismissible={ false }>
							{ __(
								'A Reveal needs a block set as its Button and one set as its Panel (select a block inside it → "Reveal part").',
								'acme'
							) }
						</Notice>
					) }
				</PanelBody>
			</InspectorControls>
			<div { ...innerBlocksProps } />
		</>
	);
}

// Two components, so each always calls the same hooks.
export default function Edit( props ) {
	const hasInnerBlocks = useSelect(
		( select ) => select( blockEditorStore ).getBlockCount( props.clientId ) > 0,
		[ props.clientId ]
	);

	return hasInnerBlocks ? (
		<RevealEdit { ...props } />
	) : (
		<VariationPlaceholder
			blockName="acme/reveal"
			clientId={ props.clientId }
			setAttributes={ props.setAttributes }
			icon={ revealIcon }
			label={ __( 'Reveal', 'acme' ) }
			instructions={ __( 'Choose a starting layout. Everything in it is a GenerateBlocks block you can restyle.', 'acme' ) }
		/>
	);
}
```

## src/reveal/parts.js

```js
/**
 * "Reveal part" panel on GenerateBlocks blocks inside a Reveal. The choice
 * is saved in the GB block's own htmlAttributes (GB saves and renders it).
 */
import { __ } from '@wordpress/i18n';
import { addFilter } from '@wordpress/hooks';
import { createHigherOrderComponent } from '@wordpress/compose';
import { InspectorControls, store as blockEditorStore } from '@wordpress/block-editor';
import { PanelBody, SelectControl } from '@wordpress/components';
import { useSelect } from '@wordpress/data';

const PART_BLOCKS = [ 'generateblocks/element', 'generateblocks/text', 'generateblocks/shape', 'generateblocks/media' ];
const KEY = 'data-reveal-part';

const withRevealPart = createHigherOrderComponent(
	( BlockEdit ) => ( props ) => {
		const { name, clientId, attributes, setAttributes, isSelected } = props;
		const isPartBlock = PART_BLOCKS.includes( name );

		// Runs for every block in the post: only look up parents for a selected GB block.
		const insideReveal = useSelect(
			( select ) =>
				isPartBlock &&
				isSelected &&
				select( blockEditorStore ).getBlockParentsByBlockName( clientId, 'acme/reveal', true ).length > 0,
			[ clientId, isPartBlock, isSelected ]
		);

		if ( ! insideReveal ) {
			return <BlockEdit { ...props } />;
		}

		const htmlAttributes = attributes.htmlAttributes || {};
		const value = htmlAttributes[ KEY ] || '';

		const onChange = ( next ) => {
			const updated = { ...htmlAttributes };

			if ( next ) {
				updated[ KEY ] = next;
			} else {
				delete updated[ KEY ];
			}

			setAttributes( { htmlAttributes: updated } );
		};

		const help = {
			button: __( 'Gets aria-expanded. Style the open state with the nested selector &[aria-expanded="true"]. Use a Text block set to <button> where you can.', 'acme' ),
			panel: __( 'Hidden while the Reveal is closed.', 'acme' ),
		};

		return (
			<>
				<BlockEdit { ...props } />
				<InspectorControls>
					<PanelBody title={ __( 'Reveal part', 'acme' ) }>
						<SelectControl
							__next40pxDefaultSize
							__nextHasNoMarginBottom
							label={ __( 'This block is the…', 'acme' ) }
							value={ value }
							options={ [
								{ value: '', label: __( 'Nothing special', 'acme' ) },
								{ value: 'button', label: __( 'Button (opens and closes)', 'acme' ) },
								{ value: 'panel', label: __( 'Panel (shown when open)', 'acme' ) },
							] }
							help={ help[ value ] }
							onChange={ onChange }
						/>
					</PanelBody>
				</InspectorControls>
			</>
		);
	},
	'withRevealPart'
);

addFilter( 'editor.BlockEdit', 'acme/reveal-part', withRevealPart );
```

## src/reveal/templates.js

```js
/**
 * Starting layouts. Looks come from GB Pro Global Styles the plugin creates
 * (includes/global-styles/reveal.php); without GB Pro they're unstyled.
 */
import { __ } from '@wordpress/i18n';

const classes = ( part, modifier ) =>
	modifier ? [ `acme-reveal__${ part }`, `acme-reveal__${ part }--${ modifier }` ] : [ `acme-reveal__${ part }` ];

const button = ( text, modifier ) => [
	'generateblocks/text',
	{
		tagName: 'button',
		content: text,
		htmlAttributes: { 'data-reveal-part': 'button', type: 'button' },
		globalClasses: classes( 'button', modifier ),
		metadata: { name: __( 'Button', 'acme' ) },
	},
];

const panel = ( modifier ) => [
	'generateblocks/element',
	{
		tagName: 'div',
		htmlAttributes: { 'data-reveal-part': 'panel' },
		globalClasses: classes( 'panel', modifier ),
		metadata: { name: __( 'Panel', 'acme' ) },
	},
	[ [ 'generateblocks/text', { tagName: 'p', content: __( 'Anything can go in here.', 'acme' ) } ] ],
];

export const variations = [
	{
		name: 'button',
		title: __( 'Button', 'acme' ),
		description: __( 'A solid button above the content.', 'acme' ),
		isDefault: true,
		scope: [ 'block' ],
		innerBlocks: [ button( __( 'Show details', 'acme' ) ), panel() ],
	},
	{
		name: 'link',
		title: __( 'Read more link', 'acme' ),
		description: __( 'A quiet text link.', 'acme' ),
		scope: [ 'block' ],
		innerBlocks: [ button( __( 'Read more', 'acme' ), 'link' ), panel( 'plain' ) ],
	},
];
```

## src/reveal/style.scss

```scss
// Front end only (block.json viewStyle). Structure, never looks.
//
// The browser hides [hidden] with display:none, but a GB class giving the
// panel display:flex would beat that, so the closed panel needs !important.
.acme-reveal [data-reveal-part="panel"][hidden] {
	display: none !important;
}

// :where() has no specificity, so anything set in GenerateBlocks wins.
:where(.acme-reveal [data-reveal-part="button"]) {
	cursor: pointer;
}
```

## includes/class-acme-reveal-render.php

```php
<?php
/**
 * Front-end rendering for the Reveal block.
 *
 * The parts are GenerateBlocks blocks saved as static HTML. This adds the
 * disclosure semantics on the server, so the markup is right before any
 * JavaScript runs and none of it has to be saved in post content.
 *
 * @package Acme
 */

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

/**
 * Renders the Reveal block.
 */
class Acme_Reveal_Render {

	const PART  = 'data-reveal-part';
	const OWNED = 'data-reveal-owned'; // Set on parts a Reveal has claimed, so an outer Reveal skips a nested one's parts.

	/**
	 * Whether the no-JavaScript fallback has been printed.
	 *
	 * @var bool
	 */
	private static $noscript_printed = false;

	/**
	 * Render callback.
	 *
	 * @param array  $attributes Block attributes.
	 * @param string $content    Saved inner-block HTML (inner blocks already rendered).
	 * @return string
	 */
	public static function render( $attributes, $content ) {
		$config  = self::config( $attributes );
		$content = self::decorate_parts( $content, $config['startOpen'] );

		$wrapper = array(
			'class'            => 'acme-reveal',
			'data-acme-reveal' => wp_json_encode( $config ),
		);

		// Dynamic blocks have to print the HTML anchor themselves.
		if ( ! empty( $attributes['anchor'] ) ) {
			$wrapper['id'] = $attributes['anchor'];
		}

		return sprintf(
			'<div %1$s>%2$s</div>%3$s',
			get_block_wrapper_attributes( $wrapper ),
			$content,
			self::noscript()
		);
	}

	/**
	 * Every attribute, whitelisted. The view script only ever sees this.
	 *
	 * @param array $attributes Block attributes.
	 * @return array
	 */
	public static function config( $attributes ) {
		$animation = $attributes['animation'] ?? 'fade';

		return array(
			'startOpen' => ! empty( $attributes['startOpen'] ),
			'animation' => in_array( $animation, array( 'none', 'fade' ), true ) ? $animation : 'none',
		);
	}

	/**
	 * Wire the button(s) to the panel(s): ids, aria-controls, aria-expanded,
	 * hidden, and keyboard access for a button that isn't a <button>.
	 *
	 * @param string $content Inner HTML.
	 * @param bool   $open    Initial state.
	 * @return string
	 */
	private static function decorate_parts( $content, $open ) {
		if ( '' === trim( $content ) ) {
			return $content;
		}

		// First pass: each panel's id (kept if it has one), so buttons can point at them.
		$panel_ids = array();
		$processor = new WP_HTML_Tag_Processor( $content );

		while ( $processor->next_tag() ) {
			if ( 'panel' === $processor->get_attribute( self::PART ) && null === $processor->get_attribute( self::OWNED ) ) {
				$id          = $processor->get_attribute( 'id' );
				$panel_ids[] = is_string( $id ) && '' !== $id ? $id : wp_unique_id( 'acme-reveal-panel-' );
			}
		}

		$panel     = 0;
		$processor = new WP_HTML_Tag_Processor( $content );

		while ( $processor->next_tag() ) {
			$part = $processor->get_attribute( self::PART );

			if ( ! is_string( $part ) || null !== $processor->get_attribute( self::OWNED ) ) {
				continue;
			}

			$processor->set_attribute( self::OWNED, '' );

			if ( 'button' === $part ) {
				if ( 'BUTTON' === $processor->get_tag() ) {
					$processor->set_attribute( 'type', 'button' ); // Never submits a surrounding form.
				} else {
					$processor->set_attribute( 'role', 'button' );
					$processor->set_attribute( 'tabindex', '0' );
				}

				$processor->set_attribute( 'aria-expanded', $open ? 'true' : 'false' );

				if ( $panel_ids ) {
					$processor->set_attribute( 'aria-controls', implode( ' ', $panel_ids ) );
				}
			} elseif ( 'panel' === $part && isset( $panel_ids[ $panel ] ) ) {
				$processor->set_attribute( 'id', $panel_ids[ $panel++ ] );

				if ( ! $open ) {
					$processor->set_attribute( 'hidden', true );
				}
			}
		}

		return $processor->get_updated_html();
	}

	/**
	 * Without JavaScript nothing could open a closed panel, so show them all.
	 *
	 * @return string
	 */
	private static function noscript() {
		if ( self::$noscript_printed ) {
			return '';
		}

		self::$noscript_printed = true;

		return '<noscript><style>.acme-reveal [data-reveal-part="panel"][hidden]{display:block!important}</style></noscript>';
	}
}
```

## src/reveal/view.js

```js
/**
 * Front end for the Reveal block (WAI-ARIA disclosure pattern).
 *
 * The server has already rendered the closed/open state, ids and ARIA
 * attributes; this only handles clicks and keys. The config in
 * data-acme-reveal could be forged, so every value is checked again here.
 */
const SELECTOR = '.acme-reveal[data-acme-reveal]';
const reducedMotion = window.matchMedia( '(prefers-reduced-motion: reduce)' );

// Web Animations, not CSS keyframes: "remove unused CSS" optimisers can't strip them.
const ANIMATIONS = {
	none: null,
	fade: [ { opacity: 0 }, { opacity: 1 } ],
};

function readConfig( root ) {
	let config;

	try {
		config = JSON.parse( root.dataset.acmeReveal || '{}' );
	} catch ( e ) {
		config = {};
	}

	const animation = config && 'string' === typeof config.animation ? config.animation : 'none';

	// hasOwn: a forged "__proto__" or "constructor" must not reach the prototype.
	return { frames: Object.hasOwn( ANIMATIONS, animation ) ? ANIMATIONS[ animation ] : null };
}

// This Reveal's own parts, not those of a Reveal nested inside it.
function ownParts( root, part ) {
	return [ ...root.querySelectorAll( `[data-reveal-part="${ part }"]` ) ].filter(
		( element ) => element.closest( SELECTOR ) === root
	);
}

function setOpen( reveal, open ) {
	// Never leave focus inside content that's about to be hidden.
	const focused = reveal.buttons[ 0 ].ownerDocument.activeElement;

	if ( ! open && reveal.panels.some( ( panel ) => panel.contains( focused ) ) ) {
		reveal.buttons[ 0 ].focus();
	}

	reveal.open = open;
	reveal.buttons.forEach( ( button ) => button.setAttribute( 'aria-expanded', open ? 'true' : 'false' ) );
	reveal.panels.forEach( ( panel ) => {
		panel.hidden = ! open;

		if ( open && reveal.frames && ! reducedMotion.matches ) {
			panel.animate( reveal.frames, { duration: 200, easing: 'ease-out' } );
		}
	} );
}

function setup( root ) {
	if ( root.acmeReveal ) {
		return; // Already set up.
	}

	const buttons = ownParts( root, 'button' );
	const panels = ownParts( root, 'panel' );

	if ( ! buttons.length || ! panels.length ) {
		return;
	}

	const reveal = {
		buttons,
		panels,
		// The markup is the truth: the server rendered the starting state.
		open: 'true' === buttons[ 0 ].getAttribute( 'aria-expanded' ),
		...readConfig( root ),
	};

	root.acmeReveal = reveal;

	buttons.forEach( ( button ) => {
		button.addEventListener( 'click', ( event ) => {
			event.preventDefault(); // A link used as the button mustn't navigate.
			setOpen( reveal, ! reveal.open );
		} );

		if ( 'BUTTON' !== button.tagName ) {
			button.addEventListener( 'keydown', ( event ) => {
				// Space for every role="button"; Enter too, except on links (Enter already clicks them).
				if ( ' ' === event.key || ( 'Enter' === event.key && 'A' !== button.tagName ) ) {
					event.preventDefault();
					setOpen( reveal, ! reveal.open );
				}
			} );
		}
	} );
}

function init() {
	document.querySelectorAll( SELECTOR ).forEach( setup );
}

// For content added later, and for tests.
window.acmeReveal = { init };

if ( 'loading' === document.readyState ) {
	document.addEventListener( 'DOMContentLoaded', init );
} else {
	init();
}
```

## includes/global-styles/reveal.php

```php
<?php
/**
 * Default Global Styles for the Reveal's starting layouts. Base classes
 * before their modifiers. State is styled on the part: &[aria-expanded="true"].
 *
 * @package Acme
 */

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

// A function, so the helper variables don't become globals.
return static function () {
	$accent = '#1e73be'; // 4.8:1 with white text.
	$text   = '#222222';
	$focus  = array(
		'outlineWidth'  => '2px',
		'outlineStyle'  => 'solid',
		'outlineColor'  => $accent,
		'outlineOffset' => '3px',
	);

	return array(
		'acme-reveal__button'        => array(
			'display'                 => 'inline-flex',
			'alignItems'              => 'center',
			'columnGap'               => '0.5em',
			'paddingTop'              => '0.6em',
			'paddingRight'            => '1.2em',
			'paddingBottom'           => '0.6em',
			'paddingLeft'             => '1.2em',
			'borderTopWidth'          => '0',
			'borderRightWidth'        => '0',
			'borderBottomWidth'       => '0',
			'borderLeftWidth'         => '0',
			'borderTopLeftRadius'     => '4px',
			'borderTopRightRadius'    => '4px',
			'borderBottomRightRadius' => '4px',
			'borderBottomLeftRadius'  => '4px',
			'backgroundColor'         => $accent,
			'color'                   => '#ffffff',
			'fontWeight'              => '600',
			'&:focus-visible'         => $focus,
		),
		'acme-reveal__button--link'  => array(
			'paddingTop'              => '0',
			'paddingRight'            => '0',
			'paddingBottom'           => '0',
			'paddingLeft'             => '0',
			'backgroundColor'         => 'transparent',
			'color'                   => $accent,
			'textDecorationLine'      => 'underline',
			'&[aria-expanded="true"]' => array(
				'color' => $text,
			),
		),
		'acme-reveal__panel'         => array(
			'marginTop'       => '1rem',
			'paddingTop'      => '1rem',
			'paddingRight'    => '1rem',
			'paddingBottom'   => '1rem',
			'paddingLeft'     => '1rem',
			'backgroundColor' => '#f7f8f9',
			'color'           => $text,
		),
		'acme-reveal__panel--plain'  => array(
			'paddingTop'      => '0',
			'paddingRight'    => '0',
			'paddingBottom'   => '0',
			'paddingLeft'     => '0',
			'backgroundColor' => 'transparent',
		),
	);
};
```

## includes/class-acme-global-styles.php

Implements the installer rules in gb-integration.md. GB Pro internals are
guarded; re-check them against the installed GB Pro version.

```php
<?php
/**
 * Creates the layouts' default GenerateBlocks Pro Global Styles, once, on an
 * ordinary admin page load. Never overwrites a class: once created it's the
 * site's to edit or delete.
 *
 * @package Acme
 */

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

/**
 * Default Global Styles installer.
 */
class Acme_Global_Styles {

	const OPTION   = 'acme_global_styles'; // Class names already handled.
	const CATEGORY = 'Acme Blocks';
	const BLOCKS   = array( 'reveal' );

	/**
	 * Hook up.
	 */
	public static function init() {
		add_action( 'admin_init', array( __CLASS__, 'maybe_create' ) );
	}

	/**
	 * Every block's defaults, base classes before modifiers.
	 *
	 * @return array Class name => GB styles.
	 */
	public static function defaults() {
		$classes = array();

		foreach ( self::BLOCKS as $block ) {
			$file = ACME_DIR . 'includes/global-styles/' . $block . '.php';

			if ( file_exists( $file ) ) {
				$build   = require $file;
				$classes = array_merge( $classes, $build() );
			}
		}

		return $classes;
	}

	/**
	 * Whether GB Pro's Global Styles are there (internal API, so checked).
	 *
	 * @return bool
	 */
	public static function available() {
		return class_exists( 'GenerateBlocks_Pro_Styles' )
			&& method_exists( 'GenerateBlocks_Pro_Styles', 'get_class_by_name' )
			&& post_type_exists( 'gblocks_styles' );
	}

	/**
	 * Create classes this site hasn't had yet.
	 */
	public static function maybe_create() {
		// Not the background requests an admin page fires (AJAX, REST, cron).
		if ( wp_doing_ajax() || wp_doing_cron() || ( defined( 'REST_REQUEST' ) && REST_REQUEST ) ) {
			return;
		}

		$can_manage = method_exists( 'GenerateBlocks_Pro_Styles', 'can_manage_styles' )
			? GenerateBlocks_Pro_Styles::can_manage_styles()
			: current_user_can( 'manage_options' );

		if ( ! self::available() || ! $can_manage ) {
			return;
		}

		$handled = (array) get_option( self::OPTION, array() );
		$new     = array_diff_key( self::defaults(), array_flip( $handled ) );

		if ( ! $new || ! self::lock() ) {
			return;
		}

		// Another request may have just finished: re-read now that we hold the lock.
		wp_cache_delete( 'alloptions', 'options' );
		wp_cache_delete( self::OPTION, 'options' );
		$handled = (array) get_option( self::OPTION, array() );
		$new     = array_diff_key( $new, array_flip( $handled ) );

		if ( $new ) {
			self::create( $new, $handled );
		}

		delete_option( self::OPTION . '_lock' );
	}

	/**
	 * An atomic lock, so two admin tabs can't both create the classes. A lock
	 * older than a minute (a request that died) is cleared.
	 *
	 * @return bool
	 */
	private static function lock() {
		global $wpdb;

		$key = self::OPTION . '_lock';

		// phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching -- must read the database, not the cache.
		$since = $wpdb->get_var( $wpdb->prepare( "SELECT option_value FROM {$wpdb->options} WHERE option_name = %s", $key ) );

		if ( null !== $since && (int) $since < time() - MINUTE_IN_SECONDS ) {
			// phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching -- compare-and-delete.
			$wpdb->query( $wpdb->prepare( "DELETE FROM {$wpdb->options} WHERE option_name = %s AND option_value = %s", $key, $since ) );
		}

		// add_option() can't be used: it quietly succeeds if the row exists.
		// phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching -- an atomic lock.
		return 1 === (int) $wpdb->query( $wpdb->prepare( "INSERT IGNORE INTO {$wpdb->options} (option_name, option_value, autoload) VALUES (%s, %s, 'no')", $key, (string) time() ) );
	}

	/**
	 * Insert the classes, pausing GB Pro's per-save stylesheet rebuild and
	 * rebuilding once at the end. Progress is saved after each class.
	 *
	 * @param array    $classes Class name => styles.
	 * @param string[] $handled Already handled.
	 */
	private static function create( $classes, $handled ) {
		global $wpdb;

		$builder = null;

		if ( class_exists( 'GenerateBlocks_Pro_Enqueue_Styles' ) && method_exists( 'GenerateBlocks_Pro_Enqueue_Styles', 'build_css' ) ) {
			$builder = GenerateBlocks_Pro_Enqueue_Styles::get_instance();
		}

		$paused  = $builder && remove_action( 'wp_after_insert_post', array( $builder, 'build_css_file_on_save' ), 100 );
		$created = false;

		// phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching -- one read, only when there's work to do.
		$order = (int) $wpdb->get_var( $wpdb->prepare( "SELECT MAX(menu_order) FROM {$wpdb->posts} WHERE post_type = %s", 'gblocks_styles' ) );

		foreach ( $classes as $class_name => $styles ) {
			$selector = '.' . $class_name;

			if ( ! GenerateBlocks_Pro_Styles::get_class_by_name( $selector ) ) {
				$post_id = wp_insert_post(
					array(
						'post_type'   => 'gblocks_styles',
						'post_status' => 'publish',
						'post_title'  => $selector,
						'menu_order'  => ++$order, // Base classes first, so modifiers win.
						'meta_input'  => array(
							'gb_style_selector' => $selector,
							'gb_style_data'     => $styles,
							'gb_style_css'      => self::compile( $selector, $styles ),
							'gb_style_category' => self::CATEGORY,
						),
					),
					true
				);

				if ( is_wp_error( $post_id ) ) {
					continue; // Not marked handled: tried again next time.
				}

				$created = true;
			}

			$handled[] = $class_name;
			update_option( self::OPTION, $handled, false );
		}

		if ( $paused ) {
			add_action( 'wp_after_insert_post', array( $builder, 'build_css_file_on_save' ), 100, 2 );
		}

		if ( $created && $builder ) {
			$builder->build_css();
		}
	}

	/**
	 * GB styles → CSS: declarations, nested "&…"/descendant keys, @media.
	 *
	 * @param string $selector Selector.
	 * @param array  $styles   GB styles.
	 * @return string
	 */
	public static function compile( $selector, $styles ) {
		$declarations = '';
		$nested       = '';

		foreach ( $styles as $property => $value ) {
			if ( is_array( $value ) ) {
				$nested .= 0 === strpos( $property, '@' )
					? $property . '{' . self::compile( $selector, $value ) . '}'
					: self::compile( self::nest( $selector, $property ), $value );
				continue;
			}

			$declarations .= strtolower( preg_replace( '/([a-z])([A-Z])/', '$1-$2', $property ) ) . ':' . $value . ';';
		}

		return ( $declarations ? $selector . '{' . $declarations . '}' : '' ) . $nested;
	}

	/**
	 * "&" is the parent; a key without one is a descendant. Splits on
	 * top-level commas only (not those inside :is( … )).
	 *
	 * @param string $parent Parent selector.
	 * @param string $key    Nested key.
	 * @return string
	 */
	private static function nest( $parent, $key ) {
		$parts = array();
		$depth = 0;
		$part  = '';

		foreach ( str_split( $key ) as $char ) {
			if ( ',' === $char && 0 === $depth ) {
				$parts[] = $part;
				$part    = '';
				continue;
			}

			$depth += ( '(' === $char ) - ( ')' === $char );
			$part  .= $char;
		}

		$parts[] = $part;

		foreach ( $parts as $i => $selector ) {
			$selector    = trim( $selector );
			$parts[ $i ] = false !== strpos( $selector, '&' ) ? str_replace( '&', $parent, $selector ) : $parent . ' ' . $selector;
		}

		return implode( ',', $parts );
	}
}

Acme_Global_Styles::init();
```

---

## Tests to write for it (see testing.md)

- Server markup before JS: button has `aria-expanded="false"` and
  `aria-controls` = panel id; panel has `hidden`; `<button>` has
  `type="button"`.
- Click and keyboard (Enter, Space) open and close; focus returns to the
  button if it was inside the panel when closing.
- A Reveal nested in another Reveal's panel works independently.
- `startOpen` renders open; `animation: "__proto__"` forged in the config
  doesn't throw.
- Reduced motion: no animation.
- Assets: `build/reveal/view.js` loads only on pages with a Reveal.
- Editor: the demo Reveal is valid; inserting each variation is valid;
  the "Reveal part" panel appears only inside a Reveal.
