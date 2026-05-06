> **Примечание**: Этот документ создан для навигации; полное содержание вынесено в `docs/`.

# Архитектура системы — указатель

**Каноническое описание:** [system_analysis/docs/system_architecture.md](docs/system_architecture.md)

Там разделены:

- **Текущая реализация в репозитории** — MVP тестового стенда: FastAPI backend, прокси в `visual-model-service` / `text-model-service`, preview по `logo_path`, webhook Stage 2.
- **Целевая архитектура продукта** — полная постановка (Stage1 orchestration, БД, async Stage 2 и т.д.) без смешения с тем, что сейчас в публичном API.

Прикладные HTTP-контракты: [system_analysis/docs/api_contracts.md](docs/api_contracts.md).
