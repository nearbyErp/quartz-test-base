---
title: ERP Test Plans
---

# Test Plans — Obsidian ERP

Документация ручного тестирования. Markdown с структурой, совместимой с любым TMS (Qase / TestRail / Notion / TestLink) — при выборе TMS импортируется через CSV/копирование.

## Структура

```
test-plans/
├── README.md                          # этот файл
├── 00-templates/                      # шаблоны (НЕ копировать в TMS)
│   ├── bug-report-template.md         # для багов в Google Docs
│   └── test-case-template.md          # для новых test cases
├── 01-rbac-matrix.md                  # СКВОЗНАЯ матрица доступа (используется всеми)
├── 02-form-validation-checklist.md    # УНИВЕРСАЛЬНЫЙ чек-лист (применяется к каждой форме)
├── 03-smoke-pass.md                   # 30-мин smoke-круг перед глубоким тестированием
├── 04-known-bugs-index.md             # все известные баги, распределённые по зонам
└── testers/
    ├── T1-access-structure/           # Тестер 1: Авторизация, Франшизы, ЮЛ, Роли, Сотрудники-CRUD
    ├── T2-hr-and-catalog/             # Тестер 2: HR (смены/зарплата) + Каталог
    └── T3-operations/                 # Тестер 3: ТТ, Склад, Заказы, Aggregator
```

## Зоны ответственности

| Тестер | Зона | Известно багов |
|--------|------|----------------|
| T1 | Доступы и структура | ~17 |
| T2 | HR + Каталог | ~22 |
| T3 | Операционный поток | ~26 |

Полное распределение — в [04-known-bugs-index.md](04-known-bugs-index.md).

## Соглашения по ID

| Сущность | Формат | Пример |
|----------|--------|--------|
| Test case | `TC-{T#}-{AREA}-{NNN}` | `TC-T1-AUTH-001` |
| Bug | `BUG-{NNN}` (3 цифры) | `BUG-042` |
| Test run | `RUN-{YYYY-MM-DD}-{N}` | `RUN-2026-05-04-1` |

**AREA-коды:**
- T1: `AUTH`, `FRAN`, `LE` (Legal Entities), `ROLE`, `EMP`
- T2: `SHIFT`, `SCHED`, `TIME`, `PAYR`, `ACT` (Activity), `PROD`, `CAT` (Categories), `MOD` (Modifiers), `STOP`, `TECH` (Tech Cards), `PRICE`
- T3: `STORE`, `TERM` (Terminals), `TABLE`, `MENU`, `WH` (Warehouse), `RECV` (Receipts), `WO` (Write-offs), `INGR`, `ORD`, `AGG`

## Workflow тестировщика

1. **Перед циклом:** прогнать `03-smoke-pass.md` — если упало, дальше не идти, репортить blocker
2. **Открыть свою папку** `testers/T{N}-*/` и брать пакеты по порядку (по приоритету P0 → P1 → P2)
3. **При каждой форме** — обязательно прогнать `02-form-validation-checklist.md`
4. **При каждом разделе** — сверяться с `01-rbac-matrix.md` (RBAC баги = регрессия №1 по объёму)
5. **Перед заведением бага** — поиск по `04-known-bugs-index.md` (избегать дублей)
6. **Заводить баги** в Google Docs по шаблону `00-templates/bug-report-template.md` с присвоением `BUG-NNN`
7. **Линковать в test case** в поле `Linked bugs: BUG-NNN`

## Регрессионный пул

При выпуске новой версии:
1. **Smoke pass** (30 мин, любой тестер)
2. **RBAC matrix** прогон (T1, ~1 час)
3. **Полный регресс по своей зоне** каждым тестером — все P0 + P1 кейсы
4. **P2 кейсы** только если есть время или затронутая фича

Test cases помечены тегом `Regression: yes/no/conditional`. По умолчанию все functional happy-path = `yes`, edge cases = `conditional`, exploratory = `no`.

## Тестовый стенд

- **Хост:** `erp-test.nirbi.ru`
- **Деплой:** ручной через SSH, без CI/CD (см. `D:\Project\ERP\content-mirror\06-DevOps\Deployment-Runbook.md`)
- **Учётки** для всех 4 ролей: получить у тест-лида
- **БД:** PostgreSQL, 12 БД (по сервисам). Сброс данных — через тест-лида
