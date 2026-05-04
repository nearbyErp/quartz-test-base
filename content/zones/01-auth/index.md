---
title: 01 Авторизация и доступ
---

# Авторизация и доступ

Точка входа в систему: login, сессии, JWT, RBAC, permissions.

> [!info] Известно багов в зоне напрямую: 0
> Связанные RBAC-баги принадлежат другим зонам ([BUG-013…017](/04-known-bugs-index)) — туда «прилетают» 403, но природа в auth/scope-логике.

## Логин и сессии

- Login по email + паролю — успех (JWT в cookie, redirect на dashboard)
- Login — неверный пароль → ошибка «Неверный логин или пароль»
- Login — несуществующий email → та же ошибка (не раскрывает существование аккаунта)
- Account lockout — 5 неудачных попыток подряд → блок 15 минут (`locked_until`)
- Logout — инвалидация refresh token, redirect на /login
- Refresh token rotation — старый помечается `revoked=true`, новый выдаётся
- Параллельные сессии (один user в двух браузерах) — обе работают
- Истечение access token (15 мин default) — auto-refresh через refresh token
- Истечение refresh token (30 дней default) — выкидывает на /login

## Управление паролями

- Forgot password — отправка email-ссылки через SMTP (Selectel, ADR-003)
- Forgot password для несуществующего email — возвращает 200 (не раскрывает)
- Reset password — single-use токен, TTL 1 час
- Reset password — повторное использование use'нутого токена → ошибка
- Reset password — устаревший токен (>1 час) → ошибка
- Изменение пароля — bcrypt cost 12

## RBAC и permissions

См. [01 RBAC Matrix](/01-rbac-matrix) — основной regression-артефакт.

- Permission catalog: 17 секций бэк-офиса × Read/Edit + 32 POS-операции + KDS-permissions (BR 5.1)
- Edit implies Read — включение Edit включает Read автоматом
- Снятие Read автоматически снимает Edit
- 4 роли — Franchise / Franchisee / Manager / Cashier (для ИП-сценария Franchisee = N/A)
- Минимум permissions владельца партнёра (нельзя снять): `pos.access`, `stores.read`, `employees.read`
- Системная роль «Администратор» — полный набор permissions, нельзя удалить

## JWT и API-валидация

- /auth/me возвращает корректные permissions и scope (cached 60 сек в Redis)
- JWT payload: `sub`, `franchise_id`, `role_ids` (без `role` enum, BR 1.4.4)
- Старые JWT с полем `role` — игнорируются (backward compat)
- POST /internal/auth/validate — service-to-service, Redis cache
- Scope резолюция per BR 1.4.4: владелец `franchise` ЮЛ → вся франшиза; владелец `franchisee` → свои ЮЛ + ТТ; иначе — `employee_role_stores`

## POS PIN-логин (вызывается из зоны 07)

- POST /auth/pin-login — PIN + store_id
- Проверка `pos.access` permission
- Validate-pin endpoint в User Service
- Коллизия PIN решается опциональным employee_id

---

**Связано:**
- Сервис: [Auth Service](https://nearbyerp.github.io/quartz-site/03-Services/Auth-Service/Overview) (port 3001)
- ADR: [ADR-001 локальный JWT-фильтр на этапе MVP](https://nearbyerp.github.io/quartz-site/02-ADR/ADR-001-Локальный-JWT-фильтр-вместо-Auth-Service-на-этапе-MVP), [ADR-003 SMTP Selectel](https://nearbyerp.github.io/quartz-site/02-ADR/ADR-003-Email-через-Resend)
- Common: [01 RBAC Matrix](/01-rbac-matrix), [02 Form Validation](/02-form-validation-checklist)
