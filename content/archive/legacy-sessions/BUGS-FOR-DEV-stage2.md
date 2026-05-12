# Багфиксы — стадия 2 (2026-05-05, дополнение к `BUGS-FOR-DEV.md`)

Дополнение к основному документу — найденное во время Стадии 1 API-прогона + наблюдения тестировщика на POS desktop.

**Этот файл присоединяется к** `BUGS-FOR-DEV.md` — те же стенд, креды, helper. Здесь — только новые баги и расширения уже описанных.

---

# admin-bff (расширение F13 — RBAC read-leak)

## F13-ext [Critical] RBAC read-leak: ещё 3 эндпоинта (всего 7)

**К ранее найденным 4** (`/catalog/kitchen-stations`, `/kds/devices`, `/kds/settings`, `/refunds`) **добавляются:**

| Endpoint | Permission которого нет у Курьера | Особое замечание |
|----------|-----------------------------------|-------------------|
| `GET /api/v1/admin/pos/devices` | `pos.settings.edit` или эквивалент | Возвращает 13 устройств с `current_user`, `last_user` |
| `GET /api/v1/admin/kitchen-queue?store_ids=X` | `kds.access` | **⚠️ Финансовая чувствительность:** возвращает заказы с `unit_price` каждой позиции |
| `GET /api/v1/admin/external-menus` | `menu.read` | На стенде сейчас 0 записей, но доступ есть |

**Репро (Курьер petr@test.local):**
```bash
PETR=$(curl -sS -X POST "$B/auth/login" -H "Content-Type: application/json" \
  -d '{"email":"petr@test.local","password":"<…см. private/creds.md>"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d);process.stdin.on('end',()=>console.log(JSON.parse(s).data.access_token))")

curl -sS "$B/pos/devices" -H "Authorization: Bearer $PETR"          # → 200, 13 устройств
curl -sS "$B/kitchen-queue?store_ids=fe4b54a9-2cc1-458f-9d0e-338bbc51df76" -H "Authorization: Bearer $PETR"  # → 200 с unit_price
curl -sS "$B/external-menus" -H "Authorization: Bearer $PETR"        # → 200
```

**Done when:** все 7 эндпоинтов F13 (4 исходных + 3 этих) проверяют permissions; запрос Курьером → 403.

---

# admin-bff (расширение F15 — HR 500-ки)

## F15-ext [Critical] HR-эндпоинты → 500 для всех ролей (4 эндпоинта)

**К ранее найденным 2** (`/payroll`, `/shift-templates`) **добавляются:**

- `GET /api/v1/admin/shift-records` → 500
- `GET /api/v1/admin/schedules` → 500

**Репро:** любой токен (admin/Manager/Курьер) → GET → 500 `INTERNAL_ERROR`.

**Гипотеза:** общий root cause со старым F15. Если фиксится один — проверить остальные три.

**Done when:** все 4 HR-эндпоинта возвращают 200 (с правом) / 403 (без права) — никаких 500.

---

# admin-bff (расширение F4 — 404 routes)

## F4-ext [Major] Ещё 4 маршрута → 404 (route в SPA bundle, не реализован в BFF)

К ранее найденным `/admin/tables` и `/admin/warehouse` (F16) добавляются:

- `GET /api/v1/admin/shifts` → 404
- `GET /api/v1/admin/aggregators` → 404
- `GET /api/v1/admin/paykeeper` → 404
- `GET /api/v1/admin/dashboard` → 404

Все упомянуты в SPA bundle админки (вероятно, из меню/навигации), но в BFF не зарегистрированы.

**Done when:** для каждого либо реализован endpoint и возвращает 200, либо удалён из SPA bundle (страница убрана из меню).

---

# admin-bff / catalog-service (валидация модификаторов)

## F30 [Major] Группа модификаторов: max < min проходит (BUG-044 confirmed)

**Endpoint:** `POST /api/v1/admin/modifier-groups`

**Репро:**
```bash
curl -sS -X POST "$B/modifier-groups" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"name":"test","type":"group","min_amount":3,"max_amount":1,"options":[{"name":"x","price":0}]}'
# → 201 Created (хотя 3 > 1, нелогично)
```

**Где смотреть:** body validator для `modifier-groups` POST/PATCH в admin-bff. Сейчас валидируются `name` и `type` (ответ `VALIDATION_ERROR` если не передать), но числовые инварианты не проверяются.

**Связано:** `BUG-044` в `04-known-bugs-index.md` («Модификаторы — нет валидации min/max»). Backend root cause — здесь.

**Done when:** запрос с `min_amount > max_amount` → 400 VALIDATION_ERROR с подсказкой.

---

## F31 [Major] Группа модификаторов: отрицательный `min_amount` проходит

**Репро:**
```bash
curl -sS -X POST "$B/modifier-groups" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"name":"test","type":"group","min_amount":-1,"max_amount":3,"options":[{"name":"x","price":0}]}'
# → 201 Created (с min_amount: -1 в БД)
```

**Done when:** `min_amount < 0` → 400.

---

## F32 [Major] Опция модификатора: отрицательная цена проходит

**Репро:**
```bash
curl -sS -X POST "$B/modifier-groups" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"name":"test","type":"group","min_amount":0,"max_amount":3,"options":[{"name":"opt","price":-100}]}'
# → 201 Created
```

**Влияние:** в позиции заказа `total_price = unit_price + sum(modifier.price)`. Отрицательная цена опции уменьшит total — может быть использовано как скрытая скидка в обход `is_manual_discount_banned` или контроля кассира.

**Done when:** `option.price < 0` → 400.

---

## F33 + F34 [Minor] Прочие пропуски валидации модификаторов

| ID | Что проходит (не должно) |
|----|---------------------------|
| F33 | `max_amount = 999999` — нет верхнего разумного предела (реалистично ≤ ~50) |
| F34 | `options: []` — группа модификатора без опций (бессмысленный объект) |

Оба на `POST /api/v1/admin/modifier-groups`. **Done when:** оба → 400 с понятным сообщением.

---

# admin-bff / catalog-service (валидация флагов товара)

## F35 [Minor] Товар: `is_open_price=true` + `is_by_weight=true` одновременно

**Endpoint:** `POST/PATCH /api/v1/admin/catalog/products`

**Репро:**
```bash
curl -sS -X POST "$B/catalog/products" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"name":"x","category_id":"6e6d98a2-b109-47f4-9f0d-50d6f4487410","type":"dish","unit_of_measure":"кг","vat_rate":"vat20","is_open_price":true,"is_by_weight":true}'
# → 201 Created (оба флага сохранены как true)
```

**Почему противоречиво:**
- `is_by_weight=true` → цена считается `base_price × weight`
- `is_open_price=true` → кассир вводит итоговую цену вручную при добавлении в заказ

Эти два режима взаимоисключающие. Что произойдёт на POS если оба true — UX-неопределённость.

**Done when:** валидация на mutex; одновременная установка обоих → 400.

---

# POS desktop (расширение F25)

## F37 [Critical] На POS нет журнала закрытых заказов смены

Расширяет F25 «Заказ исчезает из POS после оплаты» — добавляет конкретику о структуре UI.

**Симптом:** На POS desktop закрытые заказы доступны только в виде **агрегированной аналитики смены** (сумма выручки, разбивка по способам оплаты), без детализации по конкретным заказам. Открыть карточку конкретного закрытого чека через UI нельзя.

**Backend данные есть и доступны:**
- `GET /api/v1/admin/orders` возвращает все `closed` заказы со всеми полями (`paid_at`, `fiscal_data`, `pk_*`, `items[].unit_price`)
- В POS SPA bundle есть ссылка на `/api/v1/pos/orders/recent-paid` — то есть API для списка последних оплаченных предусмотрен, но UI его либо не подключает, либо вызывает с фильтром который ничего не показывает

**Влияние:** Кассир в течение смены не может:
- Посмотреть детали конкретного оплаченного заказа
- Запустить рефанд по конкретному закрытому заказу через привычный UI flow (только через ввод суммы вручную)
- Найти заказ по сумме / времени / клиенту в UI смены
- Z-отчёт смены показывает только агрегаты, без журнала операций

**Где смотреть:** POS desktop UI:
- Раздел «Смена» / «Чеки» / «История» — должна быть таблица закрытых заказов смены
- Подключение `/api/v1/pos/orders/recent-paid` к этой таблице
- При клике на строку — карточка заказа с deep view (items, fiscal, paykeeper, RRN если есть)

**Связь:** **F25 + F37 — одна история на UI слое** (после оплаты заказ пропадает из активных + нет вкладки журнала). Backend здесь не виноват — данные есть и доступны.

**Done when:** в POS UI есть вкладка «Журнал смены» / «Закрытые» с таблицей всех closed заказов. Кассир может открыть карточку конкретного заказа со всеми деталями, начать рефанд по нему.

---

## F27-clarify [Critical] Уточнение: на POS нет кнопки отмены ни на одном статусе

Уточнение к F27. Изначально я думал что кнопка может отсутствовать только для определённых статусов. Тестировщик подтвердил: **кнопки отмены нет вообще** — ни на `new`, ни на `accepted`, ни на `ready`.

При этом backend API `POST /api/v1/admin/orders/{id}/cancel` работает с обязательным body `{"reason":"..."}` для всех трёх статусов (для `closed` — корректно отказывает, не проверял).

**Done when:** в карточке заказа в любом из не-завершённых статусов есть кнопка «Отменить» с обязательным диалогом причины.

---

# order-service (под вопросом, требует ОТДЕЛЬНОЙ проверки)

## F24 [Investigate] Пропуски в нумерации заказов — вероятно симптом F25/F37

**Наблюдение:** На Арбате на 2026-05-05 в выборке через `GET /api/v1/admin/orders` пропущены номера `#22` и `#25` (диапазон 1–31, всего 29 заказов в API-выборке). На предыдущий день (1–6) пропусков нет.

**Гипотеза тестировщика (вероятно верная):** пропущенные заказы существуют, но **не показываются** ни в админ API-выборке, ни в POS UI — это та же история, что F25/F37 (закрытые заказы скрыты от пользователя). То есть `#22` и `#25` могут быть `closed` заказами, которые попали в особое состояние, недоступное через стандартный фильтр.

**Если гипотеза верна:** F24 — это **симптом F25/F37**, не отдельный баг нумерации. Достаточно фикса F25/F37 + проверки выборки `/admin/orders` без скрывающих фильтров.

**Если гипотеза неверна (для подстраховки):** возможны альтернативы:
- PostgreSQL sequence advance до commit транзакции (failed CREATE → номер потерян)
- Hard-delete вместо soft-delete при ошибке создания
- Race-condition между двумя POS

**Что проверить разработчику:**
1. SQL: `SELECT id, status, deleted_at, created_at FROM orders WHERE store_id=... AND order_number IN ('022','025')`
2. Если записи есть с заполненным `deleted_at` или непредвиденным `status` — это подтверждает гипотезу F24=F25
3. Если записей нет вообще — это тогда отдельный баг нумерации (sequence/race)
4. Сервис creating order: убедиться что номер выдаётся **внутри транзакции** с откатом при failure

**Done when:** объяснено почему пропуски есть; либо подтверждено что F24 = F25/F37, либо найден отдельный root cause и пофикшен.

---

# Сводка для приоритизации

**Критическое (security / финансы):**
- F13-ext (RBAC leak +3 эндпоинта, особенно `kitchen-queue` с unit_price)
- F37 (журнал закрытых заказов, расширение F25)

**Major (функциональность / валидация):**
- F30, F31, F32 (валидация модификаторов — реальные дыры)
- F4-ext, F15-ext (расширение известных проблем)

**Investigate (нужна проверка в БД):**
- F24 (пропуски нумерации — вероятно симптом F25/F37)

**Minor:**
- F33, F34, F35 (валидация edge cases)

---

## Что сказать Claude Code

При прогоне через Claude Code добавить к промпту из `BUGS-FOR-DEV.md`:

> Также прочитай `./BUGS-FOR-DEV-stage2.md` — это дополнения второй стадии. Сначала пройди по основному файлу, потом по этому. F13-ext и F15-ext — расширения уже описанных багов F13 и F15 (фикс должен покрывать оба исходных и новых эндпоинтов). F37 связан с F25 — скорее всего общий фикс. F24 — investigate-задача, не код-фикс: проверить БД сначала, потом решить нужен ли отдельный фикс.
