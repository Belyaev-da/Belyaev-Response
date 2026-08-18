# Руководство администратора — Belyaev Response

## Централизованное развёртывание

### Enterprise-политики (chrome.storage.managed)

Настройте через реестр Windows (GPO) или Google Admin Console:

```json
{
  "LicenseKey": "BLZR-XXXX-XXXX-XXXX-XXXX",
  "CompanyNames": ["ООО Рога и Копыта", "rogakopyta.ru"],
  "BurnoutCheckEnabled": true
}
```

Полный список политик — в `schema.json`.

### Развёртывание через GPO (Windows)
1. Скопировать `soc-copilot/` на сетевой ресурс
2. Создать GPO: Computer Configuration → Administrative Templates → добавить шаблон
3. Прописать путь к расширению и Extension ID
4. `LicenseKey` через managed storage — аналитики не вводят ключ вручную

## Сервер лицензирования

Подробно: [`LICENSE_SERVER_SETUP.md`](LICENSE_SERVER_SETUP.md)

### Выпуск лицензии
```bash
ADMIN_API_KEY=<секрет> node scripts/issue-license.js \
  --org "Название организации" --tier pro --seats 50 --days 365 \
  --url https://belyaev.expert
```

### Мониторинг
```bash
# Список активных лицензий
curl -H "X-Admin-Key: <секрет>" https://belyaev.expert/api/admin/licenses

# Отзыв лицензии
curl -X POST -H "X-Admin-Key: <секрет>" \
  https://belyaev.expert/api/admin/licenses/BLZR-XXXX/revoke
```

### Автоотзыв при превышении лимита устройств
При попытке активации сверх `seats` — весь ключ автоматически отзывается. Причина и ID устройства-триггера фиксируются в базе. Новый ключ — через `issue-license.js`.

## Grafana-дашборд

1. Включить синхронизацию в настройках расширения каждого аналитика (или через GPO `GrafanaSyncEnabled: true`)
2. Prometheus endpoint: `https://belyaev.expert/metrics?key=<license-key>`
3. Данные: рейтинг по BelZor, динамика TP/FP/Benign, активность (псевдонимизировано)

## Экспорт/импорт настроек команды
Настройки → «Экспорт / импорт настроек команды» → выгрузить JSON с адаптерами и плейбуками → раздать новым сотрудникам. Ключи шифрования и лицензионные данные не экспортируются.

## Поддержка
belyaev.pro@mail.ru · t.me/belyaev_security
