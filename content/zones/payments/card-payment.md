# Payments · Card payment (PayKeeper)

Оплата картой через PayKeeper. Фискализация, RRN, PayKeeper integration.

## Что попадает в БД при card-оплате

Заказ #026 (id `85777593-...`, оплачен 14:59:26) даёт референс:

```json
{
  "status": "closed",
  "payment_method": "card",
  "paid_amount": 347.5,
  "paid_at": "2026-05-05T14:59:26Z",
  "completed_at": "2026-05-05T14:59:26Z",
  "rrn": null,                              // ❌ F26
  "card_last4": null,                       // ❌ F26
  "fiscal_data": {
    "fn": "9999078902018961",
    "ts": "20260505T1455",
    "fnd": "35",
    "fpd": "3512313478",
    "rnkkt": "1840159071038290",
    "shift_number": 7,
    "receipt_number": 5
  },
  "fiscal_failed": false,
  "pk_invoice_id": "20260505145510997",
  "pk_invoice_url": "https://alfa-kassa-mgm-2.server.paykeeper.ru/bill/20260505145510997",
  "pk_payment_id": 34,
  "pk_fop_receipt_key": "b5788387-0a01-40bc-8c64-f75d54a7770b"
}
```

## Findings

| ID | Severity | Title |
|----|----------|-------|
| F25 | 🔴 | Заказ исчезает из POS UI после оплаты (см. также pos/orders.md, F37) |
| F26 | 🔴 | После оплаты картой возвращаются null: rrn, card_last4 (всегда) + иногда fiscal_data, pk_fop_receipt_key |

## F26 — детальный разбор

**Симптом**: после оплаты картой через PayKeeper заказ переходит в `status: closed`, но критичные поля платежа остаются `null`.

**Что наблюдалось**:

| Поле | Заказ #026 (2026-05-05) | Заказ #008 (2026-05-06) | Должно быть |
|------|-------------------------|-------------------------|-------------|
| `payment_method` | `card` | `card` | `card` |
| `paid_amount` | 347.5 | 350 | сумма заказа |
| `paid_at` | заполнено | заполнено | timestamp |
| `pk_invoice_id` | `20260505145510997` | `20260506104235095` | id счёта |
| `pk_invoice_url` | заполнено | заполнено | URL счёта |
| `pk_payment_id` | `34` | `38` | id платежа |
| **`rrn`** | **`null`** ❌ | **`null`** ❌ | banking RRN, 12 цифр |
| **`card_last4`** | **`null`** ❌ | **`null`** ❌ | 4 цифры карты |
| `fiscal_data` | заполнено ✅ | **`null`** ❌ | объект с fn/fpd/fnd/rnkkt |
| `pk_fop_receipt_key` | заполнено ✅ | **`null`** ❌ | uuid чека |
| `fiscal_failed` | `false` | `false` | true если не получилось |

**Что значит каждое поле и почему важно**:

- **rrn (Reference Retrieval Number)** — банковский идентификатор транзакции. Без него:
  - Невозможно сверить транзакцию с банковской выпиской при разборе спора
  - Невозможно сделать возврат средств через эквайринг (банк требует RRN)
  - Аудит/бухгалтерия не могут привязать платёж к конкретной банковской операции
- **card_last4** — последние 4 цифры карты. Без них:
  - На чеке для клиента не отображается карта (PCI-compliant способ показать «вы платили картой ****1234»)
  - Кассир не может идентифицировать «какой именно картой» при возврате
- **fiscal_data** (`fn`, `fpd`, `fnd`, `rnkkt`, `shift_number`, `receipt_number`) — данные фискального чека от ФР (фискального регистратора), отправляются в ОФД. Без них:
  - **ОФД-сверка ломается** — налоговая видит платёж в эквайринге, но не видит фискальный чек = штрафы/доначисления
  - 54-ФЗ (онлайн-кассы) не выполнен
- **pk_fop_receipt_key** — ключ электронного чека (ФОП = «Фискальный Оператор»). Используется для ссылки на чек в личном кабинете и при споре.
- **fiscal_failed: false** + `fiscal_data: null` — **противоречие**: API утверждает что фискалка прошла, а данных нет. Должно быть либо fiscal_data заполнен, либо fiscal_failed=true.

**Гипотеза**: проблема в `paykeeper-adapter` — обработчик callback'ов от PayKeeper.

PayKeeper присылает уведомления как минимум в трёх событиях:
1. **Счёт создан** (`pk_invoice_id`, `pk_invoice_url`) — это работает, поля заполняются
2. **Оплата произведена** (`pk_payment_id`, `rrn`, `card_pan_masked` → `card_last4`) — работает частично: `pk_payment_id` есть, RRN/last4 нет
3. **Чек ОФД сформирован** (`pk_fop_receipt_key` + `fiscal_data` от ФР) — нестабильно (раньше работало, сейчас нет)

Скорее всего:
- На событии «оплачено» парсятся не все поля callback'а от PK. Конкретно `rrn` и `card.PAN.last4` (или как они там называются в PK API) не маппятся в order.
- На событии «фискализация» — была регрессия после 2026-05-05 (что-то поломалось между сессиями).

**Воспроизведение** (когда есть стенд с активным магазином):

```bash
# 1. Создать заказ через POS (или admin)
ORDER_ID=$(curl -sS -X POST "$B/orders" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"store_id":"…","items":[{"product_id":"…","quantity":1}]}' \
  | jq -r '.data.id')

# 2. Создать счёт PayKeeper
curl -sS -X POST "$B/orders/$ORDER_ID/pay" -H "$H" -H "Content-Type: application/json" \
  --data-binary '{"method":"card"}'
# → возвращает pk_invoice_url, переходим по ней и оплачиваем тестовой картой

# 3. Через ~5 сек после успешной оплаты
curl -sS "$B/orders/$ORDER_ID" -H "$H" | jq '.data | {rrn, card_last4, fiscal_data, pk_fop_receipt_key, fiscal_failed}'
```

**Ожидание**:
```json
{
  "rrn": "412304567890",
  "card_last4": "1234",
  "fiscal_data": { "fn": "...", "fpd": "...", ... },
  "pk_fop_receipt_key": "<uuid>",
  "fiscal_failed": false
}
```

**Факт**:
```json
{
  "rrn": null,
  "card_last4": null,
  "fiscal_data": null,
  "pk_fop_receipt_key": null,
  "fiscal_failed": false
}
```

**Где смотреть в коде**:

1. **`paykeeper-adapter`** — основная зона проблемы:
   - Обработчик callback'а PK на событии «оплачено» — расширить парсинг payload, маппить поля `rrn`, `card.PAN.last4` (имя поля уточнить в документации PK API) в `order_event` или прямой UPDATE.
   - Обработчик callback'а на событии «чек сформирован» — проверить, доходит ли callback вообще; если нет — починить subscribe; если доходит и не парсится — то же самое что в п.1.

2. **`order-service`** — приёмка событий от paykeeper-adapter:
   - Проверить, что эндпоинт/handler принимает все нужные поля и пишет в БД.
   - Логи: фильтр по `pk_payment_id=38` (заказ #008) — что прилетало, что записалось.

3. **БД заказа** — поля где должны жить:
   - `orders.rrn`, `orders.card_last4` — обычные columns, заполняются при оплате.
   - `orders.fiscal_data` — JSONB, заполняется по событию фискализации.
   - `orders.pk_fop_receipt_key` — uuid, по тому же событию.

4. **Очередь между paykeeper-adapter и order-service** (если есть Kafka/Redis) — проверить что нет дропов между сервисами на этих событиях.

**Done when**:
- Тестовый заказ оплачен картой → через ≤5 сек все 5 полей заполнены ИЛИ `fiscal_failed: true` с причиной если фискалка реально упала.
- `rrn` соответствует RRN из банковской выписки PayKeeper (можно сверить в личном кабинете PK).
- `card_last4` совпадает с последними 4 цифрами тестовой карты.
- На двух подряд оплатах поведение стабильное (нет «один раз есть, другой нет»).

**Важный нюанс — нестабильность fiscal_data**: вчерашний заказ #026 имел fiscal_data заполненным, сегодняшний #008 — нет. Возможно это **регрессия после фикса других багов 2026-05-05 → 2026-05-06**. Просьба разрабу: проверить changelog/диффы paykeeper-adapter и order-service за последние сутки, не сломали ли parsing случайно.

## Тест-кейсы

## Тест-кейсы

### TC-PAY-CARD-026 — Структура заказа после card-оплаты
**Status:** ✅ partial (заказ #026)
fiscal_data заполнен, paykeeper-поля все есть. Но F26 — RRN и card_last4 null.

### TC-PAY-CARD-026-RRN — Регресс F26 после фикса
**Status:** verify-needed
1. Card-оплата на стенде
2. Через 1-2 сек GET /orders/{id} → rrn заполнено
3. card_last4 = 4 цифры

### TC-PAY-CARD-PK-FAIL — PayKeeper недоступен → fallback
**Status:** ◯ todo
Если можно симулировать недоступность PK (например через manual block) — что POS показывает кассиру?
