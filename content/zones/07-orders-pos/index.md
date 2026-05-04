---
title: 07 Заказы и операции (Касса/POS)
---

# Заказы и операции (Касса/POS)

POS-flow кассира, lifecycle заказа, оплата, X/Z отчёты, столы dine-in на POS.

> [!warning] Известно багов: 1 RBAC + большая exploratory часть
> Покрытие POS-flow в базе минимально — это твоя зона активного поиска.

## POS PIN-логин кассира

- PIN-логин на терминале (4 цифры)
- POST /auth/pin-login с PIN + store_id
- Проверка `pos.access` permission
- Validate-pin endpoint в User Service
- Сессия после логина (JWT)

## Открытие смены

- Через ФЯ (фискальное ядро K10) — `cycleopen.json` API012
- X-отчёт пустой
- Фискальный модуль готов
- `cycleIsOpen` status field — операции блокируются до открытия

## Создание заказа

- Auto-номер per-store per-day («001», «002»...)
- Сброс счётчика ежедневно
- Добавление позиций (только в статусе `new`)
- Выбор модификаторов при добавлении (с min/max от группы)
- Применение прейскуранта ТТ (актуальная цена с момента создания позиции)
- Применение стоп-листа — товар недоступен в меню кассиру (см. [Зона 04 → Стоп-листы](/zones/04-catalog#стоп-листы-per-store))
- Денормализация product_name, unit_price, модификаторов в позиции
- Удаление позиций (только в `new`)
- Пересчёт total автоматически
- Заказ ассоциируется со столом (если dine-in)

## Оплата

- Способы: cash / card (наличные / эквайринг)
- Card_last4, RRN от платёжного шлюза
- Чек печатается через ФЯ (API012 receipt.json)
- Фискализация: ФН → ОФД → ФНС/ОИСМ
- Денежные значения как строки (`"455.00"`), не числа
- После оплаты заказ → `ready` (если только good-товары) или `in_progress` (если есть `requires_kitchen=true`)

## Заказ — статусная машина

- `new` (создан) → `in_progress` (после оплаты, есть приготовление) → `ready` (KDS отметил готов) → `closed` (выдан)
- `new` → `ready` (после оплаты, нет приготовления — все товары `requires_kitchen=false`)
- `new` / `in_progress` / `ready` → `cancelled` (с причиной)
- `closed` нельзя отменить (только refund — Phase 2)

## Отмена заказа

- Из new / in_progress / ready
- Обязательная причина (`cancel_reason`)
- Поля `cancel_reason`, `cancelled_at`
- Cashier — отмена своего; Manager — на своей ТТ; Franchise — любую

## Возвраты (Refunds)

- Refund после `closed` — Phase 2 (не реализован)

## Закрытие смены

- Через ФЯ — `cycleclose.json`
- Z-отчёт (фискальный документ)
- Сверка наличных
- Передача наличных по итогу X-отчёта
- Bank settlement (опции по эквайрингу)
- Auto-close смены через 24+ часов

## Заказы в админке

- Список с пагинацией, фильтрами (ТТ, статус, дата_от, дата_до)
- Детали заказа с позициями и оплатой
- Отмена админом / Manager
- Фильтрация ролевая: Franchise — все ТТ; Franchisee — свои; Manager — своя; Cashier — своя
- Manager → Заказы возвращает 403 ([BUG-017](/04-known-bugs-index))

## X/Z отчёты в админке (Shift Report)

- BR 2.2: внутренний `GET /internal/orders/shift-report` с агрегацией по `paid_at`
- Финансовая сводка смены: выручка, разбивка по способам оплаты, количество заказов, топ товаров
- CSV-экспорт (см. также [Зона 06 → Ведомости](/zones/06-hr#ведомости-payroll-records))

## Shift Monitor

- Авто-refresh 30 секунд
- Статусы открытых смен: «скоро 24h», «Z!»
- Live через Kafka (события `shift.closed`)

## Transaction Journal

- UI есть в админке (`TransactionJournalPage.tsx`)
- Backend `/api/v1/admin/transactions` → 404 (не реализован)

## Столы на POS (dine-in)

- Привязка стола к заказу при создании
- Перенос позиций между столами (split bill)
- Освобождение стола при закрытии заказа
- Notification при изменении статуса (через `order.status.changed` event)
- Consumer `OrderStatusConsumer` синхронизирует zal_tables

---

**Связано:**
- Сервисы: [Order Service](https://nearbyerp.github.io/quartz-site/03-Services/Order-Service/Overview) (port 3005), POS BFF (port 3022), ФЯ через API012
- Дев-доки: [Fiscal Core Integration](https://nearbyerp.github.io/quartz-site/08-Specs/POS/Fiscal-Core-Integration), [Honest Mark Integration](https://nearbyerp.github.io/quartz-site/01-Architecture/Honest-Mark-Integration)
- BR: 2.1 (Заказы), 2.2 (Shift Report)
- Сценарии: [Админка ↔ KDS ↔ POS](/scenarios/admin-kds-pos) — 15 интеграционных кейсов
