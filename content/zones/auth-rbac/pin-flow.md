# Auth-RBAC · PIN flow (POS / KDS)

PIN-логин кассира на POS и оператора на KDS. Manager-approval flow для escalations.

## Endpoints (через POS BFF, недоступны снаружи стенда)

```
POST /api/v1/pos/auth/pin                   PIN-логин
                                            требует X-Device-Id header (зарегистрированный POS/KDS device)
                                            body: {pin: "1234"}
GET  /api/v1/pos/auth/me
POST /api/v1/pos/auth/refresh
POST /api/v1/pos/auth/logout

POST /api/v1/pos/manager-auth/verify-pin    manager-approval flow
                                            возвращает X-Approval-Token для escalation действий
```

## Headers

```
Authorization:    Bearer <jwt>            (после PIN-логина)
X-Device-Id:      <device_id>            (обязателен для всех /api/v1/pos/* запросов)
X-Active-Store:   <store_id>
X-App-Version:    "0.1.1"
X-Approval-Token: <от manager-auth>      (для escalations)
```

## Findings

(пока пусто — PIN flow тестировался только из-за того что snowflake POS desktop работает на стенде)

## Тест-кейсы

### TC-PIN-001 — Корректный PIN кассира
**Status:** ⛔ blocked (pos-bff недоступен снаружи)
Через POS desktop у Александр работает. Не воспроизводимо через CLI.

### TC-PIN-002 — Неверный PIN
**Status:** ◯ todo (POS desktop)

### TC-PIN-003 — PIN-логин без `pos.access` permission
**Status:** ◯ todo · **TC-CHAIN-087**
Кассир без `pos.access` → отказ при PIN-логине

### TC-PIN-KDS-001 — KDS-оператор PIN
**Status:** ⛔ blocked (тот же эндпоинт `/pos/auth/pin`)
По спеке: KDS-оператор требует `kds.access` permission, не `pos.access`.

### TC-PIN-MANAGER-APPROVAL — Manager PIN для escalation
**Status:** ◯ todo
Какие действия требуют approval? Нужны спека / observation.
- pos.discount.apply (скидка)
- pos.item.cancel (отмена позиции после accepted?)
- pos.refund (?)
