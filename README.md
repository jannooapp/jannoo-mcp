# Jannoo for Claude Code and Codex

[Jannoo](https://jannoo.app/) is a native macOS kanban app whose boards are plain folders on disk:
a `board.json` plus one Markdown file per card, Obsidian-compatible and git-versionable.

This repository is a plugin marketplace with one plugin, **`jannoo`**, which gives an agent two things:

- **The `jannoo-cli` skill** — when and how to drive boards with the app's `jannoo` command-line tool.
- **The app's MCP server** — every CLI verb as a tool, served by `jannoo mcp` from the installed app.

**The plugin drives the installed app. It does not contain Jannoo.** Install the app first from
<https://jannoo.app/download/>. Without it, the server exits with one line pointing at the download.

## Install

Claude Code:

```sh
claude plugin marketplace add jannooapp/jannoo-mcp
claude plugin install jannoo@jannoo-mcp
```

Codex CLI:

```sh
codex plugin marketplace add jannooapp/jannoo-mcp
codex plugin add jannoo@jannoo-mcp
```

Start a new session afterwards. In Claude Code the skill is `/jannoo:jannoo-cli` and the tools are
`mcp__plugin_jannoo_jannoo__jannoo_*`; in Codex the skill is `jannoo:jannoo-cli` and the tools are
`mcp__jannoo__jannoo_*`.

Writing to a board is opt-in per board through `.jannoo/agent-policy.json`; with no policy file the
tools are read-only, and that refusal is correct. Run `jannoo agent policy` or open the board in the
app to grant it.

## For an organisation

Claude Code managed settings can register this marketplace and install the plugin on every machine:

```json
{
  "extraKnownMarketplaces": {
    "jannoo-mcp": { "source": { "source": "github", "repo": "jannooapp/jannoo-mcp" } }
  },
  "enabledPlugins": { "jannoo@jannoo-mcp": true }
}
```

The app itself is a separate rollout (the DMG, through your MDM). Codex workspace admins can import
and sync this repository as a marketplace from the ChatGPT admin console.

## How the server is found

`plugins/jannoo/scripts/jannoo-mcp` is a shell launcher. It takes the first of these that is a
Jannoo build signed by Cadence Hill LLC (team `8A6D992J4P`, checked against Apple's certificate authority) and carrying the `mcp` verb:

1. `$JANNOO_APP_PATH` — an app bundle or a helper path, when set
2. `/Applications/Jannoo.app`, then `~/Applications/Jannoo.app`
3. the `jannoo` symlink that `jannoo install` writes (`~/.local/bin`, `/usr/local/bin`), or a Homebrew link in `/opt/homebrew/bin`
4. Spotlight, by bundle identifier `app.jannoo.mac`

Steps 2–4 skip any copy under `/Volumes/` — a mounted disk image, or an app folder on another drive. To
run Jannoo from another volume, point `JANNOO_APP_PATH` at it; an explicit path is used after it passes the
same signature check. The server bytes are whatever the
installed app is; the app's own updater keeps them current, so this plugin's version changes only when
the launcher or the skill changes.

Environment reaches the helper unchanged: `JANNOO_BOARDS_DIR` scopes which boards are visible,
`JANNOO_AGENT` names the agent in receipts.

## Already have a hand-written entry?

`jannoo agent install mcp-claude-code` / `mcp-codex` write a `jannoo` server entry into a project's
`.mcp.json` or Codex's `config.toml`. You do not need both: in Claude Code the two entries load the same
tools twice under two prefixes. In Codex, a `[mcp_servers.jannoo]`
table in `config.toml` takes precedence over the plugin's server of the same name — if that table is
disabled rather than removed, the plugin's server stays hidden too, so remove it to switch to the plugin.

## Layout

```
.claude-plugin/marketplace.json     Claude Code marketplace (Codex reads it too)
.agents/plugins/marketplace.json    Codex marketplace
plugins/jannoo/
  .claude-plugin/plugin.json        Claude Code manifest → hosts/claude-code.json
  .codex-plugin/plugin.json         Codex manifest       → hosts/codex.json
  hosts/claude-code.json            command: ${CLAUDE_PLUGIN_ROOT}/scripts/jannoo-mcp
  hosts/codex.json                  command: ./scripts/jannoo-mcp, cwd: .
  scripts/jannoo-mcp                the launcher
  skills/jannoo-cli/SKILL.md        the skill, identical to the one the app installs
```

## Support

Questions and problems with the plugin: <support@jannoo.app>, or an issue on this repository.
Security reports: <security@jannoo.app> — see `SECURITY.md`.

MIT licence for the files in this repository. The app is a separate, licensed product.
