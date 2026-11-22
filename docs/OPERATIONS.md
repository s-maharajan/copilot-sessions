# Operations — `cs`

Installing cs, running it, keeping it healthy, and cutting a release. `cs` is a
local terminal application, so "operations" here means the lifecycle of an
installation on a machine, not a fleet.

## 1. What cs is

A single Python package over one SQLite file. There is no server, no client,
no daemon, no background process and no telemetry. Running `cs` opens the
store read-only, renders, and exits.

That has one consequence worth stating plainly: **there is nothing to monitor
and nothing to roll back at runtime.** A user upgrades by installing a new
version, and the old one is still on disk.

## 2. Requirements

| | |
|---|---|
| Python | 3.10 or newer |
| Runtime dependencies | None — the standard library only |
| The store | GitHub Copilot CLI's session store, opened read-only |
| The terminal | A terminal that can do colour and, for the interactive UI, a mouse protocol |

`cs doctor` checks all of it, including the store, its schema, the event logs,
the config and cache directories, the terminal, the mouse protocol and the
glyph mode, and prints pass, warn or fail with a fix for each.

## 3. Installing

### From source — the local installer

```bash
git clone https://github.com/s-maharajan/copilot-sessions.git
cd copilot-sessions
./install.sh                    # symlinks bin/cs into ~/.local/bin
./install.sh /usr/local/bin     # or a directory of your choice
```

`install.sh` checks that `python3` exists and is 3.10 or newer, creates the
target directory, symlinks `bin/cs` into it, and makes the launcher
executable. It then checks whether the target directory is on your `PATH` and
warns if it is not.

It also warns about the other half of the problem: **being on `PATH` is not the
same as winning on `PATH`.** If an earlier entry already answers `cs` — a
Homebrew install, say — the symlink looks inert. `which -a cs` lists them all.

### From source — pip

```bash
pip install .                # installs the `cs` command
```

### Without installing

```bash
python -m cs recent          # run straight from a checkout
```

`bin/cs` is a thin launcher that puts the checkout on `sys.path` and calls
`cs.cli:run`, which is also the `[project.scripts]` entry point in
`pyproject.toml`.

## 4. What cs writes, and where

| What | Where | Written? |
|---|---|---|
| The session store | `$COPILOT_HOME/session-store.db`, default `~/.copilot/session-store.db` | **Never** — opened `mode=ro` |
| The per-session event logs | `$COPILOT_HOME/session-state/<id>/events.jsonl` | Never — read-only, counted |
| The settings file | `$CS_CONFIG_HOME/cs/settings.json`, default `~/.config/cs/settings.json` | Yes — theme, pins, notes, daily budget |
| The `#N` shortcuts | `~/.config/cs/.cs-last-index` | Yes |
| The event digest | `$XDG_CACHE_HOME/cs/events-digest.json`, default `~/.cache/cs` | Yes — counts and timestamps only |
| The asset counts | `$XDG_CACHE_HOME/cs/asset-counts.json` | Yes |
| The ignore list | `$COPILOT_HOME/.cs-ignore` | **Never** — read only |

Nothing cs writes is placed inside `COPILOT_HOME`. That directory belongs to
Copilot, and cs has no business writing next to the store it only reads.

To move the settings or the cache, set `CS_CONFIG_HOME` or `XDG_CACHE_HOME`.
Both are honoured for the run they are set on, and nothing is migrated between
locations.

### The environment

| Variable | Effect |
|---|---|
| `COPILOT_HOME` | Where the store and `session-state/` live. Defaults to `~/.copilot`. |
| `CS_CONFIG_HOME` | Where cs keeps its own settings. Defaults to `$XDG_CONFIG_HOME/cs`, else `~/.config/cs`. |
| `XDG_CONFIG_HOME` | The parent of the default config directory. |
| `XDG_CACHE_HOME` | Where the event digest and the asset counts are cached. Defaults to `~/.cache/cs`. |
| `PAGER` | The pager for a long report. Defaults to `less`, with `-R -F -X`. |
| `COLUMNS` | The terminal width. The layout caps at 96 columns. |
| `CS_THEME` | The theme for this run, overriding the saved one. |
| `CS_GLYPHS` | `ascii` swaps every emoji for a plain marker. |
| `CS_MOTION` | `off` keeps every screen still — no animation, no count-up. |
| `CS_REDACT` | `0` shows raw text instead of masking. The default is `1`. |
| `CS_WHY` | `1` shows the lessons every time. `--why` does the same for one command. |
| `TERM` | `dumb` suppresses colour at import time. |

Several of these are read at import time — the theme, the glyph mode, the
motion setting and the colour decision — so they must be set **before** the
process starts.

## 5. Verifying an installation

```bash
cs --version        # cs 2.4.0
cs doctor           # the full check, with a fix for anything that fails
cs recent           # your sessions from the last 7 days
```

`cs doctor` is the health check. It reports on the Python version, the store
and its schema, the event logs, the config and cache directories, the
terminal, the mouse protocol and the glyph mode.

If the home screen's status line says `schema changed · cs doctor`, Copilot
changed the store's schema under cs. That is reported, not fatal: a missing
column is a missing answer, not a broken query.

## 6. Performance and scale

- The store is opened `mode=ro`, so cs never contends with Copilot for a
  write lock.
- Computed month and year summaries are memoised in-process.
- The event-log digest is cached under `XDG_CACHE_HOME` and refreshed on a
  schedule; **only counts, names and timestamps are cached** — never tool
  output, prompts or replies.
- The digest cache is keyed by each log's path, modification time and size, and
  carries a `DIGEST_VERSION`, so a changed or truncated log invalidates its
  entry rather than serving stale counts. It is written atomically, and a
  failure to write is harmless.
- There is no index cs can add, because cs never writes.

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `cs` hangs with no output | It is waiting on a pager | Run with `PAGER=cat` |
| `cs: command not found` | The install directory is not on `PATH` | Add it, or call the absolute path |
| `cs --version` shows the wrong version | Another `cs` is earlier on `PATH` | `which -a cs`; put yours first, or remove the other |
| "schema changed · cs doctor" | Copilot changed the store's schema | Not fatal; `cs doctor` lists the difference |
| Every emoji is a blank box | The terminal or font cannot draw them | `CS_GLYPHS=ascii` |
| The output is uncoloured | `TERM=dumb`, or the output is piped | Expected — colour is suppressed at import time |
| A column is misaligned | The terminal is very narrow | `COLUMNS` sets the width; the layout caps at 96 columns |
| The settings are not remembered | The config directory is not writable | `CS_CONFIG_HOME` to a writable directory; `cs doctor` reports this |
| The store cannot be read | Copilot is not installed, or `COPILOT_HOME` is wrong | `cs doctor` names the path it looked at |

## 8. Releasing

The version lives in **two** places, and both must be moved together:

- `pyproject.toml` → `version`
- `cs/__init__.py` → `__version__`

The release procedure:

1. Run the gate: `ruff check cs tests` and `python -m unittest discover -s tests`.
2. Move the `Unreleased` section of `CHANGELOG.md` under the new version
   heading, with the date, following [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
3. Bump the version in `pyproject.toml` and `cs/__init__.py`.
4. Commit, tag, and publish the GitHub release.

There is no publish workflow and no PyPI release automation in this
repository: a release is a commit, a tag and a GitHub release, and users
install from the tag with `pip install .` or the local installer.

The `Unreleased` section is the working area. Nothing ships from it until a
release moves it.

### What a release must not contain

This repository is public, so a release is permanent. Before tagging:

- [ ] `ruff check cs tests` is clean.
- [ ] `python -m unittest discover -s tests` passes.
- [ ] The version is bumped in **both** `pyproject.toml` and `cs/__init__.py`.
- [ ] `CHANGELOG.md` has the release under a version heading with a date.
- [ ] No personal path, machine name, token, employer detail or real session
      content has entered a tracked file, a commit message or a screenshot.
- [ ] The screenshots in the README were regenerated if a view changed.

## 9. Security

The threat model is in [../SECURITY.md](../SECURITY.md). The operational
summary:

- **The store is read-only.** There is no write path, and a change that adds
  one will be declined.
- **No network code.** Nothing is uploaded, copied or phoned home. `cs
  rollup` is built to be shared — counts and rates only, with repositories
  replaced by a salted hash whose salt stays in the settings file.
- **Credentials are masked at the render edge**, so nothing secret-shaped
  reaches the screen, the scrollback, a screen-share, a file or a pipe.
- **Stored text is treated as untrusted input.** Terminal control sequences
  are stripped before anything is drawn.
- **CI scans the history** for committed credentials with gitleaks, and runs
  CodeQL, Semgrep, OSV-Scanner and Snyk on every push and pull request.

To report a vulnerability, follow [../SECURITY.md](../SECURITY.md). It does not
go in the issue tracker.

## 10. Related documents

| Document | Covers |
|---|---|
| [../README.md](../README.md) | What cs is, and what it does. |
| [../ARCHITECTURE.md](../ARCHITECTURE.md) | How the modules split and why. |
| [../CONTRIBUTING.md](../CONTRIBUTING.md) | The contract: the two commands, the module seams, what needs a test. |
| [../docs/DEVELOPMENT.md](DEVELOPMENT.md) | The development loop. |
| [../SECURITY.md](../SECURITY.md) | The threat model and private reporting. |
| [../CHANGELOG.md](../CHANGELOG.md) | What changed, and when. |
