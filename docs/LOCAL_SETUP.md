# Локальная установка

## Требования

Docker Engine/Desktop с Linux-контейнерами и Compose v2; сеть для скачивания образов и зависимостей. Ollama с моделью Qwen требует отдельного места под веса и достаточной RAM/VRAM; скорость зависит от оборудования, CPU-режим может быть медленным. Для запуска без контейнеров нужны Python 3.12, uv, Node.js 24/npm и отдельный PostgreSQL 17/PostGIS 3.6. Команды ниже выполняются из корня клона в PowerShell.

## Конфигурация и модель

```powershell
Copy-Item .env.example .env
ollama pull qwen3:4b-instruct-2507-q4_K_M
ollama list
```

Ollama должен работать (`ollama serve` в отдельном терминале, если сервер ещё не запущен). API в Docker использует `http://host.docker.internal:11434`; на Linux настройте доступность адреса хоста отдельно. Не открывайте Ollama в интернет. `LLM_PROVIDER=stub` предназначен только для тестов и не подтверждает работу Qwen.

`.env.example` содержит только локальные шаблоны. Задайте собственный `POSTGRES_PASSWORD`; не публикуйте `.env`. Для отраслевого пути установите `BA_INDUSTRY_ENABLED=1` и `BA_FINANCE_V2_ENABLED=1`. Агент и acquisition остаются отдельными opt-in. Если порты заняты, измените `API_PORT`/`WEB_PORT`.

## Docker Compose

Используйте новое имя проекта, чтобы создать отдельную БД и volume:

```powershell
docker compose -p business-ai-hackathon config --quiet
docker compose -p business-ai-hackathon up --build --detach --wait
docker compose -p business-ai-hackathon ps
Invoke-RestMethod http://127.0.0.1:8000/healthz
Invoke-RestMethod http://127.0.0.1:8000/readyz
```

Сервисы: `db`, одноразовый `migrate`, `api`, `web`; `db-tests` доступен в профиле test. PostGIS инициализируется в новой БД; `migrate` выполняет `alembic upgrade head` до запуска API. БД не публикует порт на хост. `readyz` проверяет зависимости приложения, но сам по себе не доказывает успешную генерацию Qwen. Откройте http://127.0.0.1:5173 и http://127.0.0.1:8000/docs.

Пустая БД не включает OSM. Операторский импорт описан в [GEO_DATA](GEO_DATA.md); синтетика — в [DATA_PROVIDERS](DATA_PROVIDERS.md). Синтетические данные не являются обязательным скрытым fallback. Worker acquisition включается только с проверенным registry-файлом и отдельным операторским токеном через `compose.acquisition.yaml`.

## API и frontend отдельно

```powershell
uv sync --locked --project apps/api
npm --prefix apps/web ci
$env:PGHOST='127.0.0.1'
$env:PGPORT='5432'
$env:PGDATABASE='business_ai'
$env:PGUSER='business_ai'
# PGPASSWORD задайте приватно в окружении для вашей отдельной локальной БД.
$env:OLLAMA_BASE_URL='http://127.0.0.1:11434'
$env:BA_INDUSTRY_ENABLED='1'
$env:BA_FINANCE_V2_ENABLED='1'
Push-Location apps/api
uv run --locked alembic upgrade head
uv run --locked uvicorn app.main:app --host 127.0.0.1 --port 8000
Pop-Location
```

Native API использует экспортированные PG-переменные; Compose `.env` автоматически в этот процесс не загружается. Требуется отдельная доступная PostgreSQL/PostGIS: compose-БД по умолчанию на хост не опубликована. В другом терминале запустите `npm --prefix apps/web run dev`. Vite проксирует API на `127.0.0.1:8000`; при другом порте задайте `BA_TEST_API_URL`.

## Миграции и остановка

```powershell
docker compose -p business-ai-hackathon run --rm migrate alembic current
docker compose -p business-ai-hackathon run --rm migrate alembic check
docker compose -p business-ai-hackathon down
```

`down` сохраняет volume. Не используйте `down --volumes` для данных, которые нужно сохранить. Не подключайте существующий volume другой установки; entrypoint отказывает непроверенному старому PGDATA. Native процессы останавливаются Ctrl+C. [Миграции](MIGRATIONS.md), [безопасность](SECURITY_AND_LIMITATIONS.md).
