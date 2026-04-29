# Quality Review Loop — `/pdlc:roadmap`

Полный алгоритм quality review PLAN: запуск reviewer-субагента, обработка score, IMPROVE-итерация, max-iterations gate, формат записи review.

---

### 6. Quality Review Loop (обязательно!)

После создания PLAN запусти независимый ревью:

```
┌──────────────────────────────────────────┐
│  REVIEW SUBAGENT (чистый контекст)        │
│  INPUT:  SPEC + PRD (исходные документы)  │
│  OUTPUT: созданный PLAN                   │
│  → Оценка 1-10 по критериям              │
│  → Конкретные улучшения                   │
└──────────────────┬───────────────────────┘
                   ▼
           ┌───────────────┐
           │ Score >= 8?   │───YES──→ PROCEED
           └───────┬───────┘
                   NO
                   ▼
┌──────────────────────────────────────────┐
│  IMPROVEMENT SUBAGENT (чистый контекст)  │
│  → Применяет улучшения к PLAN файлу      │
└──────────────────┬───────────────────────┘
                   ▼
           ┌───────────────┐
           │ Iteration < 2?│───NO──→ PROCEED (log warning)
           └───────┬───────┘
                   YES → Back to review
```

**Anti-loop safety:** Максимум 2 итерации (review+improve). После 2-й — продолжить с предупреждением.

#### Запуск Review субагента

Прочитай:
1. SPEC файл — полное содержимое
2. PRD файл (если есть parent PRD) — полное содержимое
3. Созданный PLAN — полное содержимое
4. DESIGN package (опционально, если у SPEC есть ребёнок `DESIGN-PKG`):
   - `{package.dir}manifest.yaml` — обязательно (содержит `artifacts[].components`,
     `artifacts[].entities`, `realizes_requirements` — основа для Architecture Coverage
     и Component-Item Mapping)
   - `{package.dir}README.md` — обязательно (Solution Strategy + список артефактов)
   - `{package.dir}api.md` — если присутствует (для критерия API Coverage)
   Если DESIGN-PKG нет — пропусти этот шаг и **не** добавляй conditional блок в prompt.

Запусти Task tool:
```
Task tool:
  subagent_type: "general-purpose"
  description: "Quality review PLAN-XXX vs SPEC-XXX"
  prompt: [prompt ниже]
```

Prompt для review субагента:
```
═══════════════════════════════════════════
SYSTEM ROLE: Independent Quality Reviewer
═══════════════════════════════════════════

Ты — независимый ревьюер. Ты НЕ автор этого документа.
Твоя задача — объективно оценить OUTPUT на соответствие INPUT.

ПРАВИЛА:
1. Оценивай ТОЛЬКО по фактам из INPUT — не додумывай
2. Каждое замечание должно ссылаться на конкретное место в INPUT
3. Не хвали — только конкретные проблемы и оценки
4. Если всё хорошо — ставь высокий балл, не ищи проблемы искусственно

═══════════════════════════════════════════
INPUT (исходные документы)
═══════════════════════════════════════════

--- SPEC ---
{полное содержимое SPEC}

--- PRD (если есть) ---
{полное содержимое PRD или "N/A"}

--- DESIGN PACKAGE (если есть DESIGN-PKG ребёнок SPEC) ---
manifest.yaml:
{полное содержимое manifest.yaml или "N/A — DESIGN package отсутствует"}

README.md:
{полное содержимое DESIGN README.md или "N/A"}

api.md:
{полное содержимое api.md или "N/A — нет OpenAPI контракта"}

═══════════════════════════════════════════
OUTPUT (результат для ревью)
═══════════════════════════════════════════
{полное содержимое созданного PLAN}

═══════════════════════════════════════════
КРИТЕРИИ ОЦЕНКИ
═══════════════════════════════════════════

БАЗОВЫЕ (всегда):
- Покрытие (все секции SPEC представлены в roadmap items): X/10
- Фазирование (MVP first, инкрементальная ценность): X/10
- Зависимости (корректны, минимальны): X/10
- Гранулярность (items = 1-3 дня, 2-5 TASKs каждый): X/10
- Риски (идентифицированы, с митигациями): X/10

ДОПОЛНИТЕЛЬНЫЕ (ТОЛЬКО если DESIGN PACKAGE присутствует):
- Architecture Coverage (X/10): каждый container/component из DESIGN покрыт хотя бы
  одним roadmap item. Источник истины — `manifest.yaml`:
    • объедини `artifacts[].components` всех `c4-container`/`c4-component` записей
    • объедини `artifacts[].entities` всех `erd` записей (если применимо для item-уровня)
  Item «покрывает» component, если упоминает его по имени в Description/Deliverable
  ИЛИ имеет component_refs: с этим именем (см. ниже). Перечисли непокрытые в КРИТИЧНЫХ.
- Component-Item Mapping (X/10): для каждого roadmap item должно быть однозначно
  понятно, какие компоненты из C4 / data-model он реализует. Идеально — явное поле
  `component_refs: [name1, name2]`. Допустимо — упоминание имён в Description.
  Item, который не привязан ни к одному компоненту, понижает балл.
- API Coverage (X/10): если `api.md` присутствует — каждый OpenAPI endpoint
  (`paths.<path>.<method>`) закрыт хотя бы одним roadmap item (по path/handler/слову
  в Description либо Deliverable). Перечисли непокрытые endpoints в КРИТИЧНЫХ.
  Если `api.md` отсутствует — API Coverage НЕ оценивается, опусти его.

═══════════════════════════════════════════
ФОРМАТ ОТВЕТА
═══════════════════════════════════════════

ОЦЕНКИ (базовые):
- Покрытие: X/10 — {brief justification}
- Фазирование: X/10 — {brief justification}
- Зависимости: X/10 — {brief justification}
- Гранулярность: X/10 — {brief justification}
- Риски: X/10 — {brief justification}

ОЦЕНКИ (DESIGN — только если manifest.yaml присутствует):
- Architecture Coverage: X/10 — {brief justification, перечисли непокрытые components}
- Component-Item Mapping: X/10 — {brief justification, items без привязки}
- API Coverage: X/10 — {brief justification, непокрытые endpoints; "N/A" если нет api.md}

ИТОГО: X/10 (среднее по применимым критериям)

КРИТИЧНЫЕ ПРОБЛЕМЫ (блокеры, если есть):
1. {problem}: {what's in INPUT} → {what's missing/wrong in OUTPUT}

УЛУЧШЕНИЯ (конкретные, применимые):
1. {section/line}: {what to change} → {how to change}
2. ...

ВЕРДИКТ:
- Без DESIGN PACKAGE: PASS (среднее >= 8) | IMPROVE (среднее < 8)
- С DESIGN PACKAGE:   PASS (среднее >= 8 И Architecture Coverage >= 8)
                      | IMPROVE (иначе)

Жёсткий минимум на Architecture Coverage означает: непокрытие containers/components
не компенсируется хорошими оценками других критериев — это блокер. Roadmap не должен
терять целые куски архитектуры.
```

#### Обработка результата review

**Если PASS (score >= 8):**
- Логируй score в session-log
- Продолжай к финальному выводу

**Если IMPROVE (score < 8):**
- Запусти Improvement субагент (см. ниже)
- После улучшения — повтори review (макс. 2 итерации)

#### Запуск Improvement субагента

```
Task tool:
  subagent_type: "general-purpose"
  description: "Improve PLAN-XXX based on review"
  prompt: [prompt ниже]
```

Prompt для improvement субагента:
```
Ты получил результаты независимого ревью PLAN.
Твоя задача — применить конкретные улучшения к файлу PLAN.

ФАЙЛ ДЛЯ УЛУЧШЕНИЯ: {path to PLAN file}

РЕКОМЕНДАЦИИ РЕВЬЮ:
{полный ответ review субагента}

ИНСТРУКЦИИ:
1. Прочитай текущий файл PLAN (Read tool)
2. Примени ТОЛЬКО рекомендации из ревью — не добавляй лишнего
3. Сохрани обновлённый файл (Edit tool)
4. Верни список применённых изменений
```

#### Логирование в session-log

Добавь запись в `.state/session-log.md`:
```markdown
### Quality Review: PLAN-{ID} (from SPEC-{ID})
- Date: {today}
- Iteration 1: {score}/10 → {PASS|IMPROVE}
- Iteration 2: {score}/10 → {PASS|IMPROVE} (если была)
- Command: /pdlc:roadmap
```

