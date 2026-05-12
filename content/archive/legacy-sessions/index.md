# Сессии — хронология прогонов

| Дата | Тип | Тестировщик | Длительность | Кейсов | Findings | Файл |
|------|-----|-------------|--------------|--------|----------|------|
| 2026-05-05 | e2e | Александр + Claude | ~4 ч | 30 / 60 | 42 (10 Crit, 14 Maj, 9 Min, 9 info/retracted) | [2026-05-05-e2e](2026-05-05-e2e.md) |
| 2026-05-06 | regression | Claude (Playwright) | ~1 ч | 32 / 32 F-NN | 13 fixed, 8 still open, 6 verify-needed, 4 POS-deferred, 1 new (F46) | [2026-05-06-regression](2026-05-06-regression.md) |
| 2026-05-06 | ui-regression | Claude (Playwright + demo3) | ~30 мин | F4/F16/F45 + sidebar map | 4 retracted (F4, F32, F44, F45), 3 new (F47, F48, F49), F26 deferred | [2026-05-06-ui-regression](2026-05-06-ui-regression.md) |
| 2026-05-07 | exploratory | Claude (Playwright + demo3) + Александр (KDS на ПК) | ~2 ч | сценарий «официант ↔ стол ↔ кухня ↔ выдача» | 9 BUG (F66, F67, F69-F75: 4 Critical, 4 Major, 1 Minor) + 5 PROPOSAL (архитектурные/UX). Подтверждены F56, F61. **⏸️ Заморожена 2026-05-12** | [2026-05-07-waiter-table-walkthrough](2026-05-07-waiter-table-walkthrough.md) |
| 2026-05-12 | exploratory | Claude (Playwright) | ~3 ч | прод-стенд admin.nirbi.ru, все разделы кроме POS/KDS, +глубокий E2E/edge/concurrency/states/пагинация/импорт | 26 новых (F76-F101: 9 Critical, 4 Major, 12 Minor, 1 XSS-warning) + ряд PROPOSAL. BUG-005 retracted, BUG-018/019/023/024/051/060 подтверждены, BUG-002/006/007 verify-needed | [2026-05-12-admin-prod-walkthrough](2026-05-12-admin-prod-walkthrough.md) |

## Соглашение по сессиям

- **Имя файла:** `YYYY-MM-DD-<тип>.md`
- **Типы:** `e2e`, `regression`, `smoke`, `exploratory`, `onboarding`
- **Шаблон:** копировать [templates/session.md](../templates/session.md)
- **Что должно быть в каждой сессии:**
  - Frontmatter (дата, тип, тестировщик, длительность)
  - Цель сессии (что хотели проверить)
  - Что сделали (chronological log по этапам)
  - Findings (список F-NN с короткой характеристикой — детали в `findings.md`)
  - Прогон сценариев (галочки pass/fail если был сценарий)
  - Итог + что в `todo.md` добавить
