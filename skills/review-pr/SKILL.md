---
name: review-pr
description: 'Run a quality review on an open pull request (by PR number or linked TASK), externally via codex or internally via a clean-context self-review, and post the verdict. Use when PM mentions "review PR", "PR review", "review pull request", "quality review", "review this pr", "сделай ревью pr", or any request to evaluate an open PR before merge. Trigger liberally — under-triggering lets PRs land without a second look; over-triggering is recoverable (PM can ignore the review comments).'
argument-hint: "[PR# or TASK-XXX] [self]"
cli_requires: "task_tool, codex_cli"
fallback: self
---

# /pdlc:review-pr [PR# or TASK-XXX] [self] — PR Quality Review (external CLI or self)

Независимый quality review Pull Request. По умолчанию — через внешний reviewer CLI (OpenAI Codex, `gpt-5.3-codex`). С флагом `self` — через CLI текущего агента в отдельном процессе (чистый контекст).

> **Флаг `self`** — для случаев, когда доступна подписка только на один агент. Ревью проводится тем же CLI, но в изолированном процессе.

**Цикл:** `implement → PR → review-pr → fix замечаний → merge`

## Использование

```
/pdlc:review-pr 42             # Review PR #42 через reviewer CLI
/pdlc:review-pr TASK-001       # Найти PR для TASK-001
/pdlc:review-pr                # Review PR текущей ветки
/pdlc:review-pr 42 self        # Review PR #42 через текущий агент
/pdlc:review-pr TASK-001 self  # PR для TASK + self-review
/pdlc:review-pr self           # Текущая ветка + self-review
```

## Архитектура

```
/pdlc:review-pr 42
         |
         v
+-----------------------------------------+
|  ОСНОВНОЙ АГЕНТ                          |
|  1. Определить PR# и TASK-ID            |
|  2. Запустить Review субагент            |
+-----------------+-----------------------+
                  v
+-----------------------------------------+
|  СУБАГЕНТ (general-purpose)              |
|  1. Pre-fetch: pr-diff + pr-view (vcs.py)|
|  2. Bash: codex exec --full-auto         |
|     (или CLI текущего агента при self)    |
|  Ревьюер получает:                       |
|   - diff и PR description в промпте      |
|   - Читает TASK, parent, AGENTS.md       |
|   - Анализирует код/тесты                |
|  3. pr-comment: публикация ревью в PR    |
|  Возвращает ревью с score и findings     |
+-----------------+-----------------------+
                  v
+-----------------------------------------+
|  ОСНОВНОЙ АГЕНТ                          |
|  Score >= 8 → merge                      |
|  Score < 8 →                             |
|    Improvement субагент → fix            |
|    → Re-review (макс. 2 итерации)        |
|  → After 2 iterations: STOP + waiting_pm |
+-----------------------------------------+
```

**Anti-loop safety:** Максимум 2 итерации (review+improve). После 2-й — STOP, ждём PM.

## Алгоритм

### 1. Определить PR, TASK-ID и режим

Из аргументов извлечь:
- **PR/TASK-ID**: число (`42`) или `TASK-XXX` если указано
- **Режим**: если слово `self` присутствует в аргументах — форсировать `self`. Иначе — спросить единый OPS-011 helper:

  ```bash
  caps=$(python3 {plugin_root}/scripts/pdlc_cli_caps.py detect)
  mode=$(python3 -c 'import json,sys; print(json.loads(sys.argv[1])["reviewer"]["mode"])' "$caps")
  # OPS-007 / issue #55: surface codex-impersonator warnings so a foreign
  # `codex` binary in PATH is never silently ignored.
  warning=$(python3 -c 'import json,sys; print(json.loads(sys.argv[1])["reviewer"].get("warning") or "")' "$caps")
  [ -n "$warning" ] && echo "⚠ $warning"
  # mode: "codex" | "self" | "blocked" | "off"
  ```

  - `mode == "codex"` → использовать `codex exec` (раздел ниже).
  - `mode == "self"` → использовать CLI текущего агента (таблица ниже).
  - `mode == "blocked"` → STOP с диагностикой (`reviewer.reason`).
  - `mode == "off"` → STOP; reviewer отключён в settings, PM делает ревью руками.

  Никакого повторного `which codex` / `which {own_cli}` — helper уже видит окружение и возвращает правильный режим для любого target CLI (Qwen/GigaCode → self без явного ветвления).

- Если аргумент — число (напр. `42`) → PR #42, TASK-ID из PR body
- Если аргумент — `TASK-XXX` → найти PR по ветке/коммитам этой TASK
- Если нет аргумента (кроме `self`) → определить по текущей ветке:

```bash
python3 {plugin_root}/scripts/pdlc_vcs.py pr-list --head "$(git branch --show-current)" --format json --project-root "${PDLC_WORK_DIR:-.}" | jq -r '.[0].number // empty'
```

- Если PR не найден → ошибка:

```
═══════════════════════════════════════════
PR QUALITY REVIEW — ОШИБКА
═══════════════════════════════════════════
PR не найден.

Возможные причины:
- Ветка не запушена
- PR не создан

-> Создай PR: /pdlc:continue (автоматически) или /pdlc:pr — проверь статус
═══════════════════════════════════════════
```

Определение TASK-ID:
1. Из PR title: `[TASK-XXX]` паттерн
2. Из PR body: поиск `TASK-XXX`
3. Из коммитов PR: `[TASK-XXX]` в сообщениях

### 2. Запустить Review субагент

```
Task tool:
  subagent_type: "general-purpose"
  description: "Review PR #N for TASK-XXX"
  prompt: [см. шаблон в reference]
```

Полный шаблон промпта ревьюера — со всеми криными секциями: pre-fetch
diff/PR description через `pdlc_vcs.py`, codex-mode и self-mode (heredoc),
критерии оценки (acceptance/полнота/качество/тесты/безопасность/constraints/
system_boundary/design conformance), формат ответа (ОЦЕНКИ + ВЕРДИКТ),
публикация результата как комментарий к PR через `pr-comment` —
вынесен в:

**`skills/review-pr/references/reviewer-prompt.md`** —
читай через Read tool ПЕРЕД формированием prompt и инжектируй данные
из шага 1 (PR_NUM, TASK_ID, worktree_path, REVIEWER_NAME).

⛔ **НЕ передавай** содержимое промпта от руки. Шаблон — единая точка
обновления; правки делаются в reference-файле.

**Режим `self`** — заменить `codex exec ...` на CLI текущего агента:

| Агент | Команда |
|---|---|
| Claude Code | `cat <<PROMPT \| claude -p` (heredoc без кавычек — переменные раскрываются) |
| Codex CLI | `codex exec --full-auto -m gpt-5.3-codex -c model_reasoning_effort='"high"' "PROMPT"` |
| Qwen CLI | `cat <<PROMPT \| qwen-code --allowed-tools=run_shell_command -p` (heredoc без кавычек) |
| GigaCode | `cat <<PROMPT \| gigacode --allowed-tools=run_shell_command -p` (heredoc без кавычек) |

> **OPS-022:** argv для self-CLI берётся из `cli-capabilities.yaml:targets.<cli>.non_interactive_args` и проверяется linter-ом (`pdlc_lint_skills.py::check_self_reviewer_tables`) на строгое равенство с ячейкой таблицы. Изменение argv делается ОДНОВРЕМЕННО в manifest и в этой таблице.

### 3. Обработать результат

- Парсить ИТОГО score и ВЕРДИКТ из ответа ревьюера
- Если ошибка (timeout, API, не установлен) → показать с рекомендациями

### 4. Если IMPROVE (score < 8) — Improvement субагент

```
Task tool:
  subagent_type: "general-purpose"
  description: "Fix PR #N based on review"
  prompt: [см. шаблон в reference]
```

Полный prompt для improvement-субагента (с инструкциями: применять только
рекомендации ревью, гонять тесты, OPS-028 verified push, OPS-010
бандлинг status в `improvement` commit) — в:

**`skills/review-pr/references/improvement-prompt.md`** —
читай через Read tool ПЕРЕД запуском улучшающего субагента.

### 5. Re-review (если был IMPROVE)

- Повторный запуск review (шаг 2) + публикация комментария в PR (шаг 3.1)
- Комментарий публикуется после **каждой** итерации — без исключений
- После 2-й итерации — STOP, ждём PM

```python
iterations = 0
while iterations < 2:
    review = run_review(pr, task_id)  # external CLI or self
    post_pr_comment(pr, review, iterations + 1)  # pdlc_vcs.py pr-comment
    iterations += 1
    if review.score >= 8:  # PASS
        merge_pr(pr)
        delete_branch()
        set_status(task_id, "done")
        # OPS-010: post-merge `status=done` — терминальная правка.
        # Бандли frontmatter TASK.md + PROJECT_STATE.json в единственный
        # `finalize` commit `[TASK-ID] Finalize status: done (PR #N)`.
        # diff: только TASK.md frontmatter + PROJECT_STATE.json. НЕ пиши
        # lastUpdated.
        break
    else:  # IMPROVE
        run_improvement(pr, review.recommendations)
        run_all_tests()
        # OPS-028: commit_and_push() =
        #   git commit ... && python3 {plugin_root}/scripts/pdlc_vcs.py git-push \
        #       --branch <branch> --project-root "${PDLC_WORK_DIR:-.}"
        # На exit=2 (push verification failed) →
        #   set_status(task_id, "waiting_pm")
        #   update_project_state(task_id, "waitingForPM",
        #       reason=f"Push failed: {json['reason']}",
        #       remote_lines=json['remote_lines'])
        #   break  # НЕ продолжаем re-review, НЕ мёржим
        commit_and_push()
else:
    # Max iterations — STOP, ждём PM
    set_status(task_id, "waiting_pm")
    update_project_state(task_id, "waitingForPM",
        reason=f"Review: score {review.score}/10 after 2 iterations")
    # OPS-010: терминальный waiting_pm без следующего семантического
    # коммита — бандли set_status + update_project_state в единственный
    # `finalize` commit `[TASK-ID] Finalize status: waiting_pm (PR #N)`.
    # diff: только TASK.md frontmatter + PROJECT_STATE.json. НЕ пиши
    # lastUpdated — поле всегда null (OPS-010 / issue #58).
    STOP  # вернуть управление PM
```

⛔ **НЕ пиши `lastUpdated`** в PROJECT_STATE.json на любом шаге review-pr —
поле зарезервировано, всегда `null` (OPS-010 / issue #58). Для времени
последнего изменения используй `git log -1 --format=%cI .state/PROJECT_STATE.json`.

### 6. Merge PR

После PASS:

```bash
# Merge PR (squash and delete branch)
python3 {plugin_root}/scripts/pdlc_vcs.py pr-merge {N} --squash --delete-branch --project-root "${PDLC_WORK_DIR:-.}"
```

```
Провайдер блокирует approve собственного PR!
Решение: После успешного quality review — merge напрямую (без approve).
```

Обновить TASK status → done в PROJECT_STATE.json.

### 7. Логирование в session-log

Добавь запись в `.state/session-log.md`:
```markdown
### PR Quality Review: PR #{N} (TASK-{ID})
- Date: {today}
- Reviewer: {REVIEWER_NAME}
- Iteration 1: {score}/10 → {PASS|IMPROVE}
- Iteration 2: {score}/10 → {PASS|IMPROVE} (если была)
- Command: /pdlc:review-pr
- Result: merged | improvements_applied
```

## Формат вывода

Все варианты вывода (PASS с первой итерации, IMPROVE → PASS, STOP после
2 итераций, ошибка ревьюера) — в:

**`skills/review-pr/references/output-formats.md`** —
читай через Read tool ПЕРЕД печатью результата. В режиме `self`
заменяй "PR QUALITY REVIEW" → "PR REVIEW (SELF)" и `Reviewer:` →
CLI текущего агента (см. шаблоны в reference-файле).

## Интеграция с автономным циклом

Когда вызывается из `/pdlc:continue` или `/pdlc:implement`:

```
1. Ревьюер (Codex CLI или CLI текущего агента в режиме `self`) получает чистый контекст
2. Субагент pre-fetch'ит diff и PR description, ревьюер читает TASK, parent, AGENTS.md
3. Оценивает PR diff vs TASK requirements (независимый reviewer)
4. Если PASS → основной агент делает merge
5. Если IMPROVE → improvement субагент исправляет → re-review
6. После 2 итераций с score < 8 → STOP, waiting_pm (PM decides)
7. После merge → статус TASK → done
```

**Режим Codex (по умолчанию):** ревью делает OpenAI Codex CLI — другая модель, другой провайдер, полностью независимое второе мнение.

**Режим `self`:** ревью делает тот же агент, но в изолированном CLI-процессе (чистый контекст). Не полноценное "второе мнение", но независимость от текущей сессии.

## Важно

- Ревьюер запускается в автономном режиме — сам навигирует проект (Codex: `--full-auto`, self: зависит от CLI)
- Diff и PR description pre-fetch'атся субагентом и передаются в промпт ревьюера
- Improvement субагент наследует модель parent — качественное применение рекомендаций
- Максимум 2 итерации review+improve — anti-loop safety
- После 2 итераций — STOP + waiting_pm, PM решает дальнейшие действия
- PM не делает code review — это автоматизированный процесс
- GitHub не позволяет approve свой PR — merge напрямую после PASS
- Timeout 300s — достаточно для анализа PR
