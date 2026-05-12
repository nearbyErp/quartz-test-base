# Багфиксы — стадия 3 (2026-05-05, дополнение к stage1+stage2)

Дополнение к **двум предыдущим файлам** (`BUGS-FOR-DEV.md` + `BUGS-FOR-DEV-stage2.md`). Здесь — только новое из стадии 3 + правильные endpoints обнаруженные в реверсе SPA-bundle.

**Стенд, креды, helper — те же** (см. `BUGS-FOR-DEV.md` шапку).

---

# 🔑 Правильные endpoints контракта каталога

Это **не баги**, а контекст для разработчика — какие endpoints правильные. Без этого знания легко полезть «фиксить» PATCH product (он же 200 OK!) и запутаться. Связано с F6a, F41, F44.

| Что | Endpoint | Body | Замечание |
|-----|----------|------|-----------|
| Цена товара | `PATCH /api/v1/admin/catalog/price-lists/{id}/items` | `{"items":[{"product_id","price"}]}` | Возвращает `{updated_count}` |
| Цена опции модификатора | `PATCH /api/v1/admin/catalog/price-lists/{id}/modifier-items` | `{"items":[{"modifier_option_id","price"}]}` | **Отдельный endpoint!** Не путать с /items |
| Привязка модификатора к товару | `POST /api/v1/admin/catalog/products/{id}/modifiers` | `{"modifier_group_id","binding_type":"free"}` | binding_type = `free` или `structural` |
| Стоп товара в ТТ | `POST /api/v1/admin/catalog/stop-lists/stores/{store_id}/products` | `{"product_id","reason"?}` | reason нужно валидировать (F39) |
| Снять стоп товара | `DELETE /api/v1/admin/catalog/stop-lists/stores/{store_id}/products/{product_id}` | — | 204 |
| Стоп категории | `POST /api/v1/admin/catalog/stop-lists/stores/{store_id}/categories` | `{"category_id","reason"?}` | |
| Снять стоп категории | `DELETE /api/v1/admin/catalog/stop-lists/stores/{store_id}/categories/{category_id}` | — | |
| Изменить статус временного меню | `PATCH /api/v1/admin/catalog/menu-availabilities/{id}` | `{"status":"active"\|"disabled"}` | regex enforced |

**При фиксах F6a / F41 / F44** — не убирать поле из body запроса (это сломает SPA). Лучше:
- Либо подключить силу записи к этим полям (если такая семантика декларируется)
- Либо отвечать 400 «Use endpoint X for this field»

---

# Новые баги стадии 3

## F6a [Critical] PATCH `/catalog/products/{id}` тихо игнорирует `base_price`

**Endpoint:** `PATCH /api/v1/admin/catalog/products/{id}`

**Репро:**
```bash
PID="<existing_product_id>"

# Отправляем base_price=100
curl -sS -X PATCH "$B/catalog/products/$PID" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"base_price":100}'
# → 200 OK
# → но updated_at в response НЕ изменился (остался от прошлого PATCH)

# Создаём заказ с этим товаром через POS
# В позиции заказа unit_price = старая цена, не 100
```

**Ожидание:** либо база цены меняется (если поле поддерживается), либо 400/422 «Use /price-lists/{id}/items to change price».
**Факт:** 200 OK, поле принято, **никаких изменений в БД** (`updated_at` не сдвигается, новые заказы используют старую цену).

**Где смотреть:** `admin-bff` PATCH product handler — поле `base_price` принимается в body (валидация проходит), но не передаётся в UPDATE статус. Решить:
1. Либо реализовать запись (если product.base_price действительно есть как fallback при отсутствии price-list записи)
2. Либо явно отвечать 400 с подсказкой про правильный endpoint (см. таблицу выше)

**Done when:** PATCH product с `base_price` либо реально меняет цену, либо отвечает 400 — никаких silent no-op.

---

## F6b [Minor] `GET /catalog/products/{id}` не возвращает текущую цену

**Endpoint:** `GET /api/v1/admin/catalog/products/{id}`

**Симптом:** Среди ~40 полей в response нет ни `base_price`, ни `price`, ни эквивалента. Чтобы узнать цену товара — нужен отдельный запрос `/price-lists/{id}/items`.

**Влияние:** Админ во вкладке «Товары» видит карточку без цены. Дополнительный round-trip при отрисовке списка товаров.

**Где смотреть:** product serializer в `admin-bff` — добавить поле `current_price` или `price` (взять из default-прейскуранта или из основного источника).

**Done when:** GET product возвращает текущую цену либо явно указывает источник (например `price_source: "price_list"`).

---

## F39 [Minor] Стоп-лист принимает `reason: null`

**Endpoint:** `POST /api/v1/admin/catalog/stop-lists/stores/{store_id}/products`

**Репро:**
```bash
curl -sS -X POST "$B/catalog/stop-lists/stores/$SID/products" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"product_id":"<id>"}'
# → 201 с reason: null
```

**Ожидание:** 400 «reason is required» — у бизнеса должна быть причина почему товар в стопе (закончился, снят с продажи, и т.д.).
**Факт:** 201, `reason: null` сохраняется.

**Контекст:** На стенде есть пример с reason заполненным (`Закончился острый перец`) — значит фронт умеет передавать. Но валидация на бэке отсутствует.

**Done when:** POST `/stop-lists/.../products` без `reason` → 400.

---

## F40 [Major] Правила `menu-availability` всегда скрывают категорию, независимо от окна

**Симптом:** Active правило `menu-availabilities` с временным окном **всегда** скрывает таргет категорию на POS. Нахождение текущего времени **внутри** окна не делает категорию видимой — она остаётся скрытой.

**Воспроизведение (через UI POS + API):**
1. На стенде есть правило: «Десерты вторая половина дня», окно 14:00-22:00, target = категория Десерты, status = active
2. Текущее время на стенде ~19:00 МСК — **внутри окна** → категория Десерты должна быть видна
3. **Факт:** На POS категории Десерты нет (Чизкейк, Маффин не отображаются)
4. PATCH правила в `status: disabled` → категория появляется на POS (после refresh меню)

**Сравнение поведения (оба правила active):**
| Правило | Окно | Текущее время | Ожидание | Факт |
|---------|------|----------------|----------|------|
| «Завтрак» 07-12 (категория Кофе) | 07:00-12:00 | вне окна | Кофе скрыт | скрыт ✅ |
| «Десерты вторая половина» 14-22 (Десерты) | 14:00-22:00 | в окне | Десерты видны | **скрыты ❌** |

**Гипотеза:** Логика обработки правила игнорирует время и работает как «active rule = всегда скрыть». То есть правила работают как «когда-нибудь стоп», а не «visibility window».

**Где смотреть:** Catalog Service — функция вычисления видимости категории/товара на момент запроса меню. Должна:
- Если есть active rule с окном → показывать категорию **только в окне**
- Если время вне окна → скрывать
- Если правила нет → показывать всегда

**Done when:** «Десерты вторая половина дня» становится видимой 14:00-22:00 и скрытой в остальное время.

---

## F41 [Major] PATCH `/catalog/products/{id}` тихо игнорирует `modifier_group_ids`

**Endpoint:** `PATCH /api/v1/admin/catalog/products/{id}`

**Репро:**
```bash
curl -sS -X PATCH "$B/catalog/products/$PID" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"modifier_group_ids":["<modifier_group_id>"]}'
# → 200 OK, в response modifiers: []  (пусто — поле проигнорировано)
```

**Правильный endpoint:** `POST /catalog/products/{id}/modifiers` body `{modifier_group_id, binding_type:"free"}` — это работает.

**Ожидание:** либо PATCH product с modifier_group_ids перепривязывает группы (как объявляет body shape), либо 400 «Use /products/{id}/modifiers».
**Факт:** 200, поле молча игнорируется. Аналогично F6a.

**Done when:** либо реализовать массовую привязку через PATCH, либо отвечать 400.

---

## F44 [Major] POST `/modifier-groups` тихо игнорирует `options[].price`

**Endpoint:** `POST /api/v1/admin/modifier-groups`

**Репро:**
```bash
curl -sS -X POST "$B/modifier-groups" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"name":"X","type":"group","min_amount":0,"max_amount":2,"options":[{"name":"A","price":10}]}'
# → 201 Created с options: [{name:"A",price:undefined,...}]
# → в /price-lists/{id}/modifier-items опция имеет price: 0
```

**Правильный endpoint:** `PATCH /catalog/price-lists/{id}/modifier-items` body `{items:[{modifier_option_id, price}]}`.

**Ожидание:** либо POST modifier-groups с options[].price реально записывает цену в default-прейскурант, либо 400 «Use price-list endpoint to set option prices».
**Факт:** 201, поле принято, но цена в БД остаётся 0. Аналогично F6a, F41 — третий случай тихого no-op.

**Где смотреть:** modifier-group create handler. Контракт с UI расходится: SPA шлёт options[].price, бэк принимает body, но не передаёт цены в price-list.

**Done when:** либо POST с options[].price записывает цены в default-прейскурант, либо 400.

---

## F45 [Minor] DELETE modifier-group оставляет orphan-записи в `price-list/modifier-items`

**Симптом:** После `DELETE /api/v1/admin/modifier-groups/{id}` соответствующие записи в `price-list/modifier-items` **не удаляются**. На стенде сейчас 8 orphan-записей от тестовых групп `[CHAIN-TEST]` (полный список ниже).

**Репро:**
```bash
# 1. Создать группу
GID=$(curl -sS -X POST "$B/modifier-groups" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"name":"test","type":"group","min_amount":0,"max_amount":1,"options":[{"name":"x","price":0}]}' \
  | jq -r .data.id)

# 2. Удалить
curl -sS -X DELETE "$B/modifier-groups/$GID" -H "$H"
# → 204

# 3. Проверить price-list
curl -sS "$B/catalog/price-lists/5dcc6666-.../items" -H "$H" \
  | jq '.data.modifier_items[] | select(.modifier_group_name == "test")'
# → запись всё ещё там
```

**Влияние:** Накопление orphan-данных в БД. На стенде уже 8 таких записей.

**Где смотреть:** `DELETE /modifier-groups/{id}` handler — добавить каскадное удаление из `price_list_modifier_items` (или эквивалент).

**Done when:** после DELETE modifier-group `/items.modifier_items` не содержит записей этой группы; стенд очищен от 8 orphan записей.

---

# Подтверждённые ✅ работающие сценарии (для контекста, не фиксить)

Эти кейсы прошли успешно — разработчику полезно знать что не сломано:

| Кейс | Что проверено |
|------|---------------|
| TC-CHAIN-018 | Изменение цены товара через `PATCH /price-lists/{id}/items` применяется к **новым** заказам, **старые** заказы сохраняют свою цену (денормализация) |
| TC-CHAIN-020 | Стоп товара: `POST /stop-lists/.../products` → товар скрывается с POS меню в течение секунд (после refresh) |
| TC-CHAIN-021 | Снятие со стопа: `DELETE` → товар возвращается на POS |
| TC-CHAIN-022/023 | Стоп категории и снятие — backend работает (UI-проверка заблокирована F40) |
| TC-CHAIN-070 | Денормализация **имени** товара: переименование в каталоге не меняет имя в существующих позициях заказов |
| TC-CHAIN-071 | Денормализация **цены**: изменение цены через price-list не меняет `unit_price` в существующих позициях |
| TC-CHAIN-013 | Привязка модификатора к товару через `POST /products/{id}/modifiers` |
| TC-CHAIN-014 (часть max) | На POS работает enforcement `max_amount` (3-я опция при max=2 не выбирается) |

---

# Ретракции из предыдущих файлов

| Что | Стало | Причина |
|-----|-------|---------|
| F38 (POS фильтрует товары без активных KDS на станции) | ❌ снято | Реальная причина — F40 (menu-availability category Кофе скрывала Эспрессо/Капучино/Латте) |
| F43 (POS не считает цену опций при показе total) | ❌ снято | Реальная причина — F44 (опция реально стоила 0 в БД, потому что POST modifier-groups проигнорировал price=10). После правильного PATCH цены через `/modifier-items` — POS UI показал 60 корректно |

---

# F42 [Minor / UX] Подсказка при достижении max-options

**Симптом:** При попытке выбрать 3-ю опцию модификатора при `max_amount=2` — POS блокирует клик молча, без подсказки «можно выбрать только 2».

**Не блокирует операцию**, но UX можно улучшить (toast / inline текст «выбрано 2 из 2»).

**Where:** POS UI — модификатор-выбор компонент.

**Done when:** при достижении max — показывается короткая подсказка.

---

# Что добавить в Tech debt стенда (к уже описанному в первом файле)

В дополнение к 4 пунктам в `BUGS-FOR-DEV.md`:

5. **8 orphan modifier_items** в default-прейскуранте (`price_list_id=5dcc6666-...`):
   ```
   [CHAIN-TEST] mod-max-lt-min / opt1
   [CHAIN-TEST] mod-neg-min / opt1
   [CHAIN-TEST] mod-neg-price / opt-neg
   [CHAIN-TEST] mod-huge-max / x
   [CHAIN-TEST] mod-valid / x
   [CHAIN-TEST] Тест-выбор / Опция A
   [CHAIN-TEST] Тест-выбор / Опция B (price: 10)
   [CHAIN-TEST] Тест-выбор / Опция C
   ```
   Удалить через DB после фикса F45.

---

# Что сказать Claude Code (дополнение к промпту)

> Также прочитай `./BUGS-FOR-DEV-stage3.md` — это третья итерация дополнений.
> 
> **Главное в stage3:**
> - В шапке файла — таблица **правильных endpoints** для управления каталогом. F6a/F41/F44 — это all silent no-op на одних и тех же эндпоинтах админ-API; правильные пути там же в таблице. Фикс может быть единым: либо реализация, либо явный 400 с подсказкой.
> - F40 — баг логики `menu-availability` (правило active = категория всегда скрыта, время игнорируется).
> - F45 — orphan-cleanup при DELETE modifier-group.
> - В разделе «Подтверждённые работающие сценарии» — список того что **не нужно** трогать: денормализация заказов работает, stop-list flow работает, привязка модификаторов через POST /products/{id}/modifiers работает.
> - В «Ретракциях» — F38 и F43 из предыдущих файлов **снимаются** (не баги).
