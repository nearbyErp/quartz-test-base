---
title: ERP Test Base
---

# ERP — Тестовая база

Рабочее пространство тестирования цепочки Admin → POS → KDS платформы Obsidian ERP.

## Быстрый доступ

| Куда | Зачем |
|------|-------|
| **[CONTEXT.md](CONTEXT.md)** | **Главный living-doc — стенды, наполнение, блокеры, хронология сессий.** Читать первым |
| [findings.md](findings.md) | Все известные баги (BUG-NNN backlog + F-NN наши, единая таблица) |
| [todo.md](todo.md) | План дальнейшего |
| [zones/](zones/) | Тест-кейсы по доменам приложения |
| [scenarios/](scenarios/) | Сквозные e2e сценарии (ссылаются на zones) |
| [checklists/](checklists/) | Универсальные чек-листы (smoke, form-validation) |
| [reference/stand.md](reference/stand.md) | URL, креды, тестовые id, helper-curl |
| [reference/endpoints.md](reference/endpoints.md) | Карта реальных API endpoints |
| [templates/](templates/) | Шаблоны новых TC, багов, сессий |
| [screenshots/](screenshots/) | Скриншоты с прогонов, по папкам `<session-id>/` |
| [archive/dev-handoffs/](archive/dev-handoffs/) | Узкоцелевые документы для разработчика (KDS-blocker, POS-onboarding и т.п.) |
| [archive/legacy-sessions/](archive/legacy-sessions/) | Старые sessions/* и SUMMARY-* (frozen, для истории) |

## Структура

```
test-base/
├── index.md                 ← навигация (ты здесь)
├── CONTEXT.md               ← главный living-doc (стенды, состояние, хронология)
├── findings.md              ← все баги
├── todo.md                  ← план дальнейшего
├── zones/                   ← тест-кейсы по доменам (catalog/orders/payments/kds/pos/auth-rbac)
├── scenarios/               ← e2e сценарии
├── checklists/              ← универсальные (smoke, form-validation)
├── reference/               ← стенд + endpoints
├── templates/               ← шаблоны
├── screenshots/             ← скриншоты прогонов
└── archive/
    ├── dev-handoffs/        ← узкоцелевые документы для разраба
    └── legacy-sessions/     ← старые sessions/* и SUMMARY-*
```

## Как мы работаем

### Начало сессии
1. **Открыть `CONTEXT.md`** — посмотреть актуальное состояние стенда + активные блокеры + открытые TODO.
2. По ходу прогона — пополнять секцию текущей сессии в `CONTEXT.md` плотными таблицами «# / Задача / Статус».
3. Скриншоты — `screenshots/<YYYY-MM-DD>/F<NN>-<desc>.png`.

### Найденный баг
1. Присвоить следующий F-NN.
2. Добавить строку в `findings.md` (severity, status=open, краткое описание, zone, source).
3. В `CONTEXT.md` в текущей сессии — короткая запись + ссылка на findings.

### Узкоцелевой документ для разраба
- Когда находка требует подробного контекста (логи, гипотезы, что проверить) → отдельный файл в `archive/dev-handoffs/`. Ссылается из `CONTEXT.md`.

### Регресс по фиксу
1. Найти баг в `findings.md` — поставить `verify-needed`.
2. Прогнать соответствующий TC из `zones/<zone>/<feature>.md` или просто воспроизвести.
3. Фикс работает → `fixed`. Не работает → запись в CONTEXT сессии.

## Конвенции

### ID
- **`F-NN`** — наши e2e-сессионные. Следующий: `F104`.
- **`BUG-NNN`** — backlog regression-баги (legacy).
- **Сессии в CONTEXT** — секция `## Сессия YYYY-MM-DD — <тема>`.

### Severity
- 🔴 **Critical** — блокирует операцию, потеря данных, security-leak.
- 🟡 **Major** — функциональный баг, missing feature, важное расхождение со спекой.
- 🟢 **Minor** — UX, валидация без блокера, doc-rot.
- ⚪ **Info / by-design** — наблюдение, не баг.

### Status
- `open` — нашли, не фиксили.
- `in-progress` — разработчик в работе.
- `verify-needed` — фикс задеплоен, нужен прогон регресса.
- `fixed` — подтверждён фикс.
- `retracted` — отозван (не баг при детальном разборе).

## Текущий статус (на 2026-05-12)

- **103 finding** (F76–F103 за сегодня + F1-F75 в legacy).
- **2 активных блокера**: F102/F103 (PATCH employees 500), F78/F100 (PayKeeper-adapter).
- Стенд PROD/ТТ Smoke 001 наполнен для POS/KDS happy-path (см. CONTEXT).
- Что в плане далее: POS/KDS happy-path → 2026-05-13.
