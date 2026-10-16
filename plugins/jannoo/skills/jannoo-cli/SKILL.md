---
name: jannoo-cli
description: >
  Read AND change Jannoo kanban boards from the shell with the `jannoo` command-line tool. Use
  whenever the user asks about their board or tickets — "what should I work on today", "give me my
  daily plan", "what's overdue", "what's blocked", "triage the new cards", "write my stand-up",
  "summarize what changed since yesterday", "audit the board before I stop", "what's in Backlog /
  In Progress", "find the card about X", "is this board valid" — and whenever they ask you to
  change one: "add a card", "move it to Done", "mark it complete", "set the due date", "tag it",
  "attach this file", "link these two cards", "archive the finished cards", "bring that card back",
  "delete that card", "duplicate that card", "comment on that card", "create a board". Boards are
  plain folders on disk; this tool reads and writes them through the app's own code, so what it
  reports is exactly what the app shows and what it writes is byte-for-byte an app write. Writing is OPT-IN per board via
  `.jannoo/agent-policy.json` — no policy file means read-only, and that refusal is correct
  behaviour. It also drives the RUNNING app — "show me that card in Jannoo", "reload the board", "sync
  it now" — through `jannoo open`, `reload` and `sync --now`. NOT for editing board files by hand.
allowed-tools: "Bash Read"
---

# Reading and changing a Jannoo board with `jannoo`

A Jannoo board is a **folder on disk**: a `board.json` (columns, labels, custom-field definitions)
plus one Markdown file per card under `cards/<card-id>.md`. The `jannoo` tool reads those folders
through the app's own code, so its answers match what the app shows, whether or not the app is
running.

## The one rule

**Never invent a flag.** This file names the verbs and the flags that are guaranteed. For anything
else — a filter, a sort, a field selector — run `jannoo <verb> --help` and use what it prints. A
guessed flag either errors with exit `64` or, worse, is accepted and means something else. The
fallback that always works is `--json` plus `jq`: get the whole answer, filter it yourself.

## Step 0 — is the tool installed?

```sh
jannoo boards list        # if this runs, you are set
```

If `jannoo: command not found`, the tool ships **inside the app** and is runnable in place:

```sh
'/Applications/Jannoo.app/Contents/Helpers/jannoo' boards list
```

**Use that full path for the rest of the session.** It works from anywhere, needs no setup, and is
always correct.

To put it on the user's `PATH` — **their call, not yours; offer it, do not run it unasked**:

```sh
'/Applications/Jannoo.app/Contents/Helpers/jannoo' install
```

That symlinks the tool into `/usr/local/bin` (or `~/.local/bin`, printing the one `PATH` line to
add). It is idempotent, so re-running it is safe, and it refuses to replace a real file it did not
put there. **Jannoo cannot do this from the app**: the app is sandboxed and so is anything it
launches, which is why Settings ▸ General shows a command to copy rather than a button.

If the app is somewhere other than `/Applications`, ask — do not guess a path.

**Setting a project up for writing** is one command, and it asks before it does anything:
`jannoo agent install claude-code` copies this skill into the project's `.claude/skills/` and, after
a y/N confirmation, writes a permissive `agent-policy.json` into the board's control directory
(`.jannoo/`, or `.plank/` on a board no new build has opened yet). `codex` and `cursor` write the
equivalent file for those hosts. It never overwrites either file silently. Offer it; do not run it
unasked.

**The policy alone** is `jannoo agent policy [<board>] --read-only` or `--writable`. `--read-only`
writes `{"readOnly": true}` at once — narrowing needs no consent, and it replaces the file, scopes
included. On a board with no policy file it writes one, so widening that board later needs the
interactive y/N below. `--writable` writes the same permissive file `agent install` does, through the
same confirmation: never over an existing file without `--force` (diff-first until `--apply`), and
the `--yes` rule below applies to it word for word. Either one on a board whose file already says so
changes nothing. Offer it; do not run it unasked.

**You cannot `--yes` your way past a policy that already says no.** If the board already has an
`agent-policy.json` that is *more restrictive* than the permissive one — `readOnly`, a `boards`
scope that excludes this board, a `columns` or `fields` scope, or a file that will not parse —
replacing it needs the **interactive** y/N, and `--yes` is refused with `policy.replace_needs_consent`
(exit `2`). That is correct behaviour: relay it and let the user answer in their own terminal, or
offer to edit the JSON by hand. An absent policy, or one already permissive for this board, is
unaffected.

## Step 1 — find the board

```sh
jannoo boards list --json
```

It unions three sources: boards the app has registered, a scan of the boards folder and of
`$JANNOO_BOARDS_DIR`, and the current directory if it holds a `board.json`. A folder that has moved
is dropped and reported.

- **One board** → just work.
- **Several** → ask which, unless the user named one. Do not silently pick.
- **`--board <path>`** addresses a folder directly and needs no registry. Use it when the user gives
  you a path, and pass it to every later call in that session.

## The verbs

`--help` on any of them prints its own flags and exit codes.

| Command | What you get |
|---|---|
| `jannoo boards list` | every board found — id, title, path, and which source found it |
| `jannoo boards show [<id\|path\|title>]` | one board's columns, labels, swimlanes, custom fields, and `status` — the live sync state the running app publishes: `live`, a `reason` when not live (`app_not_running`, `written_by_another_run`, `unreadable`), and a `github` / `linear` member each carrying `state` (`idle`, `syncing`, `paused`, `inactive` = bound but not being synced by the app right now, `error`, `serverUnavailable`, `unreachable`, `rateLimited`), `lastSyncedAt` and `conflictCount`. `null` until the app has synced the board. A not-live answer is the LAST-KNOWN state — trust a state only when `live` is true. `--format detailed` adds the archive census, quarantined files, conflict copies, the GitHub binding and the agent policy |
| `jannoo boards add <path>` | registers a board folder the app has never opened, so later calls can name it by id or title. Writes the registry, never a board |
| `jannoo boards validate [<id\|path\|title>]` | checks the board-format contract: referenced ids exist, `relatedTo` symmetric, `parent` one level, attachment bytes present, ISO-8601 UTC. **Exits 2 on any error-severity finding.** `--repair` makes it a write — see Writing, below |
| `jannoo cards list` | cards on one board, in board order, filtered — see the filter flags below. `--archived` lists the board's **archived** cards instead (same shapes, same filters, newest completion first), each with the etag `cards unarchive --if-match` takes |
| `jannoo cards show <card-id>` | one card. `--format detailed` adds the notes body, checklist, links, relations, custom fields and preserved frontmatter keys |
| `jannoo cards search <query>` | text search over titles, notes, label names and custom-field values — the same matcher the app's ⌘K palette uses. Each hit names its `boardID`, `boardTitle`, `boardPath` |
| `jannoo agents install` | drops the `AGENTS.md` format guide into a board folder. Never replaces an existing one silently: `--force` shows a diff against the current guide and `--force --apply` writes it. **Refused on a read-only board** — see Writing, below |
| `jannoo log` | replays `.jannoo/agent-log.jsonl`. An absent or empty ledger is a success with no entries; one that exists and will not parse refuses |
| `jannoo install` | symlinks the tool onto the user's `PATH`. Idempotent; refuses to replace a real file it did not put there |

**Writes** — every one needs the board's policy to allow it (see Writing, below):

| Command | What it does |
|---|---|
| `jannoo cards create --title <t>` | new card. `--column ID\|TITLE` (default: the leftmost/intake column) · `--notes <s>` or `--notes-file <p>` · `--labels a,b` · `--priority <rung>` · `--estimate S\|M\|L\|XL` · `--due DATE` · `--start DATE` · `--assignee <name>` · `--parent <card-id>` · `--blocked-by <ids>` · `--links <urls>` |
| `jannoo cards edit <id> [<id>…]` | `--title` · `--notes`/`--notes-file` · `--labels a,b` (replace) or `--add-label`/`--remove-label` · `--priority` · `--estimate S\|M\|L\|XL\|none` · `--due`/`--clear-due` · `--start`/`--clear-start` · `--add-link`/`--remove-link` · `--checklist-add "<title>"` · `--checklist-check`/`--checklist-uncheck`/`--checklist-remove <item-id\|exact title>` · `--field <name\|id>=<value>` (**one per call**; an empty value, `--field Risk=`, clears it) · `--assignee` |
| `jannoo cards move <id> --column <id\|title>` | moves the card to the bottom of that column |
| `jannoo cards complete <id>` | the real completion: moves to Done, stamps `completedAt`, and spawns the next occurrence of a recurring card. **Use this, not `cards move --column Done`** — a move does not spawn the recurrence. `--if-match` takes the card's own etag, spawn or no spawn |
| `jannoo cards relate <a> <b> [--remove]` | the symmetric `relatedTo` link, written on both cards |
| `jannoo cards relate <card> --parent <id\|none>` | the parent/epic link. One level only: an epic cannot have a parent, a child cannot have children |
| `jannoo cards relate <card> --blocked-by <ids\|none>` | replaces the blocker list |
| `jannoo cards attach <id> <file>` | copies the file into the board's `attachments/` and writes the reference in the same write |
| `jannoo cards detach <id> <attachment>` | takes one attachment off the card — the path the card stores (`attachments/spec.pdf`), or its bare file name when only one attachment has it (two with that name: `usage.attachment_ambiguous`, pass the path). Only the reference goes: **the file stays on disk**, as it does in the app. Dry-run by default |
| `jannoo cards archive <id>` | archives the card through the app's own archive transaction: `archive/<id>.md` is written and made durable, then the card leaves the board and `cards/<id>.md` goes. Reversible with `cards unarchive`. Only a card in **Done** is archived by hand (`archive.not_done` otherwise — `cards complete` it first), as in the app. A card the board's Linear sync holds a link for is refused (`archive.provider_owned`), and every card while that sync file will not read (`archive.provider_indeterminate`) — the app refuses the same ones. A card with children is refused (`card.has_children`); one already archived changes nothing. Dry-run by default. The answer's `data.archiveEtag` is the `--if-match` for a later unarchive |
| `jannoo cards unarchive <id>` | brings an archived card back through the app's own unarchive transaction: `cards/<id>.md` is written and made durable, then `archive/<id>.md` is removed (kept, and `data.archiveCopy` says so, if it changed meanwhile). `--column <id\|title>` picks the column; without it the card returns to the column it was archived from, or the first column if that one is gone. The policy's `columns` scope is checked on that destination. Its completion stamp restarts from where it lands; no other card file is written, as in the app. **On a GitHub-synced board**, a linked card brought back into an open column while its issue is closed gets that issue **reopened** on the next sync — say so before you `--apply`, which needs `--allow-remote` only when the dry run lists an effect in `data.remote` (rule 6). `--if-match` takes the archived card's etag from `cards list --archived`. A card already on the board changes nothing. Dry-run by default |
| `jannoo cards duplicate <id>` | copies the card the way the app's Duplicate does: a new card just below it, titled `<title> (copy)`, with the same fields — except comments and reactions. **Attachments are shared, not copied**: the copy names the same files. A card related to others is refused (`card.has_related` — the copy's relation would be one-sided); remove the relation first. Applies at once. `--if-match` takes the original card's etag; `data.cardEtags` has the copy's. On a synced board the copy becomes a new issue (on Linear, only in a column the binding maps), so it needs `--allow-remote` when `--dry-run` lists that effect in `data.remote` (rule 6) |
| `jannoo cards comment <id> --body <text>` | adds a comment to the card's thread as the app's composer does: authored `you`, trimmed, never empty. `--body-file <path>` instead of `--body` for longer text (`-` is not stdin — write a file). The comment is local until a sync posts it: **on a synced board the next sync posts it** to the linked issue (or to the issue that sync creates), so it needs `--allow-remote` when `--dry-run` lists that effect in `data.remote` (rule 6). Applies at once; `--if-match` takes the card's etag |
| `jannoo cards delete <id>` | deletes the card the way the app's Delete does. **Permanent**: its file is removed, not moved to the Trash, and a CLI write has no undo — offer `cards archive` first, which keeps it. Cards that named it as a blocker or relation keep a stale id (`data.staleReferences`). A card with children is refused (`card.has_children`) until their parent is cleared. Dry-run by default: show the user which card it names before you `--apply` |
| `jannoo boards create <path> --title <t>` | new board. `--columns a,b,c` (the LAST one is Done) · `--labels a,b` · `--agents-file` to drop the format guide in. Refuses a folder that already holds a board or card files, in dry-run as well as apply; takes no `--if-match` |
| `jannoo boards add-label <name>` | adds a tag to the palette. `--color #RRGGBB` (omitted: the first starter-palette colour) · `--emoji <one emoji>` (drawn in place of the colour dot; anything that is not one emoji is refused). A name the board already has is refused — the CLI resolves a label by name |
| `jannoo boards edit-label <name\|id>` | edits a tag the board already has: `--name` · `--color #RRGGBB` · `--emoji <one emoji>` or `--clear-emoji`. A name another tag has is refused. No card file changes (cards carry the tag by id), but every card carrying it reads the new name — and on a GitHub board with label sync the dry run counts the linked issues whose labels the next sync resends (`data.github.linkedIssues`, one `issue.labels` effect each in `data.remote`; a card carrying it that no issue is linked to yet changes the issue the next sync creates, `issue.create.changed`, not counted) — the apply needs `--allow-remote` only when `data.remote` lists an effect (rule 6) |
| `jannoo boards delete-label <name\|id>` | deletes a tag from the palette **and from every card carrying it** — the dry run names each of those cards (`data.cards`), whose files the apply rewrites. On a GitHub board with label sync a linked card carrying it loses that label on the next sync (`issue.labels` in `data.remote`), so the apply needs `--allow-remote` only when `data.remote` lists an effect (rule 6) |
| `jannoo boards add-column <title>` | adds a column **immediately before Done** and renumbers the ranks. Done keeps its identity and **no card is touched**; there is no `--before`/`--after`, because the only other placement makes the new column Done and clears `completed:` across the old one. Refused on a board with no columns, or one whose two highest ranks tie |
| `jannoo boards add-field <name> --type text\|date` | adds a custom-field definition. Only the types this build's inspector can edit are offered — ask `--help` rather than assuming the list |
| `jannoo boards edit-field <name\|id>` | renames and / or retypes a custom field — at least one of `--name <new>` · `--type <type>`. A name another field has is refused. `--type` takes only the types `add-field` offers (a type this build hides is refused with `add-field`'s reason), but an existing field of a hidden type may be retyped to one it offers — that is how to rescue one. **A retype CLEARS the field's value on every card**, no value converted: the dry run lists each one (`data.clearedValues`) — show the user before you `--apply`. `--type` the field already has is a no-op that says `already <type>` and clears no value (the app's own retype would clear them — this is the CLI's one difference, and no app control retypes a field). No `--options`. Custom-field values are not synced, so no `--allow-remote` |
| `jannoo boards delete-field <name\|id>` | deletes a custom field **and every card's value for it**, and clears the board's at-a-glance field if it was this one (`data.atAGlanceCleared`). The dry run lists every card holding a value, with the value (`data.cards`). A field of a type this build cannot create may still be deleted. Not synced, so no `--allow-remote` |
| `jannoo boards rename-column <id\|title> <new title>` | renames a column in place — its id, rank and colour stay, and no card file is written. The column is the first word (quote a title with spaces); every word after it is the new title. A title another column already has is refused. **On a GitHub-synced board** the dry run names the `status:<old>` → `status:<new>` change and counts the linked issues in that column the next sync relabels (`data.github.linkedIssues`, one `issue.state` effect each in `data.remote`) — possibly none: renaming the binding's default column sends nothing for a card that has already synced there. A card in the column that no issue is linked to yet is created with the column's label, so the rename changes that create (`issue.create.changed`, not counted in `linkedIssues`) — the default column's too. Relay a relabel before you `--apply`, which needs `--allow-remote` only when `data.remote` lists an effect (rule 6) |
| `jannoo boards move-column <id\|title> --before\|--after <id\|title>` | moves a column next to another and renumbers the ranks. **Done never changes**: moving the Done column, or placing anything after it, is refused (`column.done_fixed`) — the rightmost column IS completion. `--before Done` is fine. No card is touched |
| `jannoo boards edit-column <id\|title> --color <#RRGGBB\|none>` | sets or clears a column's accent colour. Id, rank and title stay, no card is touched, nothing is synced. `rename-column` renames |
| `jannoo boards delete-column <id\|title>` | deletes a column and **re-homes its cards to the leftmost remaining column** — the dry run names every re-homed card and where it goes (`data.rehomed`, `data.destination`); relay that list. **Deleting the Done column makes the next column Done**: its cards become done, stamped now (`data.becameDone`), and the cards re-homed out of Done stop being done (`data.noLongerDone`; on a two-column board they land in the new Done and stay done) — the dry run warns. A sync binding that names the column as its Done or default column is warned about (`data.syncBindings`) and left as it is — tell the user to re-pick it in the app. The board's last column is refused (`column.last`). On a synced board a re-homed linked card can change its issue's state — deleting Done reopens a linked closed issue — so the apply needs `--allow-remote` only when `data.remote` lists an effect (rule 6) |
| `jannoo boards set` | the board-level settings, at least one of: `--title <t>` (board.json only — the folder keeps its name) · `--prefix <CODE>` (2–5 letters; `EXP` makes a linked card read `EXP-345`; `--prefix ""` clears it) · `--sort <mode>` (ask `--help` for the modes; `issueNumber` is refused with `sort.needs_github_binding` on a board with no readable GitHub binding, as the app's Sort menu hides it there) · `--archive-after-days <N\|off>` · `--at-a-glance <field\|none>` (the one custom field every card surfaces; a field the board does not define is `field.not_found`, exit `3`). A value the app would have silently changed is refused. **A threshold that would archive cards names them** in the dry run (`data.archival` — the cards eligible now, which move the next time Jannoo opens or saves the board; more may qualify by then): show the user that list, because `--apply` is their consent |
| `jannoo boards copy-schema --from <board>` | copies another board's setup onto this one: its columns (titles, order, which is Done), tags, custom fields, prefix, sort and archive-after days — not its title. **Done matches Done by role**: this board's Done keeps its id and stays last, and takes the source's Done title; every other column, tag and field matches by name, case-insensitively, and **takes the source's exact spelling** — a rename that keeps its id (on a GitHub board the dry run warns about a label the rename moves, `data.github`). **Nothing is deleted and no id changes** — this board's own columns, tags and fields are kept (a kept column stays after the one it followed), and a source item with no match is added with a new id. A matched tag takes the source's colour and glyph (no glyph on the source clears it), a matched single-select field gains the missing options, and a source with no prefix clears this board's. A part this board cannot take is **skipped with a `warning:` row** (`data.skipped`) and the rest applies — a field of another type, a field type this build cannot create, sort `issueNumber` with no GitHub binding, a column title a kept column already has: relay those. A name either board carries twice is refused (`schema.duplicate_name`). The FWRITER → MANX job is `rename-column "To Do" Backlog`, then `copy-schema --from FWRITER`, then `set --prefix MANX` |
| `jannoo boards validate --repair` | fixes the **four mechanical findings** and nothing else: an asymmetric `relatedTo` (the back-link is written), and a `relatedTo` / `blockedBy` / `parent` naming its own card (the self-reference is removed). A **stale id is never removed** — the format tolerates one and an archived card's id returns on unarchive. Refused beside a duplicate card id or a manifest breach. **It does not promise writability**: read `data.writable` and `data.findings`, not the exit code |

**Driving the app** — these move the RUNNING Jannoo app, not the files. They write no board data,
need no agent policy, and are never licence-gated:

| Command | What it does |
|---|---|
| `jannoo open` | shows the board in Jannoo, launching it if needed; a window already showing it comes forward. `--card <id>` selects a card · `--view board\|list` · `--search <text>` (`""` clears it) · `--panel sync\|conflicts`. It brings Jannoo to the front — run it when the user asks to *see* something |
| `jannoo reload` | makes the app re-read the board from disk in every window showing it, without coming forward. **Exit `5` (`drive.busy`)**: the app holds edits it has not saved, so nothing was re-read — wait a moment and retry. Exit `0` with `data.ackDetail` `board.not_open` means no window showed the board |
| `jannoo sync --now` | starts the board's GitHub / Linear sync now. A round trip with the remote, so it needs `--allow-remote` (or `"remote": true`) like any remote effect — rule 6; an unbound board is `sync.not_bound`. The ack marks the sync's **start**, not its result — read the result from `jannoo boards show --json` (`data.status`) — and the app syncs only boards a window shows (`board.not_open` — run `jannoo open` first); a binding paused in Jannoo acks `sync.disabled` (exit `2` `drive.refused`) — nothing was started, and only the user can resume it in the app; `sync.not_ready` (exit `2` `drive.refused`) means Jannoo had only just started — nothing was started; wait a moment and run it again. Bare `jannoo sync` is exit `64` |

**The board must sit directly inside the app's data location**, or the verb refuses before sending
anything (`drive.board_not_in_app`, exit `2`) — the app would never see it. Each waits for the app's
answer (5 s; 30 s when it had to launch Jannoo); `--no-wait` returns once the URL is handed over.
**`drive.no_ack` (exit `2`) means the URL was sent and may still land — do not resend it**;
`jannoo log` shows a late ack. `drive.refused` (exit `2`) carries the app's reason in `data.ackDetail` —
`card.not_found` there means the board opened without that card. `drive.app_unavailable` (exit
`3`) means Jannoo could not be reached at all.

`cards show` and `cards search` span **every visible board** unless `--board` narrows them, and an
ambiguous card id is refused rather than guessed. A visible folder that will not open is not left
out silently: it is named in `data.skippedBoards` (`path` + `reason`) and in a `skipped:` line, so
you can tell an answer that covered everything from one that did not.

### Global flags

`--json` (the `{ok, data, error, receipt}` envelope) · `--format concise|detailed` ·
`--board ID|PATH` · `--limit N` · `--after CURSOR` · `--help` · `--version`.

**These are the only flags that may stand BEFORE the verb.** `jannoo --json cards list` and
`jannoo cards list --json` are the same command, and so are the other five. Every other flag —
`--apply`, `--column`, `--if-match`, all of them — belongs after the verb: written in front of it
you get `64` with a hint saying where it goes, never a command that quietly means something else.
Flags and no verb (`jannoo --json`) is `64` as well; only a bare `jannoo` prints the help.

**`--if-match ETAG` is not one of them.** It is taken by the verbs that have an etag to match, and
it is a usage error (`64`) on the ones that do not — `boards list` (a union of boards has no etag),
`install`, and `boards create` (nothing exists yet to match). On `cards search` it needs the
search to resolve to ONE board — `--board`, or a machine with a single visible board; across
several it is a usage error, because there is no single etag. It is never accepted and ignored, so a `0` back
from a command carrying it means the guard really held. Each verb's `--help` lists it where it
applies.

### `cards list` filters — let the CLI do the work

`--column ID|TITLE` · `--label ID|NAME` · `--priority RUNG[,RUNG…]` · `--due-before DATE` ·
`--due-after DATE` · `--blocked [false]` · `--done [false]` · `--archived`

Two things worth knowing, because they remove most of the work:

- **`--column` and `--label` take a TITLE or NAME, not just an id** — `--column Backlog` is enough,
  no id lookup first.
- **Every card already carries computed `done` and `blocked` booleans**, so `--done false` means
  "not in the Done column" and `--blocked true` resolves the blocker chain for you. Do not re-derive
  either from `blockedBy` or from column ranks.

`--priority` rungs are `none`, `low`, `medium`, `high`, `urgent`; comma-separate to combine.
`--estimate` is a write flag only (`cards create` / `cards edit`) — `cards list` has no estimate
filter. It takes `S`, `M`, `L`, `XL` (any case); `--estimate none` clears it.
Dates take a plain `2026-06-25`.

### Two traps in the output

- **`--limit` defaults to 50 — and to 15 under `--format detailed`.** A longer list is silently
  truncated, so check `data.hasMore` and either page with `--after <data.nextCursor>` or pass
  `--limit 0` for no limit. A count taken from a truncated page is simply wrong.
- **`priority` is frequently `none` even on a board that has priorities**, because many boards keep
  them in a `singleSelect` custom field instead (the bundled sample board does — a "Priority" field
  with `P0 - Critical` … `P3 - Low`). `--priority high,urgent` returns nothing on such a board. Run
  `jannoo boards show --json` first; if you see a priority-shaped custom field, read priorities from
  `customFields` under `--format detailed` (each entry carries the definition's `name`) rather than
  from the `priority` key. The same goes for `estimate`: a board can keep sizes in a custom field
  *named* "Estimate" instead. The two never touch — `--estimate` writes the built-in key,
  `--field Estimate=M` writes the custom field.

### Exit codes

`0` did what it said · `2` refused — a validation finding, a broken invariant, or the agent
policy · `3` the board, card or file does not exist · `4` `--if-match` did not match, or the board
changed on disk under a write · `5` busy — Jannoo holds unsaved edits, so `jannoo reload` did not
re-read the board, or a write Jannoo would have applied touches what it has not saved yet: nothing
was written, retry shortly · `64` the command line does not parse. A write the running app takes
has a few answers of its own — **When Jannoo has the board open**, under Writing, lists them.

Branch on the code, not on prose. **`64` almost always means you guessed a flag** — `--overdue`
exits 64, measured. Run `--help` and use what it prints.

### Environment

`JANNOO_BOARDS_DIR` — **colon-separated** extra directories to scan for board folders.
`JANNOO_AGENT` — the agent host the CLI was installed for; it lands in write receipts.

## Writing — six rules

**1. Writing is opt-in per board.** With no `.jannoo/agent-policy.json` in the board folder the
board is **read-only** and every write verb refuses with `policy.read_only` and exit `2`. That
refusal is correct behaviour, not a bug: relay its `hint` and let the user decide. Reads never need
a policy. The file can also narrow what is allowed — `boards`, `columns`, `fields` — and a write
outside a scope refuses with `policy.<thing>_out_of_scope` naming it. Under a `columns` or `fields`
scope a schema write that would widen it is refused too (`policy.schema_out_of_scope`): a column
added, renamed, moved, recoloured or deleted, and any board-wide change — `boards set`, `boards edit-label`,
`boards delete-label`, `boards edit-field`, `boards delete-field`, `boards copy-schema`.

**2. Quote the etag on anything the user might act on, and hand it back.** Every card and board
carries an `etag` (the SHA-256 of its bytes on disk). Pass it as `--if-match <etag>` on the write
and the write is **refused with exit `4`** if the file changed under you — someone typing in the app,
Obsidian saving, a `git pull`. **Read, then write with `--if-match`.** Without it you are overwriting
whatever is there now, which may not be what you read. Single-card verbs want that card's etag; a
multi-card or schema write wants the board's. Every applied write hands the new one back in
`data.cardEtags`, so a chain of writes needs no re-read.

**3. Some writes are dry-run by default.** `cards edit` with more than one id, `cards edit --field`,
`cards delete`, `cards detach`, `cards archive`, `cards unarchive`, `boards create` into a folder that already exists, **every `boards` schema write** (`add-*`, `edit-*`, `delete-*`, `rename-column`, `move-column`, `set`, `copy-schema`), and
`boards validate --repair` all print the **unified diff they would write** and change nothing. Add
`--apply` to perform them. `--dry-run` forces that behaviour on any write verb — use it when you are
unsure, and show the user the diff. One caveat on the schema writes: the new id is minted when the
write is applied, so the id in a dry run's diff is not the id the apply lands. Nothing else about
the diff differs.

**4. Read the receipt.** Every applied write returns a `receipt` in the `--json` envelope and appends
it to `.jannoo/agent-log.jsonl`: `requestID`, `op`, the card ids, the file paths, `etagBefore` /
`etagAfter`, the timestamp, and which agent did it. `jannoo log` replays the trail. Set
`JANNOO_AGENT` so your writes are attributable.

**5. Never hand-edit a card file as a workaround.** If a verb refuses, the refusal is the answer —
report it. The app may have the board open, and a hand edit bypasses every check above.

**6. A write that reaches GitHub or Linear needs the user's consent.** On a board bound to GitHub or
Linear the app's next sync sends some writes upstream: a new card becomes an issue, a move into Done
closes one, a retitle, a notes edit, a tag change or a new comment is pushed, a renamed column or tag
can relabel linked issues, an unarchive into an open column reopens one. **An edit to a card no issue
is linked to yet counts too**: the next sync creates that card's issue, and the edit goes up in the
create — its title, notes, tags or column (a move, a completion, and on GitHub a rename of its column),
and on Linear also its priority, due date and parent. `data.remote` lists that as
`issue.create.changed`; a card new to the board is `issue.create`. Such a write is **refused
with `policy.remote_needs_opt_in`** (exit `2`) unless the call passes `--allow-remote` or the policy
sets `"remote": true`. A dry run never refuses for it: it prints a `remote:` line per effect
(`data.remote.effects`, each with its `target` — `owner/repo#N`, a Linear id, or the issue a create
will make) and says when performing it needs the flag. **Show the user that list and get their
go-ahead before you add `--allow-remote`** — an issue closed or a comment posted is seen by other
people, and the CLI cannot take it back. **The flag follows `data.remote.effects`, not the verb**: a
write with no remote effect needs no flag, bound board or not — a checklist tick, a start date, a
relation (on Linear, not the parent of a card no issue is linked to yet), a delete or an archive (a
pending create just goes away), a rename of Done, of a column no card is in, or of the binding's
default column when every card in it is linked and has already synced there, or a create into a Linear
column the binding does not map. **The one exception is an unreadable sync file**: while
a bound board's GitHub or Linear sync file will not read, nobody can say what a write would send, so
`data.remote` lists one `unknown` effect for it and **every** write on that board needs the flag until
the file is repaired. `jannoo sync --now` is gated the same way; its effect is the round trip
itself — one `sync.round_trip` per bound provider in `data.remote.effects`.

**Re-running a write is safe.** A write whose result the board already holds exits `0` with
`data.changed: false`, touches no file and produces no receipt — so "did that land?" is answered by
running it again, not by guessing. And `data.cardIDs` / the receipt's `paths` tell you what the
write ACTUALLY touched, which can be more than you named: completing a recurring card with relations
rewrites the counterparts that link to its next occurrence.

**When Jannoo has the board open.** If Jannoo is running and lists the board directly inside its
data location, an applied write is handed to the app instead of saved by the CLI — a dry run never
is, it stays a diff against disk. A window showing the board applies it as **one undo step** (Edit ▸
Undo reverts it; `data.undoLabel` names the menu item) and saves it; the answer carries
`data.via: "app"` and the receipt's `detail` is `via.app`. With no window on the board the CLI saves it itself, as
usual. `cards archive` and `cards unarchive` are always saved by the CLI. The answers this adds:

- **Exit `5` `drive.busy`** (`data.ackDetail` `board.busy`) — Jannoo holds unsaved edits to a file
  this write touches. Nothing was written; retry in a moment, once its autosave has landed. Nearly
  every write also rewrites `board.json`, which every in-app edit dirties too — so **while the user
  is typing in the app, a write to ANY card on that board can answer busy**, and **a clean
  `--dry-run` can still be followed by exit `5` on `--apply`**. Retry shortly; never fall back to
  editing the files.
- **Exit `4` `write.stale`** — the board changed on disk after the CLI read it, so Jannoo refused the
  write. **Exit `4` `write.moved_during_route`** — the same, found by the CLI while it waited for
  Jannoo's answer. Nothing was written either way: re-read and retry.
- **Exit `2` `write.not_taken`** — Jannoo did not take the write in time and it was withdrawn: nothing
  was written, safe to retry. **Exit `2` `write.unknown`** — Jannoo took it and did not confirm it, or
  applied it and its save failed (`data.ackDetail` `write.save_failed`): it **may have landed**. Read
  the board, or `jannoo log`, before you retry — a second `cards create`, `comment` or `complete`
  would do it twice. **Exit `2` `board.gone`** — the board's folder went away while the write was in
  flight. **Exit `2` `write.request_failed`** — the CLI could not write its request into
  `.jannoo/requests/`; nothing was sent or written, safe to retry once the folder is writable.
- **Exit `2` `drive.refused`**, the reason in `data.ackDetail` — `write.expired` (Jannoo reached the
  write after its deadline), `board.unavailable` (the board is being deleted, or will not read),
  `write.no_request`, `url.dropped`. Nothing was written; safe to retry.
- **Exit `3` `drive.app_unavailable`** — Jannoo could not be reached at all. Nothing was written.
- A write Jannoo saved whose folder is not quite as asked — a deleted card's file that would not
  delete — exits `0` with `data.saveReport` and a `note:`, the same words the CLI's own save prints
  (the receipt's `detail` is `write.save_partial`; its `failedPaths` names the file).

**When Jannoo cannot take it** — not running, or the board not one it lists in its data location —
the CLI saves, and an app that has the board open reconciles your write exactly as it reconciles
Obsidian. The one collision — the user is typing in the very card you are editing — keeps their text
and preserves your bytes as `cards/<uuid>.conflict-<stamp>.md`; a SCHEMA write (a label, column or
custom field, in `board.json`) against unsaved app changes is copied aside as
`board.conflict-<stamp>.json` and is not applied. Nothing is lost; nothing is silent. A write the CLI
saved has no undo step, so say what you changed.

## Reading a card

`--format concise` gives `id`, `title`, `slug`, `etag`, `column`, `priority`, `estimate`, `due`,
`labels`, `blocked`, `done` — enough for every recipe here. `--format detailed` adds `notes`,
`checklist`, `links`, `relations`, `customFields`, `attachments`, `comments`, `reactions`, `start`,
`rank`, `createdAt`, `updatedAt`, `completedAt`, `path` and `extraFrontmatter`.

- **`done`** is computed: the card sits in the Done column (the rightmost, highest-`rank` one).
  Position is status; there is no `done` flag on disk.
- **`blocked`** is computed from `blockedBy` — true while any blocker is still outside Done.
- **`etag`** is the SHA-256 of the card's file bytes. Quote it when you report something a user may
  act on; a write will take it back as `--if-match`.
- **`updatedAt`** is what "since yesterday" means — but it is a file timestamp, not an audit log.

## Recipe 1 — daily plan

What to work on now: overdue, due soon, blocked, high-priority.

1. `jannoo boards show --json` — the columns, and whether a priority-shaped custom field exists.
2. Ask the CLI each question directly, always with `--done false` so finished work never appears
   (substitute today's and next week's dates):
   - overdue — `jannoo cards list --json --done false --due-before <today>`
   - due soon — `jannoo cards list --json --done false --due-after <today> --due-before <today+7d>`
   - blocked — `jannoo cards list --json --done false --blocked`
   - high — `jannoo cards list --json --done false --priority high,urgent`
3. Check `data.hasMore` on each. If true, re-run with `--limit 0` before you quote a count.
4. If the high-priority call comes back empty and step 1 showed a priority custom field, re-read it
   from `customFields` with `--format detailed` instead.
5. A card can appear in more than one bucket. Report it once, in the most urgent one, and say why.
6. Report in that order, shortest useful form: title, column, due date, and the reason it is listed.
   Lead with a one-line count ("3 overdue, 2 due this week, 1 blocked").
7. **Sanity-check dates before alarming anyone.** If every dated card on the board is overdue by
   roughly the same amount, treat it as a demo or fixture board and say so rather than reporting a
   crisis — the bundled sample board is exactly this shape, every due date within days of every
   other. The tell is the clustering, not how old it is: a few months past reads the same as years.
8. Close with what you would move. **Propose, then act only when asked** — and when they say yes,
   move each card with the etag you just read:
   `jannoo cards move <id> --column "In Progress" --if-match <that card's etag>`
   (`jannoo cards complete <id> --if-match <etag>` for finished work — `complete`, not a move to
   Done, or a recurring card never comes back). Exit `4` means the card changed while you were
   reporting: re-read it and ask again rather than retrying blind.

## Recipe 2 — triage new cards

Cards that arrived without the fields that make them actionable.

1. `jannoo boards show --json` — the columns in order, plus the labels and custom fields this board
   actually uses. The first column is the intake lane; anything arriving from outside lands there.
2. `jannoo cards list --json --column <that column's title> --done false --format detailed` — the
   detailed form, because what is MISSING is the question and the concise form hides most fields.
3. For each, name what is missing **against what this board actually uses**: no `due`, no priority
   (the key *or* the custom field), no `labels`, an empty `notes`, no parent in `relations` where
   every sibling has one. Do not demand a field the board never populates.
4. Report one line per card: title, id, what is missing, and **your proposed value** — the label you
   would add, the priority you would set, the column you would move it to. Proposals, not actions.
5. Flag duplicates: `jannoo cards search "<distinctive phrase from the title>"` catches the same
   ticket filed twice, and reports which board each hit is on. Name both ids; let the user decide.
6. **Only when the user accepts a proposal**, apply it one card at a time with the etag from step 2:
   `jannoo cards edit <id> --priority high --add-label bug --due 2026-10-01 --if-match <etag>`
   A batch — several ids in one `cards edit` — is **dry-run by default**: it prints the diff and
   writes nothing until you add `--apply`. Show the user that diff before you pass `--apply`.

## Recipe 3 — stand-up summary since a timestamp

1. Establish the window. Default to the last working day; take the user's word over a guess.
2. `jannoo cards list --json --format detailed --limit 0` and keep cards whose `updatedAt` is at or
   after the timestamp. There is no `--updated-since` flag — do not invent one.
3. `jannoo log --json` adds *what* changed rather than only *that* it changed. An absent ledger is a
   success with no entries, which is normal on a board nothing has written to.
4. Group by where they ended up: **landed** (`done: true`), **in flight** (a middle column),
   **queued** (back in the intake column).
5. Report three short lists, then one line of risk: anything overdue or blocked that did not move.
6. **`updatedAt` is a file timestamp.** A card the user only read still shows as updated if anything
   rewrote its file. Say "changed", not "worked on", unless `jannoo log` says otherwise.

## Recipe 4 — end-of-session audit

1. `jannoo boards validate --json` first, and **read the exit code**: `0` clean, `2` at least one
   error-severity finding, `3` board not found. Report its findings verbatim — they name the file
   and the rule — and keep its severities rather than flattening them.
2. **Stale blockers.** `jannoo cards list --json --blocked --format detailed`: for each, check
   `relations` and report blockers that are already done (the block is over, the field is stale) and
   ids that resolve to nothing (the blocker was deleted).
3. **Unlinked items.** `--format detailed --limit 0`: cards with no `labels`, no parent on a board
   where parents are the norm, no `links` where cards normally carry one, and epics with no children.
4. **Drifting work.** `--done false` plus a middle column, with `updatedAt` older than a week.
5. Report as a short checklist of findings, each with the card id so the user can act. End with the
   validate verdict. **Nothing in steps 1–4 changes a file.**
6. If the user asks you to clear a stale blocker, do it one card at a time and with the etag:
   `jannoo cards relate <card> --blocked-by none --if-match <etag>` (or a narrower list). A write is
   refused outright while `boards validate` reports an error-severity finding, so fix the board
   first — the refusal names the findings.
7. **If step 1 found error-severity findings, offer `jannoo boards validate --repair`.** Show the
   dry run's diff first, then `--repair --apply` once the user agrees. It fixes only the four
   mechanical ones (an asymmetric `relatedTo`, and a `relatedTo` / `blockedBy` / `parent` naming its
   own card) and never deletes a stale id.
8. **Then run `jannoo boards validate` again, and use ITS exit code as the board's state.** A repair
   exits `0` for "the repair did what it said", not for "this board is clean" — those are different
   claims, and a board can carry an error `--repair` has no answer for straight through a perfectly
   successful one. In the repair's own output that state is `data.writable` and `data.findings`; do
   not read it off the exit code.

## Reporting style

Lead with the answer. A count, then the list, then what you would do. Give card **titles** for
humans and keep **ids** to hand for anything the user might act on. Long lists get the top few and
a count of the rest — not fifty lines of dump. If a verb exits non-zero, say which verb, which exit
code, and what it means from the table above, rather than retrying blind.
