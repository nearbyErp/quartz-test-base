# Test Case Template

Формат портируется в Qase / TestRail / Notion / TestLink через CSV-импорт или копирование. Каждый test case — отдельная секция в файле, начинается с `## TC-...`.

## Шаблон одного кейса

```markdown
## TC-{T#}-{AREA}-{NNN} — {Краткое название}

**Type:** Smoke | Functional | Negative | Edge | Exploratory
**Priority:** P0 | P1 | P2 | P3
**Regression:** yes | no | conditional
**Roles:** Franchise | Franchisee | Manager | Cashier | All
**Linked bugs:** BUG-NNN, BUG-NNN | —
**Tags:** auth, crud, validation, ...

### Preconditions
- Войти под ролью {role}
- Существует {entity} с {properties}
- Перейти в {section}

### Steps
1. {Атомарное действие}
2. {Атомарное действие}
3. {Действие, вызывающее проверяемое поведение}

### Expected
- {Что должно произойти}
- {Какой response от API}
- {Какое состояние UI}

### Postconditions (опционально)
- {Что нужно откатить после кейса}
- {Что должно остаться в БД}
```

## Приоритеты

- **P0** — критический функционал, без которого невозможно работать (логин, создание ЮЛ, RBAC)
- **P1** — основной happy-path фич (CRUD сущностей, базовая валидация)
- **P2** — edge cases, негативные сценарии, нестандартные комбинации
- **P3** — UX/визуальные/малозначимые

## Типы

- **Smoke** — быстрая проверка «вообще живо ли»
- **Functional** — проверка по требованиям (happy path)
- **Negative** — проверка валидации и обработки ошибок
- **Edge** — граничные значения, нестандартные комбинации
- **Exploratory** — без формальных шагов, фиксируется в логе сессии

## Regression

- **yes** — попадает в каждый регрессионный прогон (P0 functional, RBAC checks, критичные negative)
- **conditional** — прогоняется только если затронута соответствующая область (по changelog)
- **no** — прогоняется один раз при первом тестировании фичи (exploratory, эстетические)

## Roles

Если кейс должен работать одинаково для всех ролей-владельцев — `Roles: All` + явно укажи в Preconditions сценарии scope-различий.
Если кейс специфичен для роли — указать роли через запятую.
Если кейс именно про RBAC негатив — `Roles: {роль которая НЕ должна иметь доступ}` + Type: Negative.

## Пример заполнения

```markdown
## TC-T1-AUTH-001 — Login by valid email/password

**Type:** Smoke
**Priority:** P0
**Regression:** yes
**Roles:** All
**Linked bugs:** —
**Tags:** auth, login

### Preconditions
- Существует активный сотрудник с email `franchise@test.local` / паролем `Test12345`
- Открыт `/login`

### Steps
1. Ввести `franchise@test.local` в поле Email
2. Ввести `Test12345` в поле Password
3. Нажать «Войти»

### Expected
- Redirect на `/dashboard`
- Cookie `access_token` установлена
- `GET /auth/me` возвращает 200 с `{ id, email, franchise_id, role_ids, permissions }`
- В шапке отображается имя сотрудника
```

## Группировка

Файлы тестера группируются по фичам (`01-authorization.md`, `02-franchises.md` и т.д.). Внутри файла кейсы идут от smoke → functional → negative → edge.

При импорте в TMS:
- Файл = Suite
- `## TC-...` = Test Case
- Поля до `### Steps` = Custom fields
