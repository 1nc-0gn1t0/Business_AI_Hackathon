# Тестирование

## Окружение

Python 3.12, uv, Node.js 24/npm, prek; Docker Compose v2 для PostGIS. Зависимости устанавливаются из lock-файлов:

```powershell
uv sync --locked --project apps/api
npm --prefix apps/web ci
```

Тесты шифрованного backup требуют реальный age и age-keygen. В CI они скачиваются с проверкой закреплённого SHA. Для локального запуска:

```powershell
$ageRoot = ./scripts/install_age.ps1 -Directory (Join-Path $env:TEMP ('business-ai-age-' + [guid]::NewGuid().ToString('N')))
$env:BA_AGE_BINARY = Join-Path $ageRoot 'age.exe'
```

На Linux используйте бинарник `age` вместо `age.exe`. Тесты генерируют временные синтетические identities; не добавляйте их в Git. При проблеме доступа к общему pytest temp задайте `PYTEST_ADDOPTS=--basetemp=<новый-временный-каталог>`; в Windows используйте `/` в пути этой переменной.

## Unit, контракты и расширенные проверки

```powershell
uv run --locked --project apps/api pytest apps/api/tests -m "not extended"
uv run --locked --project apps/api pytest apps/api/tests -m extended
prek run --all-files
```

Обычный pytest без `BA_RUN_DB_TESTS=1` пропускает DB-интеграцию. Unit-тесты проверяют валидацию, права, согласия, расчёты, протокол LLM, лимиты и операторские инструменты; сетевые/LLM ответы контролируются fixtures. Extended — pairwise/property-based проверки. Prek запускает Ruff lint/format, mypy, pytest, ESLint, TypeScript и Vitest. Скрипты backup отдельно проверяются Ruff в CI.

## PostgreSQL/PostGIS и миграции

Только новый Compose-проект и новый volume, без подключения данных другой установки:

```powershell
docker compose -p business-ai-hackathon-test up --build --detach --wait db migrate
docker compose -p business-ai-hackathon-test run --rm migrate alembic check
docker compose -p business-ai-hackathon-test --profile test run --build --rm db-tests
docker compose -p business-ai-hackathon-test down
```

`db-tests` включает `BA_RUN_DB_TESTS=1`, использует настоящий PostGIS, создаёт и удаляет отдельные БД с UUID. Проверяются миграции пустой и заполненной БД, ограничения, конкуренция, owner/consent fences, snapshots, финансы и U-06. Проверку миграций с fault injection нельзя запускать на рабочем сервере. `down` оставляет тестовый volume; его удаление требует отдельного осознанного решения.

## Frontend и браузер

```powershell
npm --prefix apps/web run lint
npm --prefix apps/web run typecheck
npm --prefix apps/web test
npm --prefix apps/web run build
npx --prefix apps/web playwright install chromium
npm --prefix apps/web run test:e2e
```

Обычный Playwright использует mock API и временный Vite на 4173. Это не live-проверка модели. `e2e/fixtures/industry.json` содержит только DTO синтетических бизнес-сценариев, перенесённые из удалённого внутреннего QA-артефакта для сохранения тестов. Исторические времена выполнения и сырые QA-результаты не включены.

## CI

`ci.yml` проверяет Windows/Linux, lock/requirements, prek, сборку, extended и mock Chromium; отдельная job собирает Compose, проверяет готовность, защиту PGDATA и интеграционные тесты. `supply-chain` проверяет Python/npm, секреты и контейнеры. Собственные секреты репозитория для базовых проверок не нужны. Успех security gate не закрывает [существенные ограничения](SECURITY_AND_LIMITATIONS.md).

## Проверка публичного снимка 01.11.2026

Результаты ниже относятся к этой отдельной директории, а не перенесены из истории разработки.

- Установка uv/npm из lock-файлов: выполнена.
- Compose config: успешно; Alembic head `0012_financial_scenarios`, upgrade новой БД и `alembic check`: успешно.
- Extended pytest: 102 passed.
- Playwright с mock API: 49 passed.
- Vite production build: успешно, остаётся предупреждение о крупном MapLibre-чанке.
- npm audit: 0 vulnerabilities на момент проверки.
- Финальный `prek run --all-files`: все 7 hooks успешно (Ruff lint/format, mypy, pytest, ESLint, TypeScript, Vitest). Первые два запуска выявили локальные проблемы temp: недоступный общий каталог и разбор Windows-пути с обратными слешами. После выделения каталога вне репозитория с `/` все проверки прошли; код приложения не менялся.
- PostGIS-интеграция: 176 passed, новая изолированная БД и отдельный Compose project.
- Production Docker API/web build и smoke: `/readyz` — database/PostGIS/schema ok, web HTTP 200. Для smoke использован явный `LLM_PROVIDER=stub`; это не live-проверка Qwen. Тестовые сервисы остановлены, volume сохранён.
- Gitleaks directory scan публичной директории: находок нет; дополнительно проверяются файлы единственного начального коммита.
- Ссылки Markdown на локальные файлы: битых ссылок нет. Файлы `.env`, дампы, private keys, локальные пути и внутренние QA-материалы не включены.

Live Qwen, импорт актуального OSM и новый Trivy-скан образов в рамках подготовки публикации не проводились. Исторические сведения о контейнерных рисках отдельно отмечены в документе безопасности.
