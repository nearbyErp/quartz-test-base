# quartz-test-plans

Документация ручного тестирования Obsidian ERP, опубликованная как статический сайт через [Quartz 4](https://quartz.jzhao.xyz).

🌐 **Сайт:** https://nearbyerp.github.io/quartz-test-plans/

## Структура

```
content/
├── README.md                          # entry point
├── 00-templates/
│   ├── bug-report-template.md         # формат для Google Docs
│   └── test-case-template.md          # TMS-portable формат
├── 01-rbac-matrix.md                  # сквозная матрица доступа (4 роли × 13 разделов)
├── 02-form-validation-checklist.md    # универсальный чек-лист (6 секций × ~50 проверок)
├── 03-smoke-pass.md                   # 30-мин круг "вообще живо"
├── 04-known-bugs-index.md             # 70 багов с ID, severity, распределением по T1/T2/T3
└── testers/                           # (в работе) пакеты test cases по тестировщикам
    ├── T1-access-structure/
    ├── T2-hr-and-catalog/
    └── T3-operations/
```

## Деплой

Автоматически через GitHub Actions при push в `main`. См. `.github/workflows/deploy.yml`.

## Локальный preview

```bash
npm install
npx quartz build --serve
# → http://localhost:8080
```

## Как обновлять контент

1. Правишь markdown в `content/`
2. `git add content/ && git commit -m "..." && git push`
3. Через ~1 минуту изменения на сайте

## Связанные проекты

- **`nearbyErp/quartz-site`** — dev-документация ERP-платформы (отдельный репо, отдельный сайт)
- **Bug tracker** — Google Docs (см. `content/00-templates/bug-report-template.md`)

## Стек

Quartz 4 (форк https://github.com/jackyzha0/quartz). Все credits — оригинальному автору.
