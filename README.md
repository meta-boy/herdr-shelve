# shelve

An LRU shelf for [herdr](https://herdr.dev) agent tabs.

Every idle coding agent is a live process holding 200 MB to 1 GB, with its MCP
servers. `shelve` keeps only the `keep` most recently used agent tabs awake. When
a tab falls off the end, its agents are quit and their sessions recorded; the
tab, its panes and their shells stay where they are, marked `💤`. Focus the tab
and each agent comes back in the same pane with `--resume <session>`, history
intact.

It works like browser tab discarding: a fixed number of tabs stay hot, and
nothing is lost.

```
shelve: keep 8 agent tabs awake

   1 w2:t4    fix-login-bug                            awake    focused
   2 w1:t1    api-refactor                             awake
   ...
   8 w1:t3    write-docs                               awake
  💤 w1:t2    💤 flaky-test-hunt                        shelved  claude
  💤 w2:t1    💤 review-pr-42                           shelved  claude, codex
  📌 w2:t2    📌 deploy-watch                           pinned
```

Other herdr sleep plugins (`herdr-agent-hibernate`, `herdr-sleeper`,
`herdr-sleep`, `herdr-park`) sleep a pane after an idle timeout or on demand.
`shelve` caps how many agent tabs are awake at once and evicts the least
recently focused, which bounds memory no matter how many tabs you open.

## Install

```sh
herdr plugin install meta-boy/herdr-shelve
herdr integration install claude      # and/or codex, opencode: herdr learns session ids from these
```

Requires herdr ≥ 0.9.1, Linux or macOS, and Python 3.9+ on the server's `PATH`
(stdlib only; 3.11+ to read `config.toml`).

## How it works

- **Use** is a `tab.focused` event. The focused tab moves to the front of the
  recent list; if it was shelved, its agents are resumed first.
- **The limit counts tabs, not panes**, and only tabs with an agent in them. A
  tab with three agents is one slot, and all three shelve and wake together.
- **Rebalance** runs on focus, whenever an agent settles to `idle`/`done`, and on
  server start. If more than `keep` agent tabs are awake, the oldest safe ones
  are shelved until it fits. The focused tab is never shelved.
- **Shelving** a pane writes its record first (kind, session, cwd, launch flags),
  then stops the agent, waits until herdr drops it and only the shell is left in
  the foreground, and renames the pane and tab with `💤`.
- **Waking** uses `herdr agent start --kind <kind> -- <flags> <resume args>`, so
  nothing is typed into the shell; herdr quotes the command for the pane's
  shell. The record is deleted only once herdr reports the pane running that
  same session again.

### A tab is only shelved when all of these hold

- every agent pane is `idle` or `done` (never `working`, `blocked`, `unknown`);
- every agent has a session id **and** a transcript on disk (an agent you
  haven't messaged yet has nothing to resume);
- every other pane is a bare shell at its prompt (a dev server, `psql`, an
  editor or a watcher keeps the tab awake);
- the agent kind has a profile (below);
- the tab is not pinned (`📌`) or marked as a watcher (`👁`).

Anything herdr can't tell counts as "no".

## Agents

| Kind | Stopped with | Resumed with | Tested |
| --- | --- | --- | --- |
| `claude` | SIGTERM (clean exit, unsent draft discarded) | `--resume <id>` + allowed launch flags | yes, herdr 0.9.5, Claude Code |
| `opencode` | SIGTERM | `--session <id>` | not yet |
| `codex` | `ctrl+c` ×3 (a signal leaves codex's keyboard protocol on) | `resume <id>` | not yet |

Launch flags such as `--model`, `--permission-mode`, `--add-dir` or
`--dangerously-skip-permissions` are carried over on resume. Anything not on the
profile's `replay` list (a positional prompt, `--print`, `--fork-session`, …) is
dropped and logged.

Any other agent herdr recognises and reports a session for can be added in
`config.toml`:

```toml
[kinds.qwen]
exit = "signal"                       # or "keys" with exit_keys = [["ctrl+c"], ["ctrl+c"]]
process_names = ["qwen"]
resume = ["--resume", "{session}"]
transcript = "~/.qwen/sessions/*{session}*"   # optional; omit to trust herdr's session id
replay = { "--model" = 1 }
```

## Configure

`config.toml` lives in the plugin config dir (`herdr plugin config-dir meta-boy.shelve`):

```toml
keep = 8              # agent tabs kept awake
dry_run = false       # log what would be shelved, change nothing
pin_marker = "📌"     # tabs whose label starts with this never shelve
skip_markers = ["👁"]
shelf_marker = "💤"
exit_timeout = 40     # seconds to wait for an agent to exit
wake_timeout = 60
```

Start with `dry_run = true` and check the log to see what it would do to your
tabs.

## Use

Bind the actions in `~/.config/herdr/config.toml`:

```toml
[[keys.command]]
key = "prefix+s"
type = "plugin_action"
command = "meta-boy.shelve.pin"
description = "pin/unpin this tab"

[[keys.command]]
key = "prefix+S"
type = "plugin_action"
command = "meta-boy.shelve.shelve"
description = "shelve this tab now"
```

| Action | |
| --- | --- |
| `meta-boy.shelve.pin` | toggle `📌` on the tab label |
| `meta-boy.shelve.shelve` | shelve the current tab now (same safety checks); switch away and back to wake it |
| `meta-boy.shelve.unshelve-all` | resume every shelved tab (the limit applies again on the next rebalance, so raise `keep` or disable the plugin first if you want them all back) |
| `meta-boy.shelve.rebalance` | rebalance now |

The `status` popup pane (`herdr plugin pane open --plugin meta-boy.shelve --entrypoint status`) lists
every agent tab in recency order. `./herdr-shelve status` and
`./herdr-shelve log` do the same from a shell in the plugin's directory.

State lives in herdr's plugin state dir, per herdr session: `state.json` (recent
list and shelf records), `shelve.log`.

## Limits

- Shelving drops in-memory state that is not in the transcript: running
  subagents, background tasks, an unsent draft. An idle agent has none of the
  first two.
- Renaming tabs makes an auto-numbered tab label (`5`) a manual one.
- Waking a long session reloads its transcript and can take a few seconds.
- If herdr restarts, pane ids can be reused. A shelf record is only trusted when
  its tab still carries the `💤` label; otherwise it is kept as an orphan and
  `status` prints the command to resume it by hand. Records are never silently
  deleted.

Credit to Anshuman's [Putting Claude tabs to sleep](https://anshuman.sh/articles/putting-claude-tabs-to-sleep/)
for the idea, and to `djbclark/herdr-sleeper`'s
[LESSONS.md](https://github.com/djbclark/herdr-sleeper/blob/main/LESSONS.md) and
`dalogax/herdr-agent-hibernate` for the hard-won safety rules.

## License

MIT
