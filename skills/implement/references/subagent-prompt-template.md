# Subagent Prompt Template — `/pdlc:implement`

Этот файл — полный шаблон промпта, который основной агент `/pdlc:implement`
формирует для general-purpose субагента (Task tool) при M/L-задачах.

SKILL.md `### 3. Формирование prompt для субагента` ссылается сюда; здесь
лежит исходный текст шаблона со всеми секциями.

---

### 3. Формирование prompt для субагента

Используй следующий шаблон:

```
Реализуй задачу {TASK-ID}: {task_title}

═══════════════════════════════════════════
КОНТЕКСТ ПРОЕКТА
═══════════════════════════════════════════

Patterns (следуй этим паттернам):
{patterns из knowledge.json или "Не определены"}

Anti-patterns (избегай):
{antiPatterns из knowledge.json или "Не определены"}

Decisions (учитывай):
{decisions из knowledge.json или "Нет зафиксированных решений"}

Glossary (ubiquitous language — source of truth для именования):
{knowledge.glossary как список "term — definition (source)" или "Glossary пуст"}

TERMINOLOGY (ОБЯЗАТЕЛЬНО):
- Используй ТОЧНО эти термины в названиях классов, функций, переменных, полей,
  тестов и комментариях. Один концепт — одно имя project-wide.
- Если в glossary есть "Session" — НЕ изобретай "UserSession", "SessionRecord",
  "AuthState". Не вводи синонимы существующих терминов.
- `synonyms_to_avoid` в записи glossary — буквальный blacklist имён.
- Если для нужного концепта нет термина — придерживайся convention проекта;
  при сомнении flag в waiting_pm, не плоди дубликаты.

Key files:
{keyFiles из knowledge.json или "Изучи структуру проекта"}

═══════════════════════════════════════════
ТРЕБОВАНИЯ ЗАДАЧИ
═══════════════════════════════════════════

{полное содержимое TASK файла}

═══════════════════════════════════════════
⛔ ТОЧНОЕ СЛЕДОВАНИЕ ИНСТРУКЦИЯМ ЗАДАЧИ
═══════════════════════════════════════════

CRITICAL: Реализуй задачу СТРОГО по инструкциям в TASK файле.

- Если таск говорит "используй X" — используй X, НЕ подставляй альтернативу Y
- Если таск говорит "удали/замени X на Y" — удали X и используй Y
- Если таск описывает порядок операций — соблюдай ИМЕННО этот порядок
- НЕ "оптимизируй" подход, даже если видишь "лучший" вариант в существующем коде

Если ты считаешь что инструкция таска ошибочна или есть лучший путь —
верни waiting_pm с объяснением, а НЕ реализуй свою версию молча.

═══════════════════════════════════════════
СВЯЗАННЫЕ ДОКУМЕНТЫ
═══════════════════════════════════════════

{содержимое родительского FEAT/SPEC/BUG если есть}

═══════════════════════════════════════════
ТРЕБОВАНИЯ ИЗ SPEC (resolved через parent chain)
═══════════════════════════════════════════

Эта TASK реализует следующие требования parent SPEC:

{для каждого composite FR/NFR из TASK.requirements (формат `{DOC}.FR-NNN`):}

### {DOC_ID}.{FR-NNN}: {title}
**EARS Statement:** {statement}
**Acceptance criteria:**
{Gherkin scenarios — Given/When/Then}

(Если TASK.requirements: [] — этот блок: "N/A — TASK не привязан к SPEC requirements")

⛔ **НЕ делай `grep -r 'FR-NNN' .` по проекту** — parent chain уже резолвит
scope однозначно. `FR-007` в разных top-level документах (PRD vs FEAT vs SPEC)
— это **разные требования**. При сомнении — спроси PM, в каком именно
документе работаем.

═══════════════════════════════════════════
ARCHITECTURE CONTRACTS (из DESIGN package)
═══════════════════════════════════════════

{релевантные секции из api.md / data-model.md / sequences.md по TASK.design_refs}

(Если design_refs: [] — этот блок: "N/A — у parent SPEC нет DESIGN package")

═══════════════════════════════════════════
ASSUMPTIONS AND CONSTRAINTS (из SPEC §4)
═══════════════════════════════════════════

Assumptions (A-N):
{assumptions из SPEC §4.1 или "N/A"}

Constraints (C-N):
{constraints из SPEC §4.2 или "N/A"}

ИНСТРУКЦИИ:
- Constraints — нерушимые. Код обязан быть совместим со всеми constraints.
- Assumptions — если assumption можно проверить программно (например,
  "API возвращает user_id в JWT"), добавь defensive validation/assert в код.
  Если нельзя — пропусти, но не нарушай assumption молча.

═══════════════════════════════════════════
SYSTEM BOUNDARY (из SPEC frontmatter)
═══════════════════════════════════════════

system_boundary: {system_boundary из SPEC frontmatter или "N/A"}
external_systems: {список external_systems из SPEC frontmatter или "N/A"}

ИНСТРУКЦИИ (если system_boundary не N/A):
- Ты работаешь ВНУТРИ {system_boundary}. Внешние системы = клиенты/адаптеры.
- НЕ реализуй код внешних систем. Реализуй НАШУ сторону интеграции:
  адаптеры, клиенты, маппинг протоколов.
- Для тестов: mock/stub внешних систем, НЕ реальные вызовы.
- Если TASK требует работу с external system — реализуй клиент/адаптер
  на нашей стороне, не сервер/логику внешней системы.

═══════════════════════════════════════════
РАБОЧАЯ ДИРЕКТОРИЯ (WORKTREE)
═══════════════════════════════════════════

⚠️ Ты работаешь в git worktree!

WORKTREE_PATH:      {worktree_path}
EXPECTED_BRANCH:    {expected_branch}     ← для OPS-001 PRE-COMMIT GUARD
PDLC_GIT_BRANCHING: true                   ← обязательный mode-signal для guard

ПРАВИЛА:
1. ВСЕ операции с кодом — в WORKTREE_PATH
2. Команды: cd "{worktree_path}" && <команда>
3. .state/ файлы: {worktree_path}/.state/ (локальная копия)
4. НЕ переключай ветки! Worktree привязан к одной ветке.
5. git commit/push — только после PRE-COMMIT GUARD (см. секцию ниже).
6. НЕ создавай новые артефакты (TASK/FEAT/ADR) — counters.json недоступен.
   Если нужен новый артефакт → верни waiting_pm.
7. Бери команды тестирования/линтинга из knowledge.json (testing.*).
   НЕ изобретай команды — используй ТОЛЬКО то, что задано в проекте.

   Примеры вызова в worktree для разных стеков:

   # Python (если .venv/ есть в worktree через симлинк)
   cd "{worktree_path}" && .venv/bin/pytest tests/ -x -q
   cd "{worktree_path}" && .venv/bin/ruff check src/

   # Java/Scala (Gradle)
   cd "{worktree_path}" && ./gradlew test
   cd "{worktree_path}" && ./gradlew check

   # Node.js/TypeScript
   cd "{worktree_path}" && npm test
   cd "{worktree_path}" && npx eslint .
   cd "{worktree_path}" && npx tsc --noEmit

   # Go
   cd "{worktree_path}" && go test ./...
   cd "{worktree_path}" && golangci-lint run

   # Rust
   cd "{worktree_path}" && cargo test
   cd "{worktree_path}" && cargo clippy

   ⛔ ЗАПРЕЩЕНО:
   ⛔ Абсолютные пути: /Users/.../Projects/.../.venv/bin/python
   ⛔ Изобретать команды — бери из knowledge.json (testing.*)
   ⛔ Присвоение в начале: WT="/path" && cd "$WT" && ...

   {Если .venv/ присутствует в worktree — дополнительные Python-ограничения:}
   ⛔ python -m <tool>: .venv/bin/python -m pytest  (вызывай инструмент напрямую)
   ⛔ python -c "...": .venv/bin/python -c "import ..."
   ⛔ Голый pytest/ruff/mypy без .venv/bin/ (без активации venv — не на PATH!)

(Блок добавляется в prompt ТОЛЬКО при workspaceMode: "worktree".
 Если worktree не используется — блок не включать.)

═══════════════════════════════════════════
КОМАНДЫ ДЛЯ ТЕСТИРОВАНИЯ И ПРОВЕРОК
═══════════════════════════════════════════

{Блок добавляется ТОЛЬКО если хотя бы одно поле testing.* заполнено в knowledge.json}

Используй ИМЕННО эти команды (из knowledge.json), НЕ изобретай свои:

Тесты: {testing.testCommand или "НЕ ЗАДАНО — регрессионные тесты будут пропущены"}
Type check: {testing.typeCheckCommand или "не задано"}
Lint: {testing.lintCommand или "не задано"}

Для worktree всегда добавляй cd "{worktree_path}" && перед командой.

⛔ ЗАПРЕЩЕНО (для worktree):
   ПРАВИЛЬНО:   cd "{worktree_path}" && {testing.testCommand}
   ПРАВИЛЬНО:   cd "{worktree_path}" && ./gradlew test
   ПРАВИЛЬНО:   cd "{worktree_path}" && npm test
   НЕПРАВИЛЬНО: cd "{worktree_path}" && /абсолютный/путь/к/инструменту  (абсолютные пути!)
   НЕПРАВИЛЬНО: cd "{worktree_path}" && выдуманная-команда  (только из knowledge.json!)
   НЕПРАВИЛЬНО: WT="/path" && cd "$WT" && ...  (присвоение в начале запрещено!)

{Если testing.strategy == "tdd-first" И testCommand задан И task-scoped run разрешим — инлайнить блок ниже.
 Если testing.strategy == "test-along", отсутствует, testCommand не задан, или task-scoped run невозможен — НЕ включать этот блок.
 Source-of-truth: references/test-authoring-protocol.md}

═══════════════════════════════════════════
⛔ TDD-FIRST ПРОТОКОЛ (testing.strategy: "tdd-first")
═══════════════════════════════════════════

Ты ОБЯЗАН реализовать задачу в ДВА ЭТАПА:

### ЭТАП 1: RED — ТЕСТЫ (до написания кода реализации)

Источники тестов (по приоритету):
1. Gherkin scenarios из SPEC (FR-NNN → Given/When/Then) — каждый Scenario → 1 тест
2. Acceptance criteria checklist из TASK — каждый AC → минимум 1 тест
3. Design contracts из design_refs (api.md, data-model.md) → контрактные тесты
4. Assumptions/constraints из SPEC §4 → defensive/negative тесты

Действия:
1. Сгенерируй тесты, покрывающие ВСЕ источники выше
2. Запусти ТОЛЬКО новые тесты (task-scoped run):
   - Команда из секции ## Verification в TASK (первая тестовая команда)
   - Или derive file-scoped: pytest → `pytest tests/test_<module>.py`, jest → `jest <file>`, etc.
3. Классифицируй падения:
   - Syntax/import/compilation error → ИСПРАВЬ harness, перезапусти
   - Assertion failures → ОК, это ожидаемый red
   - Все тесты прошли (vacuous pass) → ⚠️ Проверь что тесты реально тестируют новое поведение
4. Перед коммитом выведи RED CHECKLIST:

```
───────────────────────────────────────────
RED CHECKLIST (test-authoring)
───────────────────────────────────────────
[✓/✗] Добавлены/обновлены только тесты и минимальный harness (stubs)
[✓/✗] Новые тесты компилируются/парсятся без ошибок
[✓/✗] Новые тесты падают по ожидаемой причине (assertion failures, NOT import/syntax error)
[✓/✗] Production code НЕ реализован на этом этапе
[✓/✗] Источники тестов: покрыты все AC и Gherkin из TASK/SPEC
───────────────────────────────────────────
```

5. Коммит: `[{TASK-ID}] Add failing tests for {TASK-ID}`

⛔ НЕ ПИШИ КОД РЕАЛИЗАЦИИ НА ЭТОМ ЭТАПЕ!
   Только тестовые файлы + минимальные stubs (пустые функции/классы) чтобы тесты компилировались.

### ЭТАП 2: GREEN — РЕАЛИЗАЦИЯ (чтобы тесты прошли)

1. Напиши код, который делает тесты из этапа 1 зелёными
2. Можно добавить дополнительные edge-case тесты
3. Все тесты (из этапа 1 + новые) должны проходить
4. Выполни полный SELF-REVIEW CHECKLIST (см. ниже)
5. Коммит: `[{TASK-ID}] Implement {TASK-ID}`

⛔ ПРАВИЛО ФИЛЬТРАЦИИ:
   - Red phase: допустима фильтрация (file/test target) — ТОЛЬКО новые тесты
   - Regression (шаг 2 полного цикла): фильтрация ЗАПРЕЩЕНА — без изменений

═══════════════════════════════════════════

═══════════════════════════════════════════
SELF-REVIEW (ОБЯЗАТЕЛЬНО ВЫВЕСТИ перед коммитом!)
═══════════════════════════════════════════

⛔ ПЕРЕД КОММИТОМ ты ОБЯЗАН:

1. Перечитать ВСЕ изменённые файлы (используй Read tool)
2. ВЫВЕСТИ этот чеклист с результатами проверки:

```
───────────────────────────────────────────
SELF-REVIEW CHECKLIST
───────────────────────────────────────────
[✓/✗] Hardcoded values: нет паролей/ключей/URL
[✓/✗] Error handling: async обёрнут в try/catch
[✓/✗] Patterns: код соответствует patterns
[✓/✗] Anti-patterns: нет нарушений antiPatterns
[✓/✗] Terminology: имена классов/функций/полей соответствуют knowledge.glossary
       (нет синонимов для канонических терминов; нет имён из synonyms_to_avoid)
[✓/✗] Tests: тесты добавлены/обновлены
[✓/✗] TDD: тесты написаны ДО реализации (если testing.strategy: "tdd-first")
       RED CHECKLIST пройден | Коммит 1: failing tests | Коммит 2: implementation
       (N/A если strategy: "test-along")
[✓/✗] Каждое composite FR/NFR из требований реализовано в коде (поштучно):
       ✓/✗ SPEC-001.FR-001: <EARS statement> → <file:function>
       ✓/✗ SPEC-001.FR-002: <EARS statement> → <file:function>
       ... (по списку TASK.requirements, composite IDs из parent SPEC/PRD/FEAT)
[✓/✗] DESIGN CONFORMANCE (если design_refs non-empty И design_waiver != true):
       Для каждого файла из design_refs:
       ✓/✗ <artifact>: реализация совпадает с контрактом
       
       Если есть расхождение (DESIGN-DEVIATION):
       ⛔ ОБЯЗАТЕЛЬНО:
         1. Обнови затронутый design-артефакт в ТОМ ЖЕ коммите/PR
            (design docs — source of truth, drift недопустим)
         2. Добавь в PR description секцию "Design Updates":
            ## Design Updates
            - DESIGN-NNN/api.md: <что изменилось>
            - DESIGN-NNN/data-model.md: <что изменилось>
         3. DESIGN-DEVIATION комментарий в коде — audit trail, НЕ удалять
       
       (N/A если design_refs пуст или design_waiver: true)
[✓/✗] Acceptance criteria (ПОШТУЧНО):
       ✓/✗ AC1: <описание> → <file:line>
       ✓/✗ AC2: <описание> → <file:line>
       ... (каждый критерий отдельно!)
───────────────────────────────────────────
```

3. Если хотя бы один [✗] — ИСПРАВЬ перед коммитом
4. После исправления — повтори self-review

⚠️ КОММИТ БЕЗ ВЫВОДА CHECKLIST = НАРУШЕНИЕ ПРОТОКОЛА!

═══════════════════════════════════════════
ФОРМАТ КОММИТА
═══════════════════════════════════════════

test-along: [{TASK-ID}] краткое описание
tdd-first коммит 1: [{TASK-ID}] Add failing tests for {TASK-ID}
tdd-first коммит 2: [{TASK-ID}] Implement {TASK-ID}

⚠️ Перед КАЖДЫМ коммитом — обязательный PRE-COMMIT GUARD (OPS-001), см. ниже.

═══════════════════════════════════════════
⛔ ЗАПРЕЩЁННЫЕ git-команды (HARD BOUNDARIES)
═══════════════════════════════════════════

В рамках реализации TASK ты работаешь ТОЛЬКО в своей feature-ветке
(или worktree, привязанном к ней). ЗАПРЕЩЕНО:

- git checkout main / master / switch main
- git push origin main / origin master / --force в main
- git merge / git rebase onto main
- git branch -D / git push origin --delete
- git commit / git add / git push с current_branch ≠ EXPECTED_BRANCH
  (см. PRE-COMMIT GUARD ниже)
- ⛔ NEVER git add -f / git add --force на gitignored путях (.gigacode,
  .qwen, .codex, .worktrees и т. д.). «Кроме X» = исключение, не фокус.
  NB: исключение по `.claude/` — только файл `.claude/settings.json`,
  не директория целиком.

После self-review ты ВОЗВРАЩАЕШЬ JSON-результат и БОЛЬШЕ НИЧЕГО:
  — НЕ ищешь следующую TASK
  — НЕ «готовишь main к следующей задаче»
  — НЕ запускаешь новый цикл
  — НЕ пытаешься сделать merge/push/delete
Твоя задача ОДНА. Возврат управления — это конец.

Если ты запущен для TASK, которая уже в review (PR создан или нет) —
это bug диспетчера основного агента. Верни JSON
{"status":"blocked","reason":"OPS-008: subagent spawned for review-stage TASK"}
и больше ничего не делай. НЕ пытайся «докидать», НЕ пытайся мержить.

Merge — ответственность PM. Если в процессе ты обнаружишь, что main
опередила feature-ветку — НЕ мёржи, верни `waiting_pm` с описанием.

═══════════════════════════════════════════
⛔ PRE-COMMIT GUARD (OPS-001 — ОБЯЗАТЕЛЬНО перед КАЖДЫМ git commit/push/add)
═══════════════════════════════════════════

MODE (PDLC_GIT_BRANCHING): {git_branching_mode}   ← "true" | "false", инжектируется основным агентом
EXPECTED_BRANCH:           {expected_branch_or_NA}   ← инжектируется ТОЛЬКО при MODE=true
WORK_DIR:                  {worktree_path_or_NA}     ← инжектируется ТОЛЬКО при MODE=true

ПЕРВЫЕ bash-команды в твоей работе (до любого git). Экспортируй ВСЁ, что
дал основной агент — даже если одна из переменных кажется «необязательной»:

```bash
# При MODE=true (gitBranching: true):
export PDLC_GIT_BRANCHING="true"
export PDLC_EXPECTED_BRANCH="{expected_branch}"
export PDLC_WORK_DIR="{worktree_path_or_dot}"

# При MODE=false (gitBranching: false, legacy):
export PDLC_GIT_BRANCHING="false"
# (PDLC_EXPECTED_BRANCH и PDLC_WORK_DIR НЕ устанавливаются)
```

ПЕРЕД каждым `git commit`, `git push`, `git add` ты ОБЯЗАН выполнить:

```bash
MODE="${PDLC_GIT_BRANCHING:-}"
EXPECTED="${PDLC_EXPECTED_BRANCH:-}"
WORK="${PDLC_WORK_DIR:-.}"

case "$MODE" in
  true)
    if [ -z "$EXPECTED" ]; then
      echo "⛔ OPS-001: PDLC_GIT_BRANCHING=true, но PDLC_EXPECTED_BRANCH пуст — bug"
      exit 1
    fi
    CURRENT=$(cd "$WORK" && git rev-parse --abbrev-ref HEAD)
    if [ "$CURRENT" != "$EXPECTED" ]; then
      echo "⛔ OPS-001: cwd=$WORK current=$CURRENT, expected=$EXPECTED — коммит запрещён"
      exit 1
    fi
    echo "✓ pre-commit guard OK: cwd=$WORK branch=$CURRENT"
    ;;
  false)
    echo "ℹ️ pre-commit guard skipped: PDLC_GIT_BRANCHING=false (legacy)"
    ;;
  *)
    # fail-closed: отсутствие явного mode-signal = баг (truncation/dropout/bug)
    echo "⛔ OPS-001: PDLC_GIT_BRANCHING не выставлен (ожидалось 'true'|'false'). Commit запрещён."
    exit 1
    ;;
esac
```

⚠️ Критично: `cd "$WORK"` ОБЯЗАТЕЛЕН. В режиме git worktree корень
репозитория остаётся на main/master — это нормальное поведение. Ветка
задачи видна ТОЛЬКО внутри `{worktree_path}`. Без `cd` guard даст ложный
fail.

⚠️ **Fail-closed модель.** Отсутствие `PDLC_GIT_BRANCHING` НЕ трактуется как
"безопасно". Для legacy-режима основной агент ОБЯЗАН явно выставить
`PDLC_GIT_BRANCHING=false`; пустое/неизвестное значение mode = баг (truncation
prompt-а, dropout инструкций, забытый export) → guard fail-closed, коммит
запрещён. Это защита ровно от того класса ошибок, которые вызвали OPS-001.

Если guard упал — НЕ ретрай, НЕ `git checkout`, НЕ создавай ветку сам,
НЕ пытайся "починить" через `export PDLC_GIT_BRANCHING=false` — это
реинтродукция OPS-001. Верни JSON:
```json
{"status": "blocked", "reason": "OPS-001: mode=<mode> expected=<expected> current=<current> cwd=<work>"}
```

═══════════════════════════════════════════
⛔ POST-PUSH VERIFICATION (OPS-028 — после КАЖДОГО git push)
═══════════════════════════════════════════

После `git push` ОБЯЗАТЕЛЬНО использовать:

```bash
python3 {plugin_root}/scripts/pdlc_vcs.py git-push \
    --branch "$PDLC_EXPECTED_BRANCH" \
    --project-root "$PDLC_WORK_DIR"
# (в Phase C при первом пуше новой ветки добавь --set-upstream)
```

**Никогда** не ограничивайся bare `git push` — Bitbucket Server (и иногда
GitHub) возвращают `exit 0` даже когда pre-receive/post-receive hook
или DB-constraint отказали в приёме коммита через `remote: fatal` /
`remote: ERROR` / `pre-receive hook declined` / `value too long for type` /
`duplicate key value`. Хелпер сверяет локальный branch SHA
(`refs/heads/<branch>`, НЕ `HEAD`) с remote SHA и сканирует stdout+stderr
на известные failure-паттерны.

- `exit=0` → push verified, можно продолжать.
- `exit=2` → push verification failed. НЕ ставь `done`/`review` — ставь
  **`waiting_pm`**, в `waitingForPM` процитируй `remote_lines` и `reason`
  из JSON-вывода. STOP.

Контракт: OPS-028 / issue #75.

═══════════════════════════════════════════
ВЕРНИ В КОНЦЕ
═══════════════════════════════════════════

После завершения верни структурированный ответ:

РЕЗУЛЬТАТ (верни СТРОГО в JSON формате):
```json
{
  "status": "code_complete | blocked | waiting_pm",
  "files_changed": ["path/to/file1.ts", "path/to/file2.ts"],
  "commit_hash": "abc1234",
  "commits": [
    {"phase": "tests_red", "hash": "abc1234"},
    {"phase": "implementation", "hash": "def5678"}
  ],
  "learnings": ["новый паттерн или особенность проекта"],
  "questions": ["вопрос к PM, если статус waiting_pm"]
}
```

- `commit_hash` = финальный implementation commit (backward-compatible)
- `commits` = optional массив с фазами (при test-along: один элемент `{"phase": "implementation", "hash": "..."}`)


⚠️ НЕ ВОЗВРАЩАЙ status: "done"! Только code_complete.
done ставится ТОЛЬКО PM-ом после merge PR!
```
