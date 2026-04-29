# Git Branching Behavior — `/pdlc:continue`

Specifics поведения continue в worktree mode и inplace mode:
branch naming, worktree lifecycle (создание, симлинки, cleanup),
inplace fallback, статус в .md frontmatter после merge.

Связанный source-of-truth для `compute_expected_branch(TASK)` —
`skills/implement/references/branch-and-worktree-setup.md` (правила
именования общие для обоих скиллов).

---

## Git Branching поведение

Проверь `settings.gitBranching` и `settings.workspaceMode` в PROJECT_STATE.json.

При `gitBranching: true`:

### Branch naming

**Для TASK от FEAT/BUG/DEBT/CHORE (стандартный режим):**
- Несколько TASK одного родителя → одна ветка
- `feat/FEAT-XXX-slug`, `fix/BUG-XXX-slug`, `debt/DEBT-XXX-slug`, `chore/CHORE-XXX-slug`

**Для TASK от PLAN (режим плана):**
- Каждая TASK = отдельная ветка
- `plan/PLAN-XXX-TASK-YYY-slug`
- После выполнения **каждой** TASK — сразу PR

**Определение режима:** проверь `parent` в TASK файле:
- `parent: PLAN-XXX` → режим плана
- `parent: FEAT-XXX` / `BUG-XXX` / etc. → стандартный режим

### Worktree mode (workspaceMode: "worktree")

Каждая задача получает изолированный git worktree:

```
1. branch = feat/FEAT-001-slug
2. dir = feat__FEAT-001-slug (нормализация / → __)
3. path = .worktrees/feat__FEAT-001-slug/
4. git worktree add {path} -b {branch}
5. Копировать .state/ (кроме counters.json!)
6. Симлинк .claude/ (fallback: cp -r)
7. Все операции — в {worktree_path}
```

**Пропуск занятых задач:** перед выбором следующей задачи проверь `git worktree list --porcelain` — если для TASK уже есть активный worktree (сопоставление по имени ветки → TASK-ID), пропусти задачу.

**Graceful fallback:** если `git worktree add` не проходит → откат на `git checkout -b` с предупреждением.

### Inplace mode (workspaceMode: "inplace" или отсутствует)

- `git checkout -b {branch_name}` (прежнее поведение)

### Статус в .md frontmatter

**⚠️ При КАЖДОМ изменении статуса TASK — обновляй ОБА источника:**
- `.state/PROJECT_STATE.json` (локальная копия в worktree)
- TASK `.md` файл frontmatter (committed, source of truth для `/pdlc:sync`)

```
code_complete → Edit task .md: status: in_progress + Update PROJECT_STATE
create PR     → Edit task .md: status: review + Update PROJECT_STATE
merge         → Edit task .md: status: done + Update PROJECT_STATE
```

### Worktree lifecycle

```
create worktree → full cycle (implement → test → PR → review → merge)
  → вывести cleanup инструкцию (worktree НЕ удаляется автоматически)
  → перейти к следующей задаче (новый worktree)
```

**Итоговое сообщение для каждой задачи:**
```
Worktree: {worktree_path} (сохранён для правок по ревью)
Cleanup: git worktree remove {worktree_path} --force && git worktree prune
Сверка: /pdlc:sync --apply (из основного репо после merge всех PR)
```

При `gitBranching: false`:
- Коммиты идут в текущую ветку (main)
- Без feature branches

