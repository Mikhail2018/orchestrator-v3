# Polisade Orchestrator — Пакет 3 рефакторинг

## Что сделано

Рефакторинг крупных `SKILL.md` файлов: вынос справочного материала
(шаблоны промптов, форматы вывода, протоколы, расширенные руководства)
в `references/<name>.md`. SKILL.md остаются как алгоритм + якоря
+ ссылки на references. Никаких изменений семантики, имён команд,
схем артефактов, контрактов VCS.

## Сводка изменений

| Скилл | Было | Стало | Δ | % |
|---|---:|---:|---:|---:|
| **implement** | 2 018 | 1 198 | -820 | -40.6% |
| **design** | 1 198 | 623 | -575 | -48.0% |
| **tasks** | 1 060 | 540 | -520 | -49.1% |
| **spec** | 886 | 363 | -523 | -59.0% |
| **roadmap** | 684 | 272 | -412 | -60.2% |
| **continue** | 660 | 406 | -254 | -38.5% |
| **prd** | 570 | 189 | -381 | -66.8% |
| **review-pr** | 523 | 292 | -231 | -44.2% |
| **Итого по 9 скиллам** | **9 597** | **3 881** | **-5 716** | **-59.6%** |
| **Итого по всем 24 скиллам** | **10 132** | **6 416** | **-3 716** | **-36.7%** |

Остальные 16 скиллов не трогали — они уже компактные (≤300 строк).

## Новые reference-файлы (всего 19)

```
skills/implement/references/
  branch-and-worktree-setup.md         (203 строки) — OPS-001 GUARD, branch naming, worktree setup
  output-formats.md                    (200) — все шаблоны вывода
  subagent-prompt-template.md          (493) — полный шаблон промпта субагента

skills/tasks/references/
  subagent-prompts.md                  (562) — три варианта (PLAN-item, SPEC/FEAT, BUG/DEBT/CHORE)

skills/review-pr/references/
  reviewer-prompt.md                   (158) — шаблон reviewer prompt + pr-comment publishing
  improvement-prompt.md                (46)  — improvement subagent
  output-formats.md                    (141) — варианты вывода

skills/continue/references/
  resume-procedure.md                  (73)  — Phase A/B/C/D OPS-008
  git-branching-behavior.md            (86)  — worktree/inplace lifecycle
  output-formats.md                    (163) — все варианты вывода

skills/spec/references/
  subagent-prompt.md                   (312) — полный шаблон спека-субагента
  quality-review-loop.md               (164) — review loop алгоритм
  output-formats.md                    (98)  — варианты вывода

skills/roadmap/references/
  subagent-prompt.md                   (182)
  quality-review-loop.md               (208)
  output-formats.md                    (67)

skills/design/references/
  subagent-prompt.md                   (292)
  quality-review-loop.md               (227)
  output-formats.md                    (103)

skills/prd/references/
  prd-structure-guide.md               (405) — 13 секций PRD + правила depth + примеры + чек-лист
```

## Сохранённые контрактные якоря в SKILL.md

Линтер `pdlc_lint_skills.py` требует определённых литералов **именно в SKILL.md**.
Все они сохранены:

- `implement/SKILL.md`:
  - `═══ OPS-010: КОНТРАКТ ВИДОВ КОММИТОВ ═══` heading ✓
  - Обе finalize-templates дословно (`Finalize status: {new-status} (PR #{N})` и без суффикса) ✓
  - `НЕ пиши lastUpdated` гард ✓
  - `pdlc_vcs.py pr-create` literal ✓
  - 6 inline `# OPS-010:` (требуется ≥3) ✓
  - `git add -f` упоминания внутри bullet'ов с ⛔ NEVER маркером ✓
- `continue/SKILL.md`:
  - `НЕ пиши lastUpdated` гард ✓
  - 6 inline `# OPS-010:` (требуется ≥2) ✓
- `review-pr/SKILL.md`:
  - `НЕ пиши lastUpdated` гард ✓
  - 3 inline `# OPS-010:` (требуется ≥2) ✓
  - **Self-reviewer table** (`**Режим \`self\`**` + `| Агент | Команда |`) — OPS-022 ✓
- `review/SKILL.md`:
  - Self-reviewer table — OPS-022 ✓ (не трогали)

## Проверки совместимости

Все прошли:

- `python3 scripts/pdlc_lint_skills.py` → **0 errors, 2 warnings** (те же baseline-warnings про deprecated `codex-review`/`codex-review-pr`)
- `python3 tools/convert.py . --out /tmp/qwen-test --overlay tools/qwen-overlay` → **успешно, 7 warnings** (все baseline)
- `python3 tools/validate.py /tmp/qwen-test` → **14 passed, 0 failed**
- Конвертер автоматически переписывает пути `skills/*/references/*` в `${PDLC_PLUGIN_ROOT}/assets/*/references/*` для Qwen — references прозрачно работают и в Qwen, и в GigaCode.

## Не сделано (вынесено в Пакеты 1, 2, 4)

Пакет 3 — только структурный рефакторинг. **Поведение не изменилось.**
Реальная экономия токенов начнётся в Пакете 1 (Sliced Context Builder),
где основной агент будет инжектировать в субагентов **только релевантные**
секции references вместо целых файлов SPEC/PRD/knowledge.json.

## Что в архиве

`polisade-orchestrator-pkg3.zip` — полная рабочая копия плагина после
рефакторинга. Можно распаковать и протестировать на реальном проекте
через `/plugin install pdlc --scope project`.

