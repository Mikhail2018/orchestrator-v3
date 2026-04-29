---
name: roadmap
description: 'Build an implementation PLAN (PLAN-NNN) from a technical SPEC — milestones, phases, dependencies — via a clean-context subagent. Use when PM mentions "create PLAN", "plan from SPEC", "roadmap", "implementation plan", "milestones", "create roadmap", "сделай план", or any request to sequence SPEC work into execution phases. Trigger liberally — under-triggering lets the agent jump straight to TASKs without a milestone view; over-triggering is recoverable (PM can delete or regenerate).'
argument-hint: "[SPEC-XXX]"
cli_requires: "task_tool"
---

# /pdlc:roadmap [SPEC-XXX] — План реализации через субагент

Создание плана реализации на основе технической спецификации через изолированный субагент.

## Использование

```
/pdlc:roadmap SPEC-001   # План для конкретной спеки
/pdlc:roadmap            # Выбрать из доступных ready SPEC
```

## Когда нужен roadmap

**Нужен PLAN:**
- Крупная инициатива с несколькими фазами
- Сложные зависимости между задачами
- Работа на несколько недель
- Требуется координация компонентов

**Не нужен PLAN (иди сразу в /pdlc:tasks):**
- Простая фича (2-5 задач)
- Линейная последовательность работ
- Нет сложных зависимостей

## Архитектура с субагентом

```
┌─────────────────────────────────────────────────────────────┐
│  PM: /pdlc:roadmap SPEC-001                                 │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  1. Валидация: SPEC со статусом ready                       │
│  2. Читает SPEC + связанный PRD                             │
│  3. Читает knowledge.json                                   │
│  4. Формирует prompt с системным промптом                   │
│  5. Запускает Task tool: subagent_type="general-purpose"    │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  СУБАГЕНТ general-purpose (чистый контекст)                 │
│                                                             │
│  System role: Product Delivery Roadmap Architect            │
│  Input: SPEC + PRD + project context                        │
│                                                             │
│  Делает:                                                    │
│  1. Анализирует техническую спецификацию                    │
│  2. Разбивает на логические фазы                            │
│  3. Определяет roadmap items с зависимостями                │
│  4. Выявляет критический путь                               │
│  5. Создаёт PLAN файл                                       │
│  6. Возвращает: путь, summary, items count                  │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  1. Обновляет PROJECT_STATE.json                            │
│  2. Обновляет counters.json                                 │
│  3. SPEC.children += PLAN (статус НЕ меняется)              │
└─────────────────────────────────────────────────────────────┘
```

## Алгоритм работы основного агента

### 1. Валидация

1. Прочитай `.state/PROJECT_STATE.json`
2. Найди SPEC со статусом `ready`
3. Если указан SPEC-XXX — проверь что он `ready`
4. Если не указан:
   - Если есть один ready SPEC → используй его
   - Если несколько → покажи список и спроси какой
   - Если нет ready SPEC → сообщи и предложи /pdlc:spec

```
Нет готовых спецификаций для создания плана.

Доступные действия:
   → /pdlc:spec для создания спецификации
   → /pdlc:state для обзора проекта
```

### 2. Подготовка контекста

Прочитай и собери:
1. **SPEC файл** — полное содержимое
2. **Связанный PRD** (если есть parent PRD)
3. **Design package (опционально)**: если SPEC имеет ребёнка типа `DESIGN-PKG`, прочитай:
   - `{package.dir}README.md` — обзор и Solution Strategy
   - `{package.dir}{package.manifest}` (всегда `manifest.yaml`) — **machine-readable** список components, sub-artifacts и `realizes_requirements`. Используется в Phase 6 review для критериев Architecture Coverage / Component-Item Mapping. Без manifest эти критерии не считаются.
   - `{package.dir}api.md` (если есть) — OpenAPI контракт. Используется в Phase 6 для критерия API Coverage.

   Передай README.md, manifest.yaml и api.md в субагент как дополнительный контекст: ему легче декомпозировать задачи, когда есть точный API и список containers/components. Если у DESIGN-PKG статус `draft` или `waiting_pm` — выведи предупреждение PM (но не блокируй выполнение). `package.dir` и `package.manifest` бери из `PROJECT_STATE.json` записи DESIGN-PKG.
4. **Knowledge base** (`.state/knowledge.json`):
   - `projectContext` — описание проекта
   - `techStack` — технологии
   - `patterns` — используемые паттерны
   - `decisions` — принятые решения
5. **Шаблон плана** (`docs/templates/plan-template.md`)

### 2.5. Design Gate (условная блокировка)

1. **Определи source SPEC:**
   - Аргумент `/pdlc:roadmap` — всегда SPEC-NNN → `spec_id` = аргумент

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
   - Собери `needed_artifacts` set

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

При выборе waiver: PLAN создаётся, но downstream задачи
получат design_waiver: true.
═══════════════════════════════════════════
```

6. **Дождись ответа PM:**
   - PM выбирает `/pdlc:design` → прервать создание плана, PM запускает design
   - PM выбирает waiver:
     a. Добавь `design_waiver: true` в SPEC frontmatter (persistent marker)
     b. Последующий `/pdlc:tasks PLAN-NNN` увидит waiver на SPEC → не переспросит

7. **Если `needed_artifacts` пустой → SKIP** (design не нужен)

### 3. Формирование prompt для субагента

Полный шаблон промпта (system role, разбивка на phases, scope per phase,
items с FR/NFR mapping, dependencies, success criteria, system_boundary)
— в:

**`skills/roadmap/references/subagent-prompt.md`** —
читай через Read tool ПЕРЕД формированием prompt.

⛔ **НЕ передавай** содержимое промпта от руки.

### 4. Запуск субагента

Используй Task tool:
```
Task tool:
  subagent_type: "general-purpose"
  description: "Create PLAN from SPEC-XXX"
  prompt: [сформированный prompt выше]
```

### 5. Обработка результата

После завершения субагента:

1. **Вычисли next-id для PLAN** по протоколу из
   `skills/tasks/references/compute-next-id.md`
   (единый max по `.state/counters.json`, `PROJECT_STATE.artifactIndex`
   и file-scan `docs/plans/PLAN-*.md`). При **Counter drift** — АБОРТ
   с рекомендацией `python3 {plugin_root}/scripts/pdlc_sync.py . --apply --yes`.
2. **Write-guard.** Перед сохранением PLAN-файла, сгенерированного
   субагентом, проверь, что `docs/plans/PLAN-{N}-slug.md` не существует
   и что `PLAN-{N}` нет в `state.artifactIndex`. При коллизии — АБОРТ.
3. Инкрементируй счётчик PLAN (`counters.json[PLAN] = N`).
4. Обнови `.state/PROJECT_STATE.json`:
   - Добавь PLAN в `artifacts` со статусом `ready`
   - Добавь PLAN в `readyToWork`
   - Обнови SPEC: добавь PLAN в `children`
   - **НЕ меняй SPEC.status на `done`!** SPEC — living document по ISO/IEC/IEEE 29148:
     - Если SPEC.status == `ready` — оставить `ready` (или поднять до `accepted`, если PM баселайнит)
     - Если SPEC.status == `accepted` — оставить `accepted`
     - Если SPEC.status == `draft` — это ошибка PM, выведи warning «SPEC должен быть ready/accepted перед roadmap»
     - Правило consistent с `/pdlc:spec` и `/pdlc:design` Phase 7: parent верхнеуровневый артефакт не закрывается из-за создания child

### 6. Quality Review Loop (обязательно!)

Полный алгоритм quality review PLAN — в:

**`skills/roadmap/references/quality-review-loop.md`** —
читай через Read tool ПОСЛЕ создания PLAN и ПЕРЕД переводом в `ready`.

Loop обязателен: PLAN не переходит в `ready` без прохождения review.

## Формат вывода

Все варианты вывода — в:

**`skills/roadmap/references/output-formats.md`** —
читай через Read tool ПЕРЕД печатью соответствующего блока.

## Структура roadmap item

Каждый item в PLAN должен содержать:

```markdown
### MVP-1.1: Project scaffolding

**Description:** Настройка базовой структуры проекта

**Deliverable:**
- Инициализированный проект с TypeScript
- Настроенный ESLint/Prettier
- Базовая структура директорий

**Dependencies:** None

**Complexity:** S

**Notes:** Использовать template из internal-tools

<!-- Опциональные поля (рекомендуются если есть DESIGN PACKAGE — улучшают
     Component-Item Mapping в Phase 6 review): -->
**component_refs:** [auth-service, api-gateway]   <!-- из manifest.yaml C4 -->
**realizes_requirements:** [SPEC-001.FR-001, SPEC-001.FR-005]   <!-- composite IDs из SPEC секций 5/6 -->
```

## Нумерация items

Формат: `{PHASE}-{NUMBER}.{SUB}`

Примеры:
- `MVP-1.1` — первый item фазы 1 (Setup)
- `MVP-2.3` — третий item фазы 2 (Core)
- `POLISH-4.2` — второй item фазы 4 (Polish)

Это позволяет:
- Группировать items по фазам
- Легко ссылаться в зависимостях
- Сортировать по порядку

## Важно

- Roadmap items — это НЕ задачи, а chunks работы для декомпозиции
- Каждый item декомпозируется в 2-5 TASK командой `/pdlc:tasks`
- Фазы должны давать инкрементальную ценность
- Критический путь определяет минимальное время реализации
- При создании PLAN SPEC.children += PLAN, но **статус SPEC НЕ меняется** (SPEC — living document, ISO/IEC/IEEE 29148)
- Субагент работает в чистом контексте — передавай весь контекст
- **E2E + test-kit items** — обязательны только если `quality.e2e.enabled == true` в `.state/knowledge.json`. Пути и expectations берутся из того же конфига
- **DESIGN PACKAGE traceability** — если у source SPEC есть ребёнок `DESIGN-PKG`, основной агент в Phase 2 читает `manifest.yaml` (источник списков components/entities/endpoints) и пробрасывает его в субагент plan + Phase 6 review. Review применяет conditional критерии Architecture Coverage / Component-Item Mapping / API Coverage и требует Architecture Coverage ≥ 8 как hard floor. Roadmap items могут опционально содержать `component_refs:` и `realizes_requirements:` — это улучшает оценку Component-Item Mapping
