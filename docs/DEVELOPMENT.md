# Development guide — `cs`

The day-to-day working guide: setting up, running the two commands CI runs,
adding a command, and regenerating the screenshots.

**[CONTRIBUTING.md](../CONTRIBUTING.md) is the contract.** This document is the
companion to it: the mechanics of the loop, and the things that are easy to get
wrong. Where the two disagree, CONTRIBUTING.md wins.

## 1. Setup

```bash
git clone https://github.com/s-maharajan/copilot-sessions.git
cd copilot-sessions
python -m unittest discover -s tests     # the tests use a synthetic store
pipx install 'ruff==0.15.22' && ruff check cs tests
```

**Python 3.10 or newer.** No runtime dependencies and no development
dependencies — `ruff` is installed outside the repository, in a venv or with
`pipx`, and is pinned to the version CI uses.

There is nothing to build, nothing to install and nothing to configure. `cs`
runs from the checkout:

```bash
python -m cs            # the app
python -m cs --help
```

## 2. The two commands

CI runs exactly these two, in this order:

```bash
ruff check cs tests
python -m unittest discover -s tests
```

Two traps live here:

- **The tests are `unittest`, not `pytest`.** `pytest` may appear to work and
  will collect the wrong things. There is no `conftest.py` and no
  `tests/__init__.py` — deliberately, because `unittest discover -s tests` puts
  `tests/` on `sys.path` and the fixtures are imported as
  `from support import _build_store, Screen`.
- **`ruff` is not a dependency.** It is not in `pyproject.toml` and must not be
  added there. CI installs it with `pipx install 'ruff==0.15.22'`.

The workflow runs the test step with `if: always()`, so a lint failure cannot
hide a test failure — but lint before you push anyway.

### Running part of the suite

```bash
python -m unittest discover -s tests -p "test_render.py"      # one file
python -m unittest discover -s tests -v                       # verbose
python -m unittest discover -s tests -p "test_render.py" -v
```

`unittest discover` takes a `-p` pattern, not a dotted module path — `tests/`
is not a package, so `python -m unittest tests.test_render` does not import.

## 3. How the tests work

`tests/support.py` is not a test file. It builds the synthetic store and the
fake curses screen that every other file imports.

- **The store is synthetic.** `setUp` builds a throwaway store in a temporary
  directory and points `COPILOT_HOME` at it, so the tests never read your real
  Copilot data and behave identically on any machine.
- **The screen is fake.** A fake `Screen` serves a queue of keys and records
  every frame as `{(y, x): text}`, so an assertion reads the screen the way a
  user would:

  ```python
  screen = Screen([curses.KEY_RIGHT, ord("q")])
  _listing_tui(screen, rows, "Sessions")
  assert "sorted by turns ↓" in screen.frames[-1][(0, 0)]
  ```

  The fake honours `nodelay`: an empty queue returns `-1` instead of blocking,
  which is exactly the condition that distinguishes a bare Esc from the start
  of a mouse report. Mouse behaviour is tested by feeding the actual byte
  sequences a terminal sends.

## 4. Where a test goes

`tests/` is split by subject and each file says at the top what it is
responsible for. `CONTRIBUTING.md` has the full table of which file covers
what — follow it rather than inventing a new file.

Three of the existing files are structural rather than behavioural, and they are
the ones that keep working when you add something:

- **`test_surface.py` reads the command list out of `cli._dispatch`.** A new
  command is smoke-tested the day it is added, against a full store, an empty
  one and one from an older Copilot. If it needs an argument, add it to that
  file's `ARGUMENTS` table, or the smoke test runs it bare and fails on the
  usage line.
- **`test_practice_rules.py` fails until a new `coach` rule has a fixture**
  that fires it and one that does not.
- **The credential corpus in `test_governance.py` must name every rule in
  `redact._RULES`.** A pattern that stops matching is silent by construction:
  the audit simply reports less, which reads as good news.

## 5. Adding a command

1. **Put the query in `db.py`.** It returns plain tuples, never formatted
   strings. If the answer is a judgement rather than a fact, it belongs in
   `signals.py` and must carry its evidence.
2. **Put the inference in `signals.py`** (about one session) or `practice.py`
   (about a window of sessions). Neither may draw anything.
3. **Draw it in `ui.py`.** `ui.py` knows about colour, boxes, tables and text
   fitting, and nothing about sessions.
4. **Expose it in `export.py`** if it should also be machine-readable. Masking
   applies to both edges — a file is more exposed than a screen, not less.
5. **Wire it in `cli/`.** Add the argument to `test_surface.py`'s `ARGUMENTS`
   table.
6. **Never print stored text directly.** Session content is untrusted input; it
   goes through `redact.py` and the sanitising path in `ui.py` first.
7. **Add a test** for anything with a branch, a loop, a parser or a credential
   path — and a width test if it draws a table. A new table has to hold its
   shape at 40, 60, 80, 100 and 140 columns.

The seams are the design. The full table of which module owns what, and what
it must not know about, is in [CONTRIBUTING.md](../CONTRIBUTING.md); the
reasoning is in [ARCHITECTURE.md](../ARCHITECTURE.md).

## 6. Credentials and masking

A new credential rule in `redact.py` needs **two** cases in the corpus in
`tests/test_governance.py`: one string that must mask, and one near-miss that
must **not**. Use a synthetic value and mark the line `# gitleaks:allow` — the
corpus lives there because `cs audit` is the view that scans for secrets, so
the rule and its caller are tested together.

Masking is applied to the whole text before it is broken into lines, so a rule
that spans lines — a PEM block, say — is only recognisable as one when its
BEGIN and END are still in the same string.

## 7. Regenerating the screenshots

The images in the README are real runs against a synthetic store:

```bash
python3 docs/img/make_screens.py
```

The script drives the real UI in a pty and sets `PAGER=cat`, so a long report
is not handed to `less` — `less` would find the pty on `/dev/tty` and sit
waiting for a keypress that never comes.

Check the result in a real terminal. Glyphs fall back to blank in some fonts,
and a block character used as a cursor can render as nothing at all.

## 8. Debugging tips

| Symptom | Where to look |
| --- | --- |
| `cs` hangs and the terminal is stuck | It is waiting on a pager. Run with `PAGER=cat`. |
| A view shows nothing | `cs doctor` — it checks the store, the schema, the event logs, the config and cache directories, the terminal and the glyph mode. |
| "schema changed · cs doctor" on the home screen | Copilot changed the store's schema. `cs doctor` lists the difference; it is not fatal. |
| A figure looks wrong | The store is live — it changes while you query it. Two numbers a minute apart are not comparable. |
| A column is misaligned | The table has to hold its shape at 40, 60, 80, 100 and 140 columns. The shared width test will find it. |
| An emoji is a blank box | The terminal or font cannot draw it. `CS_GLYPHS=ascii` is the fallback. |
| A `sqlite3.ProgrammingError` about placeholders | The `?1`-style named placeholder with a sequence argument. This becomes an error on Python 3.14; keep one placeholder style per query. |
| A test cannot import `support` | The tests are run with `unittest discover -s tests`, which puts `tests/` on `sys.path`. Do not add an `__init__.py`. |

## 9. Previewing a view without a pager

```bash
PAGER=cat COLUMNS=92 python -m cs <command>
```

`PAGER`, not `CS_PAGER` — nothing in the package has ever read `CS_PAGER`.
`COLUMNS` does work, through `shutil.get_terminal_size()`.

## 10. Before you open a pull request

- [ ] `ruff check cs tests` is clean.
- [ ] `python -m unittest discover -s tests` passes.
- [ ] The change has a test, if it has a branch, a loop, a parser or a
      credential path.
- [ ] A new command is in `test_surface.py`'s `ARGUMENTS` table.
- [ ] A new `coach` rule has a firing and a non-firing fixture.
- [ ] A new `redact` rule has a masking case and a near-miss.
- [ ] A new table holds its shape at 40, 60, 80, 100 and 140 columns.
- [ ] The screenshots are regenerated if a view changed.
- [ ] Nothing personal, machine-specific or real-session-derived has entered a
      tracked file, a commit message or a screenshot.
