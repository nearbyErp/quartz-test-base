# POS/KDS onboarding на проде — блокер

> Контекст для разработчика (и его Claude Code).
> Дата: 2026-05-12. Стенд: **admin.nirbi.ru** (PROD).
> Учётка тестирования в админ-панели: `admin@erp.local / <password — см. private/creds.md>`.

## TL;DR

На проде сломан весь модуль регистрации устройств (POS + KDS). Конечный пользователь не может довести ни POS, ни KDS до рабочего состояния.

---

## Симптомы

### 1. POS-устройство при регистрации
**Сценарий:** запустили POS-Desktop, ввели URL франшизы → email/password менеджера → выбрали ТТ «ТТ Smoke 001» → попытка зарегистрировать кассу.
**Идентификатор устройства, который POS отправляет:** `POS-814800`.
**Результат:** на экране POS появляется только текст **"Internal Server Error"** без request_id, кода, или другой детали.
**ФН на этом шаге не вводится** — это первичная регистрация устройства, не привязка фискального накопителя.

### 2. KDS-устройство при логине
**Сценарий:** запустили KDS-приложение, ввели реквизиты.
**Результат:** на экране ошибка `Route not found: /api/v1/admin/auth/login`.

### 3. Админ-панель `/admin/devices` (browser, под admin@erp.local)
- Вкладка «POS-устройства» → `GET /api/v1/admin/pos/devices` → **500**
- Вкладка «KDS-устройства» → `GET /api/v1/admin/kds/devices` → **500**
- UI обоих табов скрывает 500 за пустыми списками "Нет зарегистрированных устройств" — оператор не понимает, что бэк лежит.

---

## Что уже пробовали

1. Разраб (Сано) накатил `erp-pos` (POS-bff + POS-frontend desktop).
   - После наката `/api/v1/admin/pos/devices` всё ещё **500**.
   - Регистрация POS на устройстве всё ещё **Internal Server Error**.

2. Сано подтвердил: `erp-admin` уже накатан, **это чисто фронтенд** (admin-frontend), не bff.

3. Сано отметил: на логине POS тоже 500 (видит на бэке).

4. Логин **в админ-панель через браузер работает** под `admin@erp.local` — значит admin-auth endpoint жив. У POS, видимо, другой login endpoint (например `/api/v1/pos/auth/login` или `/api/v1/admin/auth/login`, который ему 500 отдаёт).

---

## Что известно про архитектуру (из локальной документации, может устарела)

| Service | Repo | Port (test VPS) |
|---|---|---|
| auth-service | erp-auth-service | 3001 |
| user-service | erp-user-service | 3002 |
| store-service | erp-store-service | 3003 |
| catalog-service | erp-catalog-service | 3004 |
| order-service | erp-order-service | 3005 |
| warehouse-service | erp-warehouse-service | 3008 |
| aggregator-service | erp-aggregator-service | 3013 |
| admin-bff / admin-frontend | erp-admin | 3020 / 80 |
| pos-bff / pos-frontend | erp-pos / erp-pos-desktop | 3022 / 80 |

Источник: `content-mirror/06-DevOps/Deployment-Runbook.md` (8 дней назад).

⚠️ В этой таблице admin-bff и admin-frontend идут в одном репо `erp-admin`. Но Сано сейчас говорит, что `erp-admin` — это **чисто фронт**. Значит либо документация устарела, либо admin-bff переехал в другой репо. **Нужно подтвердить, где сейчас находится admin-bff.**

---

## Гипотезы причин

В порядке убывания вероятности:

1. **admin-bff недокачен на прод.** Эндпоинты `/api/v1/admin/pos/devices` и `/api/v1/admin/kds/devices` логически принадлежат admin-bff (как proxy/aggregator). Если он не накатан до актуальной версии — оба 500. Нужно найти, в каком сейчас репо живёт admin-bff (после переезда из erp-admin), и накатить его.

2. **`pos-bff` фактически не задеплоился.** Сано говорит «накатил», но: (а) /api/.../pos/devices до сих пор 500, (б) login на POS тоже 500. Стоит **проверить статус последнего workflow `erp-pos` в GitHub Actions** — мог упасть на build/push step. Также проверить `docker compose ps` и логи `pos-bff` контейнера.

3. **user-service / auth-service / device-service лежат.** Если эндпоинт `/admin/pos/devices` в admin-bff проксируется на отдельный сервис устройств, и тот недокачен — admin-bff отдаст 500 от upstream. Стоит посмотреть **логи admin-bff в момент запроса** на `/api/v1/admin/pos/devices` — там будет видно, где он умирает.

4. **Связано с PayKeeper-инфраструктурой.** На том же стенде:
   - `GET /api/v1/admin/paykeeper/accounts/by-store/<id>` → 500 (ТТ → вкладка Интеграции)
   - `GET /api/v1/admin/paykeeper/accounts?status=active` → 502 (`/admin/employees/import-from-paykeeper`)
   - `GET /api/v1/admin/catalog/menu/<store_id>` → 500 (вкладка Меню ТТ)

   Возможно один общий upstream-сервис (или payments/devices/integration модуль) на проде вообще не поднят, а admin-bff проксирует на него и отдаёт 500.

---

## Что нужно от разраба

1. **Подтвердить, где сейчас admin-bff** (репо + последний коммит, который накатан на прод).
2. Запустить деплой того сервиса, который держит маршруты `/api/v1/admin/pos/devices` + `/api/v1/admin/kds/devices` + `/api/v1/admin/paykeeper/*` + `/api/v1/admin/catalog/menu/<store>`. Скорее всего это один и тот же сервис.
3. Посмотреть **логи admin-bff** в момент запроса на `GET /api/v1/admin/pos/devices` — stack trace покажет точную причину за минуту.
4. Если деплой через GitHub Actions — глянуть, **успешно ли** прошёл последний workflow на проде (особенно для `erp-pos` после его наката).

---

## Идентификаторы для воспроизведения

- **Стенд:** `https://admin.nirbi.ru`
- **Учётка admin (web):** `admin@erp.local / <password — см. private/creds.md>` (Администратор, all_franchise)
- **ТТ Smoke 001:** `id 25e6ced6-729a-4e35-8d25-8e6696164ac8` (Опубликована, ООО Франшиза Главное)
  - Один уже зарегистрированный «Терминал 1»: ФН `9999000099990001`, РН ККТ `0001234567000001` (тестовые значения, не реальные).
- **Столовая №1:** `id 7cb8f789-cd7b-44a2-91d1-78a774d326b3` (Черновик, 0 терминалов)
- **Столовая №2:** `id 6e6eeb9e-3101-43e9-a6c7-a5c2d8f8e27e` (Черновик, 0 терминалов)
- **POS-устройство, которое регистрируется:** id `POS-814800`.

---

## Связанные находки в тест-базе (контекст, не блокеры)

См. `test-base/findings.md` за 2026-05-12:

- **F78** — ТТ → Интеграции → `GET paykeeper/accounts/by-store/<id>` 500.
- **F86** — ТТ → Меню → `GET catalog/menu/<store>` 500, UI прячет за «Нет товаров в меню».
- **F100** — `/admin/employees/import-from-paykeeper` → `GET paykeeper/accounts` 502.

Все эти 500/502 могут быть симптомами **одного и того же недокаченного сервиса**. Если так — после правильного наката они отвалятся все вместе.
