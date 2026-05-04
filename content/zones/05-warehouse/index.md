---
title: 05 Склад
---

# Склад

Ингредиенты, техкарты блюд, приёмки, списания, остатки.

> [!warning] Известно багов: 2 (1 Critical блокирует приёмки + 1 RBAC)

## Ингредиенты

- Справочник CRUD (см. также [Зона 04 — Каталог → Ингредиенты](/zones/04-catalog#ингредиенты-справочник))
- Единицы измерения с конвертацией

## Конвертация единиц

- Per-product / per-ingredient
- Например, "бутылка" → "мл" с фактором конвертации
- Поддержка кастомных единиц

## Техкарты блюд

- Состав ингредиентов с gross/net weight
- Cold loss / hot loss percentages
- Sort order
- CASCADE delete с tech card
- Связь либо с product (Catalog), либо с ingredient — взаимоисключаемы
- См. также [Зона 04 — Каталог → Техкарты](/zones/04-catalog#техкарты-блюд-per-product)

## Приёмки (Receipt Acts)

- Создание — статус draft
- Выбор ТТ → должен подгрузиться склад
- «Склад не найден для выбранной ТТ» — блокирует создание ([BUG-066](/04-known-bugs-index))
- Построчный ввод ингредиентов с количеством и unit price
- Расчёт total amount
- Подтверждение проводки (draft → posted)
- После posting — обновление stock_balances и создание stock_batches

## Списания (Write-off Acts)

- Создание — статус draft
- FIFO батчей при выборе ингредиента
- Причины: порча / собственные нужды / просрочка / другое
- Подтверждение проводки (draft → posted)
- После posting — обновление stock_balances

## Остатки (Stock Balances)

- Per warehouse / ingredient агрегаты
- Average cost (взвешенная цена)
- Auto-update при posting документов
- Просмотр текущих запасов с фильтрами

## Auto-creation склада при создании ТТ

- При создании ТТ должен автосоздаваться склад (через event `store.created`, BR 1.14)
- Один склад на ТТ
- BUG-066 может означать что склад не автосоздаётся → проверить наличие склада для каждой ТТ

## RBAC

- Все вкладки Склада → 403 у Manager ([BUG-016](/04-known-bugs-index))
- По спеке: Manager должен иметь доступ к Складу своей ТТ
- Cashier — 403 везде

---

**Связано:**
- Сервис: [Warehouse Service](https://nearbyerp.github.io/quartz-site/03-Services/Warehouse-Service/Overview) (port 3008)
- BR: 1.9 (Техкарты), 1.14 (Склад)
- Common: [01 RBAC Matrix](/01-rbac-matrix)
