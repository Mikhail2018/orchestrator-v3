# Reviewer Prompt — `/pdlc:review-pr`

Полный шаблон промпта для reviewer-субагента. SKILL.md `### 2. Запустить
Review субагент` ссылается сюда.

Здесь живёт:
- Pre-fetch последовательность (`pr-diff` + `pr-view` через `pdlc_vcs.py`)
- Codex-режим (codex exec) и self-режим (heredoc)
- Полный текст промпта ревьюера со всеми критериями
- Шаг публикации ответа как комментария к PR (`pr-comment`)

---

### 2. Запустить Review субагент

```
Task tool:
  subagent_type: "general-purpose"
  description: "Review PR #N for TASK-XXX"
  prompt: [см. ниже]
```

Субагент выполняет через Bash (timeout: 300000ms) из корня текущего проекта:

**Шаг 1: Pre-fetch diff и PR description (субагент, до вызова ревьюера):**

```bash
TASK_ID="{TASK-XXX}"
PR_NUM="{N}"

PR_DIFF=$(python3 {plugin_root}/scripts/pdlc_vcs.py pr-diff "${PR_NUM}" --project-root "${PDLC_WORK_DIR:-.}")
PR_DESC=$(python3 {plugin_root}/scripts/pdlc_vcs.py pr-view "${PR_NUM}" --fields title,body,files --format json --project-root "${PDLC_WORK_DIR:-.}")
```

**Шаг 2: Передать данные в промпт ревьюера:**

**Режим `self`** — заменить `codex exec ...` на CLI текущего агента:

| Агент | Команда |
|---|---|
| Claude Code | `cat <<PROMPT \| claude -p` (heredoc без кавычек — переменные раскрываются) |
| Codex CLI | `codex exec --full-auto -m gpt-5.3-codex -c model_reasoning_effort='"high"' "PROMPT"` |
| Qwen CLI | `cat <<PROMPT \| qwen-code --allowed-tools=run_shell_command -p` (heredoc без кавычек) |
| GigaCode | `cat <<PROMPT \| gigacode --allowed-tools=run_shell_command -p` (heredoc без кавычек) |

Агент определяет свой CLI по системному контексту. Heredoc **без кавычек** (`<<PROMPT`, не `<<'PROMPT'`), чтобы `${PR_DIFF}`, `${PR_DESC}`, `${TASK_ID}` раскрылись.

> **OPS-022:** argv для self-CLI берётся из `cli-capabilities.yaml:targets.<cli>.non_interactive_args` и проверяется linter-ом (`pdlc_lint_skills.py::check_self_reviewer_tables`) на строгое равенство с ячейкой таблицы.

**Режим Codex (по умолчанию):**

```bash
cd {worktree_path_or_project_root} && codex exec \
  --full-auto \
  -m gpt-5.3-codex \
  -c model_reasoning_effort='"high"' \
"
Ты — независимый ревьюер кода. Проведи quality review Pull Request #${PR_NUM} на соответствие требованиям задачи ${TASK_ID}.

PR DESCRIPTION:
${PR_DESC}

PR DIFF:
${PR_DIFF}

Алгоритм:
1) Diff и описание PR уже предоставлены выше — используй их
2) Найди файл задачи ${TASK_ID} в репозитории (tasks/)
3) Прочитай задачу целиком (включая metadata/frontmatter)
3.5) Если TASK.design_refs non-empty И TASK.design_waiver != true:
   - Прочитай КАЖДЫЙ файл из design_refs (конкретные файлы:
     api.md, data-model.md, sequences.md, state-machines.md, etc.)
   - Сравни PR diff против design-контрактов:
     • API endpoints: paths, methods, request/response schemas, status codes
     • Data model: entities, fields, types, relationships
     • Sequences: порядок вызовов, error paths
     • States: состояния, переходы
   - Если PR добавляет/меняет endpoint/entity/flow, отсутствующий в design:
     проверь что design-артефакт ОБНОВЛЁН в этом же PR
4) Определи родительскую задачу и восстанови intent
5) Используй AGENTS.md как системные гайдлайны проекта
6) Проверь каждый изменённый файл на качество кода
7) Прочитай тесты — оцени покрытие новой функциональности

Критерии оценки (1-10):
- Acceptance criteria: все ли требования из TASK выполнены
- Полнота: нет ли пропущенных частей задачи
- Качество: паттерны, error handling, naming, архитектура
- Тесты: покрытие, edge cases, корректность assertions
- Безопасность: нет hardcode, injection, секретов в коде
- Constraints compliance: если parent SPEC имеет секцию 4 (Constraints C-N),
  проверь что PR не нарушает ни одного constraint (например, если C-1
  фиксирует PostgreSQL — код не использует другую СУБД; если C-2 — GDPR —
  данные EU users не утекают за пределы EU-region)
- System boundary compliance: если parent SPEC имеет `system_boundary` и
  `external_systems` в frontmatter — проверь что PR НЕ содержит:
  (1) production-кода внешних систем (только клиенты/адаптеры на нашей стороне),
  (2) модификаций consumed-контрактов (`docs/contracts/consumed/`),
  (3) реализации логики внешних систем вместо интеграционных адаптеров
- Design conformance: если TASK.design_refs non-empty И TASK.design_waiver != true —
  проверь что реализация соответствует контрактам из design_refs (шаг 3.5);
  если PR содержит drift (новые endpoints/entities/flows не из design) —
  проверь что design-артефакты ОБНОВЛЕНЫ в этом же PR;
  N/A если design_refs пуст или design_waiver: true

Формат ответа:
ОЦЕНКИ:
- Acceptance criteria: X/10 — {обоснование}
- Полнота: X/10 — {обоснование}
- Качество: X/10 — {обоснование}
- Тесты: X/10 — {обоснование}
- Безопасность: X/10 — {обоснование}
- Constraints compliance: X/10 — {обоснование, или N/A если нет constraints в SPEC}
- System boundary: X/10 — {обоснование, или N/A если нет external_systems в SPEC}
- Design conformance: X/10 — {обоснование, или N/A если нет design_refs или design_waiver}
- ИТОГО: X/10

КРИТИЧНЫЕ ПРОБЛЕМЫ (блокеры, если есть):
1. {file:line}: {проблема} → {как исправить}

УЛУЧШЕНИЯ (конкретные):
1. {file:line}: {что изменить} → {как изменить}

ВЕРДИКТ: PASS (>= 8) | IMPROVE (< 8)
"
```

**Шаг 3: Опубликовать ответ ревьюера как комментарий к PR:**

После получения ответа от ревьюера — сразу опубликовать сырой результат в PR:

```bash
python3 {plugin_root}/scripts/pdlc_vcs.py pr-comment "${PR_NUM}" --body-stdin \
  --project-root "${PDLC_WORK_DIR:-.}" <<'REVIEW_EOF'
## 🤖 Quality Review — Iteration {iteration_number}

**Reviewer:** {REVIEWER_NAME}
**Task:** ${TASK_ID}

---

{сырой ответ ревьюера}

---

_Automated review by Polisade Orchestrator_
REVIEW_EOF
```

Если `pr-comment` завершился ошибкой — залогировать warning и продолжить работу.

**Важно для субагента:**
- `{worktree_path_or_project_root}` — если задача выполнялась в worktree, используй путь worktree. Иначе — корень проекта.
- `{TASK-XXX}` — ID задачи из шага 1
- `{N}` — номер PR из шага 1
- `{REVIEWER_NAME}` — в режиме Codex: `OpenAI Codex CLI (gpt-5.3-codex)`; в режиме self: `{Agent Name} (self-review)` (напр. `Claude Code (self-review)`)
- Субагент pre-fetch'ит diff и PR description через `pdlc_vcs.py` и передаёт в промпт ревьюера. Ревьюер навигирует проект для чтения TASK, parent, AGENTS.md и исходного кода.
- После получения ответа — обязательно опубликовать его как комментарий к PR через `pdlc_vcs.py pr-comment` (шаг 3)
