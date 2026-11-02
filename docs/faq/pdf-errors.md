# Ошибки разбора PDF

!!! warning "Черновик"
    Описывает поведение до живой приёмки записи в ZenMoney; детали могут измениться.

Сначала определите, где видна ошибка.

## A. Бот ответил одной фразой, preview нет

Код в этом случае пишется только в лог бота (терминал, где он запущен):

| Ответ бота | Строка в логе | Что делать |
|---|---|---|
| `Не удалось определить банк выписки по структуре файла...` | `bank detection refused: bank_unknown` | Если банк точно Forte/Kaspi/Halyk: `/bank <банк>` и пришлите файл ещё раз первым файлом. Если не поможет — формат не поддержан. |
| `Структура файла подходит больше чем одному банку...` | `bank_ambiguous` | `/bank` не поможет. Передайте разработчику обезличенный образец. |
| `Выбранный банк не совпадает с банком, определённым по структуре файла...` | `bank_conflict` | Вы указали в `/bank` не тот банк. `/bank` без аргумента и пришлите заново. |
| `Не удалось прочитать файл как выписку поддерживаемого банка.` | `bank detection refused: pdf_unreadable` (или `pdf_empty`, `pdf_too_large`, `pdf_too_many_pages`) | Нужен оригинальный PDF из приложения или интернет-банка: не скан, не фото, не «печать в PDF», до 200 страниц. Длинный период разбейте на части. |
| `Разбор выписки занял слишком много времени.` | `parse timeout, worker killed` | Лимит — от 60 с до 5 мин в зависимости от размера. Разбейте период на части. |
| `Не удалось обработать выписку.` | `parse worker failed: ...` / `import failed: ...` | Повторите один раз. Повторяется — передайте разработчику время и строку лога. |
| `Нужен PDF-файл выписки.` / `Файл слишком большой...` | — | Только `.pdf` до 10 МиБ. |
| `Выписки содержат слишком много операций...` | `parsed rows over the batch cap` | Разбейте пакет: присылайте меньше файлов за раз. |

## B. Preview пришёл, но `Запись невозможна: parse_error`

Код и пояснение — в списке «Обработать вручную»: `заблокировано (parse_error): <код>: <текст>`.

| Коды | Что значит | Что делать |
|---|---|---|
| `balance_mismatch` | Остаток на начало + сумма строк ≠ остаток на конец. | Этот файл не импортировать. Для вклада Forte — самая свежая накопительная выписка ([Forte](../banks.md#forte)). |
| `geometry_missing`, `table_geometry_mismatch`, `band_unclassified`, `row_not_parsed`, `row_text_mismatch`, `row_ambiguous`, `rows_incomplete`, `rows_dropped`, `no_rows` | Бот не может доказать, что прочитал **все** строки таблицы. | Скачайте выписку заново. Если повторяется на новом формате банка — нужен образец и доработка парсера. |
| `table_schema_unknown`, `table_schema_conflict`, `table_unrecognized`, `bank_mismatch` | Таблица операций в незнакомом формате. | То же: новый макет банка → доработка парсера. |
| `row_malformed`, `row_bad_date`, `row_bad_amount`, `row_bad_original_amount`, `row_currency_mismatch`, `row_out_of_period`, `row_flow_ambiguous`, `row_sign_conflict` | Одна строка не по формату (номер строки и страницы — в тексте). | Сравните строку в PDF. PDF вручную не редактируйте: запросите новый экспорт. |
| `summary_missing`, `summary_unrecognized`, `summary_inconsistent`, `summary_period_mismatch`, `summary_currency_mismatch`, `summary_total_mismatch`, `turnover_mismatch`, `available_mismatch`, `available_inconsistent` | Итоговая сводка выписки не совпадает со строками. | Скачайте выписку заново или за другой период. |
| `account_not_found`, `account_invalid`, `account_conflict`, `multiple_accounts`, `row_account_ambiguous`, `row_account_unresolved`, `row_amount_inconsistent` | Не удаётся однозначно определить счёт или валюту. | Проверьте, что это выписка по одному поддерживаемому продукту; затем `account_map`. |
| `currency_not_found`, `currency_unsupported`, `header_currency_mismatch`, `header_balance_invalid`, `period_not_found` | Шапка выписки не распознана или валюта не поддержана (Kaspi — только KZT). | Скачайте выписку заново; для другой валюты нужна доработка. |
| `parse_internal_error` | Внутренняя ошибка парсера Halyk. | Передайте разработчику. |

## C. Предупреждения {#warnings}

Они не блокируют запись, но видны в списке «вручную» как `предупреждение парсера (<код>): ...`: `row_unknown_type` (строка не классифицирована и в ZenMoney не пойдёт), `row_zero_amount`, `rows_pending_hold_excluded`, `balance_interest_unreconciled`, `balance_not_verified`, `header_period_invalid`, `totals_not_found`, `card_not_found`, `account_inferred` (Halyk: счёт списания выбран по итогам выписки, [Halyk](../banks.md#halyk)).

!!! tip "Образец для разработчика"
    Настоящую выписку никуда не публикуйте. Если нужен образец нового формата, сделайте обезличенную копию: ФИО, ИИН, IBAN, номера карт, суммы и описания замените вымышленными, структуру (таблица, колонки, шапка, сводка) сохраните.
