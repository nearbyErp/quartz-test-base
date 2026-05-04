---
title: 06 HR-цикл (расписание, зарплата)
---

# HR-цикл (расписание, зарплата)

Шаблоны смен, расписание, time tracking, формулы зарплаты, ведомости, активность сотрудников.

> [!warning] Известно багов: 11 + 1 cross-cutting (X03)
> 3 Critical, 4 Major, 4 Minor.

## Шаблоны смен

- CRUD: name, start_time, duration_minutes
- Лимит 4 шаблона на ТТ (проверка на уровне приложения)
- Редактирование шаблона — 500 при сохранении ([BUG-018](/04-known-bugs-index), [BUG-019](/04-known-bugs-index))

## Расписание (плановые смены)

- CRUD только future-only (нельзя на прошлые даты)
- Назначение из шаблона (template_id)
- UNIQUE (employee_id, store_id, date)
- Создание смены «Вручную» (без шаблона) → 422 ([BUG-020](/04-known-bugs-index))
- Страница расписания выглядит неполноценно — требуется доработка UX ([BUG-022](/04-known-bugs-index))

## Time tracking (фактические смены)

- Clock in/out на POS — отсутствует ([BUG-021](/04-known-bugs-index)) — feature gap
- Ручной ввод плановой смены — единственный способ сейчас
- Корректировки: increase/decrease с обязательным комментарием
- Status: `on_schedule` / `off_schedule` / `missed` / `unplanned` (вычисляется)
- Auto-close через 24+ часов (`auto_closed = true`)
- `break_duration_minutes` = `break_end - break_start`
- Date = дата clock_in (для ночных смен — день начала)

## Формулы зарплаты

- Hourly / fixed / mixed
- По роли (default) / индивидуальные (перекрывают ролевую)
- Иерархия: индивидуальная > ролевая > нет
- UNIQUE (role_id) WHERE employee_id IS NULL — одна формула на роль
- UNIQUE (employee_id) WHERE NOT NULL — одна индивидуальная
- Per-ТТ ставки (store_id) — deferred (всегда NULL в MVP)
- Создание формулы по роли с Save → 500 ([BUG-023](/04-known-bugs-index))
- Поле «Ставка в час» — отображается технический код рядом с label ([BUG-026](/04-known-bugs-index), [BUG-X03](/04-known-bugs-index))
- Поле «Ставка в час» — принимает отрицательные значения (-1) ([BUG-027](/04-known-bugs-index))

## Ведомости (Payroll Records)

- Расчёт за период (1 месяц)
- Статусы: calculated → confirmed → paid (forward only, нельзя откатить)
- formula_snapshot — JSON на момент расчёта (не обновляется при изменении формулы)
- UNIQUE (employee_id, store_id, period_start)
- Подтверждение ведомости с ошибкой — несмотря на 400 разрешает ([BUG-025](/04-known-bugs-index))
- Экспорт CSV — 422, файл не выгружается ([BUG-024](/04-known-bugs-index))
- Колонка «Действия» — отображается технический код ([BUG-028](/04-known-bugs-index), [BUG-X03](/04-known-bugs-index))

## Активность сотрудников (Dashboard)

- Дашборд по часам, начислениям, отработанным сменам
- Live через Kafka shiftMonitor (Terminals dashboard)
- Страница «Терминалы» — переосмыслить или удалить ([BUG-029](/04-known-bugs-index))
- Фильтр ролей в списке сотрудников — «Все роли (permissions)» — убрать слово «permissions» ([BUG-X03](/04-known-bugs-index))

## Юридические детали сотрудника (расширение)

- 1:1 с employee
- ИНН (12 цифр), паспорт (серия/номер), ВУ (номер/срок), СНИЛС (XXX-XXX-XXX XX)
- Все поля nullable
- Доступ только владельцам (Franchise / Franchisee их сотрудников)
- ФЗ-152 чувствительность

---

**Связано:**
- Сервис: [User Service — HR-расширение](https://nearbyerp.github.io/quartz-site/03-Services/User-Service/Overview) (port 3002)
- BR: 1.4.1 (расписание, учёт времени, зарплата)
- Common: [02 Form Validation](/02-form-validation-checklist) — у каждой формы
