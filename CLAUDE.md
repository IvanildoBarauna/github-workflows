# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository hosts **reusable GitHub Actions workflows** consumed by other repositories via `workflow_call`. There is no application code, no test suite, and no build step here — the deliverables are the YAML files under `.github/workflows/`.

## Architecture

Each workflow under `.github/workflows/` is intended to be referenced from a caller repository using:

```yaml
jobs:
  publish:
    uses: IvanildoBarauna/github-workflows/.github/workflows/<file>.yml@<ref>
    secrets:
      PYPI_TOKEN: ${{ secrets.PYPI_TOKEN }}
```

### `tests-python-poetry.yml`

Roda testes Python com Poetry + (opcional) upload de cobertura para Codecov, suportando matrix de versões. Notas:

- **Matrix via JSON**: `python-versions` é uma string contendo um array JSON (`'["3.10"]'` para versão única, `'["3.9","3.10","3.11"]'` para matrix). Isso permite uma única interface tanto para callers single-version quanto multi-version sem mudar o reusável.
- **`coverage` no test-command**: o reusável instala Poetry em venv (`virtualenvs-in-project: true`), então o `test-command` default já prefixa `poetry run`. Callers que customizarem o comando precisam manter o `poetry run` ou mover `coverage` para o ambiente global.
- **Codecov opcional**: `run-codecov: false` pula o step de upload por completo; o secret `CODECOV_TOKEN` é declarado como `required: false` para que callers sem Codecov não precisem passá-lo.
- **`working-directory`**: callers monorepo (ex.: `ivanildobarauna.dev` com backend em `backend/`) usam esse input. O caminho do `coverage.xml` enviado ao Codecov já é concatenado com a working-directory.
- **Gate `github.actor != 'actions[bot]'`**: replica o padrão dos workflows originais para evitar loops quando bots abrem PRs.

### `publish-python-package.yml`

Publishes a Poetry-based Python package to PyPI and creates a matching GitHub Release. Key design decisions to preserve when editing:

- **Trigger gate**: the `publish` job only runs when a PR has been **merged** AND carries the label specified by `label-gate` (default `deployable`). Both conditions are enforced in the job-level `if:`. Callers must hand off `pull_request` events (typically `types: [closed]`) for this gate to evaluate correctly.
- **Checkout ref**: uses `github.event.pull_request.merge_commit_sha` rather than the default branch HEAD, so the published artifact matches exactly what was reviewed in the PR.
- **Version source**: `poetry version -s` reads the version from the caller's `pyproject.toml`. The same value drives both the PyPI upload and the `v<version>` Git tag/Release name — do not introduce a second source of truth.
- **Environment**: the job runs inside a GitHub Environment (default `Production`) so that environment protection rules (required reviewers, secret scoping) apply on top of the label gate.
- **Required secret**: `PYPI_TOKEN`. `GITHUB_TOKEN` is consumed automatically for the release step and requires the workflow-level `contents: write` permission already declared.

## Conventions

- Inputs are documented in Portuguese (`description:` fields) — keep that style when adding new inputs.
- Default versions (`python-version`, `poetry-version`) are pinned as strings; bump them deliberately and treat changes as breaking for callers.
- When adding a new reusable workflow, expose configuration via `workflow_call.inputs`/`secrets` rather than hardcoding caller-specific values.
