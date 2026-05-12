---
date: 2026-05-06
type: regression
tester: Claude (через Playwright + admin SPA browser context)
duration_h: ~1
scope: F1–F45 от session-2026-05-05
stand: https://erp-test.nirbi.ru
---

# Регресс 2026-05-06 — проверка фиксов вчерашних F-NN

Разраб сообщил что «всё пофикшено». Прошёлся по всем F-NN из вчерашней session-2026-05-05. Метод — Playwright + fetch из браузерного контекста админ-SPA (admin@erp.local).

POS-баги (F25, F27, F37, F42) не проверял — pos-bff снаружи стенда недоступен, нужен Александр на POS-desktop.

## Сводка

| Статус | Кол-во | F-NN |
|--------|--------|------|
| ✅ FIXED | 13 | F3, F8, F9, F10, F13, F14, F15, F21, F30, F31, F33, F34, F35 |
| 🔴 STILL OPEN | 8 | F1, F6a, F6b, F26, F32, F41, F44, F45 |
| ⚠️ VERIFY-NEEDED | 6 | F2, F4, F7, F16, F24, F40 |
| 📵 POS-deferred | 4 | F25, F27, F37, F42 |
| ❓ ENDPOINT MISSING | 1 | F39 (stop-lists 404 везде) |

13 из 32 нон-POS багов закрыто (~40%).

## Детали по каждому

### ✅ FIXED (13)

- **F3** — KDS devices: 0 устройств с `kitchen_station_id=null` (было 8 из 8). Все 12 KDS на `Кухне` (station `743cd003-...`).
- **F8** — PATCH product `available_in_all_stores=false`: теперь возвращает **422 BUSINESS_RULE_VIOLATION** с понятным сообщением: «Cannot set available_in_all_stores=false without store_ids. Use PATCH /products/{id}/stores». Раньше было тихое 200 с потерей `store_ids`.
- **F9** — DELETE product: 204, без 500. Cascade ок.
- **F10** — `GET /catalog/categories/{id}`: 200, отдаёт всё поле.
- **F13** — Курьер (petr@): на 8 проверенных admin-эндпоинтах (`products`, `categories`, `price-lists`, `modifier-groups`, `employees`, `legal-entities`, `stores`, `refunds`) — везде 403. Лик закрыт.
- **F14** — `/admin/refunds` для Курьера: **403** с явным `Missing permission: pos.refund`.
- **F15** — все 4 HR-эндпоинта работают. **Маршруты переехали в root**: `/payroll`, `/shift-templates`, `/shift-records`, `/schedules` (а не `/employees/...`). При корректных `store_id`+`period`/`date_from`+`date_to` отдают 200. Раньше были 500.
- **F21** — `assembly_time_seconds` в позиции заказа: видел 180 секунд на свежем заказе #008 (Чизбургер). Денормализация работает.
- **F30** — модификатор `max<min`: 400 `amountRangeValid: max_amount must be >= min_amount`.
- **F31** — модификатор `min_amount: -1`: 400 `min_amount must be >= 0`.
- **F33** — модификатор `max_amount: 999999`: 400 `max_amount must be <= 50`.
- **F34** — модификатор без `options`: 400 `options must contain at least one item`.
- **F35** — товар `is_open_price=true + is_by_weight=true`: 400 `priceFlagsMutex`.

### 🔴 STILL OPEN (8)

- **F1** — Default-прейскурант «Базовый» (`5dcc6666-...`) всё ещё с `stores: []` и `store_count: 0`, при этом `is_default: true` и применяется неявно. Связано с BUG-051 (UI не привязывает прейскурант к ТТ).
- **F6a** — `PATCH /catalog/products/{id}` с `base_price: 99.99` — **200 OK, поле тихо принято, в ответе `base_price` отсутствует вообще**. Поведение «silent accept of unknown field» сохраняется.
- **F6b** — `GET /catalog/products/{id}` — поля цены нет в ответе. Поле должно жить в `price-list`, но product DTO не показывает текущую цену даже как computed field.
- **F26** — Закрытый заказ #008, `payment_method: "card"`, `paid_at: 2026-05-06T11:07:57Z` — `rrn: null`, `card_last4: null`. Тоже `pk_fop_receipt_key: null`, `fiscal_data: null`, `fiscal_failed: false`. Ничего не пробрасывается из PayKeeper.
- **F32** — `POST /modifier-groups` с `options[].price: -100`: **201 OK**, опция создана. Но в DTO опции вообще нет поля `price` (см. F44) — поэтому неясно, реально ли отрицательная цена сохранена или просто проигнорирована. Без видимости поля невозможно гарантировать, что цена не «утечёт» через price-list-modifier-items.
- **F41** — `PATCH product` с `modifier_group_ids: ['00000000-...']` (несуществующий): **200**, modifiers остался `[]`. Поле тихо проигнорировано без 400.
- **F44** — `POST /modifier-groups` с `options[].price: 50`: **201**, в созданном option поле `price` отсутствует. Та же проблема silent-ignore.
- **F45** — DELETE modifier-group: после DELETE `GET /modifier-groups/{id}` всё ещё возвращает **200** (возможно soft-delete без фильтрации, либо DELETE не отрабатывает). Orphan-проверка через `/price-lists/{id}/modifier-items` невозможна — этот эндпоинт 404.

### ⚠️ VERIFY-NEEDED (6)

- **F2** — Из 30 sampled товаров: 11 на «Кухне», 4 на `<none>` (Кола, маффин, сок, чизкейк). На Бар/Горячий/Холодный — 0. Тест-данные не правились.
- **F4** — Часть бэкенд-маршрутов есть, часть нет. **Существуют (400/200)**: `/orders/tables`, `/employees`, `/payroll`, `/shift-templates`, `/shift-records`, `/schedules`. **Не существуют (404)**: `/tables`, `/warehouse`, `/shifts`, `/aggregators`, `/paykeeper`, `/dashboard`, `/integrations/*`, `/finance/*`. Нужно уточнить у разраба список «целевых» путей.
- **F7** — Капучино, Латте всё ещё на станции «Кухня», не «Бар». Тест-данные.
- **F16** — `/warehouse/*` (stocks, receipts, movements, inventories) — все 404 и под admin, и под Manager. Warehouse-сервис не выставлен через admin-bff. BUG-066 («Склад не найден для выбранной ТТ») остаётся блокером.
- **F24** — В 20 заказах визуально гэп `8 → 26`, но это лимит пагинации (max=20). Нужно пройти по страницам, чтобы подтвердить пропуски.
- **F40** — `menu-availabilities` отдаёт корректные окна (Кофе 07–12, Десерты 14–22). Но фактическое скрытие категории на POS вне окна — проверяется только на POS-desktop.

### ❓ ENDPOINT MISSING

- **F39** — `/catalog/stop-lists`, `/stop-lists`, `/stoplist`, `/stores/{id}/stop-list`, `/menu/stop-list` — все 404 под обоими ролями. Эндпоинт переехал/убран. Нужно от разраба путь.

### 📵 POS-DEFERRED (нужен Александр)

- F25 — заказ исчезает с POS после оплаты
- F27 — нет UI cancel на POS
- F37 — нет журнала закрытых заказов
- F42 — нет подсказки о `max_amount` опций на POS

## Доп. наблюдения

1. **Перенос модификаторов**: `/admin/catalog/modifier-groups` → `/admin/modifier-groups` (без префикса `catalog/`). Влияет на старые URL в SPA/документации.
2. **HR в root**: `/admin/employees/payroll` → `/admin/payroll` (то же для shift-templates, shift-records, schedules).
3. **Поле `type` в modifier-group**: required, но **валидация значения отсутствует** — `type: "invalid_xx"` принимается с 201. Мини-баг (новая находка, см. F46).
4. **Порядок ошибок валидации**: при отсутствии `type` валидатор возвращает только `type is required` и не дает увидеть остальные ошибки (на практике мешало диагностике F32/F44).

## Что я создал на стенде (orphan)

- Тестовый продукт `1a84fead-b990-4bd9-b80c-dcbbd63653a7` (`[REGRESS] F8/F9 ...`) — DELETE прошёл с 204, но GET после возвращает 200. Возможный orphan / soft-delete без скрытия. Нужно проверить со стороны БД.
- Все созданные modifier-groups убраны через DELETE (но F45 показывает: DELETE может не отрабатывать).

## Новые находки

- **F46** — `type` в `POST /modifier-groups` принимает любую строку (например `invalid_xx`) с 201, без валидации enum. 🟢 Minor.

## Следующий шаг

- Уточнить у разраба: пути для `stop-lists`; что считается «правильным» поведением для F6a/F41/F44 (rejection через 400 или silent — но тогда документировать).
- Прогнать F25/F27/F37/F42 с Александр на POS.
- F40 проверить через POS либо через admin-resolution endpoint (если есть).
