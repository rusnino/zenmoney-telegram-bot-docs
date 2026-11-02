# CLI

!!! warning "Черновик"
    Описывает поведение до живой приёмки записи в ZenMoney; детали могут измениться.

Кроме Telegram-бота есть консольный импорт операций из JSON: `python -m zenbot.zenmoney_client.cli`. Он нужен, когда операции уже получены не из PDF (например, подготовлены вручную или другим инструментом). Как и бот, по умолчанию работает в dry-run.

!!! warning "JSON не проверяется парсером"
    Для PDF бот доказывает полноту строк и сверяет остатки. Для JSON этого нет: за полноту и правильность списка отвечаете вы. Поэтому флаг `--trust-raw-complete` обязателен, без него CLI откажется работать.

## Формат входного файла

Пример с вымышленными данными:

```json
{
  "account_map": [
    {"bank": "kaspi", "bank_account_ref": "KZ000000000000000000", "currency": "KZT",
     "zenmoney_account_id": "00000000-0000-0000-0000-000000000000"}
  ],
  "transactions": [
    {"bank": "kaspi", "bank_account_ref": "KZ000000000000000000", "booking_date": "2026-01-15",
     "amount": "-1500.00", "currency": "KZT", "description": "Покупка Example Shop"}
  ]
}
```

`amount` — строка с десятичной суммой, расход со знаком минус.

## Запуск

Окружение — то же, что у бота ([Переменные окружения](configuration/env.md)): `ZENBOT_ENV`, токен ZenMoney и каталог данных своего режима. В режиме `test` обязателен `--config` с секцией `[test_guard]`.

Dry-run:

```bash
set -a; . ./.env.test; set +a
uv run --frozen python -m zenbot.zenmoney_client.cli ops.json --trust-raw-complete --config config.bot.test.toml
```

Вывод:

```text
to_create=1 duplicates=0 already_present=0 manual=0 blocked=0 plan_hash=<хэш>
dedup_mode=strict by_account={...}
DRY-RUN: nothing was sent. Re-run with --apply --confirm-hash <plan_hash> to write.
```

Запись (нужно ещё `ZENBOT_WRITE_ENABLED=true`): повторите с `--apply --confirm-hash <plan_hash>` из dry-run. Если план изменился, запись будет отклонена.

```bash
uv run --frozen python -m zenbot.zenmoney_client.cli ops.json --trust-raw-complete --config config.bot.test.toml \
  --apply --confirm-hash <plan_hash>
```

Результат: `sent=N verified=M import_id=K errors=[...]`. Номер импорта `K` — тот же, что в [аудит-логе](usage/rollback.md).

## Параметры

| Параметр | Что делает |
|---|---|
| `input` | Путь к JSON-файлу. |
| `--trust-raw-complete` | Обязателен: вы гарантируете, что список полный и верный. |
| `--config <файл>` | TOML с секцией `[dedup]` (без неё — строгий режим для всех счетов). Обязателен при `ZENBOT_ENV=test`. |
| `--apply` | Записать в ZenMoney (по умолчанию dry-run). |
| `--confirm-hash <hash>` | Хэш плана из предыдущего dry-run; обязателен с `--apply`. |

## Отличия от бота

- Переводы между своими счетами CLI не предлагает и не объединяет: кандидат в перевод остаётся во «вручную» или блокирует план.
- Проверки стыковки остатков между выписками нет.
- Блокировки, которые в боте снимаются кнопкой «принять риск» (`balance_discontinuity`, `snapshot_malformed`), здесь снять нельзя: используйте бота или исправьте записи в ZenMoney.
- Коды возврата: `0` — успех или dry-run, `1` — запись с ошибками, `2` — отказ конфигурации или режима, `3` — запись отклонена.

## Проверочные команды

Другие разовые команды из этой документации:

- проверка конфига — [Конфиг счетов](configuration/accounts.md);
- список счетов ZenMoney — [Id счетов ZenMoney](configuration/accounts.md#zenmoney-ids);
- проверка токена и сети — [Ошибки ZenMoney API](faq/api-errors.md#check-token);
- запросы к аудит-логу — [Аудит-лог](usage/rollback.md).
