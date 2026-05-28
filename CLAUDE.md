# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Watson (`td-watson` on PyPI) — a CLI for tracking time spent on projects. Python 3.6+, distributed as a single `watson` console script (entry point: `watson.__main__:cli`).

## Common commands

- Install dev environment: `make install-dev` (runs `pip install -r requirements-dev.txt` and `python setup.py develop`).
- Full virtualenv with a sandboxed data dir: `make env` — creates `.venv/` and pins `WATSON_DIR=$(CURDIR)/data` in the activate script so dev runs of `watson` don't touch the user's real frames/state.
- Run all tests: `py.test -vs tests/` (or `tox` for the full matrix `flake8,py35–py39`).
- Run a single test file / case: `py.test -vs tests/test_cli.py` or `py.test -vs tests/test_cli.py::test_start`.
- Lint: `tox -e flake8` (runs `flake8 --show-source watson/ tests/ scripts/`).
- Regenerate CLI docs: `make docs` (runs `scripts/gen-cli-docs.py` then `mkdocs build`).
- Regenerate shell completion scripts: `make completion-scripts` (writes `watson.completion` and `watson.zsh-completion`).

## Architecture

The codebase splits cleanly into a CLI layer and a domain layer; touching one rarely requires touching the other.

- `watson/cli.py` — All `click` commands. Uses `DYMGroup` ("did you mean") for command suggestions and a custom `MutuallyExclusiveOption` for flag conflicts. Every command instantiates a `Watson` via `create_watson()` (from `utils.py`) and calls `watson.save()` on exit. This is the only place that should depend on `click`/user-facing formatting.
- `watson/watson.py` — The `Watson` class is the single source of truth for state. It owns three on-disk JSON files inside `click.get_app_dir('watson')` (overridable via `WATSON_DIR` env var or `config_dir=` kwarg):
  - `state` — the in-progress frame (project, tags, start time), or `{}` when stopped.
  - `frames` — the list of completed frames.
  - `last_sync` — timestamp of the last successful sync to a remote Crick server.
  - `config` — INI-format config (read via `watson/config.py`).
  All writes go through `utils.safe_save` (atomic write-then-rename with a `.bak` backup) — never write these files directly.
- `watson/frames.py` — `Frame` is a `namedtuple` subclass; `Frames` is a collection with filter/group helpers and `changed` tracking. Frame timestamps are stored as UTC unix ints on disk but exposed as local-time `arrow.Arrow` objects in memory. When constructing frames or comparing times, keep that in/out boundary in mind.
- `watson/config.py` — `ConfigParser` wraps `RawConfigParser` to add `get(..., default=None)` semantics used everywhere in `cli.py`.
- `watson/autocompletion.py` — Click completion callbacks consumed by the generated shell completion scripts.
- `watson/fullmoon.py` — Lookup table of full-moon dates used by the `log`/`report` rendering. Not load-bearing for behavior.

Sync (`watson sync`, `push`, `pull`) talks to a Crick API; auth token and URL live in the `[backend]` section of the config.

## Conventions worth knowing

- Times: always go through `arrow`. The codebase consistently converts to UTC for persistence and to local for display — preserve that pattern. `Watson._parse_date` / `_format_date` are the conversion choke points.
- Errors that should reach the user as a clean message must be raised as `WatsonError` (or `ConfigurationError`); `cli.py` catches these and exits with a styled message. Don't let bare exceptions bubble out of the CLI layer.
- `py35` is still in the tox envlist but `setup.py` requires `python_requires='>=3.6'` — target 3.6+ syntax.
- Tests use `pytest`, `pytest-mock`, and `pytest-datafiles`. `tests/conftest.py` provides `watson`, `runner` (Click's `CliRunner`), and `watson_df` (loads fixture files from `tests/resources/`) — prefer these fixtures over hand-rolling a `Watson` instance.
