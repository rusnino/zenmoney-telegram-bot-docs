# Конфиг счетов (TOML)

!!! warning "Черновик"
    Описывает поведение до живой приёмки записи в ZenMoney; детали могут измениться.

Боевой конфиг делается из `config.example.toml`, тестовый — из `config.bot.test.example.toml` (то же самое плюс обязательная секция `[test_guard]`). Путь к нему — в `ZENBOT_CONFIG`. Все значения ниже — вымышленные.

## Сопоставление счетов: `[[account_map]]`

Главная часть конфига — «счёт в выписке → счёт ZenMoney». Нужна **отдельная запись на каждый счёт каждого банка и на каждую валюту**:

```toml
[[account_map]]
bank = "forte"                                              # forte | kaspi | halyk
bank_account_ref = "KZ000000000000000000"                   # полный IBAN счёта точно как в PDF
currency = "KZT"
zenmoney_account_id = "00000000-0000-0000-0000-000000000000"  # id счёта в ZenMoney
```

- `bank_account_ref` сравнивается буква в букву. У всех текущих парсеров это полный IBAN из шапки выписки.
- Halyk с мультивалютной картой: по записи на каждую пару «IBAN + валюта».
- Если для счёта выписки записи нет, импорт блокируется (`unmapped_account`). Запасного счёта нет.

## Id счетов ZenMoney {#zenmoney-ids}

Команда печатает id, валюту и название неархивных счетов, без балансов. Нужны `ZENBOT_ENV`, токен ZenMoney и каталог данных своего режима: с `.env.test` она покажет счета тестового аккаунта, с `.env.prod` — боевого.

```bash
set -a; . ./.env.test; set +a
uv run --frozen python -c 'from zenbot.config import ClientSettings; from zenbot.zenmoney_client.transport import ReadOnlyTransport; from zenbot.zenmoney_client.reader import read_snapshot; c=ClientSettings.from_env(); s=read_snapshot(ReadOnlyTransport(c.zenmoney_token)); cc={v:k for k,v in s.instruments.items()}; [print(i, cc.get(a.get("instrument"), "?"), a.get("title", ""), sep=" | ") for i,a in s.accounts.items() if not a.get("archive")]'
```

## Остальные секции

Все необязательные.

| Секция | Ключи | Смысл |
|---|---|---|
| `[forte]` | `savings_account_suffix` | Последние 4 цифры вклада Forte: пополнение с описанием «Снятие со вклада» считается поступлением с этого вклада. |
| `[forte]` | `cash_account_title` | Устаревший ключ: принимается, но не используется. Снятие и внесение наличных остаются обычными расходом и доходом. |
| `[forte.recipient_payee_map]` | `"0000" = "Имя"` | Переводы на карту с этими последними 4 цифрами получают этого получателя (`payee`). |
| `[kaspi]` | `internal_candidate_markers` | Подстроки «Деталей» пополнения, по которым оно считается кандидатом в перевод между своими счетами. По умолчанию `["с карты другого банка"]`. |
| `[kaspi.recipient_payee_map]` | `"Детали" = "Имя"` | Точное совпадение «Деталей» (без учёта регистра) → получатель. |
| `[dedup.<bank>]` | `window_days` (0..3, по умолч. 1), `min_tokens` (2..5, по умолч. 2) | Тонкая настройка поиска похожих записей. Режим на уровне банка задать нельзя (ошибка запуска). |
| `[dedup."<bank>/<IBAN>/<CUR>"]` | `mode = "cross_source"`, `evidence = "none"` + ключи выше | Режим «между источниками» только для одного счёта. `evidence = "none"` опасен: операция может быть пропущена как «уже есть» только по сумме, счёту и дате. |
| `[transfers]` | `date_window_days` (0..5, по умолч. 2), `fee_abs` (≥0, по умолч. 0), `fee_pct` (0..5, по умолч. 0.5) | Когда расход и приход на разных своих счетах предлагаются как один перевод. |

Без секции `[dedup]` все счета работают в строгом режиме `strict`: похожая на существующую операция не создаётся, а попадает в список «вручную».

### Режим «между источниками» (`cross_source`)

Нужен, если в ZenMoney те же операции уже приходят из другого источника (банковский плагин, ручной ввод). Тогда совпавшие записи пропускаются как «уже есть», а не уходят во «вручную». Риск: описание может совпасть случайно, и новая операция будет тихо пропущена. Поэтому режим включается только на конкретный счёт:

```toml
[dedup."halyk/KZ00000000000000000A/KZT"]
mode = "cross_source"
```

## Тестовый конфиг {#test-config}

Секция `[test_guard]` **обязательна** в тестовом конфиге и **запрещена** в боевом. В ней указывается ZenMoney User.id **боевого** аккаунта (не Telegram id), чтобы тестовый бот никогда не работал с боевым аккаунтом:

```toml
[test_guard]
production_principals = [123456789]   # ZenMoney User.id БОЕВОГО аккаунта; заглушка 1 не принимается

[[account_map]]
bank = "kaspi"
bank_account_ref = "KZ000000000000000000"
currency = "KZT"
zenmoney_account_id = "00000000-0000-0000-0000-000000000000"   # счёт ТЕСТОВОГО аккаунта, название "TEST ..."
```

- Каждый счёт `account_map` тестового конфига должен существовать в тестовом аккаунте и называться с `TEST ` (например `TEST Kaspi Gold`). Иначе тестовый бот не стартует (`account_unmarked`).
- В боевом аккаунте счета с префиксом `TEST ` из `account_map` недопустимы (`test_account_in_prod`).
- User.id боевого аккаунта: загрузите **боевой** `.env.prod` в новом окне терминала и выполните

    ```bash
    uv run --frozen python -c 'from zenbot.config import ClientSettings; from zenbot.zenmoney_client.transport import ReadOnlyTransport; from zenbot.zenmoney_client.reader import read_snapshot; s=read_snapshot(ReadOnlyTransport(ClientSettings.from_env().zenmoney_token)); print(sorted({a.get("user") for a in s.accounts.values()}))'
    ```

    Выведется одно число в квадратных скобках — его и впишите.

## Проверить конфиг

Без запуска бота и без печати секретов:

```bash
set -a; . ./.env.test; set +a
uv run --frozen python -c 'from zenbot.bot.settings import BotSettings; c=BotSettings.from_env(); print("config OK; accounts =", len(c.account_map), "; write_enabled =", c.write_enabled)'
```

Ожидаемо: `config OK; accounts = <число записей account_map> ; write_enabled = False`.
