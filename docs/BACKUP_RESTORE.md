# Резервирование и восстановление

Дампы содержат диалоги, профили, согласия и hashes гостевых токенов. Храните их вне Git с приватными ACL и шифрованием. Модельные веса, cluster roles, WAL/PITR и полноценное аварийное восстановление этой процедурой не покрываются.

Предпочтительный инструмент — `scripts/encrypted_backup.py` (age); `scripts/backup_postgres.py` создаёт plaintext и требует дополнительной защиты. Настройка age: `scripts/install_age.ps1 -Directory <новый-приватный-каталог>`. Private identity храните отдельно от backup, не в репозитории. Потеря identity делает восстановление невозможным.

Приватный JSON config содержит `root`, `age_binary`, публичный `recipient`; для отдельного Compose-проекта задайте `compose_files` и `project` согласно параметрам `Compose` в скрипте. Сначала изучите `--help` и конфигурацию `scripts/encrypted_backup.py`; не копируйте credentials в команды или логи.

```powershell
uv run --locked --project apps/api python scripts/encrypted_backup.py --help
uv run --locked --project apps/api python scripts/encrypted_backup.py backup --config $backupConfig
uv run --locked --project apps/api python scripts/encrypted_backup.py status --config $backupConfig
uv run --locked --project apps/api python scripts/encrypted_backup.py verify --config $backupConfig --bundle $bundle --identity $identity
```

Переменные обозначают пути к приватным файлам оператора. Создавайте новый каталог для каждого набора, проверьте age recipient и доступ к identity до эксплуатации. `verify` проверяет аутентичность/целостность; restore должен выполняться только в новую отдельную БД. Сверяйте версии PostgreSQL/PostGIS, схему, counts/hashes, согласия и исторические PDF/JSON. Обновление libc/PostGIS требует отдельного [порядка переноса](CONTAINER_UPGRADE.md).

Не восстанавливайте поверх рабочей БД, не используйте `pg_restore --clean` и не удаляйте исходный volume. Перед prune сначала просмотрите результат без `--apply`; автоматическая очистка не заменяет проверку восстановления. Scheduler-скрипты сохранены, но расписание в этой публикации не устанавливается. Тестовые faults допустимы только на созданных для них отдельных БД. [Миграции](MIGRATIONS.md), [ограничения](SECURITY_AND_LIMITATIONS.md).
