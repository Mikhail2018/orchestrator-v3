---
name: design
description: Create a doc-as-code design package from a PRD or SPEC. Conditionally generates C4 diagrams (Context/Container/Component), sequence diagrams, ER diagram + Data Dictionary, OpenAPI 3.0, AsyncAPI 3.0, ADRs, domain glossary, state diagrams, and deployment view as Mermaid-rendered Markdown files. Use when PM mentions "design", "architecture diagrams", "doc-as-code artifacts", "C4", "ERD", "OpenAPI spec", "AsyncAPI", "event-driven", "Kafka", "message broker", "sequence diagram", "state machine", "domain glossary", "ADR", or before handing a SPEC to another team. Trigger liberally — undertriggering loses architectural value, overtriggering is recoverable (PM can delete).
argument-hint: "[PRD-XXX | SPEC-XXX] [--inputs=path1.md,path2.md] [--only=c4_context,openapi,...] [--skip=deployment,state,...]"
cli_requires: "task_tool"
---

# /pdlc:design [PRD-XXX | SPEC-XXX] — Doc-as-code design package через субагент

Создание набора doc-as-code артефактов (C4, sequence, ERD, OpenAPI, AsyncAPI, ADR, glossary, state, deployment) на основе PRD или SPEC. Все артефакты — Markdown с Mermaid/YAML, нативно рендерятся в GitHub/GitLab/Notion.

## Использование

```
/pdlc:design PRD-001                                  # дизайн на основе PRD
/pdlc:design SPEC-001                                  # дизайн на основе SPEC (обогащает существующую спеку)
/pdlc:design                                           # выбрать из доступных ready PRD/SPEC
/pdlc:design PRD-001 --inputs=docs/research/market.md  # с дополнительным контекстом
/pdlc:design PRD-001 --only=c4_context,openapi         # только указанные артефакты
/pdlc:design PRD-001 --skip=deployment,state           # все, кроме указанных
```

## Когда нужен дизайн-пакет

**Создавай дизайн:**
- Новый модуль или сервис
- Архитектурное переписывание
- Перед передачей SPEC другой команде
- Любой PRD/SPEC, где есть: ≥ 2 сущности, ≥ 1 endpoint, multi-step flow, состояния, integration с внешним сервисом
- Когда PM хочет «чтобы остался след» — артефакты как ubiquitous language для команды

**Не нужен дизайн (иди сразу в `/pdlc:tasks` или `/pdlc:roadmap`):**
- Тривиальные UI-правки
- Багфиксы
- Конфиг-изменения

## Производимые артефакты (12 типов, conditional)

| # | Артефакт | Файл в package | Когда генерируется |
|---|---|---|---|
| 1 | C4 Context (Level 1) | `c4-context.md` | **MANDATORY** если `external_systems` non-empty; иначе — есть внешние акторы или integration |
| 2 | C4 Container (Level 2) | `c4-container.md` | ≥ 2 deployable units (frontend/backend/worker/DB/cache/queue) |
| 3 | C4 Component (Level 3) | `c4-component.md` | Сложный single container с явно выделяемыми компонентами |
| 4 | Sequence diagrams | `sequences.md` | Multi-step flows, OAuth, retries, compensation |
| 5 | ER diagram + Data Dictionary | `data-model.md` | ≥ 2 entities или явная схема БД |
| 6 | OpenAPI 3.0 | `api.md` | ≥ 1 REST endpoint |
| 7 | AsyncAPI 3.0 | `async-api.md` | Message broker, event-driven, WebSocket, pub/sub |
| 8 | ADR | `docs/adr/ADR-XXX-*.md` | Каждое серьёзное архитектурное решение с alternatives |
| 9 | Domain Glossary | `glossary.md` | ≥ 5 уникальных доменных терминов |
| 10 | State diagrams | `state-machines.md` | Сущность с ≥ 3 состояниями (lifecycle) |
| 11 | Deployment view | `deployment.md` | Явные NFRs (HA, multi-region, k8s) |
| 12 | Quality Scenarios | `quality-scenarios.md` | Любое NFR в source SPEC секции 6 (arc42 §10) |

Подробные триггеры — в `references/conditional-triggers.md`. Подробные шаблоны и Mermaid-примеры — в `references/<тип>-guide.md` (читать только нужные).

## Архитектура с субагентом

```
┌─────────────────────────────────────────────────────────────┐
│  PM: /pdlc:design PRD-001 [--inputs=...] [--only/skip=...]  │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  Phase 1: Parse args, validate, resolve input artifact      │
│  Phase 2: Conditional analysis → needed_artifacts set       │
│  Phase 3: Allocate IDs (DESIGN + ADRs), build file plan     │
│  Phase 4: Pack subagent context (only relevant references/) │
│  Phase 5: Launch ONE subagent with full design prompt       │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  СУБАГЕНТ general-purpose (clean context)                   │
│                                                             │
│  System role: Solution Design Architect                     │
│  Input: source artifact + parent + inputs + knowledge +     │
│         relevant references/                                │
│                                                             │
│  Делает:                                                    │
│  1. Generates glossary FIRST (seeds ubiquitous language)    │
│  2. Generates remaining artifacts следуя glossary terms     │
│  3. Creates ADRs только для серьёзных decisions             │
│  4. Возвращает: список файлов, skipped + причины, вопросы   │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  Phase 6: Holistic Quality Review Loop (max 2 iterations)   │
│  Phase 7: State updates (PROJECT_STATE, counters, ADRs)     │
│  Phase 8: Report to PM                                      │
└─────────────────────────────────────────────────────────────┘
```

## Алгоритм

### Phase 1 — Parse args & validate

1. Распарсь `$ARGUMENTS`:
   - первый позиционный аргумент: `PRD-XXX` или `SPEC-XXX` (опционально)
   - `--inputs=path1,path2,...` — дополнительные context-файлы
   - `--only=type1,type2` — whitelist (override conditional logic)
   - `--skip=type1,type2` — blacklist
2. Прочитай `.state/PROJECT_STATE.json`
3. Resolve input artifact:
   - Если ID указан: найди в `artifacts`, проверь что это `PRD` или `SPEC` и `status == ready`
   - Если не указан: покажи список ready PRD + SPEC, спроси какой использовать
   - FEAT в v1 не поддерживается — если PM передал FEAT-XXX, скажи: "Для FEAT сначала создай SPEC через /pdlc:spec, затем /pdlc:design SPEC-XXX"
4. Если `--inputs` указаны: проверь что каждый файл существует и читаем
5. Прочитай `.state/knowledge.json`

```
Нет готовых PRD или SPEC для создания дизайн-пакета.

Доступные действия:
   → /pdlc:prd для крупной инициативы
   → /pdlc:spec PRD-XXX для технической спецификации
   → /pdlc:state для обзора проекта
```

### Phase 1.5 — Валидация технического контекста (обязательный checkpoint)

**Цель:** убедиться, что `.state/knowledge.json` содержит актуальный технический контекст.
Дизайн-пакет генерирует конкретные артефакты (C4, ERD, OpenAPI, ADR) — если субагент не знает
реальный стек, он выдумает технологии, и package будет бесполезен.

1. Проверь следующие поля в `.state/knowledge.json`:

| Поле | Критичность | Что проверить |
|------|-------------|---------------|
| `projectContext.techStack` | **ОБЯЗАТЕЛЬНО** | Не пустой массив |
| `projectContext.description` | **ОБЯЗАТЕЛЬНО** | Не пустая строка |
| `projectContext.keyFiles` | желательно | Не пустой массив |
| `projectContext.entryPoints` | желательно | Не пустой массив |
| `patterns` | желательно | Не пустой массив |
| `testing.testCommand` | желательно | Не null |

2. **Если ВСЕ обязательные поля заполнены** → покажи краткую сводку и запроси подтверждение:

```
═══════════════════════════════════════════
ТЕХНИЧЕСКИЙ КОНТЕКСТ (из knowledge.json)
═══════════════════════════════════════════

Tech Stack: TypeScript, React, Node.js, PostgreSQL, Redis
Description: Платформа для управления проектами
Patterns: REST API, Repository pattern, DI
Key Files: src/index.ts, src/server.ts

Контекст актуален? [y / update]
═══════════════════════════════════════════
```

- `y` → продолжить к Phase 2
- `update` → перейти к интервью (пункт 3 ниже)

3. **Если ЛЮБОЕ обязательное поле пусто** → провести обязательное интервью:

   a. **Автодетект** — просканируй корень проекта на наличие маркеров стека:
      - `package.json` → Node.js/TypeScript (проверь `dependencies`/`devDependencies`)
      - `tsconfig.json` → TypeScript (даже без package.json, напр. Deno)
      - `go.mod` → Go
      - `pyproject.toml` / `requirements.txt` / `setup.py` → Python
      - `Cargo.toml` → Rust
      - `pom.xml` / `build.gradle` / `build.gradle.kts` → Java/Kotlin
      - `build.sbt` / `.scalafmt.conf` → Scala/sbt
      - `gradlew` / `mvnw` → JVM wrapper scripts (Gradle/Maven)
      - `application.yml` / `application.properties` → Spring Boot
      - `*.csproj` / `*.sln` → C# / .NET
      - `docker-compose.yml` → infrastructure hints (DB, cache, queue, Kafka)
      - `.env.example` → environment variables
      - `Makefile` / `Justfile` → build/test commands
      - `jest.config.*` / `vitest.config.*` / `pytest.ini` / `.rspec` → test framework
      - `playwright.config.*` → Playwright (E2E)
      - `cucumber.yml` / `features/*.feature` → Cucumber (BDD)

   b. **Предложи и спроси** — покажи обнаруженное и задай обязательные вопросы:

```
═══════════════════════════════════════════
ТЕХНИЧЕСКИЙ КОНТЕКСТ НЕ ЗАПОЛНЕН
═══════════════════════════════════════════

Обнаружено в проекте:               ← примеры для разных стеков:

──── Пример A (JVM) ────
  • build.gradle.kts → Kotlin, Spring Boot 3.2
  • application.yml → Spring Boot config
  • docker-compose.yml → PostgreSQL 16, Kafka 3.6
  • src/test/ → JUnit 5, Cucumber

──── Пример B (Node.js) ────
  • package.json → TypeScript 5, Express 4
  • playwright.config.ts → Playwright (E2E)
  • docker-compose.yml → PostgreSQL 16, Redis 7

──── Пример C (Scala) ────
  • build.sbt → Scala 3, Akka HTTP
  • .scalafmt.conf → Scala formatter
  • docker-compose.yml → PostgreSQL 16, Kafka 3.6

Обязательные вопросы (без ответов дизайн-пакет НЕ будет создан):

1. Язык(и) программирования и основные фреймворки?
   Пример A: Kotlin, Spring Boot 3.2
   Пример B: TypeScript 5, Express 4
   Пример C: Scala 3, Akka HTTP

2. База данных и хранилища?
   Пример A: PostgreSQL 16, Kafka 3.6
   Пример B: PostgreSQL 16, Redis 7
   Пример C: PostgreSQL 16, Kafka 3.6

3. Архитектурный стиль?
   (монолит / микросервисы / serverless / модульный монолит / другое)

4. Ключевые ограничения или стандарты?
   (GDPR, конкретный cloud provider, legacy интеграции...)

5. Протокол коммуникации между компонентами?
   (REST / gRPC / GraphQL / message broker / комбинация)
   Это критично для выбора OpenAPI vs AsyncAPI артефактов.

Необязательные (но полезные для качества дизайна):

6. Deployment target?
   (Docker / Kubernetes / serverless / bare metal / PaaS)

7. Ключевые файлы (entry points, конфигурация)?
   Предложение: src/app/layout.tsx, src/server.ts

═══════════════════════════════════════════
```

   c. **Дождись ответа пользователя.** Агент МОЖЕТ предложить варианты на основе
      автодетекта, но КАЖДЫЙ ответ на обязательные вопросы (1-5) должен быть
      **явно подтверждён** пользователем (архитектором). Не продолжай без ответов
      на вопросы 1-5. Пользователь может ответить кратко ("да, всё верно" — значит
      предложения приняты) или скорректировать.

   d. **Запиши подтверждённые данные** в `.state/knowledge.json`:
      - `projectContext.techStack` — массив строк (языки, фреймворки, БД, инфра)
      - `projectContext.description` — строка с описанием проекта
      - `projectContext.keyFiles` — массив путей (если пользователь указал)
      - `projectContext.entryPoints` — массив путей (если пользователь указал)
      - `patterns` — если пользователь указал архитектурные паттерны, добавь как
        массив строк (например, `["REST API", "Repository pattern", "DI"]`)
      - `testing.testCommand` — команда тестирования (если указана)

   e. Запиши обновлённый `knowledge.json` (2-space indent, stable key order).

⛔ **БЛОКЕР:** Без заполненных `techStack` и `description` переходить к Phase 2
**ЗАПРЕЩЕНО**. Дизайн-пакет без технического контекста будет содержать выдуманные
технологии в C4, ERD и OpenAPI — это хуже, чем отсутствие дизайна.

### Phase 2 — Conditional analysis (main agent, no subagent)

1. Прочитай input artifact (PRD или SPEC) полностью
2. Если parent chain существует (SPEC → PRD), прочитай parent тоже
3. Прочитай каждый файл из `--inputs`
4. **Checkpoint: границы системы.** Если source artifact (PRD или SPEC) или parent PRD упоминает внешние системы / интеграции → убедись, что информация о смежных системах передаётся в субагент для генерации C4 Context diagram. Если упоминания есть, но раздел «Внешние системы» (6A) отсутствует → зафиксируй Open Question.
5. **Trigger detection**: пройди по таблице из `references/conditional-triggers.md`. Для каждого из 12 типов артефактов — проверь свои триггеры (case-insensitive regex/keyword search). Сформируй `needed_artifacts` set.
   - **IMPORTANT**: если source SPEC содержит non-empty `external_systems` или source PRD содержит заполненную секцию §6A → `c4_context` **ОБЯЗАТЕЛЕН** (добавить в `needed_artifacts` безусловно, `--skip=c4_context` НЕ удаляет его).
6. Применить `--only` (whitelist полностью переопределяет detection) и `--skip` (вычитает из detected)
7. **Если `needed_artifacts` пуст** → exit clean без state mutation:

```
═══════════════════════════════════════════
DESIGN PACKAGE НЕ НУЖЕН
═══════════════════════════════════════════

Анализ {PRD-001 | SPEC-001} не выявил архитектурных артефактов:
- нет API endpoints
- нет entities/data model
- нет multi-step flows
- нет архитектурных decisions

Рекомендую:
   → /pdlc:tasks {PRD-001 | SPEC-001} — создать задачи напрямую
   → /pdlc:roadmap SPEC-001 — если есть SPEC и нужен план фаз
═══════════════════════════════════════════
```

8. **PM checkpoint**: покажи detected набор + краткое "почему" на каждый артефакт + список ADR-кандидатов:

```
═══════════════════════════════════════════
DESIGN PACKAGE PLAN: DESIGN-001 from PRD-001
═══════════════════════════════════════════

Будут созданы артефакты:
  ✓ c4-context.md       — внешние акторы: User, OAuth Provider
  ✓ c4-container.md     — 4 контейнера: Web App, API, PostgreSQL, Redis
  ✓ sequences.md        — 2 потока: OAuth callback, Token refresh
  ✓ data-model.md       — 3 entities: User, Session, Token
  ✓ api.md              — 6 endpoints (OpenAPI 3.0)
  ✓ glossary.md         — 12 терминов из домена auth
  ✓ ADR-003             — Mermaid over PlantUML for doc-as-code
  ✓ ADR-004             — Sessions in Redis vs DB

Пропущены (не обнаружены триггеры):
  ✗ c4-component.md     — single container не требует Level 3
  ✗ async-api.md        — нет message broker / event-driven паттернов
  ✗ state-machines.md   — нет сущностей с ≥ 3 состояниями
  ✗ deployment.md       — нет явных NFRs про инфраструктуру

Продолжить? [y / n / edit]
═══════════════════════════════════════════
```

PM выбирает:
- `y` → Phase 3
- `n` → exit без изменений
- `edit` → показать interactive picker, дать добавить/убрать, повторить confirmation

### Phase 3 — Allocate IDs and paths

1. **Вычисли next-id для DESIGN и ADR** по протоколу из
   `skills/tasks/references/compute-next-id.md`.
   Для DESIGN источник file-scan — имена директорий
   `docs/architecture/DESIGN-*/` (авторитет), не содержимое README. Для
   ADR — `docs/adr/ADR-*.md`. При **Counter drift** (любой из двух типов)
   — АБОРТ с рекомендацией
   `python3 {plugin_root}/scripts/pdlc_sync.py . --apply --yes`.
2. **Write-guard.** Перед созданием директории пакета
   `docs/architecture/DESIGN-{N}-slug/` проверь, что директория не
   существует и что `DESIGN-{N}` нет в `state.artifactIndex`. Для каждого
   ADR в наборе — аналогично для `docs/adr/ADR-{Nk}-slug.md`. При
   коллизии — АБОРТ до любого IO.
3. Инкрементируй `DESIGN` → `DESIGN-NNN`
4. Если ADR в наборе: для каждого ADR инкрементируй `ADR` → `ADR-NNN`
5. Вычисли `slug` = kebab-case от title input artifact
6. Build file plan:

```
Package dir: docs/architecture/DESIGN-{NNN}-{slug}/

Files:
  - docs/architecture/DESIGN-{NNN}-{slug}/README.md             (always)
  - docs/architecture/DESIGN-{NNN}-{slug}/manifest.yaml         (always — machine-readable index)
  - docs/architecture/DESIGN-{NNN}-{slug}/c4-context.md         (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/c4-container.md       (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/c4-component.md       (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/sequences.md          (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/data-model.md         (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/api.md                (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/async-api.md          (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/state-machines.md     (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/deployment.md         (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/glossary.md           (if in set)
  - docs/architecture/DESIGN-{NNN}-{slug}/quality-scenarios.md  (if in set)

ADRs (separate, in existing docs/adr/):
  - docs/adr/ADR-{N1}-{slug1}.md
  - docs/adr/ADR-{N2}-{slug2}.md
```

### Phase 4 — Prepare subagent context

Собери в один большой context block:

1. **Source artifact** — полное содержимое PRD или SPEC
2. **Parent artifact** — если SPEC → читать parent PRD; иначе "N/A"
3. **Extra inputs** — concatenated содержимое каждого `--inputs` файла
4. **Constraints, Assumptions, Dependencies** (из source SPEC §4, если source — SPEC):
   - Извлеки секцию 4 целиком (Assumptions A-N, Constraints C-N, Dependencies D-N)
   - Если source — PRD (SPEC ещё нет): "N/A — constraints будут определены в SPEC"
   - Constraints критичны для design decisions: если C-1 говорит "PostgreSQL only" —
     ADR НЕ должен предлагать MongoDB; если C-2 — "GDPR" — deployment view
     ОБЯЗАН показать EU-region isolation
4b. **System boundary for C4 Context** (если `c4_context` в `needed_artifacts`):
   - Из SPEC frontmatter: `system_boundary` → label для центрального `System()` блока
   - Из SPEC frontmatter: `external_systems[]` → каждый элемент становится `System_Ext()` блоком
   - Из SPEC §7.0 Integration Matrix: протоколы → labels для `Rel()` связей
   - Если source — PRD: из §6A.1 → `System()`, из §6A.2 → `System_Ext()`
5. **Project knowledge** (из `.state/knowledge.json`):
   - `projectContext.name`, `description`, `techStack`, `keyFiles`
   - `patterns` (следуй), `antiPatterns` (избегай)
   - `decisions` (учитывай существующие ADRs)
   - `architecture.activeADRs` из PROJECT_STATE — список активных ADRs (не дублируй)
6. **Relevant references** — для каждого артефакта в `needed_artifacts` прочитай соответствующий `skills/design/references/<type>-guide.md`. **Не читай гайды для skipped артефактов** — это экономит контекст.
7. **`skills/design/references/manifest-schema.md`** — ВСЕГДА (для генерации manifest.yaml)
8. **`docs/templates/adr-template.md`** — только если ADR в наборе

### Phase 5 — Launch subagent (general-purpose, clean context)

Полный шаблон промпта (system role, conditional artifact set, FR/NFR
mapping в realizes_requirements, manifest.yaml structure, ADR conditions,
glossary federation rules) — в:

**`skills/design/references/subagent-prompt.md`** —
читай через Read tool ПЕРЕД формированием prompt и инжектируй данные
из Phase 4 (Prepare subagent context).

⛔ **НЕ передавай** содержимое промпта от руки. Шаблон — единая точка
обновления; правки делаются в reference-файле.

### Phase 6 — Holistic Quality Review Loop

Полный алгоритм Holistic Quality Review для DESIGN package — в:

**`skills/design/references/quality-review-loop.md`** —
читай через Read tool ПОСЛЕ создания пакета и ПЕРЕД переводом в `ready`.

Loop обязателен: package не переходит в `ready` без прохождения review.

### Phase 7 — State updates

1. **counters.json**: инкремент `DESIGN` (уже сделан в Phase 3); ADR счётчик уже инкрементирован

2. **PROJECT_STATE.json `artifacts`** — добавь **краткую** entry для DESIGN-PKG.
   Rich-данные (realizes_requirements, components, scenarios, addresses) НЕ
   дублируются здесь — они живут только в `manifest.yaml`. PROJECT_STATE хранит
   только pointer на манифест и плоский список `{type, path}` для быстрого discovery:

```json
"DESIGN-001": {
  "type": "DESIGN-PKG",
  "title": "Design: {source title}",
  "status": "ready",
  "path": "docs/architecture/DESIGN-001-{slug}/README.md",
  "created": "{today}",
  "parent": "{SOURCE-ID}",
  "children": ["ADR-003", "ADR-004"],
  "package": {
    "dir": "docs/architecture/DESIGN-001-{slug}/",
    "manifest": "manifest.yaml",
    "artifacts": [
      {"type": "c4-context", "path": "c4-context.md"},
      {"type": "c4-container", "path": "c4-container.md"},
      {"type": "sequence", "path": "sequences.md"},
      {"type": "erd", "path": "data-model.md"},
      {"type": "openapi", "path": "api.md"},
      {"type": "asyncapi", "path": "async-api.md"},
      {"type": "glossary", "path": "glossary.md"},
      {"type": "quality-scenarios", "path": "quality-scenarios.md"}
    ]
  }
}
```

Поле `package.manifest` всегда `"manifest.yaml"` — relative path внутри `package.dir`.
Скрипты, которым нужны `realizes_requirements` или другие rich-поля, открывают
`{dir}/{manifest}` на месте.

3. **PROJECT_STATE.json — каждый созданный ADR** добавь как отдельную запись:

```json
"ADR-003": {
  "type": "ADR",
  "title": "Mermaid over PlantUML for doc-as-code",
  "status": "proposed",
  "path": "docs/adr/ADR-003-mermaid-over-plantuml.md",
  "created": "{today}",
  "parent": null,
  "children": []
}
```

4. **DESIGN-{NNN} → `readyToWork`**

5. **`architecture.activeADRs`** — append каждый созданный `ADR-{N}.id` (это поле сейчас dead, оживляется новым скиллом)

6. **Parent (PRD/SPEC)** — обнови:
   - В `.md` файле frontmatter: добавь DESIGN-{NNN} в `children:`
   - В `PROJECT_STATE.artifacts[parent_id].children`: добавь DESIGN-{NNN}
   - **Статус parent НЕ меняй** (правило из `/pdlc:spec` line 504)

7. **SPEC dedup — только если parent == SPEC:**

   Цель: устранить дублирование API/data контента между SPEC и DESIGN-PKG.
   После создания DESIGN-PKG в parent SPEC должны остаться только ссылки.

   a. **Frontmatter parent SPEC** — установи поля:
      ```yaml
      design_package: DESIGN-{NNN}
      design_waiver: false
      ```
      Если `design_waiver` был `true` (PM давал waiver ранее) — **сбрось в `false`**.
      Waiver — временная мера до создания DESIGN. Теперь design создан,
      enforcement восстанавливается для всех новых TASKs.

      (`design_package` включает Режим B для секций 7.1 / 7.2 — см. spec-template.md)

   b. **Секция 7.1 "Контракты компонентов / операций"** — если содержит
      inline-таблицу Operations:
      - Заменить таблицу на link-блок:
        ```markdown
        > **См.** [[DESIGN-{NNN}/api.md]]
        >
        > SPEC определяет требования к API на уровне operations и связанных FR.
        > Конкретные endpoints, request/response schemas, error codes —
        > в `docs/architecture/DESIGN-{NNN}-{slug}/api.md`.
        ```
      - Удалить inline-таблицу полностью
      - Если в SPEC уже link-блок (Режим B уже стоял) — просто обнови ID

   c. **Секция 7.2 "Контракты данных"** — если содержит inline-таблицу
      Entities:
      - Заменить таблицу на link-блок:
        ```markdown
        > **См.** [[DESIGN-{NNN}/data-model.md]]
        >
        > SPEC определяет требования к данным на уровне entities и связанных FR/NFR.
        > ER-диаграмма, физические типы, индексы, миграции —
        > в `docs/architecture/DESIGN-{NNN}-{slug}/data-model.md`.
        ```
      - Удалить inline-таблицу полностью

   d. **Секция 3 "Глоссарий"** (опционально, если в DESIGN-PKG есть glossary.md):
      - Установи `glossary_source: "DESIGN-{NNN}/glossary.md"` во frontmatter
      - Если в SPEC inline-таблица терминов — оставь как есть (термины
        специфичные для SPEC), но добавь заголовок:
        `**Источник:** [[DESIGN-{NNN}/glossary.md]] (плюс inline ниже)`

   ВАЖНО: эти изменения делает основной агент через Edit tool после
   успешного завершения субагента и Quality Review (PASS). Это устраняет
   единственный источник дрифта между SPEC и DESIGN.

   Не меняй FR / NFR / Open Questions / Traceability — только секции 7.1 / 7.2
   и frontmatter `design_package` / `glossary_source`.

8. **Federation glossary в knowledge.json — только если `glossary.md` создан в этом package:**

   Цель: распространить ubiquitous language из package на downstream subagents
   (`/pdlc:tasks`, `/pdlc:implement`, `/pdlc:spec`), чтобы они использовали те же
   термины и не плодили синонимы (Session vs UserSession vs SessionRecord).

   a. Прочитай `docs/architecture/DESIGN-{NNN}-{slug}/glossary.md`

   b. Извлеки термины. Glossary имеет одну запись на термин со структурой:
      `**Term** — definition` (или таблицу с колонками term/definition).
      Для каждого термина построй объект:
      ```json
      {
        "term": "Session",
        "definition": "Authenticated user state, identified by token",
        "source": "DESIGN-{NNN}/glossary.md",
        "synonyms_to_avoid": [],
        "added": "{today}"
      }
      ```
      `synonyms_to_avoid` оставляй пустым, если в glossary нет явных запретов
      («НЕ путать с …»). Если есть — извлекай.

   c. Прочитай `.state/knowledge.json`. Если поля `glossary` нет (старая схема)
      — добавь как пустой массив.

   d. Для каждого извлечённого термина:
      - Поиск по `knowledge.glossary[].term` (case-insensitive exact match).
      - **Не найден** → append новый объект.
      - **Найден И definition совпадает** → пропустить (idempotent).
      - **Найден И definition отличается** → CONFLICT:
        - НЕ перезаписывать запись автоматически.
        - Добавь warning в session-log:
          ```markdown
          ### Glossary conflict: DESIGN-{NNN}
          - Term: "{term}"
          - Existing: "{old_definition}" (source: {old_source})
          - New:      "{new_definition}" (source: DESIGN-{NNN}/glossary.md)
          - Action:   kept existing, PM should resolve
          ```
        - Включи термин в список конфликтов в Phase 8 report (waiting_pm fragment).

   e. Запиши обновлённый `.state/knowledge.json` (2-space indent, stable key order).

   f. Логирование в session-log:
      ```markdown
      ### Glossary federation: DESIGN-{NNN} → knowledge.json
      - Terms added:    {N_added}
      - Terms updated:  0     (federation никогда не перезаписывает)
      - Conflicts:      {N_conflicts}
      - Source:         docs/architecture/DESIGN-{NNN}-{slug}/glossary.md
      ```

   Если в наборе нет `glossary.md` (например, `--skip=glossary` или conditional
   trigger не сработал) — этот шаг полностью пропускается.

### Phase 8 — Report to PM

Все варианты вывода (success, with reviews, blocked) — в:

**`skills/design/references/output-formats.md`** —
читай через Read tool ПЕРЕД печатью соответствующего блока.

## References (per-artifact guides)

Дополнительные гайды загружаются субагентом по нужде, по одному на тип артефакта:

| Reference | Когда читать |
|---|---|
| `references/artifact-catalog.md` | Всегда (компактная таблица всех типов) |
| `references/conditional-triggers.md` | Phase 2 (расширенная таблица триггеров) |
| `references/manifest-schema.md` | Всегда в Phase 5 (subagent создаёт manifest.yaml последним) |
| `references/c4-guide.md` | Если c4_context, c4_container или c4_component в наборе |
| `references/mermaid-sequence.md` | Если sequence в наборе |
| `references/mermaid-er.md` | Если erd в наборе |
| `references/mermaid-state.md` | Если state в наборе |
| `references/mermaid-deployment.md` | Если deployment в наборе |
| `references/openapi-guide.md` | Если openapi в наборе |
| `references/asyncapi-guide.md` | Если asyncapi в наборе |
| `references/adr-guide.md` | Если adr в наборе |
| `references/glossary-guide.md` | Если glossary в наборе |
| `references/quality-scenarios-guide.md` | Если quality_scenarios в наборе |

Это сознательное отступление от Polisade Orchestrator-конвенции одно-файловых скиллов. `/pdlc:design` единственный, кто производит 12 разнородных артефактов; модульность references/ даёт progressive disclosure (грузить только нужное).

## Важно

- Субагент работает в чистом контексте — передавай весь нужный контекст в prompt
- Glossary генерируется первым и seeds ubiquitous language для всего package
- Quality Review — холистический (один ревью на весь package), не per-file
- ADR хранятся в `docs/adr/` (стандарт MADR), а не внутри package dir
- Sub-артефакты НЕ имеют своих ID; они адресуются путём в package dir
- Только DESIGN-{NNN} и ADR-{N} занимают counters.json
- Парент-артефакт (PRD/SPEC) НЕ меняет статус после генерации дизайна
- Sub-артефакты НЕ имеют поля `status` (наследуется от DESIGN-PKG)
- При conditional analysis: conservatism rule — при сомнении ВКЛЮЧАЙ артефакт
- `architecture.activeADRs` в PROJECT_STATE.json — источник правды о live решениях, обновляется на каждый созданный ADR
- `manifest.yaml` рядом с README.md — machine-readable source of truth о структуре package; PROJECT_STATE.json `package` хранит только pointer на manifest, без дублирования rich-данных
- `knowledge.glossary` в `.state/knowledge.json` — federation назначение для терминов package'а; пополняется в Phase 7 на каждый созданный `glossary.md` и читается downstream-субагентами (`/pdlc:tasks`, `/pdlc:implement`, `/pdlc:spec`) как ubiquitous language project-wide. Конфликты НЕ перезаписывают существующие записи — только сигнализируют через session-log и Phase 8 report
