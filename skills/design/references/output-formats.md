# Output Formats — `/pdlc:design`

Phase 8 — Report to PM: форматы вывода для всех точек жизненного цикла.

---

### Phase 8 — Report to PM

#### При успешном создании (status=ready)

```
═══════════════════════════════════════════
DESIGN PACKAGE СОЗДАН
═══════════════════════════════════════════

ID: DESIGN-001
Source: {PRD-001 | SPEC-001}
Package: docs/architecture/DESIGN-001-{slug}/
Status: ready

АРТЕФАКТЫ ({N} файлов):
  ✓ README.md (включает Solution Strategy — 5 ключевых решений)
  ✓ manifest.yaml (machine-readable индекс — для doctor/codex/roadmap review)
  ✓ c4-context.md         — System Context
  ✓ c4-container.md       — 4 containers
  ✓ sequences.md          — 2 flows
  ✓ data-model.md         — 3 entities + dictionary
  ✓ api.md                — 6 OpenAPI endpoints
  ✓ async-api.md          — 3 Kafka channels, 6 events (AsyncAPI 3.0)
  ✓ glossary.md           — 12 terms
  ✓ quality-scenarios.md  — 4 measurable scenarios (Q1-Q4 для NFR-001..NFR-004)

ADR СОЗДАНЫ:
  ✓ ADR-003: Mermaid over PlantUML
  ✓ ADR-004: Sessions in Redis vs DB

GLOSSARY FEDERATION (если был glossary.md):
  ✓ knowledge.glossary: +12 терминов из DESIGN-001/glossary.md
  ✓ Конфликтов: 0
  (downstream subagents tasks/implement/spec теперь видят словарь)

ПРОПУЩЕНЫ:
  ✗ c4-component.md     — single container не требует Level 3
  ✗ async-api.md        — нет message broker / event-driven
  ✗ state-machines.md   — нет lifecycle сущностей
  ✗ deployment.md       — нет явных NFRs

───────────────────────────────────────────
QUALITY REVIEW
───────────────────────────────────────────
Iteration: 1/2
Score: 8.7/10
  • Artifact Coverage:    9/10
  • Requirement Coverage: 9/10
  • Consistency:          9/10
  • Depth:                8/10
  • Source Alignment:     9/10
  • Clarity:              8/10
Вердикт: PASS
───────────────────────────────────────────

═══════════════════════════════════════════
СЛЕДУЮЩИЙ ШАГ:
   → /pdlc:roadmap {SPEC-XXX} — план фаз с учётом дизайна
   → /pdlc:tasks {PRD/SPEC-XXX} — создать задачи (subagent учтёт api.md)
   → Открой docs/architecture/DESIGN-001-{slug}/README.md в IDE
═══════════════════════════════════════════
```

#### При улучшении после ревью

```
─────────────────────────────────────────
QUALITY REVIEW
─────────────────────────────────────────
Iteration 1: Score 6.4/10 → IMPROVE
  Применено 5 улучшений
Iteration 2: Score 8.4/10 → PASS
─────────────────────────────────────────
```

#### При наличии вопросов (waiting_pm)

```
═══════════════════════════════════════════
DESIGN PACKAGE ТРЕБУЕТ УТОЧНЕНИЙ
═══════════════════════════════════════════

ID: DESIGN-001 (status: draft)
Package: docs/architecture/DESIGN-001-{slug}/
Source: PRD-001

Создан как draft. Вопросы для PM:
1. {Вопрос 1}
2. {Вопрос 2}

═══════════════════════════════════════════
СЛЕДУЮЩИЙ ШАГ:
   → Ответь на вопросы
   → /pdlc:unblock для продолжения
═══════════════════════════════════════════
```

