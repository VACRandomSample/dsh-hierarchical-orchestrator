# 03 — Архитектура Hierarchical Orchestrator (v2)

**Версия Harness:** 0.2.2-rc.1 · **Механизм:** только штатные подсистемы (Subagent runtime, Agent Presets, AGENTS.md). Собственный прокси/маршрутизатор моделей отсутствует; OmniRoute не задействован. **v2 — строго аддитивная композиция: ни один штатный пресет/строка не переопределяется, кроме двух host-строк конфигурации (`subagent` и `subagent-model-selection-settings`); существующие сессии затронуты быть не могут.**

```
Пользователь
   │  (сессия с пресетом HO · Orchestrator — depth 0)
   ▼
Уровень 0 · ORCHESTRATOR
   subagent (spawn, maxDepth 2, allowedAgentPresets = 8 ролей, agent_preset)
   │
   ├──────────────► Уровень 1 · TECH LEAD (preset-ho-tech-lead)
   │                   subagent: allowedAgentPresets=[explorer, backend, frontend, test]
   │                   ├─► Уровень 2 · EXPLORER (read-only: fs read, search, web)
   │                   ├─► Уровень 2 · BACKEND DEVELOPER (fs+shell, свои пути)
   │                   ├─► Уровень 2 · FRONTEND DEVELOPER (fs+shell, свои пути)
   │                   └─► Уровень 2 · TEST ENGINEER (fs+shell, тесты/фикстуры)
   │
   └──────────────► Уровень 1 · QUALITY LEAD (preset-ho-quality-lead)
                       subagent: allowedAgentPresets=[explorer, test, reviewer, auditor]
                       ├─► Уровень 2 · EXPLORER
                       ├─► Уровень 2 · TEST ENGINEER
                       ├─► Уровень 2 · CODE REVIEWER (read-only)
                       └─► Уровень 2 · SECURITY AUDITOR (read-only)
```

## 1. Как это работает (нативные механизмы)

| Механизм | Реализация |
|---|---|
| Управляемые активации | штатный `@deepseek-ai/dsh-subagent` + бэкенд `spawn` (in-process, continuable). Каждый вызов `subagent` возвращает child id; runtime доставляет уведомление о завершении родителю и освобождает ресурсы |
| Глубина 2 | host-настройка `subagent.maxDepth=2` (строка `subagent`) + `maxDepth: 2` в конфиге делегирующих инструментов. Проверка выполняется runtime при каждом старте: `Error: subagent depth 3 exceeds maxDepth 2` |
| Пул 6 | host-настройка `subagent.maxActiveSubagents=6` — общий пул на всё дерево (включая ожидающих лидов); переполнение → отказ запуска («active child limit: 6») |
| Роли | 9 Agent Presets (`preset-ho-*`) с персонами и наборами инструментов; выбор роли — штатный параметр `agent_preset` инструмента (enum из `allowedAgentPresets`); ребёнок без `agent_preset` наследует ревизию пресета родителя |
| Запрет делегирования воркерам | двойной: в пресетах уровня 2 делегирующий инструмент не смонтирован, плюс runtime-лимит глубины 2 |
| Read-only роли | `tool-fs {write:false, edit:false, applyPatch:false}` + отсутствие shell/jobs в составе пресета explorer/reviewer/auditor — пишущих инструментов в каталоге нет |
| Модели | политика `subagent-model-selection-settings {enabled, allowedModels: zai/*}` фиксируется в новой корневой сессии и наследуется; выбор маршрута вне политики → явная ошибка; `list_subagent_models` для обзора; preflight валидирует маршрут живым адаптером до создания ребёнка. Наследование маршрута родителя — дефолт. Fallback выключен |
| Межагентное общение | `send_message` (ребёнок → прямому родителю и наоборот), `interrupt_agent`, `list_agents` (`tool-subagent-control`) |
| Правила и контракты | глобальный `~/.dsh/AGENTS.md` (цепочка user-global инструкций) + персоны ролей |
| Зависшие задачи | runtime: `interrupt_agent` отменяет ход; dispose гасит поддерево; слот возвращается после cleanup; завершение доставляется уведомлением |

## 2. Политика решений по размеру задачи (в персонах)

Trivial → без субагентов · Small → ≤1 исполнитель · Medium → лид + 1–2 исполнителя · Large → 2 лида + до 4 исполнителей (лимит 6 общий).

## 3. Контракт делегирования (advisory, соблюдение проверяет лид)

Запрос: `task_id, role(agent_preset), objective, context, allowed_paths, forbidden_actions, expected_outputs, acceptance_criteria, retry_limit≤2`.
Ответ: `task_id, status, summary, changed_files, tests, risks, handoff`.
Нативный `outputSchema` spawn-бэкенда в 0.2.2-rc.1 существует на уровне API (`request.outputSchema`), но делегирующий инструмент его не экспортирует — поэтому структура текстовая и валидируется лидом. Это задокументировано как ограничение, не как API.

## 4. Иерархия владения записью

Один писатель на путь назначается лидом в `allowed_paths` (advisory). Технически: разные дети работают в параллельных активациях; редактирование защищено версией файла (`FS_STALE_VERSION` при конкурирующем изменении после чтения) — штатный механизм `edit` снижает риск тихой перезаписи. Полной блокировки путей на уровне fs в этой версии нет — отмечено в ограничениях.
