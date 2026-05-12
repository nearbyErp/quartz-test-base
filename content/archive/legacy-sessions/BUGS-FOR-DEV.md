# Багфиксы по итогам тестирования цепочки Admin → POS → KDS (2026-05-05)

Найдено 15 багов. Сгруппированы по сервису. Каждый — самодостаточный: репро, ожидание, факт, где смотреть, критерий «готово». Все воспроизведены на стенде.

## Тестовый стенд

```
Base URL:    https://erp-test.nirbi.ru
Admin BFF:   POST /api/v1/admin/auth/login  →  email + password → JWT (HS256, exp 15min)
POS BFF:     /api/v1/pos/* — слушает только локально на VPS, снаружи НЕ доступен
```

**Test creds (стенд, не для prod):**
- `admin@erp.local` / `<password — см. private/creds.md>` — Franchise admin, scope=all_franchise, 50+ permissions
- `petr@test.local` / `<password — см. private/creds.md>` — Курьер, scope=Арбат, perms: `customers.create_quick, customers.read, orders.read, pos.access`
- `ivan@test.local` / `123456` — Менеджер ТТ, scope=3 ТТ, 41 perm
- `anna@test.local` / `123456` — Кассир, в admin-bff не пускают (by-design)
- POS PIN: `1234` / `4321`
- KDS PIN: `1234` / `4321`

**Полезные id:**
- Арбат-флагман: `fe4b54a9-2cc1-458f-9d0e-338bbc51df76`
- Бауманская (другое ЮЛ): `47c128b9-1f7d-4f01-b6ad-795795b50e7a`
- Сокольники: `6e02dffe-e344-4859-909b-68059cb39385`
- Кухня (станция): `743cd003-7b65-4642-835d-1a1b8e1631e1`
- Категория «Бургеры»: `6e6d98a2-b109-47f4-9f0d-50d6f4487410`

**Helper для curl:**
```bash
TOKEN=$(curl -sS -X POST https://erp-test.nirbi.ru/api/v1/admin/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@erp.local","password":"<password — см. private/creds.md>"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d);process.stdin.on('end',()=>console.log(JSON.parse(s).data.access_token))")
H="Authorization: Bearer $TOKEN"
B="https://erp-test.nirbi.ru/api/v1/admin"
```

---

# admin-bff (catalog routes)

## F8 [Critical] PATCH product теряет `store_ids` при `available_in_all_stores: false`

**Endpoint:** `PATCH /api/v1/admin/catalog/products/{id}`

**Репро:**
```bash
# создаём товар
curl -sS -X POST "$B/catalog/products" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"name":"f8-test","category_id":"6e6d98a2-b109-47f4-9f0d-50d6f4487410","type":"dish","requires_kitchen":true,"kitchen_station_id":"743cd003-7b65-4642-835d-1a1b8e1631e1","unit_of_measure":"шт","vat_rate":"vat20","base_price":250}'
# → запоминаем id

# выключаем "доступно во всех точках" + указываем store_ids
curl -sS -X PATCH "$B/catalog/products/{id}" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"available_in_all_stores":false,"store_ids":["fe4b54a9-2cc1-458f-9d0e-338bbc51df76"]}'

# проверяем
curl -sS "$B/catalog/products/{id}" -H "$H"
```

**Ожидание:** `"available_in_all_stores": false, "store_ids": ["fe4b54a9-..."]`
**Факт:** `"available_in_all_stores": false, "store_ids": []` — товар недоступен **нигде**

**Где смотреть:** handler PATCH product в `admin-bff` (катch-фраза в bundle: функция передаёт `available_in_all_stores=i.available_in_all_stores`, но переданный массив store_ids видимо обнуляется до записи в БД). Проверить порядок: сначала записывается `available_in_all_stores=false`, потом отдельный handler стирает `store_ids` потому что флаг false. Логика должна быть наоборот: если `false` — записать `store_ids` как переданный массив.

**Связано:** UI-симптом в `04-known-bugs-index.md` — BUG-031 («после Save галочка возвращается»). Backend root cause — здесь.

**Done when:** PATCH с обоими полями сохраняет `store_ids` как переданный массив; e2e-тест на этот сценарий зелёный.

---

## F9 [Critical] DELETE product → 500 для товара в состоянии F8

**Endpoint:** `DELETE /api/v1/admin/catalog/products/{id}`

**Репро:** воспроизвести F8 → попробовать удалить:
```bash
curl -sS -X DELETE "$B/catalog/products/{id}" -H "$H" -i
# → HTTP 500 {"error":{"code":"INTERNAL_ERROR"}}
# затем GET того же id → тоже 500 (товар в неконсистентном состоянии)
```

**Ожидание:** 204 No Content + soft-delete (`deleted_at` заполнен)
**Факт:** 500 INTERNAL_ERROR; товар остаётся orphan-ом в БД; повторный GET тоже 500

**Где смотреть:** soft-delete query видимо предполагает `JOIN` или `EXISTS` по `product_stores` таблице. Если `store_ids = []` — falls. Логи `catalog-service` за время репро покажут точный exception.

**На стенде сейчас застрявший:** `id=f9cfbec8-5dba-4de2-9ee5-453076d1d2e2` (`[CHAIN-TEST] Бургер RENAMED`) — удалить через DB после фикса, проверить что новые товары удаляются OK.

**Done when:** DELETE возвращает 204 для товара в любом состоянии; стенд очищен; e2e-тест: создать товар → F8 PATCH → DELETE → 204.

---

## F4 [Major] `/admin/tables` → 404 (route registered, not implemented)

**Endpoint:** `GET /api/v1/admin/tables` (и `?store_id=...`, `/stores/{id}/tables`, `/halls`)

**Репро:**
```bash
curl -sS "$B/tables" -H "$H"
# → 404 {"error":{"code":"NOT_FOUND","message":"Route not found"}}
```

**Факт:** SPA bundle админки содержит ссылку на `/api/v1/admin/tables`, но в admin-bff route не зарегистрирован.

**Где смотреть:** route registration в `admin-bff/src/routes/` — проверить наличие `tables.routes.ts`. Также проверить `admin-bff` что endpoint вообще задизайнен или это deferred feature. Если фича отложена — убрать из SPA bundle (не показывать «Столы» в меню).

**Done when:** либо endpoint реализован и возвращает 200 со списком столов; либо удалён из SPA bundle.

---

## F6 [Major] `base_price` не возвращается через admin product API

**Endpoint:** `GET /api/v1/admin/catalog/products/{id}` (и POST/PATCH тоже)

**Репро:**
```bash
curl -sS "$B/catalog/products/{any_product_id}" -H "$H" | grep -o '"[a-z_]*price[a-z_]*"'
# → только "is_open_price"
# Поля base_price НЕТ
```

**Ожидание:** в response есть `base_price` (или эквивалент типа `price`), либо документация явно говорит «цены через прейскурант, не через продукт»
**Факт:** в response 40+ полей продукта, ни одного с ценой. POST/PATCH принимают `base_price`, отвечают 200 — но не возвращают. В `GET /catalog/price-lists/{id}` цен тоже нет.

**Цены при этом существуют** — позиции в заказах имеют корректные `unit_price` (например, Шаурма с говядиной = 297.5).

**Где смотреть:**
- `admin-bff` сериализатор продукта (`product.serializer.ts` или похоже) — поле просто не включено в response
- catalog-service: где физически хранится `base_price` — отдельная таблица (`product_prices`?) или поле продукта; admin-bff должен его вытаскивать
- Сравнить с `pos-bff` `GET /api/v1/pos/catalog/menu` — там цена точно есть, skopiroвать паттерн

**Связано:** F1 (price-list не привязан к ТТ; default не имеет цен). Скорее всего это разрыв admin/pos — POS видит цены, admin не видит.

**Done when:** `GET /admin/catalog/products/{id}` возвращает поле `base_price` (или эквивалент). Админ видит цену товара в карточке без необходимости лезть в POS.

---

## F10 [Major] `GET /catalog/categories/{id}` → 404 (асимметрия CRUD)

**Endpoint:** `GET /api/v1/admin/catalog/categories/{id}`

**Репро:**
```bash
# существующая категория Бургеры:
curl -sS "$B/catalog/categories/6e6d98a2-b109-47f4-9f0d-50d6f4487410" -H "$H"
# → 404 Route not found

# при этом PATCH и DELETE работают:
curl -sS -X PATCH "$B/catalog/categories/{id}" -H "$H" -H "Content-Type: application/json" --data-binary '{"name":"X"}'
# → 200
```

**Где смотреть:** `admin-bff/src/routes/categories.routes.ts` — отсутствует `GET /:id`. Добавить handler по аналогии с PATCH.

**Done when:** GET одной категории по id возвращает 200 с полным объектом (включая `children`, `is_active`, `is_available_*`).

---

## F16 [Major] `/admin/warehouse` → 404 даже у Manager с `warehouse.read+edit`

**Endpoint:** `GET /api/v1/admin/warehouse`

**Репро:**
```bash
curl -sS "$B/warehouse" -H "$H"
# → 404 Route not found
```

**Где смотреть:** `admin-bff/src/routes/` — отсутствует `warehouse.routes.ts` либо роут не зарегистрирован в root router. Проверить что warehouse-service даёт API и admin-bff его проксирует.

**Связано:** BUG-016 (Manager → Склад → 403). На самом деле там не 403 а 404 — значит проблема в маршрутизации, не в RBAC.

**Done when:** все warehouse эндпоинты доступны через `/api/v1/admin/warehouse/*` и enforce `warehouse.read` permission.

---

# admin-bff (RBAC / security)

## F13 [Critical] RBAC read-leak: 4 эндпоинта доступны без permissions

**Endpoints (для роли БЕЗ соответствующих perms):**
- `GET /api/v1/admin/catalog/kitchen-stations` — нужен `kds.access`
- `GET /api/v1/admin/kds/devices` — нужен `kds.access`
- `GET /api/v1/admin/kds/settings` — нужен `kds.access` / `kds.settings.edit`
- `GET /api/v1/admin/refunds` — нужен `pos.refund` (см. F14)

**Репро:**
```bash
# логин Курьером (4 perms, без kds.* и pos.refund):
PETR=$(curl -sS -X POST "$B%/auth/login" -H "Content-Type: application/json" \
  -d '{"email":"petr@test.local","password":"<…см. private/creds.md>"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d);process.stdin.on('end',()=>console.log(JSON.parse(s).data.access_token))")

curl -sS "$B/kds/settings" -H "Authorization: Bearer $PETR" -i
# → 200 OK с настройками KDS
curl -sS "$B/refunds" -H "Authorization: Bearer $PETR" -i
# → 200 OK с 4 рефандами
```

**Ожидание:** 403 FORBIDDEN
**Факт:** 200 OK с данными

**Контекст:** Mutation-RBAC работает. Например `PATCH /catalog/products` под Менеджером без `menu.edit` → 403 «No permission to edit catalog». Значит middleware есть и работает — он просто не подключен к этим 4 эндпоинтам.

**Где смотреть:** route definitions в `admin-bff/src/routes/{kitchen-stations,kds,refunds}.routes.ts` — на GET нет `requirePermission('kds.access')` / `requirePermission('pos.refund')`. Добавить.

**Done when:** для каждого из 4 эндпоинтов запрос Курьера → 403; admin/Manager с правом → 200.

---

## F14 [Critical] Refunds доступны Курьеру без `pos.refund`

Подмножество F13, но финансовая чувствительность — вынесено отдельно.

**Done when:** запрос `GET /api/v1/admin/refunds` без `pos.refund` → 403 (вместе с фиксом F13).

---

## F15 [Critical] `/admin/payroll` и `/admin/shift-templates` → 500 для всех

**Endpoints:** `GET /api/v1/admin/payroll`, `GET /api/v1/admin/shift-templates`

**Репро:**
```bash
curl -sS "$B/payroll" -H "$H" -i        # admin → 500
curl -sS "$B/shift-templates" -H "$H" -i # admin → 500
# тоже 500 у Менеджера с payroll.read и schedule.read
```

**Ожидание:** 200 (если есть perm) / 403 (если нет)
**Факт:** 500 INTERNAL_ERROR на всех ролях, включая admin

**Где смотреть:** логи `admin-bff` или соответствующего бэкенда (user-service?) за последние 24ч с фильтром `5(00|xx)` — стектрейс. Скорее всего unhandled exception в handler / dependency injection / SQL.

**Связано:** BUG-018, BUG-019, BUG-023 в `04-known-bugs-index.md` (HR backend 500-ки) — может быть тот же root cause.

**Done when:** GET возвращает 200 для пользователей с правом, 403 без права; e2e-тест прогоняется.

---

# order-service / catalog-service

## F21 [Major] `assembly_time_seconds` в позиции = null (денорм не сработала)

**Endpoint:** `GET /api/v1/admin/orders/{id}` → `items[].assembly_time_seconds`

**Репро:**
```bash
# создать заказ с товаром у которого assembly_time>0 (Двойной бургер: 240)
# проверить позицию заказа
curl -sS "$B/orders/{order_id}" -H "$H" | node -e "..."
# → items[0].assembly_time_seconds: null
```

**Ожидание:** `assembly_time_seconds: 240` (или сколько у товара)
**Факт:** `null`

**Где смотреть:** `order-service` создание позиции — должно копировать `assembly_time` из `catalog.products` в `order_items.assembly_time_seconds`. Сейчас поле в БД-схеме есть (раз null приходит, а не отсутствует), но не заполняется при INSERT.

**Симптом:** `expected_ready_at` рассчитывается одинаково для всех товаров (~+1.5 мин от создания), вместо использования реального assembly_time.

**Done when:** в новых заказах `assembly_time_seconds` равно значению `assembly_time` товара на момент создания; `expected_ready_at` корректно сдвигается.

---

## F1 [Major] Прейскурант не привязан ни к одной ТТ (модель неясна)

**Симптом:** `GET /api/v1/admin/stores` → у всех 3 ТТ `price_list_id: null`. `GET /api/v1/admin/catalog/price-lists/{id}` → `stores: []`. При этом цены в заказах есть.

**Связано:** BUG-051 (создание прейскуранта — нельзя вручную привязать к ТТ).

**Что нужно решить (продуктовый вопрос, не только код):**
- Если default-прейскурант неявно применяется ко всем ТТ → admin API должен возвращать `price_list_id = <default_id>` у каждой ТТ
- Если цены берутся из `base_price` товара → задокументировать это, и тогда F6 становится критичнее (admin не видит цены вообще)
- Если планируется явная привязка ТТ → починить BUG-051 + добавить в admin UI поле «Прейскурант» в карточке ТТ

**Done when:** контракт цен задокументирован, admin видит цену товара, привязка ТТ → прейскурант делается через UI.

---

# paykeeper-adapter / order-service

## F26 [Major] RRN и `card_last4` не заполняются после оплаты картой через PayKeeper

**Endpoint:** `GET /api/v1/admin/orders/{id}` → `rrn`, `card_last4`

**Репро:** оплатить любой заказ картой через PayKeeper на POS desktop → проверить:
```bash
curl -sS "$B/orders/{order_id}" -H "$H"
# payment_method: "card"
# paid_amount: <X>
# fiscal_data: { fn, rnkkt, ... } — заполнено
# pk_invoice_id, pk_payment_id, pk_invoice_url, pk_fop_receipt_key — заполнены
# rrn: null              ← баг
# card_last4: null       ← баг
```

**Пример заказа на стенде:** `id=85777593-dbec-4523-b78e-b73078a70f08` (#026)

**Где смотреть:** `paykeeper-adapter` — обработчик payment_completed event от PK. PayKeeper в callback возвращает RRN и маскированный PAN — сейчас они не извлекаются и не передаются в order-service.

**Влияние:** финансовая reconciliation усложняется (нет привязки к банковской транзакции), затрудняет рефанды и аудит.

**Done when:** после оплаты картой `rrn` и `card_last4` заполнены значениями от PayKeeper.

---

# POS desktop (Tauri app)

## F25 [Critical] Заказ исчезает из POS после оплаты

**Симптом:** Кассир оплачивает заказ → заказ полностью пропадает из всех видимых вкладок POS. На бэкенде заказ корректно сохраняется (status=closed, paid_at, fiscal_data, PayKeeper-поля).

**Влияние:** Кассир теряет видимость закрытых заказов в смене — нельзя посмотреть детали, инициировать рефанд через привычный flow, найти заказ.

**Где смотреть:** POS UI — найти фильтр на списке заказов или вкладку «Закрытые / История». Вероятно либо:
- Нет вкладки/фильтра «closed заказы текущей смены» — добавить
- Вкладка есть, но фильтр сломан — починить query

**Done when:** после оплаты заказ виден в списке закрытых/истории смены; можно открыть и посмотреть полную карточку.

---

## F27 [Critical Missing Feature] На POS нет кнопки отмены заказа

**Что:** Backend API `POST /api/v1/admin/orders/{id}/cancel` работает корректно. На POS desktop тестировщик не нашёл способа отменить заказ через UI.
**Влияние:** Кассир не может отменить ошибочно созданный заказ. Workaround — обращение к админу через бэк-офис.
**Воспроизведение (UI):** Создать заказ → искать кнопку «Отменить» → не находится
**Воспроизведение API (для подтверждения что backend готов):**
1. `POST /api/v1/admin/orders/{id}/cancel` с `{"reason":"<текст>"}` → 200 OK, status=cancelled
2. Без body → 400 INVALID_REQUEST_BODY (подсказывает что reason обязателен — UI должен это учесть)

---

## F3 [Critical] KDS-устройства зарегистрированы без `kitchen_station_id`

**Что:** Все 8 зарегистрированных KDS-устройств на Арбате имеют `kitchen_station_id: null`.
**Влияние:** Каждый KDS-экран показывает заказы со **всех** станций, без возможности фильтрации по кухне/бару/холодному цеху. По дизайну система предполагает ассоциацию `KDS device ⇔ kitchen station`.
**Воспроизведение:** `GET /api/v1/admin/kds/devices` → у всех `kitchen_station_id: null`
**Возможные причины:**
- В UI регистрации KDS нет поля выбора станции
- Поле есть, но не сохраняется
- Назначение делается отдельным flow которого нет
**Связь:** Вместе с F2 (все товары на одной станции) делает невозможным TC-CHAIN-032 (микс кухня+бар).

---

# Технический долг стенда

После фиксов:
1. Удалить orphan тест-продукт `f9cfbec8-5dba-4de2-9ee5-453076d1d2e2` (`[CHAIN-TEST] Бургер RENAMED`) — застрял после F9
2. Назначить кухонные станции 8 KDS-устройствам (F3)
3. Решить контракт цен (F1, F6) — задокументировать или починить
4. Назначить «барные» товары (Эспрессо, Капучино, Латте) на станцию `Бар` — иначе никакие multi-station тесты не воспроизводимы
