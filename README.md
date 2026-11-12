# dsh-hierarchical-orchestrator

**Иерархический мультиагентный оркестратор для DeepSeek Harness (`dsh` 0.2.2-rc.1).**
Two-level delegation (Orchestrator → Lead → Worker) built **only on native Harness mechanisms**: Subagent runtime, Agent Presets, AGENTS.md, and the built-in model/provider subsystem. No external model router, no proxy, no custom LLM service.

English summary below — [English](#english).

## Что это

Бандл-плагин профиля (`@local/hierarchical-orchestrator`), который добавляет:

| Компонент | Механизм |
|---|---|
| Лимиты делегирования: `maxDepth=2`, пул `maxActiveSubagents=6` (на всё дерево) | host-строка `subagent`; переполнение → `ACTIVATION_LIMIT_REACHED`, уровень 3 отклоняется runtime |
| Политика моделей детей: только нативно подключённые маршруты (по умолчанию `zai/glm-5.3-flash` + `zai/glm-5.3`) | host-строка `subagent-model-selection-settings`; фиксируется в новой сессии, fail-loud без скрытого fallback |
| 9 ролевых пресетов `HO · …` | аддитивные Agent Presets; выбор роли — штатный параметр `agent_preset` инструмента `subagent` |
| Глобальные правила команды | файл `%DSH_HOME%\AGENTS.md` (опционально, ставится вручную) |

**Важно:** бандл **не переопределяет ни один штатный пресет**. Существующие сессии не затрагиваются; сломанная декларация в худшем случае даёт диагностику в ростере. Оркестратор включается выбором пресета `HO · Orchestrator` для сессии.

## Роли

| Пресет | Глубина | Делегирует | Запись |
|---|---|---|---|
| `HO · Orchestrator` | 0 | лидам + воркерам (enum из 8 ролей) | по области |
| `HO · Tech Lead` | 1 | только explorer/backend/frontend/test | по области |
| `HO · Quality Lead` | 1 | только explorer/test/reviewer/auditor | по области |
| `HO · Explorer` | 2 | нет (инструмента нет) | **read-only по составу** |
| `HO · Backend Developer` | 2 | нет | выделенные пути (advisory) |
| `HO · Frontend Developer` | 2 | нет | выделенные пути (advisory) |
| `HO · Test Engineer` | 2 | нет | тесты/фикстуры |
| `HO · Code Reviewer` | 2 | нет | **read-only по составу** |
| `HO · Security Auditor` | 2 | нет | **read-only по составу** |

Read-only = в композиции пресета нет `write`/`edit`/`apply_patch` и shell-инструментов: писать физически нечем (это ограничение каталога инструментов, не ОС-sandbox).

## Установка

Требуется DeepSeek Harness Desktop/Web **0.2.2-rc.1** (проверено именно на этой версии) и подключённый провайдер моделей с поддержкой tool calling.

1. Склонируйте репозиторий.
2. В сессии Harness с инструментом `plugin_manager` (или через страницу Plugins):
   ```
   plugin_manager: install_bundle, target = <absolute path>\bundle
   ```
3. Перезагрузите GUI (Ctrl+R). Пресеты `HO · …` появятся в списке пресетов.
4. Создайте сессию с пресетом **`HO · Orchestrator`** (или сделайте его дефолтным в Settings → General).
5. (Опционально) скопируйте `instructions/AGENTS.global.md` в `%DSH_HOME%\AGENTS.md` — глобальные правила команды для всех агентов.

Проверка: `plugin_manager list_plugins` → строки `preset-ho-*` с `fiberPhase: active`.

## Использование

- В HO-сессии формулируйте задачу — оркестратор сам решит: сделать inline или делегировать (`agent_preset` выбирает роль).
- Явно: «Делегируй аудит безопасности в X» → `agent_preset=ho-security-auditor`.
- Лимиты: Plugins → Subagent (volatile; правки пользователя перекрывают бандл).

## Технически обеспечено vs advisory

**Enforced (конфигурацией/runtime):** глубина 2; пул 6; воркеры без делегирующего инструмента; read-only составы reviewer/auditor/explorer; маршруты детей — только из политики нативных моделей.

**Advisory (инструкции; проверяют лиды и review):** владение путями записи (один писатель на путь), лимит ретраев ≤2 → `blocked`, review до приёмки, бюджет токенов (`NOT_ENFORCED`).

## Откат

```
plugin_manager: remove_bundle, target = @local/hierarchical-orchestrator
```
плюс удалить `%DSH_HOME%\AGENTS.md`, если добавляли. Подробности — `docs/07-rollback.md`.

## Документация

- `docs/03-architecture.md` — архитектура и механизмы
- `docs/04-installation.md` — эксплуатация
- `docs/06-security-and-limitations.md` — границы гарантий и разбор инцидента дизайна v1 (почему бандл аддитивный)
- `docs/07-rollback.md` — откат
- `config/*.example.yaml` — шаблоны политик (не устанавливаются автоматически)
- `roles/*.md` — карточки ролей
- `tests/` — сценарии дымовых и приёмочных проверок

## Совместимость и ограничения

- Проверено на Desktop `0.2.2-rc.1` (win32 x64); структура строк/пакетов соответствует именно этой версии.
- Политика маршрутов детей применяется к **новым** корневым сессиям (существующие сохраняют зафиксированную).
- volatile-настройки `subagent` пользователь может переопределить через UI — это штатное поведение.

---

<a id="english"></a>
## English

Hierarchical multi-agent orchestration for DeepSeek Harness 0.2.2-rc.1: **Orchestrator → Leads → Workers** with native-only model routing and hard runtime caps (delegation depth 2, shared pool of 6 active subagents). Ships as an **additive profile bundle** — it never overrides shipped presets, so existing sessions cannot be affected. Nine role presets (`HO · Orchestrator`, `HO · Tech Lead`, `HO · Quality Lead`, `HO · Explorer`, `HO · Backend/Frontend Developer`, `HO · Test Engineer`, `HO · Code Reviewer`, `HO · Security Auditor`); depth-2 worker presets carry no delegation tool, and reviewer/auditor/explorer compositions are read-only (no write/edit/shell tools at all).

Install: clone, then in a Harness session run `plugin_manager install_bundle` with `target` = the absolute path of the `bundle/` directory. Create a session on the `HO · Orchestrator` preset. See the Russian sections above for the full guide; key docs are in `docs/`.

Verified on `dsh` 0.2.2-rc.1: nested delegation with real child ids, depth-3 runtime rejection (`subagent depth 3 exceeds maxDepth 2`), capacity rejection at the 7th concurrent child, read-only enforcement (`NO-WRITE-TOOL-CONFIRMED`), fail-loud model-route policy for new sessions.

## License

MIT — see [LICENSE](LICENSE).
