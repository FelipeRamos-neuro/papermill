# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick setup

- Python: 3.10+
- Recommended dev env:

```bash
python3 -m venv dev
source dev/bin/activate
pip install -e '.[dev]'
```

## Common commands

### Run tests

- Fast local default (single Python):

```bash
tox -e py312
```

- Full CI-like matrix (py310/311/312/313 + dist/docs/binder/manifest):

```bash
tox
```

- Run pytest directly in active environment:

```bash
pytest
```

- Run one test file:

```bash
pytest papermill/tests/test_execute.py
```

- Run one test case:

```bash
pytest papermill/tests/test_execute.py::test_execute_notebook
```

### Lint / formatting

Primary lint/format flow in this repo is pre-commit:

```bash
pre-commit run -a
```

Useful direct commands when iterating:

```bash
ruff check .
ruff format .
black --skip-string-normalization .
codespell
```

### Build and package checks

```bash
python -m build
tox -e dist
check-manifest
```

### Docs and binder checks

```bash
tox -e docs
tox -e binder
```

## Architecture: big picture

Papermill core flow is:

1. Parse CLI/options (`papermill/cli.py`)
2. Execute orchestration (`papermill/execute.py`)
3. Parameter injection (`papermill/parameterize.py`)
4. Notebook execution via engine (`papermill/engines.py`, `papermill/clientwrap.py`)
5. Read/write through IO handlers (`papermill/iorw.py`)

### Key subsystems

- **Execution API (`execute.py`)**
  - `execute_notebook(...)` is the main orchestrator used by CLI and Python API.
  - Adds built-in `pm.*` parameters, parameterizes input/output paths, injects notebook parameters, prepares metadata, executes through engine, then writes output.
  - Error handling wraps notebook failures into `PapermillExecutionError` and injects error marker cells for easier debugging in output notebooks.

- **Engines (`engines.py`)**
  - `papermill_engines` is a registry of execution backends.
  - `NBConvertEngine` is the default implementation.
  - `NotebookExecutionManager` centralizes per-cell lifecycle metadata (`pending/running/completed/failed`), autosave behavior, duration timing, and progress bar state.
  - If you add engine kwargs, make sure they are merged/cleaned consistently (`merge_kwargs`, `remove_args` usage in engine code paths).

- **Parameter translation (`translators.py`)**
  - `papermill_translators` maps kernel/language names to translator classes.
  - Translators convert Python values to language-specific source code injected into `injected-parameters` cell.
  - `PythonTranslator` optionally formats generated parameter code with Black when installed.

- **IO abstraction (`iorw.py`)**
  - `papermill_io` is a LIFO registry of path handlers.
  - Built-in schemes include local files plus cloud/remote handlers (`s3://`, `gs://`, `adl://`, `abs://`, `http(s)://`, etc., when optional deps exist).
  - New handlers must provide `read`, `write`, `pretty_path` (and optionally `listdir`).
  - Registry supports plugin extension through entry points (`papermill.io`).

- **CLI layer (`cli.py`)**
  - Keep CLI thin: options parse/coerce here, behavior implemented in `execute.py`.
  - Any new CLI option should usually be mirrored in `execute_notebook(...)` API and covered by both `test_cli.py` and `test_execute.py`.

## Extension points and expected change pattern

When adding functionality, follow these repo conventions:

- **New CLI option:** update `cli.py`, propagate to `execute.py`, add/adjust tests in `test_cli.py` and `test_execute.py`.
- **New engine:** implement engine class in `engines.py` style, register it, add tests in `test_engines.py`.
- **New translator/language support:** implement translator methods, register in `translators.py`, add tests in `test_translators.py`.
- **New IO backend:** implement handler and register in `iorw.py`, add tests in `test_iorw.py` and backend-specific tests.

## Test data layout

Tests use:

- `papermill/tests/*.py` for unit/integration behavior
- `papermill/tests/notebooks/*.ipynb` for execution fixtures
- `papermill/tests/parameters/*` for parameter input fixtures

Prefer extending existing test modules that match subsystem ownership.

## CI and toolchain notes

- CI runs `tox` across Python 3.10–3.13 plus `manifest`, `docs`, and `binder` environments (`.github/workflows/ci.yml`).
- Formatting/lint hooks are defined in `.pre-commit-config.yaml` (ruff, black, mdformat, codespell, pyproject-fmt, validate-pyproject).
- Packaging metadata, tool config, and tox env definitions live in `pyproject.toml`.
