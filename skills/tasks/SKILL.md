---
name: tasks
description: 'Decompose a PLAN / SPEC / FEAT / BUG / DEBT / CHORE into atomic TASK-NNN items ready for the implement flow, via a clean-context subagent. Use when PM mentions "create tasks", "generate TASKs", "break down into tasks", "tasks from PLAN", "break this into tasks", "декомпозиция", "сделай таски", or any request to explode upstream artefacts into executable work. Trigger liberally — under-triggering forces ad-hoc task-creation in chat that drifts from backlog conventions; over-triggering is recoverable (PM can delete).'
argument-hint: "[PLAN-XXX | SPEC-XXX | FEAT-XXX | BUG-XXX | DEBT-XXX | CHORE-XXX]"
cli_requires: "task_tool"
---

# /pdlc:tasks [PLAN-XXX | SPEC-XXX | FEAT-XXX | BUG-XXX | DEBT-XXX | CHORE-XXX] — Создание задач через субагент

Создание атомарных задач из плана, спецификации, Feature Brief, Bug Report,
Tech Debt или Chore через изолированный субагент.

## Использование

```
/pdlc:tasks PLAN-001   # Задачи из детального плана (итерация по items)
/pdlc:tasks SPEC-001   # Задачи из спецификации
/pdlc:tasks FEAT-001   # Задачи напрямую из фичи (для простых случаев)
/pdlc:tasks BUG-001    # Задача из бага (обычно 1 TASK)
/pdlc:tasks DEBT-001   # Задачи из техдолга (обычно 1-3 TASK)
/pdlc:tasks CHORE-001  # Задача из chore (обычно 1 TASK)
/pdlc:tasks            # Выбрать из доступных ready артефактов
```

## Когда что использовать

| Источник | Когда использовать | Типичное кол-во TASKs |
|----------|-------------------|-----------------------|
| PLAN | Крупная инициатива с фазами и зависимостями | 5-20 |
| SPEC | Техническая работа, требующая архитектуры | 3-10 |
| FEAT | Простая фича, понятная из описания | 2-5 |
| BUG | Багфикс с конкретным воспроизведением | 1 (реже 2-3) |
| DEBT | Рефакторинг, обычно ленивая декомпозиция после регистрации | 1-3 |
| CHORE | Простая задача, когда `--no-task` использовался при регистрации | 1 |

## Архитектура с субагентом

### Для PLAN (итерация по roadmap items)

```
┌─────────────────────────────────────────────────────────────┐
│  PM: /pdlc:tasks PLAN-001                                   │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  1. Валидация: PLAN со статусом ready                       │
│  2. Читает PLAN + SPEC + PRD                                │
│  3. Извлекает список roadmap items                          │
│  4. Запускает субагенты для ВСЕХ items (параллельно)        │
└─────────────────────────────────────────────────────────────┘
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│  СУБАГЕНТ     │ │  СУБАГЕНТ     │ │  СУБАГЕНТ     │
│  Item MVP-1.1 │ │  Item MVP-1.2 │ │  Item MVP-2.1 │
│  → 3 TASKs    │ │  → 2 TASKs    │ │  → 4 TASKs    │
└───────────────┘ └───────────────┘ └───────────────┘
            │           │           │
            └───────────┼───────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  1. Собирает ВСЕ TASK в память (НЕ сохраняет файлы)        │
│  2. Consolidated PM Checkpoint (один на весь PLAN)          │
│  3. После подтверждения — сохраняет файлы                   │
│  4. Обновляет PROJECT_STATE.json                            │
│  5. Обновляет counters.json                                 │
└─────────────────────────────────────────────────────────────┘
```

### Для SPEC/FEAT/BUG/DEBT/CHORE (один субагент)

```
┌─────────────────────────────────────────────────────────────┐
│  PM: /pdlc:tasks SPEC-001 / FEAT-001 / BUG-001 /            │
│                  DEBT-001 / CHORE-001                       │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  1. Валидация: SPEC/FEAT/BUG/DEBT/CHORE со статусом ready   │
│  2. Читает документ + knowledge.json                        │
│  3. Запускает субагент для декомпозиции                     │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  СУБАГЕНТ general-purpose (чистый контекст)                 │
│                                                             │
│  System role: Task Planner                                  │
│  Input: SPEC/FEAT/BUG/DEBT/CHORE + project context          │
│                                                             │
│  Делает:                                                    │
│  1. Анализирует требования                                  │
│  2. Читает затронутые файлы кода                            │
│  3. Декомпозирует в атомарные задачи                        │
│  4. Определяет зависимости                                  │
│  5. Проводит self-review постановки                         │
│  6. Возвращает: список TASKs                                │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  1. Собирает ВСЕ TASK в память (НЕ сохраняет файлы)        │
│  2. Consolidated PM Checkpoint если > 3 задач               │
│  3. После подтверждения — сохраняет файлы                   │
│  4. Обновляет PROJECT_STATE.json                            │
│  5. Обновляет counters.json                                 │
└─────────────────────────────────────────────────────────────┘
```

## Алгоритм работы основного агента

### 1. Валидация

1. Прочитай `.state/PROJECT_STATE.json`
2. Найди PLAN, SPEC, FEAT, BUG, DEBT или CHORE со статусом `ready`
3. Если указан ID — проверь что он `ready`
4. Если не указан:
   - Покажи список всех ready артефактов (PLAN, SPEC, FEAT, BUG, DEBT, CHORE)
   - Спроси какой использовать
   - Если нет ready → предложи альтернативы

```
Нет готовых артефактов для создания задач.

Доступные действия:
   → /pdlc:feature для добавления фичи
   → /pdlc:defect для репорта бага
   → /pdlc:spec для создания спецификации
   → /pdlc:state для обзора проекта
```

### 2. Подготовка контекста

Прочитай и собери:
1. **Исходный документ** (PLAN, SPEC, FEAT, BUG, DEBT или CHORE)
2. **Связанные документы**:
   - Для PLAN: SPEC + PRD
   - Для SPEC: PRD (если есть parent)
   - Для FEAT: ничего дополнительно
   - Для BUG: ничего дополнительно (баг самодостаточен)
   - Для DEBT: ничего дополнительно (техдолг самодостаточен, parent SPEC отсутствует)
   - Для CHORE: ничего дополнительно (chore самодостаточен, parent SPEC отсутствует)
3. **Design package (опционально)**: если SPEC имеет ребёнка типа `DESIGN-PKG`, прочитай README.md package'а и (если есть) `api.md` — это OpenAPI контракт. Передай как дополнительный контекст в субагент: точные endpoints/schemas помогают создавать корректные TASKs (правильные routes, request/response shapes, error codes). Если у DESIGN-PKG статус `draft`/`waiting_pm` — выведи предупреждение PM, но не блокируй.
4. **Out of scope и Constraints** (из source SPEC, если source — SPEC или PLAN→SPEC):
   - Извлеки **Out of scope** из SPEC §1 (Purpose & Scope)
   - Извлеки **Constraints** (C-N) и **Dependencies** (D-N) из SPEC §4
   - Передай в субагент: out-of-scope items определяют что НЕ должно стать TASK;
     constraints определяют технологические рамки для implementation steps
   - Если source — FEAT/BUG/DEBT/CHORE без SPEC в chain: "N/A"
5. **System boundary** (из SPEC frontmatter, если source — SPEC или PLAN→SPEC):
   - Извлеки `system_boundary` и `external_systems` из SPEC frontmatter
   - Если `system_boundary` задан — передай в субагент: НЕ создавай TASK для external_systems,
     реализуй ТОЛЬКО system_boundary
   - Если source — FEAT/BUG/DEBT/CHORE без SPEC: "N/A"
   **Integration Contract Pre-check** (если `external_systems` non-empty):
   - Для каждой записи в `external_systems` проверь: `contract_ref` указан и файл существует?
   - Если `contract_ref` пуст или файл не найден → добавь Open Question (НЕ блокируй генерацию):
     "Контракт для [системы] отсутствует (contract_ref: [значение]). Задачи создаются,
      но интеграционные тесты будут неполными без контракта."
   - Выведи предупреждение PM перед PM Checkpoint
6. **Knowledge base** (`.state/knowledge.json`):
   - `projectContext`, `patterns`, `antiPatterns`, `decisions`
   - `glossary` — ubiquitous language project-wide (federated из DESIGN packages). Передавай в субагент как source-of-truth для именования сущностей в коде и тестах.
7. **Шаблон задачи** (`docs/templates/task-template.md`)
8. **Текущие счётчики** (`.state/counters.json`)

### 2.5. Design Gate (условная блокировка)

1. **Определи source SPEC:**
   - source = SPEC-NNN → `spec_id` = source
   - source = PLAN-NNN → `spec_id` = parent PLAN'а (через `PROJECT_STATE.artifacts[plan_id].parent`)
   - source = FEAT-NNN / BUG-NNN → **SKIP** (design не обязателен для простых фич и багов)
   - Если SPEC не найден → **SKIP**

2. **Проверь наличие DESIGN-PKG:**
   - Проверь `design_package` field в SPEC frontmatter, ИЛИ
   - Сканируй `docs/architecture/*/manifest.yaml` на `parent: {spec_id}`
   - Если DESIGN-PKG существует:
     - Если `design_waiver: true` в SPEC → **сбрось**: установи `design_waiver: false`
       (waiver был временным, теперь design создан — enforcement восстанавливается)
     - **SKIP** (design уже сделан)

3. **Проверь SPEC frontmatter на `design_waiver`:**
   - Если `design_waiver: true` → **SKIP** (PM дал waiver, DESIGN-PKG ещё не создан)

4. **Lightweight trigger detection:**
   - Прочитай `skills/design/references/conditional-triggers.md`
   - Сканируй содержимое SPEC на trigger patterns из reference
   - Собери `needed_artifacts` set (какие типы артефактов triggered: erd, openapi, sequence, etc.)

5. **Если `needed_artifacts` НЕ пустой → БЛОКИРОВКА с waiver:**

```
═══════════════════════════════════════════
⛔ DESIGN GATE
═══════════════════════════════════════════
{spec_id} имеет архитектурные триггеры,
но DESIGN package не создан.

Обнаруженные триггеры:
  • {тип} — {краткое описание что обнаружено}
  • ...

Варианты:
  1. /pdlc:design {spec_id} — создать design package (рекомендуется)
  2. Продолжить без дизайна (explicit waiver)

При выборе waiver: задачи создаются с design_waiver: true.
Enforce-проверки Design Conformance при review
не применяются для этих задач.
═══════════════════════════════════════════
```

6. **Дождись ответа PM:**
   - PM выбирает `/pdlc:design` → прервать создание задач, PM запускает design
   - PM выбирает waiver:
     a. Добавь `design_waiver: true` в SPEC frontmatter (persistent marker)
     b. Все TASKs, создаваемые из этого SPEC, наследуют `design_waiver: true`
     c. При повторном вызове `/pdlc:tasks PLAN-NNN` → gate проверяет SPEC frontmatter → видит `design_waiver: true` → SKIP (не переспрашивает)

7. **Если `needed_artifacts` пустой → SKIP** (SPEC не имеет архитектурных триггеров, design не нужен)

### 2.6. Pre-check: design_refs mapping (основной агент)

Если DESIGN-PKG существует И `design_waiver != true`:

1. Прочитай `manifest.yaml` DESIGN-пакета
2. Извлеки все FR/NFR из source SPEC (секции 5 и 6)
3. Для каждого FR/NFR проверь: есть ли хотя бы один артефакт в `manifest.artifacts[]`
   где `realizes_requirements` содержит этот FR/NFR?
4. Если есть unmapped requirements (FR/NFR не покрыты ни одним артефактом в manifest):
   - **STOP** — НЕ запускай субагент
   - Спроси PM:
   ```
   ═══════════════════════════════════════════
   ⛔ UNMAPPED REQUIREMENTS
   ═══════════════════════════════════════════
   Следующие requirements из {spec_id} не покрыты
   ни одним артефактом в {DESIGN-NNN}/manifest.yaml:

     • FR-003 — {title}
     • NFR-002 — {title}

   Действие:
     Обновите manifest.yaml — добавьте unmapped requirements
     в realizes_requirements соответствующих артефактов.
     После обновления повторите /pdlc:tasks.
   ═══════════════════════════════════════════
   ```
   - Дождись обновления manifest.yaml PM'ом → повторить шаг 2.6
5. Если все requirements mapped → продолжить к шагу 3

Если DESIGN-PKG не существует ИЛИ `design_waiver: true` → **SKIP**

### 3. Формирование prompt для субагента

Полные шаблоны промптов (для PLAN-items, SPEC/FEAT, BUG/DEBT/CHORE) вынесены в:

**`skills/tasks/references/subagent-prompts.md`** —
читай через Read tool ПЕРЕД формированием prompt и инжектируй секции
по мере наполнения данными из шага 2 (Подготовка контекста) и 2.5/2.6
(Design Gate / design_refs mapping).

Что инжектируется:
- **Для PLAN-item:** конкретный roadmap item + контекст из SPEC + Out of Scope
  (SPEC §1) + Constraints/Dependencies (SPEC §4) + system_boundary +
  knowledge.{patterns, antiPatterns, decisions, glossary} + task-template.md +
  next_task_id (из counters.json через protocol `references/compute-next-id.md`).
- **Для SPEC/FEAT:** полное содержимое документа + те же блоки SPEC §1/§4 +
  system_boundary (если есть SPEC в parent chain) + knowledge.* + task-template.md.
- **Для BUG/DEBT/CHORE:** содержимое source-файла + три подстановки
  (`{SOURCE_TYPE}`, `{SOURCE_FILE_PATH}`, `{SYSTEM_ROLE}`) + те же
  knowledge-блоки + task-template.md.

⛔ **НЕ передавай** содержимое шаблонов от руки. Шаблон — единая точка
обновления; правки делаются в reference-файле.

Принципы декомпозиции (АТОМАРНОСТЬ, КОНКРЕТНОСТЬ, ЗАВИСИМОСТИ, ПОЛНОТА,
БЕЗОПАСНОСТЬ ИЗМЕНЕНИЙ, ВЕРИФИКАЦИЯ ЧЕРЕЗ КОД, ПОКРЫТИЕ ТРЕБОВАНИЙ,
SELF-REVIEW) — раскрыты в reference-файле подробно. SKILL.md не
дублирует их, чтобы не создавать двух источников правды.

### 4. Запуск субагента

#### Для PLAN (итерация):
```
Для каждого roadmap item в PLAN:
  Task tool:
    subagent_type: "general-purpose"
    description: "Create TASKs for {item_id}"
    prompt: [prompt для конкретного item]
```

#### Для SPEC/FEAT/BUG/DEBT/CHORE:
```
Task tool:
  subagent_type: "general-purpose"
  description: "Create TASKs from {SPEC-XXX/FEAT-XXX/BUG-XXX/DEBT-XXX/CHORE-XXX}"
  prompt: [prompt с полным документом]
```

### 5. PM Checkpoint (consolidated)

**При создании > 3 задач — ОБЯЗАТЕЛЬНАЯ остановка.**

Ключевой принцип: **ОДИН checkpoint на весь запуск**, а не на каждый roadmap item.
Все субагенты завершают работу, все TASKs собираются в память, и только потом PM видит
консолидированный обзор и принимает решение. Файлы сохраняются ТОЛЬКО после подтверждения.

#### Формат consolidated checkpoint

```
═══════════════════════════════════════════
НУЖНО РЕШЕНИЕ PM
═══════════════════════════════════════════
Контекст: Создание задач для {SOURCE-ID} ({title})

Создано задач: {N} {из M roadmap items — если PLAN}

ФАЗА 1: Setup ({K} задач)
  • TASK-001: Project scaffolding [P0]
  • TASK-002: Database setup [P0]
  • TASK-003: Auth integration [P1]

ФАЗА 2: Core ({K} задач)
  • TASK-004: User model [P1]
  • TASK-005: User service [P1]
  • TASK-006: Permission system [P1] → ждёт TASK-004

ФАЗА 3: Tests ({K} задач)
  • TASK-007: Unit tests [P2] → ждёт TASK-005
  ...

COVERAGE:
  FR покрыты: 12/12
  NFR покрыты: 4/5 (NFR-005 не покрыто ⚠️)

Зависимости: {N} cross-phase dependencies

Действия:
  1 — Сохранить все
  2 — Изменить (открыть обсуждение)
  3 — Отмена

→ "1" / "2" / "3"
═══════════════════════════════════════════
```

#### Группировка по фазам

При выводе TASKs группируй по логическим фазам в порядке выполнения:

| Фаза | Содержимое |
|------|------------|
| Setup | Scaffolding, конфигурация, зависимости |
| Core | Основная бизнес-логика, модели, сервисы |
| API | Endpoints, middleware, контроллеры |
| UI | Компоненты, страницы, стили |
| Tests | Unit, integration, e2e тесты |
| Integration | Связывание подсистем, миграции |

Если источник — PLAN с roadmap items, фазы определяются из самих items (phase из PLAN).
Если источник — SPEC/FEAT, фазы определяются из логических групп (Setup → Core → Tests).

#### Per-item mode (опционально)

Если PM явно запрашивает per-item checkpoint (например, при дебаге декомпозиции
конкретного item), основной агент может переключиться в per-item mode:
показывать checkpoint после каждого обработанного roadmap item.
Этот режим НЕ используется по умолчанию — только по явному запросу PM.

### 6. Обработка результата

После подтверждения PM в consolidated checkpoint (или сразу если ≤ 3 задач):

1. **Вычисли next-id для TASK** по протоколу из
   `skills/tasks/references/compute-next-id.md`
   (единый max по `.state/counters.json`, `PROJECT_STATE.artifactIndex`
   и file-scan `tasks/TASK-*.md`). При **Counter drift** — АБОРТ с
   рекомендацией `python3 {plugin_root}/scripts/pdlc_sync.py . --apply --yes`.
2. **Batch-режим.** После первого `next_id` внутри цикла присваивай
   `next_id, next_id+1, next_id+2, …` БЕЗ повторного чтения диска.
   Это безопасно: никто не создаёт TASK параллельно в той же сессии.
3. **Write-guard на каждый файл.** Перед `Write tasks/TASK-{k}-slug.md`
   проверь, что файл не существует и что `TASK-{k}` нет в
   `state.artifactIndex`. При коллизии — АБОРТ (до IO всего батча или
   после частичной записи: любой guard-fail останавливает оставшуюся
   пачку и сообщает, сколько файлов уже создано).
4. Инкрементируй счётчик TASK на количество созданных
   (`counters.json[TASK] = last_written_n`).
5. Обнови `.state/PROJECT_STATE.json`:
   - Добавь все TASK в `artifacts`
   - Задачи без зависимостей → `ready` + в `readyToWork`
   - Задачи с зависимостями → `ready` но НЕ в `readyToWork`
   - Обнови parent: добавь TASK в `children`
   - Если PLAN → parent статус `in_progress`
   - Если SPEC/FEAT/BUG/DEBT/CHORE → parent статус остаётся `ready`

## Формат вывода

### При создании ≤ 3 задач (без checkpoint)

```
═══════════════════════════════════════════
ЗАДАЧИ СОЗДАНЫ
═══════════════════════════════════════════

Из: FEAT-001 (Добавить экспорт в PDF)
Создано задач: 3

ГОТОВЫ К РАБОТЕ:
   • TASK-001: Создать сервис экспорта [P1]
   • TASK-002: Добавить UI кнопку [P1]

ЖДУТ ЗАВИСИМОСТИ:
   • TASK-003: Интеграционные тесты [P2]
     (ждёт: TASK-001, TASK-002)

═══════════════════════════════════════════
СЛЕДУЮЩИЙ ШАГ:
   → /pdlc:implement TASK-001 — начать реализацию
   → /pdlc:continue — автономная работа
═══════════════════════════════════════════
```

### После consolidated PM Checkpoint

```
═══════════════════════════════════════════
ЗАДАЧИ СОЗДАНЫ
═══════════════════════════════════════════

Из: PLAN-001 (MVP Implementation)
Roadmap items обработано: 5
Создано задач: 15 (1 consolidated checkpoint)

ФАЗА 1: Setup (3 задачи)
   ✓ TASK-001: Project scaffolding [P0]
   ✓ TASK-002: Database setup [P0]
   ✓ TASK-003: Auth integration [P1]

ФАЗА 2: Core (7 задач)
   ✓ TASK-004: User model [P1]
   ✓ TASK-005: User service [P1]
   ...

COVERAGE:
   FR: 12/12 ✓
   NFR: 4/5 (NFR-005 ⚠️)

ГОТОВЫ К РАБОТЕ: 5
   • TASK-001, TASK-004, TASK-008...

═══════════════════════════════════════════
СЛЕДУЮЩИЙ ШАГ:
   → /pdlc:implement TASK-001 — начать реализацию
   → /pdlc:continue — автономная работа
═══════════════════════════════════════════
```

## Структура TASK файла

```markdown
---
id: TASK-001
title: "Создать сервис экспорта PDF"
status: ready
created: 2026-02-02
parent: FEAT-001
priority: P1
depends_on: []
blocks: [TASK-003, TASK-004]
requirements: [SPEC-001.FR-001]   # composite ID из parent SPEC/PRD/FEAT секций 5/6
design_refs: []          # пути внутри DESIGN-PKG (если у parent SPEC есть design package)
---

# Задача: Создать сервис экспорта PDF

## Контекст

**Parent:** [[FEAT-001]]

**Зачем:** Пользователи хотят экспортировать отчёты в PDF для печати и sharing.

## Что нужно сделать

1. [ ] Создать `src/services/pdf-export.ts`
2. [ ] Реализовать функцию `exportToPdf(data: ReportData): Promise<Buffer>`
3. [ ] Использовать библиотеку jsPDF
4. [ ] Добавить форматирование таблиц
5. [ ] Добавить header/footer

## Файлы для изменения

- `src/services/pdf-export.ts` — создать новый файл
- `src/services/index.ts` — добавить экспорт
- `package.json` — добавить jsPDF dependency

## Критерии приёмки

- [ ] Функция возвращает валидный PDF buffer
- [ ] Таблицы корректно форматируются
- [ ] Русский текст отображается правильно
- [ ] Unit тесты покрывают основные сценарии

## Edge cases

- Пустые данные
- Очень большие таблицы (100+ строк)
- Спецсимволы в тексте

## Тесты

### Unit тесты
- [ ] `exportToPdf` с пустыми данными
- [ ] `exportToPdf` с большой таблицей
- [ ] Форматирование дат и чисел
```

## Важно

- **PM Checkpoint обязателен при > 3 задачах** — всегда **consolidated** (один на весь запуск, НЕ per-item)
- Все TASKs собираются в память до сохранения файлов. Файлы создаются ТОЛЬКО после подтверждения PM
- Задачи должны быть атомарными (можно сделать за один подход)
- Для FEAT обычно 2-5 задач достаточно
- Для BUG обычно 1 задача (фикс + тесты вместе)
- Чётко описывай что нужно сделать
- Указывай конкретные файлы где возможно
- **Субагент ОБЯЗАН читать затронутый код** — не полагаться только на описание бага/фичи
- **Субагент ОБЯЗАН провести self-review** — задача должна быть самодостаточной для автономного агента
- `/pdlc:implement` работает только с TASK
- При итерации по PLAN — запускай субагенты параллельно для ВСЕХ items, собирай результаты, показывай один checkpoint
- Субагент работает в чистом контексте — передавай весь необходимый контекст
