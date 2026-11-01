# База данных и миграции

Источники истины: [SQLAlchemy-модели](../apps/api/app/db/models.py) и [Alembic versions](../apps/api/migrations/versions). PostgreSQL хранит JSONB/обычные поля, PostGIS — пространственные данные и индексы. В снимке последняя ревизия — `0012_financial_scenarios` (точное имя см. `alembic heads`).

Основные группы: conversations/business_profiles и сообщения; analysis runs и immutable reports; географические наборы, POI, границы и пешие графы; согласия, drafts/mock receipts и аудит; registry/acquisition jobs/events; owner-scoped agent runs; отраслевые JSONB-профили; финансовые workspaces, action receipts и неизменяемые results. Внешние ключи, уникальность и CHECK-ограничения задаются миграциями. Подробные имена таблиц и поля доступны непосредственно в моделях и DDL.

`0011` добавляет отраслевой профиль, `0012` — финансовые сценарии. U-06 объединяет существующие состояния без новой миграции. Исторические snapshots и legacy-отчёты не пересчитываются при чтении.

```powershell
docker compose -p business-ai-hackathon run --rm migrate alembic heads
docker compose -p business-ai-hackathon run --rm migrate alembic upgrade head
docker compose -p business-ai-hackathon run --rm migrate alembic current
docker compose -p business-ai-hackathon run --rm migrate alembic check
```

Для заполненной установки сначала создайте проверенную приватную резервную копию и выполните upgrade на отдельной восстановленной БД. Не запускайте DDL одновременно несколькими операторами. Проверяйте FK/unique, owner-доступ и сохранность экспортов. `alembic check` выявляет ORM drift, но не сертифицирует безопасность DDL или расширений. Политика schema comparison фиксирует public и исключает объекты, принадлежащие расширениям, чтобы не предлагать удаление TIGER/PostGIS.

Рабочий downgrade запрещён по умолчанию. Разрешение `allow_test_downgrade` предназначено только для изолированных БД с предусмотренными UUID-именами. Тесты создают свои БД и удаляют только их; [Публичное техническое описание](SECURITY_AND_LIMITATIONS.md). Нельзя выполнять тесты на рабочем сервере с реальными данными. Резервирование: [BACKUP_RESTORE](BACKUP_RESTORE.md).
