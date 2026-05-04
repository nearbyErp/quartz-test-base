---
title: 08 KDS-flow и интеграции (Кухня)
---

# KDS-flow и интеграции (Кухня)

Кухонные станции, KDS-настройки, отображение заказов на кухне, связки админка↔KDS↔POS, агрегаторы, PayKeeper, Честный знак.

> [!warning] Известно багов: 0 в KDS напрямую
> Зона exploratory. Часть багов про Меню в карточке ТТ — в [Зоне 04 — Каталог](/zones/04-catalog#меню-в-карточке-тт-вычисляемое).

## Кухонные станции

- CRUD: «Hot Kitchen», «Bar», «Cold», «Grill» и т.д.
- yellow_threshold_minutes, red_threshold_minutes (color по timing)
- Привязка товара к станции через `kitchen_station_id` (см. [Зона 04 — Каталог → Товары → поля](/zones/04-catalog#товары--поля))
- Связка `requires_kitchen=true` ↔ `kitchen_station_id` обязательна

## KDS-настройки франшизы (`kds_franchise_settings`)

- Звуки, громкость, repeat_intervals
- Auto-logout после N минут неактивности
- Auto-create при первом запросе с defaults

## KDS PIN-логин (Phase 2, BR 5.1)

- PIN сотрудника с `kds.access` permission
- Сессия на устройстве
- Force logout при revoke устройства

## Регистрация KDS-устройств (Phase 2, BR 5.1)

- POST /admin/kds/devices/register с `device_id` (UUID на устройстве)
- Привязка к ТТ
- name (default = `KDS-{first 6 hex of device_id}`)
- last_user_id, current_user_id, app_version, last_seen_at
- Soft-delete через `revoked_at`
- Permission: `kds.settings.edit`

## KDS-flow приготовления

- Отображение заказов на станции (фильтр по `kitchen_station_id`)
- Только товары с `requires_kitchen=true` идут на KDS (good-товары не отображаются)
- Детали позиции: товар + модификаторы + количество
- Изменение статуса позиции: «принят» / «готов»
- Цвет ячейки по timing: нейтральный → yellow (после `yellow_threshold_minutes`) → red (после `red_threshold_minutes`)
- Звуковое уведомление о новом заказе
- Heartbeat каждые ~5 секунд

## Force logout

- Revoke device → событие `user.kds_device.revoked` (BR 5.1)
- pos-bff consumer завершает WebSocket-сессии
- Deactivate сотрудника → 401 на следующее действие
- На следующем heartbeat (≤2 мин) — выкидывает из приложения с сообщением «устройство деактивировано»

## Изменение настроек на лету

- Событие `catalog.kds_settings.updated` (Catalog Service publisher)
- В P0 не реализован consumer — KDS делает re-pull при «Применить»
- В P1 — pos-bff broadcast в WebSocket KDS-подписчикам
- kind=settings → re-pull `GET /admin/kds/settings`
- kind=station_thresholds → re-pull `GET /kitchen-stations`

## Интеграция Админка ↔ KDS ↔ POS

15 интеграционных сценариев — см. **[Сценарий: Админка ↔ KDS ↔ POS](/scenarios/admin-kds-pos)**.

Ключевые проверки:
- Полный happy path заказа с приготовлением (TC-INT-001)
- Stop-list propagation на POS (TC-INT-003)
- Multi-station routing — Бургер → Hot Kitchen, Капучино → Bar (TC-INT-004)
- Заказ с модификаторами на KDS (TC-INT-005)
- Отмена заказа во время приготовления (TC-INT-006)
- Изменение каталога во время активной смены (TC-INT-007)
- KDS device revoke и force logout (TC-INT-008)
- Изменение KDS-настроек на лету (TC-INT-009)
- Изменение порогов цвета (TC-INT-010)
- Network failure recovery (TC-INT-011)
- RBAC между POS и KDS — pos.access vs kds.access (TC-INT-012)

## Aggregator integrations (Я.Еда / МД)

- Bindings ТТ ↔ внешний агрегатор (OAuth credentials)
- Webhook-приём заказов (`POST /webhook/...`)
- Push статусов в агрегатор (`StatusChangedConsumer` потребляет `order.status.changed`)
- Connector Я.Еда — stub (M2 skeleton, ждёт контракт)
- Connector Маркет Деливери — stub
- Topics: `aggregator.orders`, `aggregator.bindings`, `aggregator.status.sync`
- M3 (планируется): consumers `catalog.product.updated`, `warehouse.product.stock.depleted` для авто-инвалидации snapshot

## PayKeeper интеграция (BR 3.3, 3.4, 3.5)

- Подключение ЛК PK к юрлицу: host (`{tsp}.server.paykeeper.ru`), login, password, informer_seed
- Permission `integrations.manage` для управления подключением
- Один ЛК PK = одно юрлицо
- Адаптер: создание invoice, фискализация
- Catalog sync ERP → PayKeeper (transactional outbox `catalog_outbox`)
- 6 топиков изменений каталога: `catalog.product.{upserted,deleted}`, `catalog.category.{upserted,deleted}`, `catalog.modifier_group.{upserted,deleted}`
- Импорт сотрудников из PayKeeper (BR 3.5) — pull-импорт через wizard

## Честный знак (маркировка)

- POS-flow с маркированными товарами (бутылка воды, молоко, пиво)
- Сканирование DataMatrix — productCode тег 1163 в чеке
- ФЯ через API012 (`startlabelschecksession.json`, `labelcheck.json`, `receipt.json`)
- ФЯ → ОИСМ напрямую (мы не ходим в Честный знак при продаже)
- Marking Service для приготовления и приёмки (УКЭП через CryptoPro)
- Тип кода: 1304 (GS1 DataMatrix)
- Лимит 128 маркированных позиций на чек
- Не для MVP, но в roadmap

---

**Связано:**
- BR: [5.1 KDS](https://nearbyerp.github.io/quartz-site/...), [3.3 PayKeeper P0 Pilot](...), [3.4 Catalog Sync](...), [3.5 Импорт сотрудников из PK](...), [4.2 Aggregator Phase 2](...)
- ADR: [KDS Phase 2 Hardware Research](https://nearbyerp.github.io/quartz-site/01-Architecture/KDS-Phase-2-Hardware-Architecture-Research)
- Сценарии: [Админка ↔ KDS ↔ POS](/scenarios/admin-kds-pos)
