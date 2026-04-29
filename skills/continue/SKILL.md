---
name: continue
description: Autonomous work — find and execute ready tasks
cli_requires: "task_tool, codex_cli"
fallback: self
---

# /pdlc:continue — Автономная работа

Найти готовые к работе артефакты и выполнять их автономно.

---

## ⛔ КРИТИЧЕСКИ ВАЖНО: Полный цикл обязателен!

```
┌─────────────────────────────────────────────────────────────┐
│  ⛔ ЗАПРЕЩЕНО ставить status: done если:                    │
│                                                             │
│     ✗ Regression tests НЕ прогнаны                          │
│     ✗ PR НЕ создан                                          │
│     ✗ Review НЕ пройден                                     │
│     ✗ Merge НЕ выполнен                                     │
│                                                             │
│  После написания кода статус: in_progress                   │
│  После создания PR статус: review                           │
│  После merge PR статус: done                                │
└─────────────────────────────────────────────────────────────┘
```

**Каждая TASK проходит ПОЛНЫЙ ЦИКЛ:**
```
1. IMPLEMENT    → код + unit tests + commit      → статус: in_progress
2. REGRESSION   → запуск ВСЕХ тестов проекта     → исправить если упали
3. CREATE PR    → push + pr-create (pdlc_vcs.py)  → статус: review
4. REVIEW LOOP  → ждать/исправлять               → повторять до approve
5. MERGE        → pr-merge + delete-branch       → статус: done
```

**НЕ ПЕРЕХОДИ к следующей TASK пока текущая не завершена полностью!**

---

## Использование

```
/pdlc:continue    # Начать автономную работу
```

## Алгоритм

1. Прочитай `.state/PROJECT_STATE.json`
2. Прочитай `.state/knowledge.json` для контекста проекта
3. Проверь `waitingForPM`:
   - Если не пусто → "Есть N вопросов к тебе. Запусти /pdlc:unblock сначала."
4. Проверь `readyToWork`:
   - Если пусто → "Всё сделано или заблокировано. /pdlc:state для деталей."
5. Если `workspaceMode: "worktree"`:
   - Выполни `git worktree list --porcelain`
   - Извлеки ветки активных worktree
   - Сопоставь ветки → TASK-ID (по конвенции именования)
   - Пропусти задачи с активным worktree (заняты другим агентом)
6. Выбери задачу по приоритету (см. ниже)
7. **Выполни ПОЛНЫЙ ЦИКЛ для задачи** (см. ниже)
8. Повтори с шага 1

## Полный цикл для TASK (максимальная автономность)

```
┌─────────────────────────────────────────────────────────────┐
│  АВТОНОМНЫЙ ЦИКЛ TASK                                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. IMPLEMENT                                               │
│     ├─ Создать ветку (gitBranching: true)                   │
│     ├─ Реализовать код через субагент                       │
│     ├─ Написать unit тесты                                  │
│     └─ Коммит                                               │
│              │                                              │
│              ▼                                              │
│  2. REGRESSION TEST                                         │
│     ├─ Запустить ВСЕ тесты проекта                          │
│     ├─ Если упали → исправить → коммит → повторить          │
│     └─ Если прошли → продолжить                             │
│              │                                              │
│              ▼                                              │
│  3. CREATE PR                                               │
│     ├─ Push ветки                                           │
│     ├─ Создать Pull Request                                 │
│     └─ TASK → status: review                                │
│              │                                              │
│              ▼                                              │
│  3.5. PRE-CHECK: REVIEWER CLI                              │
│     ├─ python3 scripts/pdlc_cli_caps.py detect             │
│     │    → reviewer.mode = codex | self | blocked          │
│     └─ mode=blocked → STOP с диагностикой                  │
│              │                                              │
│              ▼                                              │
│  4. QUALITY REVIEW (Independent)                            │
│     ├─ /pdlc:review-pr [self] для независимого ревью       │
│     ├─ Ревьюер оценивает diff vs TASK                       │
│     ├─ Если score < 8 → improve → re-review (макс 2 итер.) │
│     └─ Если PASS → merge → delete branch → TASK: done       │
│              │                                              │
│              ▼                                              │
│  5. NEXT TASK                                               │
│     └─ Вернуться к шагу 1 алгоритма                         │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  ПРЕРЫВАНИЕ ЦИКЛА (только при):                             │
│  • waiting_pm — нужно решение PM                            │
│  • blocked — неразрешимая техническая проблема              │
│  • Все задачи завершены                                     │
├─────────────────────────────────────────────────────────────┤
│  НЕ ПРЕРЫВАЙСЯ для:                                         │
│  • Падающих тестов — исправь автоматически                  │
│  • Review замечаний — исправь автоматически                 │
│  • Merge конфликтов — разреши автоматически                 │
└─────────────────────────────────────────────────────────────┘
```

### Детали каждого шага

**1. IMPLEMENT**
- Создай ветку согласно режиму (PLAN mode или стандартный)
- Запусти субагент для реализации
- Субагент пишет код + unit тесты для новой функциональности
- Коммит с сообщением `[TASK-XXX] description`

**2. REGRESSION TEST**
```
while tests_failing:
    run_all_project_tests()
    if failed:
        analyze_failures()
        fix_code()
        commit("[TASK-XXX] Fix test failures")
    else:
        break
```

**3. CREATE PR**
- Push ветки на remote
- Создать PR через `pdlc_vcs.py pr-create` (провайдер выбирается автоматически из settings.vcsProvider)
- Обновить статус TASK → `review`
  <!-- # OPS-010: правка frontmatter TASK.md + PROJECT_STATE.json task-bucket
  НЕ отдельный commit. Либо бандли в следующий `commit_and_push()`
  (IMPROVE-ветка review-loop), либо — при терминальном STOP через
  review_mode="off"/"blocked" — вынеси в единственный `finalize` commit
  `[TASK-ID] Finalize status: review (PR #N)`. НЕ пиши lastUpdated
  в PROJECT_STATE.json (OPS-010 / issue #58) — поле всегда null. -->
- ⛔ **НЕ пиши `lastUpdated`** в PROJECT_STATE.json — поле зарезервировано,
  всегда `null` (OPS-010 / issue #58).

**4. QUALITY REVIEW И MERGE (автоматически, без вопросов!)**

```
⚠️ ВАЖНО:
- PM не делает code review — это автоматизированный процесс
- НЕ СПРАШИВАЙ "Продолжить?" — делай автоматически!
- Ревью делает Codex CLI (default) или текущий агент (`self`) — независимое мнение!
- GitHub не позволяет approve свой PR — merge напрямую после PASS
```

Алгоритм:
0. Pre-check: определить reviewer CLI через единый helper (OPS-011):
   - `caps=$(python3 {plugin_root}/scripts/pdlc_cli_caps.py detect)` → JSON с `reviewer.mode`
   - `review_mode = caps.reviewer.mode` → `"codex"` | `"self"` | `"blocked"` | `"off"` (OPS-017)
   - `reason = caps.reviewer.reason` (может быть пустым)
   - `warning = caps.reviewer.warning` — OPS-007 / issue #55: непустое значение означает, что в PATH нашёлся чужой `codex` и был отбит identity-проверкой. Напечатать `⚠ {warning}` перед ветвлением по `review_mode`, чтобы самозванец не оставался silent в логе.
0a. Если `review_mode == "off"` → STOP: reviewer отключён в `settings.reviewer.mode`. TASK остаётся в `review` с созданным PR; PM делает ревью руками и выполняет merge через `/pdlc:pr merge <id>` (или закрывает PR).
0b. Если `review_mode == "blocked"` → STOP. Прочитать `reason` и подсказать соответственно:
   - Если `reason` упоминает «settings …» → проблема в настройках:
     ```
     ═══════════════════════════════════════════
     REVIEWER BLOCKED
     ═══════════════════════════════════════════
     Reason: {reason}

     Проверьте settings.reviewer.mode и settings.reviewer.cli
     в .state/PROJECT_STATE.json — текущее значение конфликтует
     с доступными CLI в окружении.

     TASK остаётся в статусе: review
     ═══════════════════════════════════════════
     ```
   - Иначе (reason не задан или указывает на отсутствие CLI) — показать варианты установки:
     ```
     ═══════════════════════════════════════════
     REVIEWER CLI НЕ НАЙДЕН
     ═══════════════════════════════════════════
     Reason: {reason or "no reviewer CLI available"}

     Quality review требует CLI ревьюера.

     Варианты:
       • Codex CLI: npm install -g @openai/codex
       • Claude Code: https://docs.anthropic.com/claude-code
       • Qwen CLI: документация Qwen

     TASK остаётся в статусе: review
     ═══════════════════════════════════════════
     ```
1. Запусти `/pdlc:review-pr` (или `/pdlc:review-pr self` если `review_mode == self`) — независимый quality review:
   - Ревьюер оценивает PR diff vs TASK requirements
   - Score 1-10 по критериям (acceptance, полнота, качество, тесты, безопасность)
2. Если score < 8 (IMPROVE):
   - Improvement субагент исправляет код
   - Прогоняет тесты, коммит, push
   - Re-review (макс. 2 итерации)
3. Если score >= 8 (PASS):
   - **Merge** (self-approve блокируется провайдером — merge напрямую)
   - `python3 {plugin_root}/scripts/pdlc_vcs.py pr-merge N --squash --delete-branch --project-root "${PDLC_WORK_DIR:-.}"`
   - Статус TASK → done
   - **Автоматически продолжи** со следующей задачей (не спрашивай!)
4. Если 2 итерации пройдены и score < 8:
   - **STOP** — НЕ мержить, НЕ переходить к следующей задаче
   - Статус TASK → waiting_pm
   - Вывести подробный отчёт и варианты для PM

```python
iterations = 0
while iterations < 2:
    review = run_review(pr_number, review_mode)  # Codex CLI or self
    iterations += 1
    if review.score >= 8:  # PASS
        run('python3 {plugin_root}/scripts/pdlc_vcs.py pr-merge N --squash --delete-branch --project-root "${PDLC_WORK_DIR:-.}"')
        task.status = "done"
        # OPS-010: post-merge `status=done` — терминальная правка frontmatter
        # TASK.md + PROJECT_STATE.json. Если семантического коммита больше нет
        # (переход к следующей ready TASK в шаге 5 запускает новый цикл), это
        # единственный `finalize` commit `[TASK-ID] Finalize status: done (PR #N)`.
        # diff: только TASK.md frontmatter + PROJECT_STATE.json. НЕ пиши lastUpdated.
        break
    else:  # IMPROVE
        run_improvement(review.recommendations)
        run_all_tests()
        # OPS-028: commit_and_push() =
        #   git commit ... && python3 {plugin_root}/scripts/pdlc_vcs.py git-push \
        #       --branch <expected_branch> --project-root "$WORK_DIR"
        # На exit=2 (push verification failed) → task.status = "waiting_pm",
        # в waitingForPM процитировать remote_lines + reason из JSON. break.
        commit_and_push()
else:
    # Max iterations — STOP, ждём PM
    task.status = "waiting_pm"
    update_project_state(task_id, "waitingForPM",
        reason=f"Review ({review_mode}): score {review.score}/10 after 2 iterations")
    # OPS-010: терминальный waiting_pm без следующего коммита. Бандли
    # task.status + update_project_state в единственный `finalize` commit
    # `[TASK-ID] Finalize status: waiting_pm (PR #N)`. diff: только TASK.md
    # frontmatter + PROJECT_STATE.json. НЕ пиши lastUpdated.
    STOP  # НЕ переходить к следующей задаче!
```

**5. NEXT TASK**
- Вернуться к началу алгоритма
- Выбрать следующую ready задачу

## Приоритет выбора (v2.1)

### Уровень 0: Завершить начатое (ОБЯЗАТЕЛЬНО сначала!)
```
⚠️ НЕ НАЧИНАЙ новую TASK пока есть незавершённые!
```
0. `review` — resume-процедура (OPS-008):

   Полный алгоритм resume (Phase A: resolve workspace, Phase B: auto-discover PR,
   Phase C: create PR если не нашли, Phase D: quality review) — в:

   **`skills/continue/references/resume-procedure.md`** —
   читай через Read tool, когда выбираешь TASK из `inReview`.

   Ключевые инварианты процедуры:
   - source of truth для `pr_url` — frontmatter TASK (НЕ PROJECT_STATE.artifacts)
   - assertion `cd "$WORK_DIR" && git rev-parse --abbrev-ref HEAD == <expected>`
   - clean working tree через `git status --porcelain` ДО любых git-операций
   - OPS-028 verified push на Phase C; exit=2 → `waiting_pm`, не `blocked`
   - `pdlc_vcs.py pr-create` failure → `waiting_pm` (не blocked, иначе
     зацикливается); сообщение содержит "pr_url_request" (`/pdlc:unblock` ловит)

1. `changes_requested` — исправить замечания code review
2. `in_progress` — доделай начатое

### Уровень 1: Следующая по порядку (в рамках PLAN)
```
Если текущая TASK из PLAN, следующая = первая ready TASK из того же PLAN.
Не прыгай на другие PLAN пока текущий не завершён или не заблокирован.
```
3. `ready` TASK из текущего PLAN (по roadmap_item order)

### Уровень 2: Прямая реализация (если нет активного PLAN)
4. `ready` TASK от BUG — исправь баги (P0 > P1 > P2)
5. `ready` TASK — реализуй задачи
6. `ready` TASK от CHORE — выполни простые задачи

### Уровень 3: Критичный техдолг
7. `ready` TASK от DEBT (P0-P1) — security/performance

### Уровень 4: Исследование
8. `ready` SPIKE — исследуй (следи за timebox)

### Уровень 5: Создание задач (если нет готовых TASK)
9. `ready` PLAN → создай задачи (`/pdlc:tasks`)
10. `ready` SPEC → создай задачи (`/pdlc:tasks`)
11. `ready` FEAT → создай задачи (`/pdlc:tasks`) или spec если сложно

### Уровень 6: Проработка
12. `ready` PRD → создай спецификацию (`/pdlc:spec`)

### Уровень 7: Обычный техдолг
13. `ready` TASK от DEBT (P2+) — обычный техдолг

## Логика выбора из нескольких ready

Если несколько артефактов одного типа:
1. Сначала по приоритету: P0 > P1 > P2 > P3
2. Затем по дате создания: старые раньше

## Условия остановки

Останавливайся и сообщай PM когда:

### Требуется решение PM
- Бизнес-выбор (приоритет, скоуп)
- Архитектурный trade-off с последствиями
- Неясные требования

→ Поставь `waiting_pm`, добавь вопрос, продолжи с другими задачами

### Техническая блокировка
- Тесты падают и непонятно почему
- Зависимость недоступна
- Ошибка окружения

→ Поставь `blocked`, продолжи с другими задачами

### Всё готово
- Нет больше `ready` задач
- Все `waiting_pm` или `blocked`

→ Выведи итоги и останови работу

## Формат вывода

Все варианты вывода (начало работы, полный цикл одной задачи, при падении
тестов с автоисправлением, при quality review с улучшением, при блокировке,
итоги сессии) — в:

**`skills/continue/references/output-formats.md`** —
читай через Read tool ПЕРЕД печатью соответствующего блока.

## Git Branching поведение

Подробности (branch naming, worktree mode lifecycle, inplace mode fallback,
статус в .md frontmatter после merge) — в:

**`skills/continue/references/git-branching-behavior.md`** —
читай через Read tool ПЕРЕД операциями с ветками/worktree.

Source-of-truth для имён веток и `compute_expected_branch(TASK)` —
общий с `/pdlc:implement`: `skills/implement/references/branch-and-worktree-setup.md`.

## Важно

- Коммить после каждой завершённой задачи
- Не накапливай много изменений
- При сомнениях — лучше спросить PM (waiting_pm)
- Обновляй PROJECT_STATE.json после каждого изменения
- `/pdlc:implement` работает только с TASK
- Следи за timebox для SPIKE

## ⛔ ЗАПРЕЩЕНО спрашивать разрешение на продолжение!

```
НЕ ПИШИ:
  "Продолжить с TASK-039?"
  "Хочешь чтобы я продолжил?"
  "Начать следующую задачу?"

ВМЕСТО ЭТОГО — просто продолжай автоматически!
```

Автономный режим означает:
- После merge PR → сразу следующая задача
- После завершения TASK → сразу следующая задача
- Не жди подтверждения PM для технических операций

**Останавливайся ТОЛЬКО при:**
- `waiting_pm` — бизнес-вопрос к PM
- `blocked` — техническая проблема которую не можешь решить
- Все задачи завершены

## GitHub Self-Approve Limitation

GitHub не позволяет approve свой собственный PR:
```
Error: Review Can not approve your own pull request
```

**Решение:** После успешного Independent Quality Review (score >= 8) делай merge напрямую:
```bash
python3 {plugin_root}/scripts/pdlc_vcs.py pr-merge N --squash --delete-branch --project-root "${PDLC_WORK_DIR:-.}"
```

Не approve собственного PR (ни через VCS CLI, ни через REST API) — это не сработает.
