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

### `deploy-compose-ssh.yml`

Sincroniza um `docker-compose.*.yaml` (e demais arquivos auxiliares) com uma VPS via SSH+rsync e executa `docker compose pull && up -d` lá. Foi extraído dos workflows originais de `ivanildobarauna.dev` (site + rollback) e `ivanildobarauna.dev.automations` (n8n). Notas:

- **Handoff via GitHub Artifact**: o caller é responsável por construir/buildar imagens, gerar o `docker-compose.prod.yaml` e o `.env`, e fazer `actions/upload-artifact@v4`. O reusável faz `download-artifact` e envia o conteúdo do diretório `./deploy/` por rsync. Isso evita escapar YAML/secrets em inputs string e mantém o caller como autor único do conteúdo.
- **`rsync` sem `--delete`**: divergência intencional do original. Os workflows antigos usavam `--delete` mas com arquivos avulsos como source, o que tornava a flag um no-op. Ao mudar para `./deploy/` (diretório), `--delete` se torna ativo e perigoso para `remote-path` com estado. A escolha conservadora é não habilitar.
- **Re-escrita do `.env` via SSH eliminada**: o padrão antigo do site fazia rsync do `.env` e depois `rm .env && echo "DD_API_KEY=..." >> .env` via SSH (redundante, já que o rsync já tinha entregue o conteúdo correto). O caller agora gera o `.env` localmente uma única vez e o rsync é a única fonte.
- **Flags opcionais para tags mutáveis**: `pull-policy-always: true` e `force-recreate: true` reproduzem o comportamento do `n8n-deploy.yml` (tags `latest`/branch+sha). O site usa SemVer imutável e não precisa.
- **`pre-deploy-command`**: comando shell opcional executado via SSH antes do `pull` — útil para o caso do n8n que faz `docker rmi -f` para forçar pull fresh.
- **`mkdir -p` do `remote-path`**: sempre executado (idempotente). O workflow de rollback original assumia o diretório existente; o reusável padroniza para criar.
- **`webfactory/ssh-agent@v0.10.0`**: fixado. Bump deliberado vs `v0.9.1` usado no site/rollback originais.
- **Secrets**: `VPS_SSH_HOST`, `VPS_SSH_USER`, `VPS_SSH_PRIVATE_KEY` são `required: true` na interface. Os valores reais ficam em `Settings → Secrets and variables` do repo caller.

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
