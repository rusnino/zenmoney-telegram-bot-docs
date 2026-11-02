# Аудит-лог и ручной откат

!!! warning "Черновик"
    Описывает поведение до живой приёмки записи в ZenMoney; детали могут измениться.

## Аудит-лог

Файл `audit.sqlite3` в каталоге данных режима (`$ZENBOT_DATA_DIR` или `$ZENBOT_DATA_DIR_TEST`): SQLite, права `600`, рядом служебные `audit.sqlite3-wal` и `audit.sqlite3-shm`. Пишется только при включённой записи.

!!! danger "После первой записи аудит-лог не удаляйте и не заменяйте пустым"
    Бот опирается на него при каждом следующем импорте: по нему он узнаёт операции, которые вы **удалили** в ZenMoney, и не создаёт их заново; там же хранятся объединённые переводы и остатки для проверки стыковки выписок. Без аудит-лога удалённые вами операции могут вернуться при повторной загрузке выписки.

### Резервная копия

Делайте её после каждого импорта или хотя бы регулярно и храните вне компьютера, где работает бот. Команда безопасна и при работающем боте (использует механизм резервного копирования SQLite, а не `cp`):

```bash
set -a; . ./.env.test; set +a
uv run --frozen python -c 'import os,sqlite3,time; d=(os.environ.get("ZENBOT_DATA_DIR") or os.environ["ZENBOT_DATA_DIR_TEST"]); dst=os.path.join(d,"audit.backup-"+time.strftime("%Y%m%d-%H%M%S")+".sqlite3"); b=sqlite3.connect(dst); sqlite3.connect(os.path.join(d,"audit.sqlite3")).backup(b); b.close(); os.chmod(dst,0o600); print("backup:", dst)'
```

Восстановление: остановите бота; сохраните в сторону **весь** текущий каталог данных, включая `audit.sqlite3-wal`, `audit.sqlite3-shm` и `.zenbot-mode`; уберите из каталога `audit.sqlite3`, `-wal` и `-shm`; положите резервную копию как `audit.sqlite3` (права `600`); метку `.zenbot-mode` оставьте своего режима. Старая копия может не знать о последних отправках: до новых импортов сверьте `/history` и операции с ZenMoney.

### Таблицы

| Таблица | Что в ней |
|---|---|
| `imports` | Импорт: номер, `mode` (`apply`), хэш плана, время (UTC), статус. |
| `created_tx` | Каждая отправленная операция: id, номер импорта, счёт ZenMoney, дата, сумма, валюта, `verified` (1 — бот увидел её в ZenMoney после записи). |
| `write_attempts` | Каждая порция записи: номер, время, `result` (`intent`, `ok`, `unknown_or_failed`), класс ошибки. |
| `planned_tx` | Полный план импорта. |
| `absorbed_tx` | Операции выписок, которые стали одним переводом. |
| `dedup_matches` | Почему операция не создана: `already_present`, `manual`, `transfer_*`. |
| `balance_anchors`, `balance_anchor_resets` | Сохранённые остатки выписок для проверки стыковки и журнал `/reset_balance`. |

### Статусы импорта {#import-statuses}

- `done` — всё отправлено и подтверждено;
- `verify_failed` — отправлено, но не всё увидено при проверке;
- `partial` — «частично записано»: бот остановился между порциями, потому что ZenMoney изменился во время записи, или порция не отправилась. Повторная загрузка того же PDF безопасна;
- `started` — процесс прервался посреди записи.

Последние три — [результат неизвестен](../faq/api-errors.md#unknown-result).

Строка в `created_tx` пишется **до** отправки: `verified=0` — «не подтверждено», а не «точно не создано». `/status` и `/history` показывают в `создано:` число таких строк, включая неподтверждённые.

### Запросы

Перед командами загрузите env своего режима. Номер импорта задаётся переменной `IMPORT_ID` (ниже `42` — пример).

Последние 10 импортов (номер | время UTC | статус | отправлено | подтверждено):

```bash
uv run --frozen python -c 'import os,sqlite3; db=os.path.join((os.environ.get("ZENBOT_DATA_DIR") or os.environ["ZENBOT_DATA_DIR_TEST"]),"audit.sqlite3"); [print(*r,sep=" | ") for r in sqlite3.connect(db).execute("SELECT import_id,created_at,status,(SELECT COUNT(*) FROM created_tx c WHERE c.import_id=i.import_id),(SELECT COUNT(*) FROM created_tx c WHERE c.import_id=i.import_id AND verified=1) FROM imports i ORDER BY import_id DESC LIMIT 10")]'
```

Операции одного импорта:

```bash
IMPORT_ID=42 uv run --frozen python -c 'import os,sqlite3; db=os.path.join((os.environ.get("ZENBOT_DATA_DIR") or os.environ["ZENBOT_DATA_DIR_TEST"]),"audit.sqlite3"); [print(*r,sep=" | ") for r in sqlite3.connect(db).execute("SELECT zenmoney_id,date,amount,currency,account_id,verified FROM created_tx WHERE import_id=? ORDER BY date,zenmoney_id",(int(os.environ["IMPORT_ID"]),))]'
```

Порции записи импорта:

```bash
IMPORT_ID=42 uv run --frozen python -c 'import os,sqlite3; db=os.path.join((os.environ.get("ZENBOT_DATA_DIR") or os.environ["ZENBOT_DATA_DIR_TEST"]),"audit.sqlite3"); [print(*r,sep=" | ") for r in sqlite3.connect(db).execute("SELECT chunk_no,started_at,finished_at,result,error_class FROM write_attempts WHERE import_id=? ORDER BY chunk_no",(int(os.environ["IMPORT_ID"]),))]'
```

## Ручной откат ошибочного импорта {#rollback}

Автоматического отката нет: бот только даёт список того, что он создал. Удаляете вы сами в приложении ZenMoney.

1. **Выключите запись**: `ZENBOT_WRITE_ENABLED=false` и перезапуск, либо остановите бота.
2. Найдите номер импорта: `/history` или ответ бота `Импорт №K`. Список операций — `/export_ids K` (`id | дата | сумма | валюта | счёт`). Сумма — как в выписке (расход со знаком минус). Счёт — id счёта ZenMoney.
3. Сделайте резервную копию аудит-лога (команда выше) и проверьте, какие операции импорта действительно есть в ZenMoney (только чтение, с названиями счетов). Загрузите `.env` того режима, в котором был импорт:

    ```bash
    set -a; . ./.env.test; set +a
    IMPORT_ID=42 uv run --frozen python - <<'PY'
    import os, sqlite3
    from zenbot.config import ClientSettings
    from zenbot.zenmoney_client.transport import ReadOnlyTransport
    c = ClientSettings.from_env()
    rows = sqlite3.connect(c.data_dir / "audit.sqlite3").execute(
        "SELECT zenmoney_id,date,amount,currency,account_id,verified FROM created_tx WHERE import_id=? ORDER BY date",
        (int(os.environ["IMPORT_ID"]),)).fetchall()
    d = ReadOnlyTransport(c.zenmoney_token).diff_read(0)
    live = {str(t["id"]).lower() for t in d.get("transaction", []) if not t.get("deleted")}
    titles = {a["id"]: a.get("title", "?") for a in d.get("account", [])}
    for zid, day, amount, cur, acc, ver in rows:
        print(zid, day, amount, cur, titles.get(acc, acc), "LIVE" if zid in live else "absent",
              "verified" if ver else "not verified", sep=" | ")
    print("total:", len(rows), "live:", sum(r[0] in live for r in rows))
    PY
    ```

4. В приложении ZenMoney откройте каждый счёт и удалите операции из строк с `LIVE`, сверяя **дату, сумму, валюту и счёт**. Если на счёте в этот день несколько одинаковых сумм, сверяйте описание с PDF. Перевод между своими счетами — одна операция, в списке её счёт — счёт списания.
5. Повторите проверку из шага 3 (без резервной копии): должно быть `live: 0`.
6. Аудит-лог руками не правьте. Факт отката фиксируется только в ZenMoney и в вашей копии.
7. Если импорт сохранил остатки ошибочной выписки, соседняя выписка получит `balance_discontinuity`. Сбросьте их: `/reset_balance KZT 1234 ГГГГ-ММ-ДД..ГГГГ-ММ-ДД`, затем то же с `confirm`.

### Повторная загрузка после отката

Бот не пишет поверх удалённых операций: удалённая операция попадёт во «вручную» с причиной `deleted_in_zenmoney` и создана не будет. Исключение — операции с `verified=0` (бот не видел их после записи), по которым ZenMoney не вернул метку удаления: они снова будут в «К созданию». Поэтому перед подтверждением сверьте «К созданию» с тем, что вы удаляли.

Значит, исправить ошибочный импорт повторной загрузкой той же выписки **нельзя**: удалённые операции бот больше не создаст. Исправленные операции вносите вручную в приложении. Это верно и после исправления конфига (например, неверного `account_map`): id операции зависит от выписки и владельца аккаунта ZenMoney, но не от счёта.
