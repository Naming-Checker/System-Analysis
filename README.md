# System Analysis

Материалы по системному анализу: тексты и диаграммы. Актуальное состояние разделено на **текущую реализацию в репозитории** (MVP тестового стенда: similarity через sidecars) и **целевую продуктовую модель** (Stage1 + async Stage2, полный Trademark pipeline).

**Ответственный за результаты по системному анализу:** Илья Печерский.

## С чего начать

| Документ | Назначение |
|----------|------------|
| [docs/system_architecture.md](docs/system_architecture.md) | **Канон** архитектуры: сначала MVP, затем целевое видение |
| [docs/api_contracts.md](docs/api_contracts.md) | Публичные HTTP контракты backend + общие схемы webhook |
| [docs/domain_model.md](docs/domain_model.md) | MVP vs целевая DDD-модель |
| [docs/bounded_contexts.md](docs/bounded_contexts.md) | Контексты, включая **ML Similarity Search (реализовано)** |

Корневой [system_architecture.md](system_architecture.md) — короткий указатель на `docs/system_architecture.md`.

## Структура каталогов

| Путь | Содержимое |
|------|------------|
| `docs/` | Markdown: контекст проекта, bounded contexts, ubiquitous language, API, доменная модель |
| `diagrams/architecture/puml/` | C4 и компонентные диаграммы: **test stand + текущий REST** и **target product** (подписано в title) |
| `diagrams/sequences/` | Актуальные sequence для MVP: `sequence_logo_similarity_and_preview.puml`, `sequence_text_similarity_search.puml` |
| `diagrams/sequences/legacy/` | Продуктовые сценарии (регистрация, нарушение, logo comparison, полный async) — **не** отражают текущий публичный API; см. [legacy/README.md](diagrams/sequences/legacy/README.md) |

## Диаграммы

Файлы `.puml` открываются в редакторе с поддержкой PlantUML или через [PlantUML](https://plantuml.com/). При необходимости сгенерируйте PNG и положите в подкаталоги `diagrams/**/png/`.
