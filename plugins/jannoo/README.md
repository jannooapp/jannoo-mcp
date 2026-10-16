# Jannoo

[Jannoo](https://jannoo.app/) is a native macOS kanban app whose boards are plain folders on disk:
a `board.json` plus one Markdown file per card, Obsidian-compatible and git-versionable. This plugin
lets Claude read and change those boards.

**The plugin drives the installed app. It does not contain Jannoo.** Install the app first from
<https://jannoo.app/download/>. macOS only.

## What the plugin contains

- **The `jannoo-cli` skill** — when and how to drive boards with the app's `jannoo` command-line
  tool: daily plans, triage, stand-ups, audits, and every kind of card change.
- **The app's MCP server** — every CLI verb as a tool. `scripts/jannoo-mcp` is a short shell
  launcher: it finds the installed `Jannoo.app`, checks that its helper is signed by Cadence Hill LLC
  with a signature anchored to Apple's certificate authority and that it answers the `mcp` verb, and
  then runs that helper as `jannoo mcp`. Nothing is downloaded, compiled, or sent anywhere. The
  server reads and writes board folders on your Mac and nothing else.

## Try these

- "What should I work on today?" — a plan from due dates, priorities, and blockers across your boards.
- "Move the card about the launch email to Done and tell me what is still open in that column."
- "Write my stand-up from what changed on the Sprint board since yesterday."

## Writing is opt-in per board

A board accepts agent writes only when it carries a `.jannoo/agent-policy.json` file. With no policy
file the server is read-only for that board, and a refusal is the correct behaviour. Every write is a
dry run until it is repeated with `apply: true`.

## Troubleshooting

- *The server exits with "Jannoo.app is not installed"*: install the app, or set `JANNOO_APP_PATH`
  to the app bundle if it lives somewhere unusual.
- *"a Jannoo was found but is not a signed build with the mcp verb"*: the copy found is not a
  current signed release. Install the current app from the download page.
- *Tools appear twice*: you also have a hand-written `jannoo` server entry from `jannoo agent
  install`. Remove one.

Support: <support@jannoo.app>. Security reports: <security@jannoo.app>.
