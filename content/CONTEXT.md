# ERP — Контекст тестирования

Главный living-doc тест-базы. Читать первым в каждой сессии.

## Стенды и учётки

| Стенд | URL | Учётка | PIN |
|---|---|---|---|
| **PROD** | https://admin.nirbi.ru/admin | `admin@erp.local / <password — см. private/creds.md>` | 9999 |
| **TEST** | https://erp-test.nirbi.ru/admin | `demo3@nirbi.ru / <password — см. private/creds.md>` | разные (см. memory `project_pos_kds_regression_2026_05_07.md`) |

Пароли и PIN — в `private/creds.md` (gitignored).

**Текущий фокус — PROD.** TEST используется только для возврата к замороженной регрессии POS/KDS (2026-05-07, F66–F76).

## Архитектура (быстро)

| Компонент | Описание |
|---|---|
| Прод-инфра | Kubernetes, namespaces `erp` (backend) и `erp-front` (admin/POS bff и frontend) + `ingress-nginx` |
| Сервисы k8s | `admin-bff`, `pos-bff` (3022), `user-service`, `store-service`, `catalog-service`, `order-service`, `aggregator-service`, `customer-service`, `warehouse-service`, `auth-service`, `paykeeper-adapter` |
| Деплой | Каждый сервис отдельно через GitHub Actions (`gh workflow run -R nearbyErp/<repo>`). One-click rollout нет. См. memory `project_deploy_no_oneclick.md` |
| Observability | Grafana (https://grafana.nirbi.ru) + Prometheus (kube-state-metrics) + Loki (nginx access-logs в namespace `ingress-nginx`) |
| Картинки товаров | MinIO `minio.nirbi.ru/erp-catalog/products/<id>/image.*`. POS-меню отдаёт только minio-URL. Внешние URL в `image_url` через PATCH не работают для POS |
| Tauri-клиенты | POS-desktop (Windows installer, repo `nearbyErp/erp-pos-desktop`) и KDS (Android/Windows, repo `nearbyErp/erp-kds`). Имеют `Referer: tauri.localhost` |

Подробный runbook деплоя (test VPS, устарело для прода) — `content-mirror/06-DevOps/Deployment-Runbook.md`.

## Состояние ТТ Smoke 001 (PROD) — на 2026-05-12

| Сущность | Количество | Заметки |
|---|---|---|
| Категории | 12 | Кофе, Напитки, Напитки холодные, Горячие напитки, Завтраки, Супы, Горячее, Гарниры, Салаты, Закуски, Десерты, Выпечка |
| Ингредиенты | 13 | Молоко 3.2%, Говядина, Куриное филе, Свинина, Форель, Картофель, Лук, Морковь, Мука, Яйцо, Сахар, Соль, Сыр |
| Модификаторы | 5 (14 опций) | Молоко (3), Сироп (3), Размер порции (3), Соус (4), Без сахара (1) |
| Товары | 67 | Тематические картинки (Wiki + Unsplash). Хранение MinIO |
| Цены в прейскуранте «Базовый» | 82 | Товары + модификатор-опции, 80–400 ₽ |
| Временные тарифы | 3 | Завтрак −15% (07–11), Бизнес-ланч −25% (12–16 Пн-Пт, категории Супы/Горячее/Гарниры), Счастливый час −10% (14–17 Пн-Пт, Кофе+Десерты) |
| Расписание меню | 1 | Завтраки видны только 07:00–11:00 Пн-Вс |
| Стоп-лист | 2 | «Латте 0.3» (Тест), «Уха из форели» (Нет рыбы) |
| POS-устройства | 1 | POS-814800, Online, текущий кассир Admin Admin |
| KDS-устройства | 1 | Зарегистрировано после фикса ingress-роутинга (см. handoff) |
| Сотрудники | 2 | Admin Admin (admin@erp.local, PIN 9999, привязан к ТТ Smoke 001), Тест Тестов (роль "Кассир", PIN неизвестен) |

ID для запросов:
- ТТ Smoke 001: `25e6ced6-729a-4e35-8d25-8e6696164ac8`
- ООО Франшиза Главное: `10000000-0000-0000-0000-000000000001`
- Прейскурант «Базовый»: `80f2a261-93a8-4615-a0e4-c001cbd0760a`
- Категории см. через `GET /api/v1/admin/catalog/categories?limit=200`

## Активные блокеры (для разраба)

| ID | Что блокирует | Где смотреть |
|---|---|---|
| **F102 / F103** | UI редактирования сотрудника висит на «Сохранение...», `PATCH /api/v1/admin/employees/<id>` → 500 на любое поле (pin, roles.stores). Обход — прямой PATCH с {pin} или {roles:[{role_id,store_ids}]}. user-service Pod здоров, баг в коде свежего билда | findings F102/F103 |
| **F78 / F100** | PayKeeper-flow заблокирован: `paykeeper/accounts/by-store` → 500 «fetch failed», `paykeeper/accounts` → 502 PAYKEEPER_ADAPTER_UNAVAILABLE. Сано отметил «Kafka не подняли». Ждём подъём Kafka | findings F78/F100 |

## Открытые вопросы / TODO

- **POS/KDS happy-path** запланирован на 2026-05-13 (план в session 2026-05-12 → этап «POS/KDS happy-path»).
- **RBAC** под Manager/Кассир/Franchisee — заблокирован отсутствием паролей к существующим тестовым аккаунтам. Без новых аккаунтов на проде — невозможен.
- **Hard-delete [TEST] Капучино 0.4** (id `d1224622-a76d-4bce-bee8-64bf2cc3411e`) — застрял в soft-delete, hard-delete endpoint вернул 404.
- **F81 (mock-чаевые)** ждёт ручного фикса разработчиком — фронт-хардкоженные данные на странице `/admin/tips`.
- **F91 (длинное имя ТТ → 500)** — backend не валидирует длину строки.

## Конвенции

| Что | Куда |
|---|---|
| Баги (все) | `findings.md` — единый список с severity, source, status |
| Скриншоты находки | `screenshots/<session-id>/F<NN>-<desc>.png` + README.md с маппингом |
| `[TEST]`-объекты | Создавать с префиксом `[TEST] `, удалять после прогона. Без новых аккаунтов на проде |
| Doc-for-dev | Узкоцелевые handoff-документы — `archive/dev-handoffs/` |
| Старые сессии (до 2026-05-12) | `archive/legacy-sessions/` (frozen, ссылка из «Сессии» ниже) |

## Принципы тестирования (из memory)

- Фокус — **user-facing сценарии через UI** (Playwright). API-аудит без UI-эффекта — пропускаем.
- BUG vs PROPOSAL — раздельная маркировка, PROPOSAL с баннером «не фиксить автоматически».
- Каждая F-NN-карточка: **user-сценарий + «Где смотреть» для разраба**. Канон — `archive/legacy-sessions/BUGS-FOR-DEV-stage4.md`.
- 500/блокер → сначала сообщение разрабу, потом — решение, заводить ли finding. См. memory `feedback_blocker_ask_first.md`.
- Не трогать инфру/деплой/конфиги самостоятельно. `gh workflow run` — только по явному запросу. См. memory `feedback_no_infra_touching.md`.
- На проде: модификация существующих объектов — только с разрешения. Тестовые объекты — с префиксом `[TEST]`, без новых аккаунтов.

---

# Хронология сессий

## Сессия 2026-05-12 — Admin walkthrough на PROD + POS/KDS onboarding + наполнение стенда

**Тестер:** Claude (Playwright) + Александр на связи. **Длительность:** ~8 ч.

### Прогон 9 разделов admin-панели (без POS/KDS)

| # | Раздел | Статус | Найденные finding'и |
|---|--------|--------|---------------------|
| 1 | Логин + sidebar map (10 разделов) | ✅ | dashboard «Роль: —» (мелочь) |
| 2 | Юридические лица | ✅ | F76, F77, BUG-005 retracted, BUG-006 verify-needed |
| 3 | Торговые точки | ✅ | F78 (Интеграции PayKeeper 500), F84 (silent submit), F89 (два «дефолтный»), F90 (concurrency last-write-wins), F91 (длинное имя → 500), F92 (email-валидация неконсистентна) |
| 4 | Сотрудники + Роли + Расписание + Шаблоны + Активность + Терминалы + Формулы + Ведомости | ✅ | F79 (кавычки в названиях), F80 (JWT stale без redirect), F93/BUG-018-19 (Save шаблона смены 500), F94/BUG-023 (Save формулы 500), F95/BUG-024 (Экспорт CSV 422), F96 (английские ошибки в UI), F97 (Отмена → history.back) |
| 5 | Каталог (Товары/Модификаторы/Категории/Прейскуранты/Тарифы/Меню/Внешние/Ингредиенты/Стоп-листы) | ✅ | F82 (КБЖУ без min=0), F83 (HEX без pattern), F85 (XSS-warning), F86/BUG-060 (ТТ→Меню 500, починен накатом), F87 (карточка без категории), F88 (двойные кавычки в confirm), F98 (Дублирование 500), F99 (+Дочерняя на корне), F101 (AI кнопка-заглушка) |
| 6 | Склад (Остатки/Приёмки/Списания) | ✅ | без находок |
| 7 | Заказы (Активные/История/Транзакции/Мониторинг смен) | ✅ | без находок |
| 8 | Финансы → Чаевые (Нетмонет) | ✅ | **F81 (CRITICAL)** — фейковые цифры на проде, hardcoded во фронте |
| 9 | Глубокий E2E + edge-cases + concurrency + восстановление | ✅ | F90, F92, F96, F101 — см. выше |

### Pre-test расчистка стенда (после reality-check)

- Создал 30 [TEST] товаров через тот же endpoint что UI → проверил пагинацию/сортировку A-Я → удалил все 30.
- Пагинация: «Показано 1–20 из 33», 2 страницы. Сортировка A-Я по Unicode (`"`<`[`). ✅

### POS/KDS onboarding (с помощью Сано)

| # | Задача | Статус |
|---|--------|--------|
| - | **Регистрация POS** — после наката `erp-pos` (Сано) endpoint `pos/devices` ожил, POS-814800 Online на ТТ Smoke 001 | ✅ |
| - | **KDS блокер `Route not found:/api/v1/admin/auth/login`** — диагностика через Grafana + Loki (nginx access-logs в ns `ingress-nginx`). ingress направлял Tauri-запросы с `Referer: tauri.localhost` всегда на `pos-bff`, который не знает admin-route. Сано пофиксил в KDS-клиенте | ✅ |
| - | **PATCH admin'у PIN=9999** через прямой PATCH (UI висел F102) | ✅ |
| - | **Привязка admin → ТТ Smoke 001** через прямой PATCH с `{roles:[{role_id,store_ids}]}` (UI висел F102) | ✅ |
| - | Дев-док про KDS-блокер для разраба → `archive/dev-handoffs/KDS-login-route-not-found-2026-05-12.md` | ✅ |
| - | Дев-док про POS-onboarding для разраба → `archive/dev-handoffs/POS-KDS-onboarding-blocker-2026-05-12.md` | ✅ |

### Наполнение стенда (через UI + batch через тот же endpoint UI)

| # | Действие | Статус |
|---|----------|--------|
| - | Renaming (убрали кавычки): «"Кофе"», «"Латте 0.3"», «"Молоко"», «"Базовый"», «"Молоко 3.2%"» | ✅ |
| - | 10 категорий: Завтраки, Супы, Горячее, Гарниры, Салаты, Закуски, Десерты, Выпечка (+ имевшиеся Кофе, Напитки, Напитки холодные, Горячие напитки) | ✅ |
| - | 12 ингредиентов (Говядина, Куриное филе, ..., Сыр) | ✅ |
| - | 4 модификатора (Сироп, Размер порции, Соус, Без сахара) | ✅ |
| - | 65 товаров по категориям (4–9 в каждой) | ✅ |
| - | 82 цены в прейскуранте «Базовый» (random 80–400 ₽) | ✅ |
| - | 3 временных тарифа (Завтрак / Бизнес-ланч / Счастливый час) | ✅ |
| - | 1 расписание меню (Завтраки 07–11 Пн-Вс) | ✅ |
| - | 1 стоп (Уха из форели «Нет рыбы») в дополнение к уже бывшему Латте 0.3 | ✅ |
| - | Картинки 68 товаров: 64 через Wikipedia summary API (точное соответствие блюду), 4 — Unsplash проверенные | ✅ |
| - | Корректировка: «Какао» + 3 чая перенесены из Кофе в Горячие напитки; «Капустный с морковью» удалён; прицельные картинки для Куриная лапша / Запечённая форель / Рис / Макароны / Овощи / Соленья / Слойка с сыром / 3 пирожка | ✅ |

### Подтверждения backlog

| ID | Старое | Новое |
|----|--------|-------|
| BUG-002 (потеря фокуса) | open | verify-needed (на admin не воспроизводится) |
| BUG-005 (вкладки ЮЛ) | open | **retracted** (UI переделан) |
| BUG-006 (404 Save франшизы) | open | verify-needed |
| BUG-007 (поиск ФИО) | open | verify-needed (на admin работает) |
| BUG-018, BUG-019, BUG-023, BUG-024 | open | **reproduced** = F93/F94/F95 |
| BUG-051 (default-прейскурант не привязан) | open | reproduced |
| BUG-060 (catalog/menu 500) | open | **fixed** (после деплоя 2026-05-12) |

### Итоги дня

- **28 новых finding (F76–F103)**: 1 Critical, 4 Major, 12 Minor, 1 XSS-warning + 9 backlog-reproductions
- **POS/KDS onboarding отдельно — 2 блокера**, оба пофикшены силами Сано
- **Стенд готов к POS/KDS happy-path** (план в этом же файле выше)
- **Не покрыто:** RBAC (нужны новые аккаунты — запрещено), импорт XLSX, загрузка фото товара через UI (делали через API), полный happy-path

---

## Старые сессии (frozen, see archive/legacy-sessions/)

| Дата | Тип | Кратко | Файл |
|------|-----|--------|------|
| 2026-05-05 | e2e | 30/60 кейсов, **42 finding** (F1-F65) | [2026-05-05-e2e](archive/legacy-sessions/2026-05-05-e2e.md) |
| 2026-05-06 | regression | F1-F65 после фиксов: 13 fixed, 8 still open, 6 verify-needed, 4 POS-deferred, 1 new (F46) | [2026-05-06-regression](archive/legacy-sessions/2026-05-06-regression.md) |
| 2026-05-06 | ui-regression | demo3, +sidebar map, 4 retracted, 3 new (F47-F49) | [2026-05-06-ui-regression](archive/legacy-sessions/2026-05-06-ui-regression.md) |
| 2026-05-06 | playbook | POS-coordination, e2e-demo3-playbook, full-coverage-plan | [pos-coordination](archive/legacy-sessions/2026-05-06-pos-coordination-playbook.md), [e2e-demo3](archive/legacy-sessions/2026-05-06-e2e-demo3-playbook.md), [full-coverage](archive/legacy-sessions/2026-05-06-full-coverage-plan.md) |
| 2026-05-07 | exploratory | demo3 waiter↔table flow, **9 BUG (F66-F75)** + 5 PROPOSAL. ⏸️ **Заморожена** 2026-05-12 (переключение на PROD admin) | [2026-05-07-waiter-table](archive/legacy-sessions/2026-05-07-waiter-table-walkthrough.md) |
| 2026-05-12 | exploratory+setup | Admin walkthrough + POS/KDS onboarding + наполнение стенда | (это CONTEXT выше) |

Оригинальные подробные логи сессии 2026-05-12 (хронология шаг-за-шагом, скриншоты, edge-case попытки): [archive/legacy-sessions/2026-05-12-admin-prod-walkthrough.md](archive/legacy-sessions/2026-05-12-admin-prod-walkthrough.md) и сводный SUMMARY: [archive/legacy-sessions/SUMMARY-2026-05-12.md](archive/legacy-sessions/SUMMARY-2026-05-12.md).
