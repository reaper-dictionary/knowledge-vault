# CI — Continuous Integration

CI — автоматическая проверка проекта после изменений в Git. Обычно CI запускается на `push` и `pull_request` и проверяет, что код можно установить, проверить линтерами, типизатором и тестами. GitHub Actions описывает CI через YAML-файлы в `.github/workflows/`. ([GitHub Docs](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions "Workflow syntax for GitHub Actions - GitHub Docs"))

## Терминология

**Workflow** — весь сценарий CI, например `.github/workflows/ci.yml`.

**Event** — событие, запускающее workflow:

```yaml
on: [push, pull_request]
```

**Job** — отдельная группа проверок. Jobs по умолчанию могут выполняться параллельно. ([GitHub Docs](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions "Workflow syntax for GitHub Actions - GitHub Docs"))

**Runner** — машина, на которой выполняется job:

```yaml
runs-on: ubuntu-latest
```

**Step** — отдельный шаг внутри job:

```yaml
- name: Tests
  run: uv run pytest
```

**Action** — готовое переиспользуемое действие:

```yaml
- uses: actions/checkout@v6
```

`checkout` загружает код репозитория на runner.

**Exit code** — код завершения команды:

```text
0     → успешно ✅
не 0  → ошибка ❌
```

Поэтому CI работает на обычных CLI-командах: Ruff, mypy, pytest и т. д.

**Matrix** — один шаблон job, который GitHub запускает несколько раз с разными значениями. ([GitHub Docs](https://docs.github.com/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow "Running variations of jobs in a workflow - GitHub Docs"))

**Lock-файл** (`uv.lock`) — фиксирует разрешённые версии зависимостей.

```bash
uv sync --locked
```

`--locked` требует, чтобы существующий lock-файл соответствовал проекту и не нуждался в изменении. ([Astral Docs](https://docs.astral.sh/uv/concepts/projects/sync/?utm_source=chatgpt.com "Locking and syncing | uv"))

---

## Минимальный CI

```yaml
name: CI

on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: echo "CI works"
```

Структура:

```text
event
  ↓
workflow
  ↓
job
  ↓
runner
  ↓
steps
```

## Python-проверки

```yaml
- name: Install dependencies
  run: uv sync --locked

- name: Ruff lint
  run: uv run ruff check .

- name: Ruff format
  run: uv run ruff format --check .

- name: Mypy
  run: uv run mypy src

- name: Tests
  run: uv run pytest
```

Важно:

```bash
pytest || true
```

маскирует ошибку pytest, поэтому в настоящем CI так делать нельзя.

## Matrix для monorepo

```yaml
strategy:
  matrix:
    service:
      - casino_service
      - identity_service
      - gateway
      - wallet_service
```

Использование переменной:

```yaml
- run: uv sync --locked --project services/${{ matrix.service }}
```

GitHub создаст четыре запуска одного job — по одному на каждый сервис. ([GitHub Docs](https://docs.github.com/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow "Running variations of jobs in a workflow - GitHub Docs"))

## Protobuf

Проверяем сами контракты:

```bash
buf lint
```

Buf предназначен в том числе для lint-проверки Protobuf-схем в CI. ([Buf](https://buf.build/docs/lint/usage/?utm_source=chatgpt.com "Linting usage guide"))

Проверяем свежесть generated-кода:

```bash
python scripts/generate_proto.py
git diff --exit-code
```

Логика:

```text
.proto изменили
→ генератор изменил *_pb2.py
→ diff появился
→ CI падает
```

Значит разработчик забыл закоммитить актуальный generated-код.

## Docker Compose

```bash
docker compose config
```

Проверяет конфигурацию Compose без запуска контейнеров.

Если нужен `.env`, в CI можно подготовить безопасный вариант:

```yaml
- run: cp .env.example .env
- run: docker compose config
```

## Главное запомнить

```text
CI = повторяем локальные проверки автоматически.

push / PR
→ install
→ lint
→ format check
→ type check
→ tests
→ contracts
→ generated-code check
→ compose check
```

**CI не исправляет ошибки — CI обнаруживает их и останавливается с ненулевым exit code.**