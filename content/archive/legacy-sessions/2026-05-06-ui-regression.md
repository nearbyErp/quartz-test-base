---
date: 2026-05-06
type: ui-regression
tester: Claude (Playwright под учёткой demo3@nirbi.ru)
duration_h: 0.5
scope: проверка user-facing проявлений F4, F16, F26, F45 через клики в SPA
stand: https://erp-test.nirbi.ru
---

# UI-регресс 2026-05-06 — реальный пользовательский сценарий

После API-регресса нужно было проверить, что из открытых багов реально видит пользователь в админ-SPA. Использовал **свежую учётку demo3@nirbi.ru** (отдельная франшиза `a0c0ffee-...3`, 0 ТТ, 0 заказов) — чтобы не трогать `admin@erp.local`, которая идёт на демо.

## Главный вывод

Из 8 «open» багов после API-регресса **3 не воспроизводятся в реальном UI**:

- **F4** (6 admin-routes 404) — это были не SPA-роуты, а API-only пути (`/admin/tables`, `/admin/warehouse` без префикса). В SPA реальные роуты — `/admin/warehouse/inventory`, `/admin/warehouse/receipt-acts` и т.д., и они все работают. → **retracted**
- **F32** (отрицательная цена опции модификатора) — в форме создания модификатора **поля `price` нет вообще**. Пользователь физически не может ввести отрицательную цену. Цена опций живёт в прейскуранте (отдельный экран). → **retracted**
- **F44** (POST modifier-groups игнорит options.price) — то же что F32: UI просто не отдаёт это поле. → **retracted**
- **F45** (DELETE оставляет orphan) — оказался **by-design soft-delete**. UI явно сообщает «Группа будет перемещена в удалённые», есть вкладки «Активные/Удалённые», `?deleted=true` корректно отдаёт удалённое. → **retracted**

**Чисто API-баги (silent-accept)** — F6a, F41, F44, F46 — это технический долг, который ловит только тот, кто пишет интеграции через API в обход UI. Реальный пользователь админки на них не наткнётся.

## Найдено три новых UI-бага

### F47 — после удаления последнего модификатора пропадают вкладки 🟢 Minor
**Сценарий**: пользователь имеет одну группу модификаторов → удаляет её через «...» → «Удалить» → подтверждает → список «Активные» становится пустым. Вместо empty-state с сохранёнными вкладками — отрисовывается главный empty-screen «Модификаторы пока не добавлены» БЕЗ вкладок «Активные/Удалённые». **Доступа к удалённым из UI больше нет.**

### F48 — Dashboard: «Роль: —» и пустой Franchise ID 🟢 Minor
**Сценарий**: после логина пользователь видит карточку «Информация о пользователе». При этом `/auth/me` возвращает `roles: [{id: ...}]` и `franchise.id`, но UI показывает «Роль: —» и оставляет «Franchise ID:» пустым. Скорее всего, UI ждёт `role.name` (которое не отдаётся), вместо того чтобы догрузить роль по id.

### F49 — `/admin/warehouse` редиректит молча 🟢 Minor
**Сценарий**: пользователь вбивает в адрес `/admin/warehouse` → 1 сек «Загрузка...» → редирект на dashboard без какого-либо сообщения. Аналогично для `/admin/finance`, `/admin/catalog`, `/admin/orders` (без подраздела). Пользовательски-непредсказуемо. Лучше показывать первый подраздел или «Выберите подраздел».

## Карта SPA-роутов сайдбара (29 шт., 0 редиректов)

Все sidebar-пункты у demo3 (полные права) живые:

- **Корневые**: Dashboard, Юридические лица, Торговые точки
- **Сотрудники**: /employees, /roles, /schedule, /schedule/templates, /activity/employees, /activity/terminals, /payroll/formulas, /payroll
- **Каталог**: /catalog/products, /catalog/modifiers, /catalog/categories, /catalog/price-lists, /catalog/time-tariffs, /catalog/menu-availability, /external-menus *(не /catalog/!)*, /catalog/ingredients, /catalog/stop-lists, /catalog/kitchen-stations
- **Склад**: /warehouse/inventory, /warehouse/receipt-acts, /warehouse/write-off-acts
- **Заказы**: /orders/active, /orders/history, /kitchen, /transactions, /shifts/monitor
- **Финансы**: /tips *(только Чаевые)*
- **Настройки KDS**: /kds-settings
- **Устройства**: /devices

## Что осталось

- **F26 (RRN/card_last4 в UI)** — не проверено. На demo3 нет заказов; для проверки нужен или admin@erp.local в чтении (не хочу трогать без согласования), или создать заказ + оплату на demo3 (длинный сценарий, нужны Александр + ТТ + товары).
- **F1, F6a, F6b, F41, F46** — чистый API silent-accept, реальному пользователю не виден.
- **F40** — нужен POS-desktop с Александр.
- **F2, F7, F16-warehouse-functional, F39** — требуют наполнения данных или уточнения у разраба.

## Артефакты

Скриншоты лежат в [`../screenshots/2026-05-06-ui-regression/`](../screenshots/2026-05-06-ui-regression/) (см. README в той папке для маппинга на F-NN).

Привязанные к багам:
- `F32-F44-modifier-form-no-price-field.png` — нет поля цены опции (→ retract F32, F44)
- `F45-soft-delete-confirmation-dialog.png` — диалог «перемещена в удалённые» (→ retract F45)
- `F47-tabs-disappear-after-last-delete.png` — исчезновение вкладок (F47, новый)
- `F48-dashboard-empty-role-and-franchise.png` — Роль/Franchise ID пустые (F48, новый)
- `F49-warehouse-silent-loading.png` + `F49-warehouse-redirect-to-dashboard.png` — молчаливый редирект (F49, новый)

Контекстные:
- `context-stop-lists-loading.png`, `context-stop-lists-store-chooser.png`
- `context-modifiers-empty-state.png`, `context-modifiers-list-with-test-item.png`, `context-modifier-actions-menu.png`

## Итог сессии

- Открытых user-facing багов после UI-проверки: **F16 (warehouse функционал не виден без выбора ТТ + молчаливый редирект → F49), F26 (отложено), F40 (POS), F25/F27/F37/F42 (POS).**
- Новые UI-баги: **F47, F48, F49**.
- Retracted: **F4, F32, F44, F45**.
- 13 fixed после API-регресса остаются fixed.
