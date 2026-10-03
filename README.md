# GenerateBlocks Add-on Builder: a skill for Claude

This skill teaches Claude (Claude Code, the Claude apps, or the API) how to
build WordPress add-ons for **GenerateBlocks 2** and **GenerateBlocks Pro**
that feel like part of GenerateBlocks:

- New blocks are **wrappers around real GenerateBlocks blocks**. You style
  everything with GB's own Styles panel, Global Styles and design tokens,
  not a second set of controls.
- Extras for GB's own blocks (animations, masks, backgrounds) are stored
  where GenerateBlocks already saves things, so content keeps working if the
  add-on is switched off.
- **Security, accessibility and performance are planned before any code is
  written**, then checked with real browser tests on a local WordPress.

It doesn't contain a plugin. It contains the *method*: the patterns, the
GenerateBlocks hooks, the gotchas, a working example block to learn from,
and a review checklist.

## What's inside

```
generateblocks-addon-builder/
├── SKILL.md                     # the rules and the workflow (Claude reads this first)
└── references/
    ├── architecture.md          # wrapper blocks, part panels, PHP rendering, extending GB blocks
    ├── gb-integration.md        # GB attributes, styles format, Global Styles, icons, tokens, GeneratePress
    ├── security.md              # what contributors can do, and how to make it harmless
    ├── accessibility.md         # APG patterns, keyboard, motion, announcements, contrast
    ├── performance.md           # load nothing where unused, no flash, lean front ends
    ├── testing.md               # WordPress Playground + Playwright + CI
    ├── starter-code.md          # a complete example "Reveal" block
    └── review-checklist.md      # before you commit or release
```

## What you need

### Must have

1. **Claude Code**, in the Claude desktop app's Code tab, the terminal, or
   VS Code. It needs a paid Claude plan (Pro or Max). On claude.ai, the
   skill can only write the code; Claude Code is what builds it, runs the
   test site and runs the browser tests.
2. **Node.js 20 or newer**: the "LTS" download from
   [nodejs.org](https://nodejs.org). This one installer covers building the
   blocks, the local WordPress test site and the browser tests.

That's it to start. You **don't** need PHP, MySQL, Local, MAMP or Docker:
[WordPress Playground](https://wordpress.org/playground/) runs WordPress
inside Node, and installs free GenerateBlocks and GeneratePress by itself.
You don't install the build or test tools (`@wordpress/scripts`,
Playground, Playwright) yourself either. Claude adds them to the project.

### Nice to have

- **Google Chrome.** The tests can use it. Without it, Playwright downloads
  its own browser (about 150 MB, once).
- **Git and a GitHub account**, for version history and automatic checks on
  every push (coding standards, Plugin Check, tests). Claude Code can walk
  you through setting these up.
- **A real WordPress site with GenerateBlocks**, ideally staging. Build and
  test in Playground, but try the plugin zip on a real site before using it
  anywhere important.
- **GenerateBlocks Pro.** Without it everything works, but the starting
  layouts are unstyled (their styles are Pro Global Styles). Pro can only be
  tested on a real site, not in Playground, because it isn't on
  wordpress.org.

### Your first build

1. Install Node.js.
2. Install the skill (below).
3. Make an empty folder, open it in Claude Code, and ask: *"Build me a
   GenerateBlocks tabs block."*

Claude asks you about security, accessibility and performance, sets up the
project, builds the block, starts a local test site at
`http://127.0.0.1:9400` and runs the tests.

## Install

First, download **`generateblocks-addon-builder.zip`** from the
[latest release](https://github.com/Calvin-Susan/generateblocks-addon-builder/releases/latest).
(Don't use the green "Code → Download ZIP" button. It wraps everything in an
extra folder.)

### Claude desktop app or claude.ai

1. Open **Settings → Capabilities** and make sure **Skills** is turned on.
2. Click **Upload skill** and choose the zip you downloaded. Don't unzip it.
3. Make sure the skill's switch is on.

### Claude Code (terminal, desktop Code tab, or IDE)

Unzip the download. You'll get a folder called `generateblocks-addon-builder`.
Move that folder into your skills folder:

- **For all your projects:** `~/.claude/skills/` (on a Mac, `~` is your home
  folder; press Cmd+Shift+. in Finder to show the hidden `.claude` folder).
- **For one plugin only:** `.claude/skills/` inside that plugin's folder.

Or in Terminal, from the folder where you unzipped it:

```bash
mkdir -p ~/.claude/skills && mv generateblocks-addon-builder ~/.claude/skills/
```

Start a new Claude Code session afterwards. Skills are picked up when a
session starts.

### Check it's working

Ask Claude: *"What skills do you have for GenerateBlocks?"* It should mention
generateblocks-addon-builder.

### Use it

Just ask for what you want, e.g. *"Build a GenerateBlocks block that
switches between monthly and annual pricing"*. Claude picks up the skill from
the request. To be sure it's used, name it: *"Use the
generateblocks-addon-builder skill to…"*.

## What to expect

Before writing code, Claude will ask you to agree:

1. **Security**: what a contributor could type that ends up on the page,
   and how each value is cleaned.
2. **Accessibility**: how each control works for keyboard and screen reader
   users.
3. **Performance**: what loads, where, and how the page avoids flashing or
   jumping.

Then it builds one feature at a time: block, layouts, tests, review, docs.

## Caveats

- Built plugins target GenerateBlocks 2.x, WordPress 6.6+ and PHP 7.4+.
- Some GenerateBlocks Pro internals the skill uses (how Global Styles are
  stored, a few editor globals) aren't official public API. The skill marks
  them and tells Claude to check them against your installed version.
- It's a starting point, not a guarantee. Review what gets built, and test
  on a staging site before production.

## Licence

MIT. Use it, change it, share it.
