---
name: spec
description: 'Generate a technical SPEC (SPEC-NNN) from an existing PRD or Feature Brief via a clean-context subagent. Use when PM mentions "create SPEC", "write spec", "turn PRD into SPEC", "functional spec", "specification from PRD", "создай spec", "напиши спеку", or any request to translate product intent into engineering-ready specification. Trigger liberally — under-triggering forces the agent to improvise design/API calls in chat without the canonical SPEC template; over-triggering is recoverable (PM can delete or regenerate).'
argument-hint: "[PRD-XXX | FEAT-XXX]"
cli_requires: "task_tool"
---

# /pdlc:spec [PRD-XXX | FEAT-XXX] — Техническая спецификация через субагент

Создание технической спецификации на основе PRD или Feature Brief через изолированный субагент.

## Использование

```
/pdlc:spec PRD-001    # Спека для крупной инициативы
/pdlc:spec FEAT-001   # Спека для фичи (если нужна архитектура)
/pdlc:spec            # Выбрать из доступных ready PRD/FEAT
```

## Когда нужна спецификация

**Нужна SPEC:**
- Новые API endpoints
- Изменения в базе данных
- Сложная бизнес-логика
- Интеграция с внешними сервисами
- Архитектурные изменения

**Не нужна SPEC (иди сразу в /pdlc:tasks):**
- UI изменения без логики
- Простые CRUD операции
- Багфиксы
- Мелкие улучшения

## Архитектура с субагентом

```
┌─────────────────────────────────────────────────────────────┐
│  PM: /pdlc:spec PRD-001                                     │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  1. Валидация: PRD/FEAT со статусом ready                   │
│  2. Читает PRD/FEAT файл полностью                          │
│  3. Читает knowledge.json                                   │
│  4. Формирует prompt с системным промптом                   │
│  5. Запускает Task tool: subagent_type="general-purpose"    │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  СУБАГЕНТ general-purpose (чистый контекст)                 │
│                                                             │
│  System role: Technical Specification Architect             │
│  Input: PRD/FEAT content + project context                  │
│                                                             │
│  Делает:                                                    │
│  1. Анализирует требования                                  │
│  2. Выявляет технические gaps → вопросы (если есть)         │
│  3. Проектирует архитектуру                                 │
│  4. Создаёт SPEC файл по структуре                          │
│  5. Возвращает: путь, summary, вопросы                      │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  ОСНОВНОЙ АГЕНТ                                             │
│  1. Если вопросы → статус waiting_pm                        │
│  2. Если готово → обновляет PROJECT_STATE.json              │
│  3. Обновляет counters.json                                 │
└─────────────────────────────────────────────────────────────┘
```

## Алгоритм работы основного агента

### 1. Валидация

1. Прочитай `.state/PROJECT_STATE.json`
2. Найди PRD или FEAT со статусом `ready`
3. Если указан ID — проверь что он `ready`
4. Если не указан:
   - Покажи список ready PRD и FEAT
   - Спроси какой использовать
   - Если нет ready → предложи `/pdlc:prd` или `/pdlc:feature`

```
Нет готовых PRD или Feature Brief для создания спецификации.

Доступные действия:
   → /pdlc:feature для создания фичи
   → /pdlc:prd для крупной инициативы
   → /pdlc:state для обзора проекта
```

### 2. Подготовка контекста

Прочитай и собери:
1. **Исходный документ** (PRD или FEAT) — полное содержимое
2. **Knowledge base** (`.state/knowledge.json`):
   - `projectContext` — описание проекта
   - `techStack` — технологии
   - `patterns` — используемые паттерны
   - `antiPatterns` — что избегать
   - `decisions` — принятые решения (ADR)
   - `glossary` — ubiquitous language project-wide (federated из DESIGN packages). Передавай в субагент: SPEC должен использовать ИМЕННО эти термины в FR/NFR/Glossary section.
3. **Шаблон спецификации** (`docs/templates/spec-template.md`)
4. **Существующий DESIGN-PKG для этого SPEC** (dedup-режим):
   - Проверь `PROJECT_STATE.artifacts` — есть ли `DESIGN-PKG` с
     `parent == {PRD-XXX или FEAT-XXX}` или среди children исходного документа
   - Если найден `DESIGN-NNN`:
     - Прочитай `docs/architecture/DESIGN-NNN-{slug}/README.md`
     - Прочитай `api.md` и `data-model.md` (если присутствуют) для контекста
     - Сохрани `existing_design_pkg = "DESIGN-NNN"` для передачи в субагент
   - Если не найден: `existing_design_pkg = null`

### 2.4. Валидация границ системы (checkpoint)

1. Проверь, содержит ли parent PRD секцию «Внешние системы и границы ответственности» (раздел 6A).
2. **Если секция есть и заполнена** → извлеки из неё информацию о смежных системах и передай в субагент как часть контекста (поле `external_systems`).
3. **Если PRD упоминает внешние системы / интеграции, но секция 6A отсутствует или пуста** → зафиксируй Open Question: "PRD упоминает внешние системы, но раздел 'Внешние системы и границы ответственности' не заполнен. Уточните границы и интеграции." Установи статус `waiting_pm`.
4. **Если PRD не упоминает интеграций и секция отсутствует** → считай систему standalone, продолжай без блокировки.

### 2.5. Валидация технического контекста (обязательный checkpoint)

**Цель:** убедиться, что `.state/knowledge.json` содержит актуальный технический контекст,
чтобы субагент НЕ фантазировал о стеке и архитектуре, а работал с подтверждённой архитектором информацией.

1. Проверь следующие поля в `.state/knowledge.json`:

| Поле | Критичность | Что проверить |
|------|-------------|---------------|
| `projectContext.techStack` | **ОБЯЗАТЕЛЬНО** | Не пустой массив |
| `projectContext.description` | **ОБЯЗАТЕЛЬНО** | Не пустая строка |
| `projectContext.keyFiles` | желательно | Не пустой массив |
| `projectContext.entryPoints` | желательно | Не пустой массив |
| `patterns` | желательно | Не пустой массив |
| `testing.testCommand` | желательно | Не null |
| `testing.lintCommand` | желательно | Не null |

2. **Если ВСЕ обязательные поля заполнены** → покажи краткую сводку и запроси подтверждение:

```
═══════════════════════════════════════════
ТЕХНИЧЕСКИЙ КОНТЕКСТ (из knowledge.json)
═══════════════════════════════════════════

Tech Stack: TypeScript, React, Node.js, PostgreSQL, Redis
Description: Платформа для управления проектами
Key Files: src/index.ts, src/server.ts
Test command: npm test
Lint command: npm run lint

Контекст актуален? [y / update]
═══════════════════════════════════════════
```

- `y` → продолжить к шагу 3
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

Обязательные вопросы (без ответов SPEC НЕ будет создана):

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
   (GDPR, конкретный cloud provider, legacy интеграции, корпоративные стандарты...)

Необязательные (но полезные для качества SPEC):

5. Команда для запуска тестов?
   Пример A: ./gradlew test
   Пример B: npm test
   Пример C: sbt test
   Cucumber (JS): npx cucumber-js
   Cucumber (JVM): ./gradlew test --tests '*Cucumber*'
   Playwright: npx playwright test

6. Команда для линтинга?
   Пример A: ./gradlew check
   Пример B: npx eslint .
   Пример C: sbt scalafmtCheck

7. Ключевые файлы (entry points, конфигурация)?
   (Предложение формируется на основе реальных файлов проекта)

═══════════════════════════════════════════
```

   c. **Дождись ответа пользователя.** Агент МОЖЕТ предложить варианты на основе
      автодетекта, но КАЖДЫЙ ответ на обязательные вопросы (1-4) должен быть
      **явно подтверждён** пользователем (архитектором). Не продолжай без ответов
      на вопросы 1-4. Пользователь может ответить кратко ("да, всё верно" — значит
      предложения приняты) или скорректировать.

   d. **Запиши подтверждённые данные** в `.state/knowledge.json`:
      - `projectContext.techStack` — массив строк (языки, фреймворки, БД, инфра)
      - `projectContext.description` — строка с описанием проекта
      - `projectContext.keyFiles` — массив путей (если пользователь указал)
      - `projectContext.entryPoints` — массив путей (если пользователь указал)
      - `patterns` — если пользователь указал архитектурные паттерны, добавь как
        массив строк (например, `["REST API", "Repository pattern", "DI"]`)
      - `testing.testCommand` — команда тестирования (если указана)
      - `testing.lintCommand` — команда линтинга (если указана)

   e. Запиши обновлённый `knowledge.json` (2-space indent, stable key order).

⛔ **БЛОКЕР:** Без заполненных `techStack` и `description` переходить к шагу 3
(формирование prompt для субагента) **ЗАПРЕЩЕНО**. Субагент без технического
контекста будет фантазировать о стеке, что приведёт к нерелевантной спецификации.

### 3. Формирование prompt для субагента

Полный шаблон промпта (system role, project context из knowledge.json,
parent intent, structure SPEC по секциям, FR/NFR mapping, EARS+Gherkin
формат, system_boundary правила, design hooks) — в:

**`skills/spec/references/subagent-prompt.md`** —
читай через Read tool ПЕРЕД формированием prompt и инжектируй данные
из шага 2 (Подготовка контекста).

⛔ **НЕ передавай** содержимое промпта от руки. Шаблон — единая точка
обновления; правки делаются в reference-файле.

### 4. Запуск субагента

Используй Task tool:
```
Task tool:
  subagent_type: "general-purpose"
  description: "Create SPEC from {PRD-XXX/FEAT-XXX}"
  prompt: [сформированный prompt выше]
```

### 5. Обработка результата

После завершения субагента:

**Если статус `ready`:**
1. **Вычисли next-id для SPEC** по протоколу из
   `skills/tasks/references/compute-next-id.md`
   (единый max по `.state/counters.json`, `PROJECT_STATE.artifactIndex`
   и file-scan `docs/specs/SPEC-*.md`). При **Counter drift** — АБОРТ
   с рекомендацией `python3 {plugin_root}/scripts/pdlc_sync.py . --apply --yes`.
2. **Write-guard.** Перед сохранением SPEC-файла, сгенерированного
   субагентом, проверь, что `docs/specs/SPEC-{N}-slug.md` не существует
   и что `SPEC-{N}` нет в `state.artifactIndex`. При коллизии — АБОРТ
   (субагент уже потратил контекст — PM должен починить state и
   перезапустить, а не молча перезаписать).
3. Инкрементируй счётчик SPEC (`counters.json[SPEC] = N`).
4. Обнови `.state/PROJECT_STATE.json`:
   - Добавь SPEC в `artifacts`
   - Добавь SPEC в `readyToWork`
   - Обнови parent: добавь SPEC в `children`

**Если статус `waiting_pm`:**
1. Сохрани SPEC как `draft`
2. Добавь в `waitingForPM` с вопросами
3. Выведи вопросы PM

### 6. Quality Review Loop (обязательно!)

Полный алгоритм quality review SPEC (запуск reviewer-субагента, обработка
score, IMPROVE-итерация, max-iterations gate, сохраняемая запись review)
вынесен в:

**`skills/spec/references/quality-review-loop.md`** —
читай через Read tool ПОСЛЕ создания SPEC и ПЕРЕД переводом в `ready`.

Loop обязателен: SPEC не переходит в `ready` без прохождения review.

## Формат вывода

Все варианты вывода (успешное создание ready, улучшение после ревью,
при наличии вопросов waiting_pm) — в:

**`skills/spec/references/output-formats.md`** —
читай через Read tool ПЕРЕД печатью соответствующего блока.


## Содержание спецификации

### Для FEAT (упрощённая спека)
- Обзор и связь с FEAT
- Изменения в API (если есть)
- Изменения в данных (если есть)
- Основные компоненты
- Критические edge cases

### Для PRD (полная спека)
- Полная архитектура
- Все API endpoints с примерами
- Модели данных с типами
- Database schema
- Безопасность
- Производительность
- План миграции
- Тестирование

## Важно

- Субагент работает в чистом контексте — передавай весь необходимый контекст в prompt
- Knowledge.json содержит паттерны проекта — субагент должен их учитывать
- Если субагент выявил gaps — это хорошо, вопросы к PM лучше чем додумывание
- Не создавай спеку если она не нужна — для простых фич иди сразу в `/pdlc:tasks`
- Спека для FEAT может быть короче чем для PRD
- При создании спеки не меняй статус родительского документа на `done`
