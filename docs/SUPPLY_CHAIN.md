# Проверки цепочки поставки

Python-зависимости зафиксированы в `apps/api/uv.lock`, npm — в `apps/web/package-lock.json`, Docker requirements содержат hashes. Runtime images и GitHub Actions закреплены digest/SHA. Изменять lock-файлы ради оформления публичной версии не требуется.

Workflow `supply-chain` выполняет pip-audit, npm audit, Gitleaks и Trivy. Контейнерный gate проверяет исправимые Critical/High, полноту и актуальность входных отчётов; неисправленные находки не становятся автоматически принятым риском. [Открытые ограничения](SECURITY_AND_LIMITATIONS.md).

Локально из корня:

```powershell
npm --prefix apps/web audit --audit-level=low --registry=https://registry.npmjs.org
gitleaks dir . --config .gitleaks-dir.toml --ignore-gitleaks-allow --redact=100 --no-banner
gitleaks git . --config .gitleaks.toml --ignore-gitleaks-allow --redact=100 --no-banner
```

Не сохраняйте сырые отчёты с содержимым потенциальных секретов в Git. `scripts/check_secret_scanner.py` проверяет детектор на синтетических значениях. `scripts/image_security.py` анализирует локальные JSON-отчёты Trivy; точные команды с закреплённой версией сканера находятся в workflow. [Лицензии](../THIRD_PARTY_NOTICES.md), [Публичное техническое описание](SECURITY_AND_LIMITATIONS.md).
