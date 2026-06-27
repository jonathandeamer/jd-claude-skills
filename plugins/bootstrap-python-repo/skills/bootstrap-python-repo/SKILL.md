---
name: bootstrap-python-repo
description: Use when creating a new Python project from scratch, setting up a Python repo skeleton, or when the user says "bootstrap", "scaffold", or "init" a Python project. Also trigger when the user says "new python app", "start a python project", "set up a python package", "create a python CLI", or is in an empty directory and wants to start coding in Python. Covers uv, pytest, ruff, pyright, git hooks, CLAUDE.md, and src layout.
---

# Bootstrap Python Repo

Set up a new Python project with uv, ruff, pytest, pyright, conventional commits, CLAUDE.md, and src layout.

Assumes the user has already created and `cd`'d into the project directory.

## When to Use

- User asks to create/bootstrap/scaffold/init a new Python project
- User wants a repo skeleton with modern Python tooling
- User is in an empty directory and wants to start a Python project

## Setup Steps

### 1. Ask the user

Before generating files, ask:

- **Project name** (default: current directory name). If the user doesn't have a name yet, don't block on it — proceed with a placeholder like `myproject`, mention you've done so, and point them at "Renaming the project later" below. A name is structurally required (it becomes the importable package and the distribution name), but it's cheap to change afterward with the steps provided.
- **One-line project description** (used in CLAUDE.md and pyproject.toml)
- **Minimum Python version** (default: `3.12`) — give a bare version like `3.12`. It becomes `requires-python = ">=<python-version>"`, the `pyright` `pythonVersion`, and the pinned `.python-version`.
- **Whether they need a `.env.example`** and if so, what variables

Don't add a license. Leave it out deliberately and tell the user it's a choice they should make later — the license governs how others may use the project, so it shouldn't be defaulted on their behalf. They can add a `LICENSE` file whenever they've decided — e.g. via GitHub's "Add license" UI, or by fetching the text for an SPDX id with `gh api /licenses/<spdx-id> --jq .body` (e.g. `mit`, `apache-2.0`).

### 2. Initialize git and uv

The user is already inside the project directory. Initialize in place with a `src` layout:

```bash
git init .
uv init --lib --no-readme --name <project-name> --python <python-version>
```

`uv init --lib` scaffolds the `src` layout for you: it creates `src/<package-name>/__init__.py`, `src/<package-name>/py.typed`, and `.python-version`. `uv` derives `<package-name>` from `<project-name>` by lowercasing and replacing dots and hyphens with underscores (e.g. `my-cool-app` → `my_cool_app`, `acme.tools` → `acme_tools`), because dots and hyphens are illegal in Python import names. Use this underscored `<package-name>` wherever the package directory is referenced below, and keep the original `<project-name>` only as the distribution name in `pyproject.toml`.

Passing `--python <python-version>` makes `uv init` use the selected floor for both the generated `requires-python` value and `.python-version`, even when a newer interpreter is available locally.

You'll overwrite the `pyproject.toml` and `.gitignore` that `uv` generated with the templates in step 3.

### 3. Create files

Generate these files, substituting `<project-name>` (distribution name, e.g. `my-cool-app`), `<package-name>` (importable package, e.g. `my_cool_app`), `<project-description>`, and `<python-version>` (bare minimum version, e.g. `3.12`) throughout:

#### `pyproject.toml`

```toml
[project]
name = "<project-name>"
version = "0.0.0"
description = "<project-description>"
requires-python = ">=<python-version>"
dependencies = []

[build-system]
requires = ["uv_build>=0.10,<0.12"]
build-backend = "uv_build"

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pyright>=1.1",
    "ruff>=0.9",
]

[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]

[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "integration: marks tests as integration tests (deselect with '-m \"not integration\"')",
]

[tool.pyright]
pythonVersion = "<python-version>"
typeCheckingMode = "standard"
venvPath = "."
venv = ".venv"
```

#### `.gitignore`

```
# Python
__pycache__/
*.pyc
*.pyo
*.pyd

# Virtual environments
.venv/
venv/

# Distribution
dist/
build/
*.egg-info/

# Tools
.ruff_cache/
.pytest_cache/
.pyright/

# Secrets
.env
```

#### `src/<package-name>/__init__.py` and `src/<package-name>/py.typed`

Already created by `uv init --lib` in step 2 — leave them in place. Note that `uv` seeds `__init__.py` with a sample `hello()` function; replace it with your real code (or empty the file) rather than shipping the placeholder. `py.typed` marks the package as typed so downstream consumers (and `pyright`) honor your annotations.

#### `tests/__init__.py`

Empty file. (`uv init --lib` does not create a `tests/` directory, so create it yourself.)

#### `.githooks/commit-msg`

```bash
#!/usr/bin/env bash
set -euo pipefail

commit_msg_file="$1"
commit_msg=$(cat "$commit_msg_file")

# Skip merge commits
if echo "$commit_msg" | grep -qE "^Merge "; then
    exit 0
fi

pattern="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build)(\(.+\))?: .+"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo ""
    echo "ERROR: Invalid commit message format."
    echo ""
    echo "Expected: <type>(<optional scope>): <description>"
    echo "Example:  feat(core): add initial implementation"
    echo ""
    echo "Valid types: feat, fix, docs, style, refactor, test, chore, perf, ci, build"
    echo ""
    echo "Your message: $commit_msg"
    echo ""
    echo "Use --no-verify to bypass (e.g. for WIP commits)."
    echo ""
    exit 1
fi
```

#### `.githooks/pre-commit`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Check for staged Python files (newline-delimited, just for the guard)
staged_py=$(git diff --cached --name-only --diff-filter=ACM | grep '\.py$' || true)

# No-op cleanly when no Python files are staged
if [ -z "$staged_py" ]; then
    exit 0
fi

# Note: ruff checks working-tree files, not the staged snapshot.
# For a single-developer workflow this is acceptable.

# Use null-delimited pipeline (handles filenames with spaces).
# The brace group with || true prevents set -o pipefail from aborting on grep's
# non-zero exit if (unexpectedly) no .py files match — xargs -0 handles empty input cleanly.
{ git diff --cached --name-only --diff-filter=ACM -z | grep -z '\.py$' || true; } | xargs -0 uv run ruff check
{ git diff --cached --name-only --diff-filter=ACM -z | grep -z '\.py$' || true; } | xargs -0 uv run ruff format --check

echo "OK"
```

#### `CLAUDE.md`

```markdown
# <Project-Name> — Claude Code Reference

## Project Overview

<project-description>

---

## Architecture

- `src/<package-name>/` — production source code (src layout)
- `tests/` — mirrors `src/<package-name>/` structure (e.g. `src/<package-name>/foo.py` → `tests/test_foo.py`)
- `.githooks/` — git hooks, installed with `git config core.hooksPath .githooks`

---

## Coding Conventions

- **Type hints required** on all function signatures (parameters and return types). No bare `def`.
- **No `Any`** without an inline comment explaining why it cannot be avoided.
- **`ruff` is enforced** via the pre-commit hook. Rules: E, F, I (isort), UP (pyupgrade). Line length: 88.
- **No `print()`** in production code. Use the `logging` module.
- **No secrets in code.** API keys and credentials go in `.env` (gitignored).

---

## Commits

- **Conventional commits** are enforced by the `.githooks/commit-msg` hook: `<type>(<optional scope>): <description>` (types: feat, fix, docs, style, refactor, test, chore, perf, ci, build). Bypass only for WIP with `--no-verify`.
- **No AI attribution.** Do not add `Co-Authored-By` trailers or any other AI/assistant attribution, and commit under the repository's own git identity — never an AI persona. Keep messages to a plain subject line (plus a body when it adds context).

---

## Testing

- **Runner:** `uv run pytest`
- **Location:** `tests/` — mirrors the `src/<package-name>/` structure.
- **Prefer unit tests.** Mock external API calls; don't make real HTTP requests in tests.
- **Integration tests** (real API calls, real network) must be marked:
  ```python
  @pytest.mark.integration
  ```
  Run only explicitly: `uv run pytest -m integration`
- Write the test first, then the implementation.

---

## Common Commands

| Task | Command |
|------|---------|
| Install deps | `uv sync` |
| Run tests | `uv run pytest` |
| Lint | `uv run ruff check .` |
| Format | `uv run ruff format .` |
| Type check | `uv run pyright` |
| Install hooks | `git config core.hooksPath .githooks` |

---

## Things to Avoid

- Broad `except Exception` blocks — catch specific exceptions.
- Mocking at levels that diverge from real API behaviour — test logic in isolation from I/O rather than deeply mocking internals.

---

## Self-Update Note

If you (Claude) add a module, establish a pattern, or make an architectural decision that isn't reflected here, suggest a CLAUDE.md update so this doc stays current.
```

Adapt this template to the project: fill in the project name and description, and if the user mentioned specific architectural details or extra conventions during step 1, incorporate them. The template is a starting point, not a rigid form.

#### `.env.example` (if requested)

```
# <describe the variable>
VARIABLE_NAME=your_value_here
```

### 4. Make hooks executable and install

```bash
chmod +x .githooks/commit-msg .githooks/pre-commit
git config core.hooksPath .githooks
```

### 5. Install dependencies

```bash
uv sync
```

### 6. Verify

```bash
uv run ruff check .
uv run pytest
```

### 7. Initial commit

Stage all files and commit:

```
chore: add project foundation
```

## Common Commands

After setup, these commands are available:

| Task | Command |
|------|---------|
| Install deps | `uv sync` |
| Add a dependency | `uv add <package>` |
| Add a dev dependency | `uv add --group dev <package>` |
| Run tests | `uv run pytest` |
| Lint | `uv run ruff check .` |
| Format | `uv run ruff format .` |
| Type check | `uv run pyright` |
| Re-install hooks | `git config core.hooksPath .githooks` |

## Renaming the project later

If the project was started with a placeholder (or just needs a new name), the name is baked into a few predictable places. To rename from `<old-name>` to `<new-name>` — where `<old_package>`/`<new_package>` are the lowercase, importable forms with dots and hyphens replaced by underscores — work through these in order:

1. Rename the package directory (preserves history): `git mv src/<old_package> src/<new_package>`
2. Update `[project] name` in `pyproject.toml` to `<new-name>`. `uv_build` derives the package location from this name, so it must match the directory in step 1.
3. Update any `import <old_package>` / `from <old_package>` references in `src/` and `tests/`.
4. Update the project name and paths referenced in `CLAUDE.md`.
5. (Optional) Rename the project directory itself.
6. Re-sync so the editable install picks up the new name: `uv sync`
7. Verify nothing broke: `uv run ruff check . && uv run pytest`

## Directory Structure

```
<project-name>/
  .githooks/
    commit-msg
    pre-commit
  src/
    <package-name>/
      __init__.py
      py.typed
  tests/
    __init__.py
  .env.example      # if requested
  .gitignore
  .python-version
  CLAUDE.md
  pyproject.toml
  uv.lock
```
