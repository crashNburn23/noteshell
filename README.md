# obsnote

A little CLI that turns your terminal into a notebook. Instead of alt-tabbing
into Obsidian to write down what you just did, you tell `obsnote` and it
writes clean markdown into your vault — notes, captured command output, an
optional local-LLM summary, or a whole marked stretch of shell history
bundled into one tidy block.

It started as "save my shell history to Obsidian" and grew into something
built for taking notes on a technical course: pages per topic, tags, and a
`mark` → do stuff → `since` workflow for turning a lab exercise into a
readable note without manually copy-pasting a terminal.

## ⚠️ Read this before you use it

**Personal project, not a security product.** Treat it like your shell
history: useful, convenient, not private.

- Redaction only filters commands *passively* captured by the shell hook.
  `obsnote run`, and any command's *output* (an env dump, a curl response
  with a token in it), goes into your vault **unfiltered**.
- Notes are plain markdown on disk. If your vault syncs anywhere (Obsidian
  Sync, iCloud, Dropbox, a git repo), whatever obsnote writes goes with it.
- Passive capture keeps a rolling plaintext shadow of your shell history
  (plus the last command's full output) under `~/.local/state/obsnote/`.
  `obsnote forget` clears it.
- Capture is off by default and only runs inside a `mark` → `since` bracket,
  or after `obsnote resume` — see [Shell integration](#shell-integration).
- It's one small module (`obsnote/cli.py`). If you're unsure what it's doing
  with your data, it's a quick read.

Don't paste real credentials into a terminal that's feeding this tool.

## Install

```bash
git clone <this repo>
cd obsnote
pip install -e .
```

## Quick start

```bash
obsnote config --vault ~/path/to/your/ObsidianVault
obsnote shell-install bash   # wires up the hook, capture starts OFF (or: zsh)
source ~/.bashrc             # or just open a new terminal
obsnote doctor               # sanity-check that all of the above worked
obsnote mark                 # turns capture on and starts a session
```

Run `obsnote doctor` any time something seems off — it checks your vault,
config, and shell hook, and says exactly what's missing.

## Mental model

Everything lands on the **active page**, a markdown file in your vault
(`obsnote page new/use/list`, or `--page` for a one-off).

- `obsnote note "..."` — a timestamped freeform note (`--tag` to tag it).
- `obsnote mark [name]` → do stuff → `obsnote since [name]` — writes
  everything typed in between as one bash block, flagging failures
  (`# exited 1`) and directory changes. `--preview` shows it first without
  writing; `--ok-only` drops failed attempts. Marks auto-number
  (`1`, `2`, ...) and `since`/`unmark` resolve to the only marker around if
  you don't name one. A successful `since` consumes its marker.
- `obsnote annotate "..."` / `obsnote summary "..."` — drop a note into the
  pending timeline, or a summary that renders above it, before you `since`.
- `obsnote run -- <cmd>` — capture one command and its output (or an LLM
  summary, `--synth`, if Ollama's running locally), instead of a whole
  marked stretch.

Every entry starts with `From obsnote: <timestamp>` plus any tags, then the
note, command, output, or history.

## Commands

| Command | What it does |
|---|---|
| `obsnote config` | show/set vault, default page, Ollama settings, redact patterns |
| `obsnote note [text]` | append a freeform note (reads stdin if omitted) |
| `obsnote run -- <cmd>` | run a command, capture output, append it |
| `obsnote mark [name]` / `unmark [name]` | set / delete a checkpoint in your shell history |
| `obsnote since [name] [--preview\|--synth\|--ok-only]` | write (or preview) everything since a marker |
| `obsnote annotate [text]` / `summary [text]` | insert a note / summary into the pending timeline |
| `obsnote undo [--page]` | remove the last obsnote entry from a page |
| `obsnote page new/use/list <name>` | create, switch, or list pages |
| `obsnote tail [--page] [-n]` | read-only peek at the last entries on a page |
| `obsnote show` | read-only status: capture state, active page, last command, markers |
| `obsnote doctor` | preflight check: vault, config, shell hook |
| `obsnote pause` / `resume` | pause/resume passive capture, with a confirmed status readout |
| `obsnote forget [--last N]` | clear captured commands/output from local state (vault untouched) |
| `obsnote shell-install` / `shell-uninstall <bash\|zsh>` | wire up / remove the passive capture hook |

Run `obsnote <command> --help` for the full flag list on any of these.

## Shell integration

`shell-install` adds a hook (`PROMPT_COMMAND` for bash, `precmd` for zsh)
that records every command's text, exit status, and working directory,
skipping obsnote's own commands.

**Capture is opt-in.** A fresh install leaves it off. `obsnote mark` turns it
on; `since`/`unmark` turns it back off once no markers are left pending. Want
it always-on instead? `obsnote resume` does that until you `pause` it.

Guardrails on top of that:

- A **leading space** keeps a command out of both shell history and obsnote's
  capture (`ignorespace`, enabled automatically).
- A **redact pattern list** silently drops commands that look like a live
  credential (`TOKEN=...`, `mysql -pSECRET`, `Authorization: Bearer ...`)
  before they touch disk — extend it with `obsnote config --redact-pattern
  '<regex>'`. A redacted command leaves a placeholder in the timeline
  (`# obsnote skipped a redacted command`) with no text stored.

`obsnote run` bypasses redaction on purpose — if you explicitly ran it
through obsnote, that's an intentional capture, not passive history.

Doing something sensitive mid-mark and don't trust the pattern list? Run
`obsnote pause`. If something already slipped in, `obsnote forget --last 5`
(or plain `forget`) scrubs it from local state.

While capture is active, the shell hook prefixes your prompt with `[● rec]`
in red. If the hook fails to write state, it leaves a warning file that
`obsnote show` will surface.

`shell-uninstall bash`/`zsh` removes the hook entirely — no more passive
capture. Explicit commands (`note`, `run`, `mark`, ...) keep working.

One more guardrail: the active page is remembered per vault. If a
project-local `.obsnote.json` points a directory at a different vault, a
page activated elsewhere is ignored there instead of writing to the wrong
vault.

## Config

Settings resolve in order: environment variables (`OBSNOTE_VAULT`,
`OBSNOTE_NOTE`, ...) → a project-local `.obsnote.json` (found by walking up
from cwd, like `.git`) → the global config at `~/.config/obsnote/config.json`
→ built-in defaults. The project-local file is handy for pointing a course
repo at a specific default page without touching your global setup.

## Status

Early, actively-changing personal tool. There's a small standard-library
test suite for the core CLI/state behavior, but no compatibility matrix or
security guarantees. If something breaks, `obsnote doctor` and `obsnote show`
are the first stop.

## License

MIT — see [LICENSE](LICENSE).
