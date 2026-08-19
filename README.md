# poh-sysreq-agent

Автономный агент системного аналитика для **sysreq-стадии** FNR-пайплайна: превращает утверждённый концепт решения (`concept.md` + вердикт дебатов) и постановку проблемы (`task.md`) в документ системных требований `system_requirements.md`, где каждое утверждение подтверждено анализом кодовой базы.

## Зачем отдельный репозиторий

Ранее роль/шаблон/чек-лист sysreq-стадии жили внутри [`poh-issue-agents`](https://github.com/po-helper-org/poh-issue-agents) (`.claude/skills/system-analyst-sysreq`, `.claude/commands/fnr-system-requirements.md`) и отлаживались только через полный прогон Temporal-пайплайна issue-агента: клон целевого репозитория, `repomix`, пять последовательных стадий `claude -p` (`task → concept → debate → sysreq → validate`), 15 минут таймаута на стадию.

Этот репозиторий выделяет sysreq-агента в отдельный контур, чтобы:

- дорабатывать и оценивать качество именно этой стадии (потеря контекста, полнота As-Is/To-Be, соответствие шаблону) без поднятия остального пайплайна issue-агента;
- гонять `claude -p` с ролью sysreq локально, на своих фикстурах, быстро итерируя над промптом;
- версионировать изменения агента отдельно и осознанно подключать конкретную ревизию в `poh-issue-agents`, а не править всё «на живую» в одном репозитории.

См. issue [po-helper-org/poh-issue-agents#70](https://github.com/po-helper-org/poh-issue-agents/issues/70) — там описан план встраивания этого репозитория обратно в `poh-issue-agents` как внешнего модуля.

## Структура

```
.claude/
  skills/system-analyst-sysreq/
    SKILL.md                              — роль и принципы системного аналитика
    examples/ideal_system_requirements.md — эталонный шаблон документа (стиль Confluence, основной)
    examples/ideal_sysreq_document.md     — альтернативный эталонный шаблон (более компактный, Jira-декомпозиция)
    resources/sysreq_validation_checklist.md — чек-лист самопроверки + hard gates
  commands/
    fnr-system-requirements.md            — slash-команда `/fnr-system-requirements <path_to_concept>`
```

Это ровно то, что `worker/Dockerfile` в `poh-issue-agents` кладёт в `/root/.claude/` образа воркера перед запуском стадии `sysreq` (`_fnr_stages` / `run_fnr_stage` в `worker/activities.py`). Контракт стадии не меняется: вход — путь к `concept.md` (и лежащий рядом `task.md`), выход — `system_requirements.md` в той же папке.

## Как гонять локально

Агент — обычный Claude Code skill + slash-команда, поэтому для локальной итерации нужен только `claude-code` (`npm install -g @anthropic-ai/claude-code`) и доступ к Anthropic-совместимому API:

```bash
# из корня целевого репозитория (или фикстуры с sa_documentation/)
cp -r .claude ~/.claude   # либо запускать claude-code с --add-dir на этот репозиторий
claude -p "/fnr-system-requirements sa_documentation/FNR/FNR_1/concept.md"
```

Для предметного тестирования качества нужны фикстуры — пары `task.md` + `concept.md` (+ желательно `repomix-output.xml` целевого кода) и ожидаемый/эталонный `system_requirements.md` для сравнения. Эта часть контура (`fixtures/`, скрипт прогона регрессии, критерии оценки) — предмет дальнейшей работы в этом репозитории, README будет обновляться по мере её появления.

## Встраивание в poh-issue-agents

Механизм подключения (git submodule / subtree / sync-шаг в CI, тянущий зафиксированный тег или коммит) выбирается и документируется в рамках [issue #70](https://github.com/po-helper-org/poh-issue-agents/issues/70) в `poh-issue-agents`. Версия должна быть зафиксирована явно (тег/коммит), а не «плавающим» HEAD — изменение промпта системного аналитика равносильно изменению кода и должно проходить ревью перед попаданием в прод-пайплайн issue-агента.
