# agentic-pipeline

Конфигурация для агентной разработки на Java: правила, скиллы и субагенты.
## Субагенты

| Имя | Путь | Назначение |
|-----|------|------------|
| **bn-java-developer** | `.cursor/agents/bn-java-developer.md` | Написание, рефакторинг и ревью Java-кода и конфигов (`.java`, `.xml`, `.yml`, `.properties`). Основной агент **не** пишет Java сам — делегирует этому субагенту. |

## Скиллы

| Имя | Путь | Когда вызывать |
|-----|------|----------------|
| **bn-maven** | `.cursor/skills/bn-maven/` | Сборка, тесты, `pom.xml`, дерево зависимостей, диагностика Maven (`/bn-maven`). |
| **bn-java-tdd** | `.cursor/skills/bn-java-tdd/` | TDD, unit/integration/e2e-тесты, цикл красный–зелёный–рефакторинг (`/bn-java-tdd`). |
| **bn-git-push-changes** | `.cursor/skills/bn-git-push-changes/` | Коммит и push по явной просьбе пользователя, безопасная работа с git (`/bn-git-push-changes`). |
| **karpathy-guidelines** | `.cursor/skills/karpathy-guidelines/` | Написание и ревью кода: простота, явные допущения, минимальный объём изменений. |

## Правила

| Файл | Назначение |
|------|------------|
| `.cursor/rules/Agent.mdc` | Глобальные ограничения: делегирование Java субагенту `bn-java-developer`, запрет правки `.gitignore`. |

## Структура

```
.cursor/
  agents/     # субагенты
  skills/     # скиллы (SKILL.md в каждой папке)
  rules/      # правила Cursor
tests/        # пример Maven/Spring Boot проекта
```
