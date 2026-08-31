# AGENTS.md — Repository Agent Instructions (Source of Truth)

This file defines the canonical coding directives for this repository.

If other instruction files exist (Copilot, IDE rules, contributor docs) and conflict with this file, follow this file and treat the others as stale.


## Table of contents

- [Project basics](#project-basics)
- [How to run code](#how-to-run-code)
- [Coding directives (Python)](#coding-directives-python)
- [Django architecture conventions](#django-architecture-conventions)
- [Tests](#tests)
- [Change workflow expectations](#change-workflow-expectations)
- [If instructions are missing or ambiguous](#if-instructions-are-missing-or-ambiguous)
- [Agent repository index](#agent-repository-index)


## Project basics

- Primary language: Python
- Target runtime: Python 3.12, as specified by `pyproject.toml`.
- Dependency / execution tool: `uv`
- Project-root is the directory containing this file (and `.git/`, and `.gitignore`).


## How to run code

- Assume user is in the project-root directory.
- Do not use `python` to run scripts.
- Run a script via: `uv run ./path_to_script.py --help`
- Run tests via:
    - `uv run ./run_tests.py`
        - Note that `run_tests.py` has usage instructions about how to run more granular tests.
- Run django management scripts via: `uv run ./manage.py THE-COMMAND`


## Coding directives (Python)

### Type hints and imports

- Use Python 3.12 type hints everywhere (functions and important variables).
- Prefer builtin generics (e.g., `list[str]`, `dict[str, int]`) over `typing.List` / `typing.Dict`.
- Prefer PEP 604 unions (e.g., `str | None`) over `Optional[str]`.
- Avoid `typing` and `annotations` imports unless strictly necessary.

### Script structure

- Structure runnable modules as:
  - `def main() -> None: ...`
  - `if __name__ == '__main__': main()`
- Keep `main()` simple: parse args / orchestrate calls only.
- Put real logic into top-level helper functions and modules (no nested function definitions).
- Rarely use more than three levels of helper calls: `main()` can call `helper_a()`, which can call `helper_b()`, which can call `helper_c()` when necessary.

### Functions and control flow

- Prefer single-return functions (use local variables and a final return).
- Do not define functions inside other functions.
- Favor clarity and explicitness over cleverness.

### Logging

- When adding a log statement, when possible, format each variable value as its label followed by a comma and a space, with the value enclosed in double backticks.
- Prefer a label that matches the variable name. Example: `log.debug(f'branch_and_commit, ``{branch_and_commit}``')`.

### HTTP and networking

- Use `httpx` for all HTTP calls.
- Do not introduce alternate HTTP libraries (e.g., `requests`, `aiohttp`) unless the repository already depends on them and there is a documented reason.

### Docstrings

- Use triple-quoted docstrings.
- Write docstrings in present tense, with triple-quotes on their own lines.
  - Good: 
    ```
    """
    Parses ...
    """
    ```
  - Avoid: `"""Parse ..."""`
- The last line of non-test function-docstrings should be: `Called by: the_caller_function()` (or, if in another class/module, `Called by: module.Class.the_caller_function()`)
- Start test-function docstring-text with "Checks..."
- For header-comments, in functions, start the comment with two hashes (e.g., `## does this`).

### Additional coding directives

- inspect the `/ruff.toml` for additional coding directives, such as `max-line-length` and `quote-style`.

### Markdown formatting

- Do not add manual line breaks within Markdown paragraphs; let the renderer wrap them naturally.
- When creating a Markdown file with more than three top-level `##` headings, add a table of contents near the top with links to those headings.


## Django architecture conventions

### View-layer responsibilities

- `project/app/views.py` should contain **only** view functions that directly handle URL endpoints.
- Every view function in `project/app/views.py` should correspond to an entry in `project/config/urls.py`.
- Views should act as **manager/orchestrator** functions:
  - Parse request input (query params, POST body, files)
  - Perform minimal validation and shaping of inputs
  - Delegate substantive work to modules under `project/app/lib/`
  - Convert returned results into the appropriate `HttpResponse` (HTML, JSON, redirects)

### Business logic placement

- Put domain logic, integrations, and reusable operations in `project/app/lib/` (not in `views.py`).
- If multiple endpoints share logic, move that shared logic into `project/app/lib/` and keep each view thin.
- Prefer pure, testable functions in `project/app/lib/` that accept plain Python values (not Django request objects)
  unless passing the request is necessary for a narrow, well-justified reason.

### Imports and dependencies

- `views.py` should primarily import:
  - Django primitives (`HttpRequest`, `HttpResponse`, `render`, `redirect`, etc.)
  - The minimal set of functions/classes from `project/app/lib/` needed for each endpoint
- Avoid creating a secondary abstraction layer inside `views.py` (no view-helper utilities); place helpers in `project/app/lib/`.


## Tests

- Use the standard library `unittest` framework (not pytest) for non-Django projects.
- Use Django's test framework for Django projects.
- New behavior should usually come with a focused test covering:
  - the happy path
  - at least one failure / edge case


## Change workflow expectations

When implementing a change (especially from an issue/task):

1. Read relevant surrounding code and match existing conventions.
2. Make the smallest correct change that satisfies the request.
3. Update tests and run: `uv run ./run_tests.py`
4. If you cannot run tests in your environment, still write/adjust tests and state what you would run.

### Commit messages

- Group related files into focused commits; do not require a separate commit for every file.
- Keep each commit message brief, with no more than ten words.
- Write messages in the present tense so they complete the phrase "This commit..." Begin with a fitting verb such as "Adds," "Implements," or "Updates."


## If instructions are missing or ambiguous

- Do not ask questions unless absolutely necessary to proceed.
- Make reasonable assumptions, state them explicitly, then implement.
- If blocked, provide:
  - what you tried
  - what you found in the repo
  - a concrete next step (command, file to edit, or minimal decision needed)


## Agent repository index

- `main.py` parses command-line input and coordinates remote checks, overwrite confirmation, cloning, Git metadata removal, sanitization, and state updates.
- `lib/repo_operations.py` contains environment parsing, persistent-state operations, Git commands, path validation, cloning, and directory removal.
- `lib/sensitive_cleanup.py` identifies text files and replaces sensitive assignments, email addresses, URLs, and hostnames. Large files use linear replacement passes, and URL or hostname work is skipped when no configured sensitive hostname marker is present.
- `tests/` uses `unittest` and mirrors the three main code areas: command orchestration, repository operations, and sensitive-content cleanup.
- Persistent state is stored in `../gather_repos_state.json`, one directory above this repository root. Do not assume the chosen output directory contains the state file.
- Update decisions compare only the remote `main` branch. A repository whose default branch differs from `main` produces a warning but is still checked and cloned from `main`.
- The operation order is clone, capture Git metadata, remove `.git` directories, sanitize files, and then save successful state. An interruption can leave an incomplete destination without updated state; a later run should replace that destination.

---
