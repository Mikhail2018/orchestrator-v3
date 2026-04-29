# Quality Review Loop — `/pdlc:spec`

Полный алгоритм quality review SPEC: запуск reviewer-субагента, обработка score, IMPROVE-итерация, max-iterations gate, формат сохраняемого review. SKILL.md ссылается сюда.

---

### 6. Quality Review Loop (обязательно!)

После создания SPEC (если статус `ready`) запусти независимый ревью:

```
┌──────────────────────────────────────────┐
│  REVIEW SUBAGENT (чистый контекст)        │
│  INPUT:  PRD/FEAT (исходный документ)     │
│  OUTPUT: созданная SPEC                   │
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
│  → Применяет улучшения к SPEC файлу      │
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
1. Исходный документ (PRD/FEAT) — полное содержимое
2. Созданную SPEC — полное содержимое

Запусти Task tool:
```
Task tool:
  subagent_type: "general-purpose"
  description: "Quality review SPEC-XXX vs {PRD-XXX/FEAT-XXX}"
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
INPUT (исходный документ)
═══════════════════════════════════════════
{полное содержимое PRD или FEAT}

═══════════════════════════════════════════
OUTPUT (результат для ревью)
═══════════════════════════════════════════
{полное содержимое созданной SPEC}

═══════════════════════════════════════════
КРИТЕРИИ ОЦЕНКИ
═══════════════════════════════════════════
- Coverage (все требования INPUT покрыты): X/10
- Specifiability (FR/NFR имеют стабильные ID, NFR измеримы): X/10
- EARS Compliance (каждое FR следует одному из 5 паттернов): X/10
- Testability (каждое FR имеет ≥ 1 Gherkin AC, falsifiable): X/10
- Language Neutrality (нет hardcoded TS/SQL/REST вне techStack): X/10
- Traceability (каждое FR ссылается на PRD-секцию; раздел 9 заполнен): X/10
- Consistency (с patterns/glossary из knowledge): X/10
- Clarity (нет двусмысленностей, "и т.д."): X/10

═══════════════════════════════════════════
ФОРМАТ ОТВЕТА
═══════════════════════════════════════════

ОЦЕНКИ:
- Coverage: X/10 — {brief justification}
- Specifiability: X/10 — {brief justification}
- EARS Compliance: X/10 — {brief justification}
- Testability: X/10 — {brief justification}
- Language Neutrality: X/10 — {brief justification}
- Traceability: X/10 — {brief justification}
- Consistency: X/10 — {brief justification}
- Clarity: X/10 — {brief justification}
- ИТОГО (среднее): X/10

КРИТИЧНЫЕ ПРОБЛЕМЫ (блокеры, если есть):
1. {problem}: {what's in INPUT} → {what's missing/wrong in OUTPUT}

УЛУЧШЕНИЯ (конкретные, применимые):
1. {section/line}: {what to change} → {how to change}
2. ...

ВЕРДИКТ: PASS (среднее >= 8 И EARS Compliance >= 7 И Testability >= 7 И Language Neutrality >= 7) | IMPROVE

Пояснение: даже при среднем >= 8, если хотя бы один из трёх критичных
критериев (EARS Compliance, Testability, Language Neutrality) ниже 7 —
вердикт IMPROVE. Эти свойства не компенсируются другими оценками.
```

#### Обработка результата review

**Если PASS (среднее >= 8 И EARS Compliance >= 7 И Testability >= 7 И Language Neutrality >= 7):**
- Логируй score в session-log
- Продолжай к финальному выводу

**Если IMPROVE (любое из условий PASS не выполнено):**
- Запусти Improvement субагент (см. ниже)
- После улучшения — повтори review (макс. 2 итерации)

#### Запуск Improvement субагента

```
Task tool:
  subagent_type: "general-purpose"
  description: "Improve SPEC-XXX based on review"
  prompt: [prompt ниже]
```

Prompt для improvement субагента:
```
Ты получил результаты независимого ревью SPEC.
Твоя задача — применить конкретные улучшения к файлу SPEC.

ФАЙЛ ДЛЯ УЛУЧШЕНИЯ: {path to SPEC file}

РЕКОМЕНДАЦИИ РЕВЬЮ:
{полный ответ review субагента}

ИНСТРУКЦИИ:
1. Прочитай текущий файл SPEC (Read tool)
2. Примени ТОЛЬКО рекомендации из ревью — не добавляй лишнего
3. Сохрани обновлённый файл (Edit tool)
4. Верни список применённых изменений
```

#### Логирование в session-log

Добавь запись в `.state/session-log.md`:
```markdown
### Quality Review: SPEC-{ID} (from {PARENT-ID})
- Date: {today}
- Iteration 1: {score}/10 → {PASS|IMPROVE}
- Iteration 2: {score}/10 → {PASS|IMPROVE} (если была)
- Command: /pdlc:spec
```

