# Output Formats — `/pdlc:roadmap`

Все варианты вывода для разных точек жизненного цикла команды.

---

## Формат вывода

```
═══════════════════════════════════════════
ПЛАН СОЗДАН
═══════════════════════════════════════════

ID: PLAN-001
Название: [Название]
Файл: docs/plans/PLAN-001-slug.md
На основе: SPEC-001
Статус: ready

Фазы:
1. Setup (3 items)
   • MVP-1.1: Project scaffolding
   • MVP-1.2: Database setup
   • MVP-1.3: Auth integration

2. Core (5 items)
   • MVP-2.1: User service
   • MVP-2.2: API endpoints
   ...

3. Integration (2 items)
   ...

4. Polish (3 items)
   ...

Всего roadmap items: 13
Критический путь: MVP-1.1 → MVP-2.1 → MVP-3.1 → MVP-4.1

Риски:
• [Риск 1 и митигация]
• [Риск 2 и митигация]

───────────────────────────────────────────
QUALITY REVIEW
───────────────────────────────────────────
Iteration: 1/2
Score: 8.2/10
  • Покрытие: 9/10
  • Фазирование: 8/10
  • Зависимости: 8/10
  • Гранулярность: 8/10
  • Риски: 8/10
  [если есть DESIGN PACKAGE:]
  • Architecture Coverage: 9/10
  • Component-Item Mapping: 8/10
  • API Coverage: 8/10  (или "N/A" если нет api.md)
Вердикт: PASS
───────────────────────────────────────────

═══════════════════════════════════════════
СЛЕДУЮЩИЙ ШАГ:
   → /pdlc:tasks PLAN-001 — создать задачи из items
   → /pdlc:continue — автономная работа
═══════════════════════════════════════════
```

