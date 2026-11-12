# 04 — Установка и эксплуатация

## Состав поставки

| Компонент | Расположение | Состояние |
|---|---|---|
| Бандл `@local/hierarchical-orchestrator` 1.0.0 | `default-workspace\hierarchical-orchestrator-bundle\` (исходник), установлен в профиль `desktop` | **установлен и применён** (`plugin_manager list_bundles`) |
| Глобальные правила | `%DSH_HOME%\AGENTS.md` | **создан**, подхватывается плагином agent-instructions во всех сессиях |
| Пресеты ролей | Loader-строки `preset-ho-*` (9 шт.) из бандла | **активны** |
| Host-лимиты | оверрайды строк `subagent` и `subagent-model-selection-settings` | **активны** |
| Корневые пресеты | оверрайды `preset-standard` и `preset-cordis` (персона оркестратора + ролевой enum) | **активны** |
| Шаблоны и документация | `harness-hierarchical-orchestrator\{docs,config,instructions,roles,tests}` | справочные (`*.example.yaml` НЕ устанавливаются автоматически) |

## Как пользоваться (v2)

1. **Сессия-оркестратор:** при создании сессии выберите пресет **`HO · Orchestrator`** (или Settings → General → Agent preset → сделайте его дефолтом для новых сессий). Только в HO-сессиях инструмент `subagent` имеет параметр `agent_preset`.
2. **Обычные сессии** (standard/cordis/ptc/minimal) не затронуты — бандл не переопределяет штатные пресеты.
3. **Явная роль:** в HO-сессии — «делегируй X роли tech-lead» → оркестратор передаст `agent_preset=ho-tech-lead`.
4. **Контроль ролей в UI:** пресеты HO · … в списке пресетов (orders 10–18).
5. **Настройка лимитов:** Plugins → Subagent (volatile `maxDepth`/`maxActiveSubagents`; правки пользователя перекрывают бандл).
6. **Модели детей:** смена политики — правка секции 2 патча бандла + новая корневая сессия.

## Идемпотентность и обновление

- Повторная установка: `plugin_manager install_bundle <каталог>` — тот же пакет требует `remove_bundle` → `install_bundle` (штатный цикл обновления, проверен в этой работе).
- Все строки объявлены только этим бандлом — удаление снимает всё разом.

## Отключение / откат

См. `docs/07-rollback.md`.
