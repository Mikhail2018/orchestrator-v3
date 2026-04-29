# Improvement Prompt — `/pdlc:review-pr`

Шаблон промпта для improvement-субагента, который вызывается на IMPROVE-ветке
(score < 8). SKILL.md `### 4. Если IMPROVE (score < 8) — Improvement субагент`
ссылается сюда.

---

### 4. Если IMPROVE (score < 8) — Improvement субагент

```
Task tool:
  subagent_type: "general-purpose"
  description: "Fix PR #N based on review"
  prompt: [prompt ниже]
```

Prompt для improvement субагента:
```
Ты получил результаты независимого quality review PR.
Твоя задача — исправить найденные проблемы.

PR ВЕТКА: {branch name}
ЗАДАЧА: {TASK-ID}

РЕЗУЛЬТАТЫ РЕВЬЮ:
{полный ответ review}

ИНСТРУКЦИИ:
1. Прочитай файлы, указанные в замечаниях (Read tool)
2. Примени ТОЛЬКО рекомендации из ревью — не добавляй лишнего
3. Запусти тесты проекта — убедись что всё проходит
4. Сделай коммит: [{TASK-ID}] Address review feedback: {summary}
   <!-- # OPS-010: это коммит вида `improvement` (OPS-010 / issue #58).
   В этот же commit-staging бандли ЛЮБЫЕ отложенные правки frontmatter
   TASK.md (`status:`) и PROJECT_STATE.json task-bucket. НЕ делай
   отдельный status-only commit перед/после. НЕ пиши `lastUpdated`
   в PROJECT_STATE.json — поле всегда null. -->
5. Push изменения через verified-helper (OPS-028 — НЕ bare `git push`):
   `python3 {plugin_root}/scripts/pdlc_vcs.py git-push --branch <branch> --project-root "${PDLC_WORK_DIR:-.}"`
   Если exit=2 — НЕ заявляй success. Верни в ответе `remote_lines` и `reason`
   из JSON-вывода и пометь итерацию как failed (PM получит `waiting_pm`).

Верни список применённых исправлений.
```

