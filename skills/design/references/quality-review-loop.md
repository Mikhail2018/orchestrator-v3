# Quality Review Loop — `/pdlc:design`

Holistic Quality Review Loop для DESIGN package: запуск reviewer-субагента, обработка score, IMPROVE-итерация, max-iterations gate.

---

### Phase 6 — Holistic Quality Review Loop

Адаптация Quality Review Loop из `/pdlc:spec` (lines 242-388), но **холистическая** (один ревью на весь package, не per-file).

```
┌──────────────────────────────────────────┐
│  REVIEW SUBAGENT (clean context)          │
│  INPUT:  source PRD/SPEC + parent (если)  │
│  OUTPUT: ВСЕ файлы package + ВСЕ ADRs     │
│  → Score 1-10 по 5 критериям              │
│  → Конкретные улучшения                   │
└──────────────────┬───────────────────────┘
                   ▼
           ┌───────────────┐
           │ Score >= 8?   │───YES──→ PROCEED
           └───────┬───────┘
                   NO
                   ▼
┌──────────────────────────────────────────┐
│  IMPROVEMENT SUBAGENT (clean context)    │
│  → Применяет improvements ко всему package│
└──────────────────┬───────────────────────┘
                   ▼
           ┌───────────────┐
           │ Iteration < 2?│───NO──→ PROCEED (log warning)
           └───────┬───────┘
                   YES → Back to review
```

**Anti-loop safety**: max 2 итерации (review + improve). После 2-й — proceed с предупреждением в session-log.

#### Запуск Review субагента

Прочитай:
1. Source PRD/SPEC + parent (если есть)
2. **ВСЕ файлы созданного package** (README + все sub-artifacts)
3. **ВСЕ созданные ADRs**

Запусти Task tool:

```
Task tool:
  subagent_type: "general-purpose"
  description: "Holistic quality review DESIGN-{NNN}"
  prompt: [prompt ниже]
```

Prompt для review субагента:

```
═══════════════════════════════════════════
SYSTEM ROLE: Independent Design Reviewer
═══════════════════════════════════════════

Ты — независимый ревьюер архитектурного дизайна. Ты НЕ автор этого package.
Твоя задача — холистически (целиком) оценить package на соответствие source artifact.

ПРАВИЛА:
1. Оценивай ТОЛЬКО по фактам из source — не додумывай
2. Каждое замечание ссылается на конкретный файл и место в source
3. Не хвали — только конкретные проблемы и оценки
4. Если всё хорошо — высокий балл, не ищи проблемы искусственно

═══════════════════════════════════════════
SOURCE ARTIFACT
═══════════════════════════════════════════
{полное содержимое source PRD или SPEC + parent}

═══════════════════════════════════════════
DESIGN PACKAGE (все файлы)
═══════════════════════════════════════════
{полное содержимое README.md package}
{полное содержимое каждого sub-artifact}
{полное содержимое каждого созданного ADR}

═══════════════════════════════════════════
COVERAGE MATRIX (для критериев Requirement Coverage и Implementation Fidelity)
═══════════════════════════════════════════

Перед оценкой Requirement Coverage:

1. Извлеки список FR-NNN и NFR-NNN из source SPEC секций 5/6
2. Извлеки realizes_requirements из frontmatter КАЖДОГО sub-artifact
3. Извлеки addresses из frontmatter КАЖДОГО созданного ADR
4. Построй матрицу: каждое требование → артефакты которые его адресуют
5. В критичных проблемах перечисли непокрытые FR/NFR явно

Перед оценкой FR/NFR Implementation Fidelity:

6. Для КАЖДОГО FR извлеки из source SPEC все конкретные значения:
   - Числа (timeout, размеры, лимиты, retry counts, TTL, expiration)
   - Поля и типы данных (что должно храниться, что возвращаться)
   - Edge cases и error conditions из EARS / Gherkin acceptance
7. Для КАЖДОГО NFR определи его категорию (performance / security / reliability / usability / …)
   и найди соответствующий sub-artifact или ADR, который его материализует
8. Сравни буквально: FR-NNN.{значение/поле/условие} ↔ DESIGN.{значение/поле/условие}.
   Любое расхождение — критичная проблема.

═══════════════════════════════════════════
КРИТЕРИИ ОЦЕНКИ (X/10 каждый)
═══════════════════════════════════════════

1. Artifact Coverage (X/10) — каждый артефакт из NEEDED set действительно создан и заполнен (не stub)
2. Requirement Coverage (X/10) — каждое FR/NFR из source SPEC адресовано хотя бы одним sub-artifact:
   - FR должен быть в realizes_requirements хотя бы одного sub-artifact
   - NFR должен быть в realizes_requirements (предпочтительно в quality-scenarios.md как arc42 §10 measurable scenario)
     ИЛИ в addresses одного из созданных ADR
   - Если quality-scenarios.md в наборе: каждое NFR из source SPEC ОБЯЗАТЕЛЬНО имеет ≥ 1 сценарий Q-NNN
3. Consistency (X/10) — имена entities в ERD == schema names в OpenAPI == payload schema bases в AsyncAPI == terms в glossary == participants в sequences == container names в C4
4. Depth (X/10) — Mermaid диаграммы детальные, не placeholder; OpenAPI имеет request/response/errors; AsyncAPI имеет channels/operations/payload schemas
5. Source Alignment (X/10) — ничего не выдумано сверх source PRD/SPEC; все требования source отражены
6. Clarity (X/10) — нет placeholders, "и т.д.", "TBD"; concrete имена и поля
7. FR Implementation Fidelity (X/10) — для каждого FR из source SPEC реализующий
   sub-artifact ТОЧНО отражает требование (не просто упомянут — буквально совпадает):
   - Числовые значения (timeout, размеры, лимиты, TTL, retry, expiration) совпадают
     до конкретных значений (30 минут != 3600 секунд если SPEC говорит «30 минут»)
   - Поля и типы в data-model совпадают с описанием в FR (если FR требует
     `user_id, action, timestamp` — все три должны быть в Log entity, не два из трёх)
   - Endpoint paths, methods, status codes в OpenAPI совпадают с тем, что описано в FR
   - Edge cases из FR.acceptance (Gherkin scenarios, EARS «WHEN/IF») отражены
     в sequence diagrams или error responses в OpenAPI
   В justification ОБЯЗАТЕЛЬНО покажи проверку для каждого FR одним из форматов:
   - "FR-001 → c4-container.md (auth-service): OK"
   - "FR-005 → data-model.md: NOT OK — поле user_id отсутствует в Log entity"
   - "FR-007 → api.md (POST /sessions): NOT OK — expires_in=3600, SPEC требует 1800 (30 минут)"
8. NFR Implementation Fidelity (X/10) — для каждого NFR из source SPEC найди
   материализацию И проверь что цифры/условия совпадают:
   - Performance NFR (latency/throughput/load) → quality-scenarios.md Q-NNN с теми же
     значениями, или явно в deployment.md/api.md (rate limits, timeouts)
   - Security NFR → отражено в OpenAPI security schemes / sequence auth flows / ADR
   - Reliability NFR (availability, RPO/RTO, fault tolerance) → deployment view
     или sequence error/retry/compensation paths
   - Usability/Maintainability/Portability → ADR addresses или quality-scenarios
   В justification покажи каждое NFR одним из форматов:
   - "NFR-002 → quality-scenarios.md Q-003: OK (rate limit 100 req/min/user)"
   - "NFR-002 → NOT FOUND — rate limit 100 req/min/user не упомянут ни в одном sub-artifact"
   - "NFR-004 → deployment.md: NOT OK — SPEC требует RTO 5 мин, deployment описывает 30 мин"

═══════════════════════════════════════════
ФОРМАТ ОТВЕТА
═══════════════════════════════════════════

ОЦЕНКИ:
- Artifact Coverage:        X/10 — {brief justification}
- Requirement Coverage:     X/10 — {brief justification, mention uncovered list если есть}
- Consistency:              X/10 — {brief justification}
- Depth:                    X/10 — {brief justification}
- Source Alignment:         X/10 — {brief justification}
- Clarity:                  X/10 — {brief justification}
- FR Implementation Fidelity:  X/10 — {per-FR check, см. формат выше}
- NFR Implementation Fidelity: X/10 — {per-NFR check, см. формат выше}
- ИТОГО:                    X/10 (среднее по 8 критериям)

КРИТИЧНЫЕ ПРОБЛЕМЫ (блокеры, если есть):
1. {file:section}: {FR-NNN | NFR-NNN}: {что требует source} → {что не так в DESIGN}
   Примеры:
   - "data-model.md: FR-005 требует поле user_id в Log entity, но Log имеет только action+timestamp"
   - "api.md POST /sessions: FR-001 требует session 30 минут, expires_in=3600 (1 час)"
   - "quality-scenarios.md: NFR-002 требует rate limit 100 req/min/user, не упомянут нигде"

УЛУЧШЕНИЯ (конкретные, применимые):
1. {file:section}: {что изменить} → {как изменить}
2. ...

ВЕРДИКТ: PASS (среднее >= 8 И Requirement Coverage >= 8 И FR Implementation Fidelity >= 8 И NFR Implementation Fidelity >= 8) | IMPROVE (иначе)

Жёсткие минимумы на Requirement Coverage и FR/NFR Implementation Fidelity означают:
непокрытие требований ИЛИ расхождение конкретных значений (числа/поля/edge cases)
не компенсируются хорошими оценками других критериев — это блокеры.
```

#### Обработка результата review

**Если PASS (score >= 8):**
- Логируй score в session-log
- Phase 7

**Если IMPROVE (score < 8):**
- Запусти Improvement субагент → re-review (max 2 итерации)

#### Запуск Improvement субагента

```
Task tool:
  subagent_type: "general-purpose"
  description: "Improve DESIGN-{NNN} package based on review"
  prompt: [prompt ниже]
```

Prompt:

```
Ты получил результаты независимого ревью design package.
Задача — применить конкретные улучшения к файлам package.

PACKAGE: docs/architecture/DESIGN-{NNN}-{slug}/

РЕКОМЕНДАЦИИ РЕВЬЮ:
{полный ответ review субагента}

ИНСТРУКЦИИ:
1. Прочитай каждый указанный в рекомендациях файл (Read tool)
2. Примени ТОЛЬКО рекомендации из ревью — не добавляй лишнего
3. Сохрани обновлённые файлы (Edit tool)
4. Верни список применённых изменений
```

#### Логирование

Добавь запись в `.state/session-log.md`:

```markdown
### Quality Review: DESIGN-{NNN} (from {SOURCE-ID})
- Date: {today}
- Iteration 1: {score}/10 → {PASS|IMPROVE}
- Iteration 2: {score}/10 → {PASS|IMPROVE}  (если была)
- Files in package: {count}
- ADRs created: {count}
- Command: /pdlc:design
```

