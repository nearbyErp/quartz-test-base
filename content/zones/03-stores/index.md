---
title: 03 Торговые точки (ТТ)
---

# Торговые точки (ТТ)

CRUD ТТ, расписание работы, POS-терминалы (как объекты в админке), столы dine-in.

> [!warning] Известно багов: 11
> Подробности — inline ниже.

## ТТ — CRUD и фильтры

- Создание ТТ: имя, адрес, координаты, юрлицо, часовой пояс
- Редактирование (Franchise — без ограничений; Franchisee — свои; Manager — статус/график своей)
- Soft delete (только для unpublished)
- Suspend/Resume через статус ЮЛ — каскадно снимает с публикации
- Публикация / снятие с публикации
- Назначение прейскуранта к ТТ
- Снятие прейскуранта (возврат к default)
- UNIQUE (legal_entity_id, name) WHERE deleted_at IS NULL
- Поиск по адресу ([BUG-052](/04-known-bugs-index) — не работает)
- Фильтр статус — есть «Приостановлена», но функционала остановки нет ([BUG-058](/04-known-bugs-index))
- Фильтр по городу — требует 100% совпадение (не префикс) ([BUG-059](/04-known-bugs-index))
- Валидация полей Телефон/Адрес/Название — отсутствует ([BUG-054](/04-known-bugs-index))
- RBAC: создание ТТ для Franchisee → 403 «Only franchise role» ([BUG-015](/04-known-bugs-index))
- RBAC: Manager → ТТ возвращает 403 ([BUG-014](/04-known-bugs-index))

## График работы

- 7 дней недели (день=0 Пн ... день=6 Вс)
- Часы открытия/закрытия (HH:MM)
- Выходной (`is_closed=true`) — open/close должны быть NULL
- Работа через полночь (`close_time < open_time`) — допускается
- Чек-бокс «Одинаковый для всех дней» — должен делать колонку «Выходной» неактивной ([BUG-056](/04-known-bugs-index))
- Колонка «Круглосуточно» — отсутствует ([BUG-053](/04-known-bugs-index))
- UNIQUE (store_id, day_of_week)

## Часовые пояса

- IANA timezone (`Europe/Moscow` default)
- Магадан (UTC+11) — нужно добавить ([BUG-057](/04-known-bugs-index))
- Новосибирск (UTC+7) — убрать ([BUG-057](/04-known-bugs-index))

## Координаты

- Latitude (-90, 90) decimal(10,7)
- Longitude (-180, 180) decimal(10,7)
- Вставка координат из 2gis — должны заполняться оба поля одним действием ([BUG-055](/04-known-bugs-index))

## POS-терминалы (как объекты в админке)

- CRUD на ТТ
- Привязка по `fs_number` (заводской номер ФН), UNIQUE глобально
- `rn_kkt` (регистрационный номер ККТ) — опционально
- Статус active/inactive
- last_seen_at (heartbeat от устройства)
- Internal API: lookup terminal by ФН (для bind по ФН)
- Soft delete

## Столы (Dine-in)

- CRUD: number, label, capacity (default 4), position_x/position_y на canvas
- Статусы: free / occupied / reserved (CHECK `status IN`)
- UNIQUE (store_id, number) WHERE deleted_at IS NULL
- Назначение/освобождение стола (привязка `current_order_id`)
- Резервирование с `reserved_note` и `reserved_until`
- Cancel reservation
- Назначение официанта на стол (BR 3.2, поле `current_waiter_id`)
- Назначение официанта → 404 ([BUG-064](/04-known-bugs-index))
- При назначении показываются ВСЕ сотрудники, должны быть только официанты ТТ ([BUG-063](/04-known-bugs-index))
- Перемещение стола в карте зала → белая страница до reload ([BUG-065](/04-known-bugs-index))
- Internal API для POS Desktop (Phase 1)

---

**Связано:**
- Сервис: [Store Service](https://nearbyerp.github.io/quartz-site/03-Services/Store-Service/Overview) (port 3003)
- BR: 1.5 (POS-терминалы), 3.2 (Назначение официанта)
- Common: [01 RBAC Matrix](/01-rbac-matrix), [02 Form Validation](/02-form-validation-checklist)
