# Переменные окружения

!!! warning "Черновик"
    Описывает поведение до живой приёмки записи в ZenMoney; детали могут измениться.

Бот читает настройки **только из переменных окружения процесса**. Файл `.env` сам по себе не загружается: перед запуском загрузите его в shell. **Один процесс — один режим, один файл окружения.** Не держите боевые и тестовые переменные в одном файле или в одной сессии терминала: бот откажется запускаться.

## Режим → что читается → что запрещено

| `ZENBOT_ENV` | Токен ZenMoney | Токен Telegram | Каталог данных | Конфиг (`ZENBOT_CONFIG`) | Задано → отказ запуска |
|---|---|---|---|---|---|
| `prod` | `ZENMONEY_TOKEN` | `TELEGRAM_BOT_TOKEN` | `ZENBOT_DATA_DIR`, метка `.zenbot-mode` = `prod` | боевой, **без** `[test_guard]` | `ZENMONEY_TEST_TOKEN`, `TELEGRAM_TEST_BOT_TOKEN`, `ZENBOT_DATA_DIR_TEST`, `ZENBOT_MODE` |
| `test` | `ZENMONEY_TEST_TOKEN` | `TELEGRAM_TEST_BOT_TOKEN` (отдельный бот) | `ZENBOT_DATA_DIR_TEST`, метка `.zenbot-mode` = `test` | тестовый, **с** `[test_guard]` | `ZENMONEY_TOKEN`, `TELEGRAM_BOT_TOKEN`, `ZENBOT_DATA_DIR`, `ZENBOT_MODE` |

- `ZENBOT_ENV` обязательна, значения по умолчанию нет. Ровно `prod` или `test`, строчными, без пробелов.
- Старая переменная `ZENBOT_MODE` переименована в `ZENBOT_ENV`. Если она задана, бот откажется запускаться с подсказкой.
- «Задано» означает непустое значение. Пустые строки `NAME=` из `.env.example` не мешают.
- Один и тот же токен в боевой и тестовой переменной — отказ (`same_token`).
- Каталоги данных режимов не могут совпадать или быть вложены друг в друга (`data_dir_overlap`).
- Каталог с меткой другого режима — отказ (`data_dir_mode`).

После старта бот читает аккаунт ZenMoney и проверяет, что токен от аккаунта своего режима: в `test` все счета из `account_map` должны называться с `TEST `, в `prod` ни один не должен. Проверка повторяется при каждом чтении, в том числе перед записью.

## Все переменные

| Переменная | Режим | Обязательна | Что это и где взять |
|---|---|---|---|
| `ZENBOT_ENV` | оба | да | `prod` или `test`. |
| `TELEGRAM_BOT_TOKEN` | prod | да | Токен **боевого** бота от `@BotFather` (`/newbot`). |
| `TELEGRAM_TEST_BOT_TOKEN` | test | да | Токен **отдельного тестового** бота: создайте второго бота в `@BotFather`. Тестовые preview никогда не придут в боевой чат и наоборот. |
| `ZENMONEY_TOKEN` | prod | да | OAuth access token **боевого** аккаунта ZenMoney. Бот его не выпускает и не обновляет. |
| `ZENMONEY_TEST_TOKEN` | test | да | Токен **тестового** аккаунта ZenMoney. |
| `ZENBOT_DATA_DIR` | prod | да | Абсолютный путь к боевому каталогу данных с правами `700`. Там аудит-лог `audit.sqlite3`; **после первой записи его нельзя удалять** ([аудит-лог](../usage/rollback.md)). |
| `ZENBOT_DATA_DIR_TEST` | test | да | Абсолютный путь к тестовому каталогу данных, отдельному от боевого. |
| `OWNER_USER_ID` | оба | да | Ваш числовой Telegram **user id** (не id чата и не @username). Как узнать — ниже. |
| `ZENBOT_CONFIG` | оба | да | Абсолютный путь к TOML-конфигу счетов ([Конфиг счетов](accounts.md)). |
| `ZENBOT_WRITE_ENABLED` | оба | нет | Запись в ZenMoney. По умолчанию выключена. Включают **только** точные значения `true`, `1` или `yes`: строчными, без пробелов и кавычек. `TRUE`, `True`, `on`, `true ` (с пробелом) и пустое значение — выключено, `/status` покажет DRY-RUN. |
| `ZENBOT_MAX_TX_PER_PLAN` | оба | нет | Максимум операций в одном импорте, целое `1..5000`, по умолчанию `2000`. **Пустое значение — ошибка запуска**: удалите строку или укажите число. |
| `ZENBOT_WRITE_CHUNK_SIZE` | оба | нет | Сколько операций отправляется за один запрос, целое `1..18`, по умолчанию `18`. Пустое значение — ошибка запуска. Менять обычно не нужно. |

## Примеры

`.env.test` (значения в угловых скобках замените на свои; кавычки не нужны; боевые переменные сюда **не добавляйте**):

```dotenv
ZENBOT_ENV=test
TELEGRAM_TEST_BOT_TOKEN=<токен тестового бота>
ZENMONEY_TEST_TOKEN=<токен ТЕСТОВОГО аккаунта ZenMoney>
OWNER_USER_ID=<ваш user id>
ZENBOT_DATA_DIR_TEST=/home/you/zenbot-test-data
ZENBOT_CONFIG=/home/you/zenmoney-telegram-bot/config.bot.test.toml
ZENBOT_WRITE_ENABLED=false
```

`.env.prod`:

```dotenv
ZENBOT_ENV=prod
TELEGRAM_BOT_TOKEN=<токен боевого бота>
ZENMONEY_TOKEN=<токен боевого аккаунта ZenMoney>
OWNER_USER_ID=<ваш user id>
ZENBOT_DATA_DIR=/home/you/zenbot-data
ZENBOT_CONFIG=/home/you/zenmoney-telegram-bot/config.prod.toml
ZENBOT_WRITE_ENABLED=false
```

Загружайте файл в **новую** сессию терминала. Боевой и тестовый файлы в одной сессии не загружайте: для смены режима откройте новое окно.

```bash
set -a; . ./.env.test; set +a
```

Токены не вставляйте в код, TOML, issue, PR, чаты и в саму командную строку. Если токен Telegram попал в чужие руки, перевыпустите его: `@BotFather` → `/revoke`.

## Как узнать `OWNER_USER_ID`

1. Заполните в `.env.test` хотя бы `TELEGRAM_TEST_BOT_TOKEN` (или в `.env.prod` — `TELEGRAM_BOT_TOKEN`). User id у вас один для обоих ботов.
2. Убедитесь, что бот **не запущен**: иначе он сам заберёт сообщения.
3. Напишите своему боту в Telegram `/start`.
4. Выполните:

```bash
set -a; . ./.env.test; set +a
uv run --frozen python - <<'PY'
import os, requests
token = os.environ.get("TELEGRAM_TEST_BOT_TOKEN") or os.environ["TELEGRAM_BOT_TOKEN"]
try:
    r = requests.get(f"https://api.telegram.org/bot{token}/getUpdates", timeout=30).json()["result"]
    print(sorted({u["message"]["from"]["id"] for u in r if "message" in u}))
except Exception as e:  # текст ошибки requests содержит URL с токеном — печатаем только тип
    print("ERROR:", type(e).__name__)
PY
```

Будет выведен список вида `[123456789]`. Если чисел несколько, ботом писали несколько людей — выберите своё. Пустой список `[]` — сообщение не дошло, повторите шаг 3. `ERROR: ...` — неверный токен или нет сети.

!!! danger "Не публикуйте вывод"
    Полный ответ `getUpdates` и любые ошибки с адресом `api.telegram.org/bot...` содержат токен бота или тексты сообщений. Никуда их не вставляйте. Если токен всё же засветился — `@BotFather` → `/revoke` и новый токен в `.env`.

## Что значит ошибка при запуске {#startup-errors}

| Сообщение | Причина |
|---|---|
| `configuration error: env_missing: ZENBOT_ENV is not set ...` / `env_invalid: ZENBOT_ENV must be exactly ...` | Не загружен `.env`, или в `ZENBOT_ENV` не ровно `prod`/`test`. |
| `configuration error: mode_renamed: ZENBOT_MODE was renamed to ZENBOT_ENV ...` | Осталась старая `ZENBOT_MODE`: удалите её, задайте `ZENBOT_ENV`. |
| `configuration error: foreign_token: ZENBOT_ENV=prod: TELEGRAM_TEST_BOT_TOKEN must not be set` (и аналогичные) | В окружении переменная другого режима. Откройте новое окно терминала и загрузите только один `.env`. |
| `configuration error: same_token: ...` | Одинаковый токен в боевой и тестовой переменной. |
| `configuration error: no_token: ...` / `no_data_dir: ...` / `missing required environment variables: ...` | Нет обязательной переменной своего режима (названа в сообщении). |
| `configuration error: foreign_data_dir: ...` / `data_dir_overlap: ...` | Задан каталог данных другого режима, или каталоги режимов совпадают/вложены. |
| `configuration error: data_dir_mode: ...` | Каталог данных принадлежит другому режиму (метка `.zenbot-mode`). Укажите каталог своего режима. |
| `configuration error: data_dir_marker: ...` | Метку `.zenbot-mode` не удалось прочитать или записать (права, только чтение, пустая метка). |
| `configuration error: config_mode: ...` | В `prod` указан конфиг с `[test_guard]` или в `test` — конфиг без него. |
| `configuration error: no_production_identity: ...` / `config_invalid: ...` / `production_account: ...` | Тестовый конфиг: не заполнен или неверно заполнен `[test_guard]`, или в тестовом `account_map` счёт боевого аккаунта ([Конфиг счетов](accounts.md#test-config)). |
| `startup refused: account_missing: ...` | `prod`: ни одного счёта из `account_map` нет в аккаунте ZenMoney — скорее всего, токен от другого аккаунта. |
| `startup refused: test_account_in_prod: ...` | `prod`: счёт из `account_map` называется с `TEST ` — похоже, в боевом режиме токен тестового аккаунта. |
| `startup refused: account_unmarked` / `owner_ambiguous` / `production_principal` / `production_account` | `test`: счёта из `account_map` нет или его название не начинается с `TEST `; у аккаунта не один владелец; токен от боевого аккаунта. |
| `startup failed (TransportError), see log id ...` | При старте не удалось прочитать ZenMoney: неверный токен или нет сети ([проверка токена](../faq/api-errors.md#check-token)). |
| `configuration error: OWNER_USER_ID must be an integer Telegram user id` | В `OWNER_USER_ID` не число (например `@username`). |
| `configuration error: ZENBOT_MAX_TX_PER_PLAN must be an integer in 1..5000` | Пустое или неверное значение лимита. |
| `configuration error: ZENBOT_WRITE_CHUNK_SIZE must be an integer in 1..18` | Пустое значение, не число или больше 18. |
| `configuration error: account_map: unknown \`bank\` value ...` | Опечатка в `bank` (допустимо `forte`, `kaspi`, `halyk`). |
| `configuration error: account_map entry lacks key ...` | В записи `[[account_map]]` не хватает поля. |
| `configuration error: [dedup...] ...` / `[transfers]: ...` | Неверная секция TOML ([Конфиг счетов](accounts.md)). |
| `configuration error: cannot read ZENBOT_CONFIG: FileNotFoundError` (или `PermissionError`, `TOMLDecodeError`) | Неверный путь, нет прав на чтение или синтаксическая ошибка в TOML. |
| `startup failed (InvalidToken), see log id ...` | Неверный токен Telegram своего режима. |
