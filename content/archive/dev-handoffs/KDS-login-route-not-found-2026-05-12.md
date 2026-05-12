# KDS — Route not found при логине

> Контекст для разработчика (и его Claude).
> Дата: 2026-05-12. Стенд: **admin.nirbi.ru** (PROD).
> Учётка тестирования: `admin@erp.local / <password — см. private/creds.md>`.

## TL;DR

При регистрации/логине на KDS-устройстве экран показывает:

```
Route not found:/api/v1/admin/auth/login
```

POS-устройство на этом же стенде с этой же учёткой регистрируется и работает. KDS — нет.

---

## Что точно знаем (воспроизводимо)

### 1. Backend на запрос POST на этот URL отвечает корректно

```
POST https://admin.nirbi.ru/api/v1/admin/auth/login
Content-Type: application/json
Body: {"email":"admin@erp.local","password":"<password — см. private/creds.md>"}

→ 200 OK
{"data":{"user":{"id":"...","email":"admin@erp.local","franchise":{...},"scope":{"type":"all_franchise"},...}}}
```

То есть **endpoint живой и принимает POST**. Это подтверждено как из admin-frontend (логин в браузере работает), так и прямым fetch через DevTools.

### 2. Backend на GET (или любой не-POST) этого пути отвечает ровно тем сообщением, которое видит KDS

```
GET https://admin.nirbi.ru/api/v1/admin/auth/login

→ 404
{"error":{"code":"NOT_FOUND","message":"Route not found"}}
```

Дословное совпадение текста `"Route not found"` с тем что показывает KDS.

### 3. Других различий со стендом нет

- POS-устройство `POS-814800` зарегистрировано на ТТ Smoke 001, **Online**, версия 0.1.1, текущий кассир — Admin Admin.
- В админке `/admin/devices` оба эндпоинта (`pos/devices`, `kds/devices`) возвращают 200 (пустой массив для KDS — попыток регистрации не дошло).
- На том же `admin.nirbi.ru` логин из браузера работает.

### 4. Kafka

Сано отметил, что Kafka на проде не поднята. **Маловероятно что это причина именно KDS-ошибки** — мой POST на тот же endpoint возвращает 200, значит auth-route жив и не зависит от Kafka синхронно. Но Kafka скорее всего нужно поднять для других этапов прогона (paykeeper-adapter, события заказов, и т.п.).

---

## Главная гипотеза

**KDS-клиент шлёт не-POST метод** (предположительно GET) на `/api/v1/admin/auth/login`. Backend возвращает 404 «Route not found», KDS форматирует его на экране как `Route not found:/api/v1/admin/auth/login` (добавляет URL к сообщению).

Уверенность: ~70–80%. Чистого доказательства нет, потому что:
- У нас нет network-логов с самого KDS-устройства.
- Текст с URL мог быть полностью сформирован клиентом на основе любого 404 — тогда метод может быть и POST, но на **другой path**.

### Альтернативные гипотезы

1. **Опечатка в path** в KDS-клиенте. Например, `/Api/v1/...` (большая `A`), `/api/v1/admin/auth/sign-in`, `/api/v1/admin/login` без `/auth/`, и т.п. Сервер на любой не существующий route возвращает то же `Route not found`.

2. **Неправильный baseURL** в KDS-конфиге. Например, клиент захардкоден на старый test-домен или с другим префиксом. Если на нём такого route нет — тот же 404.

3. **CORS / preflight**. KDS-клиент на Tauri обычно не блокируется CORS, но если запрос проходит через какой-то proxy — может быть.

---

## Что нужно проверить разрабу

В коде `erp-kds` найти **логин-компонент** (форма ввода email/password менеджера на первом экране регистрации) и проверить:

1. **HTTP метод** в fetch/axios/native-tauri вызове — должен быть POST. Если GET — баг найден.
2. **Путь**: должен быть строго `/api/v1/admin/auth/login` (lowercase `api`).
3. **baseURL** — должен указывать на `https://admin.nirbi.ru` (или ту переменную окружения, что включена в билд).
4. **Headers**: `Content-Type: application/json`.
5. **Body**: `{"email": "...", "password": "..."}`.

Если код выглядит правильно — добавить временный лог `console.log(method, url, body)` перед вызовом, пересобрать APK/installer, посмотреть что реально шлётся в попытке логина.

### Также — версия билда

Последний successful workflow в `nearbyErp/erp-kds`:
- **Build Windows installer** — 2026-05-12 10:17 UTC, success, 5m14s, ветка main, push trigger.

Это и есть та сборка, которая сейчас установлена и даёт ошибку. Перебилд без фикса в коде даст тот же бинарь.

---

## Идентификаторы для воспроизведения

- **Стенд:** `https://admin.nirbi.ru`
- **Учётка admin (web):** `admin@erp.local / <password — см. private/creds.md>` — логин через эту учётку в браузер работает.
- **ТТ Smoke 001:** id `25e6ced6-729a-4e35-8d25-8e6696164ac8` (Опубликована, ООО Франшиза Главное).
- **POS на этой же ТТ** регистрируется без проблем.

---

## Кратко для Сано (одной строкой)

> Backend на POST `/api/v1/admin/auth/login` → 200 (логин работает). На GET → 404 «Route not found» (ровно как видит KDS). Проверь в `erp-kds` логин-форме: HTTP метод, точный path, baseURL.

---

## UPDATE 2026-05-12 ~15:40 MSK — финальный диагноз по nginx access-logs

Достал access-logs из Loki по `{namespace="ingress-nginx"}`. Однозначно:

```
12:18:42  POST /api/v1/admin/auth/login   404  Referer: http://tauri.localhost/  UA: Edg/148  →  upstream: erp-front-pos-bff-3022   ← KDS, упал
12:18:57  POST /api/v1/admin/auth/login   404  Referer: http://tauri.localhost/  UA: Edg/148  →  upstream: erp-front-pos-bff-3022   ← KDS, упал
12:15:40  POST /api/v1/pos/auth/login     200  Referer: http://tauri.localhost/  UA: Edg/148  →  upstream: erp-front-pos-bff-3022   ← POS, OK
12:17:03  POST /api/v1/admin/auth/login   200  Referer: https://admin.nirbi.ru/admin/login  UA: Chrome/147  →  upstream: erp-front-admin-frontend  ← web-админка, OK
```

**KDS-клиент шлёт POST правильно**, на `/api/v1/admin/auth/login`. Но **ingress направляет всё с Referer `tauri.localhost` в pos-bff** (`erp-front-pos-bff-3022`), независимо от пути. pos-bff знает только `/pos/auth/login`, поэтому отдаёт 404 «Route not found».

POS не палится на этом, потому что использует `/pos/auth/login` — путь совпадает с тем что pos-bff обслуживает.

### Это инфра/ingress баг, не клиентский

- Перебилд `erp-kds` не поможет.
- Нужно править правила маршрутизации в k8s ingress.

### Почему на тесте работало

На test-VPS (185.152.93.77) деплой через docker-compose, без отдельного pos-bff в виде k8s-сервиса. Вероятно один admin-bff обслуживал и `/admin/*`, и `/pos/*`. После переезда на k8s + ingress разделение admin-bff/pos-bff появилось, но правила ingress для admin-маршрутов от Tauri-клиентов не учитывают, что Referer всё равно `tauri.localhost`.

### Для девопса (что попросить)

> На проде KDS-клиент шлёт POST `/api/v1/admin/auth/login` с Referer `http://tauri.localhost/`. По логам ingress-nginx (см. namespace `ingress-nginx`, controller pod `ingress-nginx-controller-6797f4dc8c-bkbd6`) этот запрос уходит в upstream `erp-front-pos-bff-3022`, который такого route не знает → 404. POST с Referer `admin.nirbi.ru/admin/login` уходит в `erp-front-admin-frontend` и проходит.
>
> Нужно: правило ingress для пути `/api/v1/admin/*` должно направлять в admin-bff **независимо от Referer/Host source-клиента**. Команды для диагностики:
> ```
> kubectl -n erp-front get ingress -o yaml
> kubectl -n ingress-nginx get cm -o yaml | head -100
> ```
> Скорее всего в одном из Ingress-объектов есть правило c host="admin.nirbi.ru" + path /api/v1/admin/* → admin-frontend-svc, а fallback (или другой ingress) с более широким матчем — на pos-bff.
