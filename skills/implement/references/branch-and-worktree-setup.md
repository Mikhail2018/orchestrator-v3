# Branch and Worktree Setup (OPS-001 GUARD)

Этот файл — source-of-truth для шага `1.7. [OPS-001 GUARD] Branch/worktree setup`
из `/pdlc:implement` и для правил branch naming, на которые опирается
`compute_expected_branch(TASK)`.

SKILL.md содержит обзорную ссылку на этот файл; полный алгоритм с assertion'ами
живёт здесь.

---

## 1. Зачем этот шаг (инвариант)

⛔ Шаг ОБЯЗАТЕЛЕН для S, M, L задач при `gitBranching: true`.
Пропуск = OPS-001 (коммит не в ту ветку, в частности в main).

На выходе шага должен выполняться **expected-branch invariant**:

```
cd "$WORK_DIR" && git rev-parse --abbrev-ref HEAD == compute_expected_branch(TASK)
```

где `compute_expected_branch(TASK)` — детерминированная функция по `parent` из
TASK frontmatter (см. §3 ниже).

**ВАЖНО для worktree mode.** `$WORK_DIR` = `worktree_path` (в корне репо ветка
остаётся на `main` — это нормальное поведение `git worktree`). Guard и все
последующие git-инспекции ВСЕГДА выполняются внутри `$WORK_DIR`.

---

## 2. Алгоритм шага

```
1. expected = compute_expected_branch(TASK)

2. Если workspaceMode == "worktree" И gitBranching: true:
     — git worktree add .worktrees/<dir> -b <expected>    (если ветки нет)
     — git worktree add .worktrees/<dir> <expected>       (если ветка уже есть)
     — WORK_DIR = .worktrees/<dir>
   Иначе если gitBranching: true (inplace):
     — git checkout <expected>                            (если ветка уже есть)
     — git checkout -b <expected>                         (если новая)
     — WORK_DIR = project_root
   Иначе (gitBranching: false, legacy):
     — Инвариант отключён. Пропустить шаг, коммит в текущую ветку.
     — Экспортировать ТОЛЬКО явный мод-флаг: export PDLC_GIT_BRANCHING=false
     — PDLC_EXPECTED_BRANCH и PDLC_WORK_DIR НЕ выставляются.

3. Assertion ВНУТРИ WORK_DIR (для worktree — критично!):
     current = run(f'cd "{WORK_DIR}" && git rev-parse --abbrev-ref HEAD').stdout.strip()
     assert current == expected, \
         f"OPS-001: cwd={WORK_DIR} current={current}, expected={expected}"
   Если assertion упал → STOP с диагностикой, НЕ продолжать к Шагу 2.

4. Экспортировать для всех последующих bash-вызовов (основной агент и субагент).
   Fail-closed модель: bash-guard всегда требует ЯВНЫЙ signal, никогда не
   "fall-through по умолчанию" (это ловит truncation/dropout в prompt для слабых моделей).

     Если gitBranching: true:
       export PDLC_GIT_BRANCHING="true"
       export PDLC_EXPECTED_BRANCH="<expected>"
       export PDLC_WORK_DIR="<WORK_DIR>"

     Если gitBranching: false:
       export PDLC_GIT_BRANCHING="false"
       (остальные НЕ выставляются)

   Guard-сниппет перед каждым commit/push/add читает PDLC_GIT_BRANCHING:
     — "true" → проверить CURRENT == EXPECTED, fail иначе
     — "false" → pass-through (инвариант отключён по дизайну)
     — unset/другое → ⛔ fail (bug: основной агент не экспортировал mode)
```

⛔ **Не использовать** `git.current_branch()` или `git rev-parse …` без явного
`cd "$WORK_DIR"` — в worktree mode корневой репо возвращает `main`, это даёт
ложный fail.

---

## 3. Branch naming (source of truth для `compute_expected_branch`)

**Для TASK от FEAT/BUG/DEBT/CHORE (стандартный режим):**
- Несколько TASK одного родителя → одна ветка
- `feat/FEAT-XXX-slug`, `fix/BUG-XXX-slug`, `debt/DEBT-XXX-slug`, `chore/CHORE-XXX-slug`

**Для TASK от PLAN (режим плана):**
- Каждая TASK = отдельная ветка
- `plan/PLAN-XXX-TASK-YYY-slug`

**Логика определения режима:**
1. Прочитай `parent` из TASK файла
2. Если parent начинается с `PLAN-` → режим плана (ветка per TASK)
3. Иначе → стандартный режим (ветка per parent)

---

## 4. Создание ветки: worktree vs checkout (детали реализации)

**Если `workspaceMode == "worktree"` И `gitBranching: true`:**

```python
def setup_worktree(project_root, branch_name):
    if workspace_mode != "worktree" or not git_branching:
        run(f"git checkout -b {branch_name}")  # fallback
        return project_root

    worktrees_root = f"{project_root}/.worktrees"
    dir_name = branch_name.replace("/", "__")
    worktree_path = os.path.join(worktrees_root, dir_name)

    # Проверить существующий worktree для ветки
    existing = parse_git_worktree_list()
    if branch_name in existing:
        return existing[branch_name]  # переиспользовать

    try:
        mkdir -p {worktrees_root}
        git worktree add {worktree_path} -b {branch_name}
    except:
        # Graceful fallback
        warn("git worktree add failed, falling back to git checkout -b")
        run(f"git checkout -b {branch_name}")
        return project_root

    # Копировать .state/ (КРОМЕ counters.json!)
    # ⚠️ ВАЖНО: каждую команду выполняй ОТДЕЛЬНЫМ Bash-вызовом!
    # НЕ объединяй в одну цепочку через && с переменными —
    # это ломает матчинг permissions в settings.json.
    mkdir -p {worktree_path}/.state
    cp .state/PROJECT_STATE.json {worktree_path}/.state/
    cp .state/knowledge.json {worktree_path}/.state/
    cp .state/session-log.md {worktree_path}/.state/ 2>/dev/null || true
    # ⚠️ counters.json НЕ копируется — глобальный ресурс

    # .claude/ уже в worktree через git (tracked directory) — НЕ нужен симлинк!

    # Симлинк dependency-каталогов (если есть) — чтобы инструменты были доступны из worktree
    # .venv — Python (ruff/pytest/mypy), node_modules — JS/TS, vendor — Go/PHP/Ruby
    for dep_dir in [".venv", "node_modules", "vendor"]:
        if os.path.isdir(f"{project_root}/{dep_dir}"):
            ln -s {project_root}/{dep_dir} {worktree_path}/{dep_dir}

    return worktree_path
```

**Шаги выполнения:**
1. Определи имя ветки по правилам §3
2. Нормализуй имя папки: `/` → `__` (например `feat/FEAT-001-auth` → `feat__FEAT-001-auth`)
3. Worktree path: `.worktrees/{dir_name}/` (внутри проекта, добавлена в `.gitignore`)
4. Проверь `git worktree list --porcelain` — если worktree для ветки уже существует, переиспользуй
5. Если ветка существует без worktree: `git worktree add {path} {branch}` (без `-b`)
6. Если ветка новая: `git worktree add {path} -b {branch}`
7. Скопируй `.state/` файлы (кроме `counters.json`!)
8. `.claude/` уже в worktree (tracked в git) — **НЕ создавай симлинк и НЕ копируй!**
9. Симлинк dependency-каталогов: для каждого из `.venv`, `node_modules`, `vendor` — если есть в project_root → `ln -s {project_root}/{dep_dir} {worktree_path}/{dep_dir}`
10. Все последующие операции выполняй в `{worktree_path}`
11. **[OPS-001 GUARD] Post-setup assertion** ВНУТРИ `{worktree_path}`:
    ```
    current = run(f'cd "{worktree_path}" && git rev-parse --abbrev-ref HEAD').stdout.strip()
    assert current == branch_name, \
        f"OPS-001: cwd={worktree_path} current={current}, expected={branch_name}"
    ```
    НЕ использовать `git.current_branch()` без явного `cd "{worktree_path}"` —
    в worktree mode корень репо возвращает `main`, это даст ложный fail.
12. Экспортировать для последующих bash-вызовов и для инжекции в prompt субагента:
    ```
    PDLC_GIT_BRANCHING=true
    PDLC_EXPECTED_BRANCH=<branch_name>
    PDLC_WORK_DIR=<worktree_path>
    ```

**Если `workspaceMode != "worktree"` или `gitBranching: false`:**
- `gitBranching: true, workspaceMode: inplace` → `git checkout -b {branch_name}`
  + post-setup assertion: `git rev-parse --abbrev-ref HEAD == branch_name`
  + export `PDLC_GIT_BRANCHING=true`, `PDLC_WORK_DIR=project_root`, `PDLC_EXPECTED_BRANCH=branch_name`
- `gitBranching: false` → инвариант отключён, но **ОБЯЗАТЕЛЬНО** export
  `PDLC_GIT_BRANCHING=false` (явный positive signal для guard'а).
  PDLC_EXPECTED_BRANCH и PDLC_WORK_DIR НЕ выставляются. Guard видит
  "false" → pass-through. Если флаг не выставлен вообще → guard fail-closed
  (защита от truncation/dropout в prompt).

---

## 5. Legacy mode (gitBranching: false)

Инвариант expected-branch **отключён**, но основной агент ОБЯЗАТЕЛЬНО
экспортирует явный positive signal:

```
export PDLC_GIT_BRANCHING="false"
```

`PDLC_EXPECTED_BRANCH` и `PDLC_WORK_DIR` не экспортируются. Pre-commit guard
видит `MODE=false` → pass-through с info-сообщением. Добавь в prompt субагента:
"Коммить прямо в текущую ветку; PDLC_GIT_BRANCHING=false экспортируй первой
bash-командой."

⛔ **Важно:** отсутствие `PDLC_GIT_BRANCHING` (вообще не выставлен) bash-guard
трактует как bug — fail-closed. Это защита от truncation/dropout в prompt
(OPS-001 amplification сценарий на слабых моделях). Только явный
`PDLC_GIT_BRANCHING=false` отключает guard.
