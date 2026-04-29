# Resume Procedure (OPS-008) — `/pdlc:continue`

Этот файл — source-of-truth для resume-процедуры, которая запускается на
Уровне 0 приоритета `/pdlc:continue` для TASK в статусе `review`.

SKILL.md `## Приоритет выбора → Уровень 0` ссылается сюда.

Процедура состоит из четырёх фаз: A (resolve workspace), B (auto-discover PR),
C (create PR если не нашли), D (quality review).

---

0. `review` — resume-процедура (OPS-008):

   **Phase A — Resolve workspace** (не полагаемся на env предыдущей сессии):
   1. `expected = compute_expected_branch(TASK)` — детерминировано из TASK
      frontmatter (parent + slug), правила именования — в implement §1.7
   2. Если `workspaceMode == "worktree"`:
      - Найти worktree по expected_branch:
        `git worktree list --porcelain` → найти `branch refs/heads/{expected}`
      - Если найден → `WORK_DIR` = путь worktree (всё уже настроено)
      - Если НЕ найден (worktree удалён/pruned) →
        - `git worktree add .worktrees/<dir> <expected>` (ATTACH без `-b`)
        - Скопировать `.state/` (кроме `counters.json`!):
          ⚠️ Каждую команду выполнять ОТДЕЛЬНЫМ Bash-вызовом
          (НЕ цепочкой через `&&` — ломает matching permissions в settings.json):
          `mkdir -p {wt}/.state`
          `cp .state/PROJECT_STATE.json {wt}/.state/`
          `cp .state/knowledge.json {wt}/.state/`
        - `.claude/` уже в worktree (tracked) — НЕ симлинк
        - Симлинк dep-каталогов: `.venv`, `node_modules`, `vendor`
          (если есть в project_root → `ln -s`)
        - `WORK_DIR` = путь нового worktree
      Иначе (inplace):
      - `git checkout <expected>` → `WORK_DIR` = project_root
   3. OPS-001 assertion: `cd "$WORK_DIR" && test "$(git rev-parse --abbrev-ref HEAD)" = "<expected>"`
      Если упало → STOP, `blocked` reason=OPS-001
   4. Clean working tree: `cd "$WORK_DIR" && git status --porcelain`
      Если не чисто → `waiting_pm` reason="uncommitted changes in resume workspace"
   5. Экспортировать: `PDLC_GIT_BRANCHING`, `PDLC_EXPECTED_BRANCH`, `PDLC_WORK_DIR`

   **Phase B — Auto-discover PR:**
   1. Прочитать `pr_url` из TASK frontmatter (source of truth;
      НЕ из PROJECT_STATE.artifacts — его schema не расширяем)
   2. Если `pr_url` заполнен → перейти к Phase D
   3. Если пуст → discover:
      `python3 {plugin_root}/scripts/pdlc_vcs.py pr-list --head <expected_branch> --state OPEN --format json --project-root "${PDLC_WORK_DIR:-.}"`
      (провайдер выбирается автоматически из settings.vcsProvider; GitHub — через gh, Bitbucket — через REST API)
   4. Если discovery нашёл PR → записать `pr_url` в TASK frontmatter → Phase D

   **Phase C — Create PR** (если Phase B не нашла):
   1. `python3 {plugin_root}/scripts/pdlc_vcs.py git-push --branch <expected_branch> --set-upstream --project-root "$WORK_DIR"`
      — **OPS-028**: verified push (не bare `git push`). Хелпер сверяет
      local SHA с remote SHA через `git ls-remote` и сканирует вывод на
      `remote: fatal` / `remote: ERROR` / `pre-receive hook declined` /
      `value too long for type` / `duplicate key value` / `! [rejected]` /
      `non-fast-forward` / `failed to push`.
      - exit=0 → продолжить (step 2: pr-create)
      - exit=2 → push verification failed: status → **`waiting_pm`**, в
        `waitingForPM` процитировать `remote_lines` и `reason` из JSON-вывода.
        STOP. **НЕ** `git merge`, **НЕ** `git push origin main`,
        **НЕ** `branch -D`.
   2. `python3 {plugin_root}/scripts/pdlc_vcs.py pr-create --title "[TASK-XXX] ..." --body-file <PR_BODY_FILE> --head <expected_branch> --project-root "${PDLC_WORK_DIR:-.}"`
   3. При успехе → записать `pr_url` в TASK frontmatter → Phase D
   4. При ошибке (нет `.env` для Bitbucket / токен невалиден / VCS CLI недоступен):
      status → **`waiting_pm`** (НЕ `blocked` — иначе зацикливается)
      Добавить в `waitingForPM` вопрос:
      `"TASK-XXX: автоматическое создание PR не удалось. Ветка: <expected_branch>. Создайте PR вручную через web UI и запустите /pdlc:unblock чтобы указать URL. Для диагностики VCS: /pdlc:doctor --vcs."`
      ⛔ **НЕ** `git merge`, **НЕ** `git push origin main`, **НЕ** `branch -D`.
      STOP

   **Phase D — Quality review** (существующая логика без изменений):
   запусти Independent Quality Review, merge после PASS
