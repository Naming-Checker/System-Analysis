> **Примечание**: Этот документ был создан с помощью ИИ (AI-generated).  
> При внесении изменений вручную, пожалуйста, сохраняйте эту пометку или обновляйте её соответствующим образом.

# Bounded Contexts

Документ фиксирует предварительное разбиение системы на bounded contexts в терминах DDD.

---

## 1. Naming Registration Check Context

**Назначение**: Проверка нового текстового нейминга на возможность регистрации в РФ.

**Внутри контекста**:
- входной нейминг;
- класс МКТУ;
- поиск кандидатов;
- вычисление фонетического, семантического, графического и текстового сходства;
- формирование `top-200` конфликтующих результатов;
- расчет итогового процента сходства.

**Основные термины**:
- `Submitted Naming`
- `Nice Classification Class`
- `Conflict List`
- `Similarity Percentage`
- `Confusing Similarity`

---

## 2. Trademark Infringement Check Context

**Назначение**: Проверка потенциального неправомерного использования товарного знака.

**Внутри контекста**:
- сравнение одного текстового обозначения с другим;
- оценка процента сходства;
- подготовка результата для юриста как сигнала о возможном нарушении.

**Основные термины**:
- `Infringement Check`
- `Designation Comparison`
- `Similarity Percentage`
- `Conflict Risk`

**Примечание**: Этот контекст близок к `Naming Registration Check Context`, но у него другой бизнес-сценарий и другой формат результата.

---

## 3. Logo Comparison Context

**Назначение**: Сравнение логотипов и комбинированных обозначений.

**Внутри контекста**:
- сравнение визуальных объектов;
- оценка графического сходства;
- возможный отдельный сервис с учетом требований по времени отклика.

**Основные термины**:
- `Logo`
- `Graphic Similarity`
- `Logo Comparison`

**Примечание**: На текущем этапе это расширение, но в архитектуре его стоит учитывать как отдельный bounded context.

---

## 4. Naming Preprocessing Context

**Назначение**: Очистка и нормализация сырых данных базы перед индексированием и поиском.

**Внутри контекста**:
- извлечение основного нейминга из длинных текстовых описаний;
- удаление чисел, общепринятых аббревиатур и родовых обозначений;
- использование `mark_disclaimer`;
- получение канонической формы обозначения.

**Основные термины**:
- `Naming Preprocessing`
- `Preprocessed Naming`
- `Non-protectable Element`
- `Distinctiveness Assessment`

---

## 5. Similarity Engine Context

**Назначение**: Вычисление всех видов сходства и итоговой similarity score.

**Внутри контекста**:
- фонетические метрики;
- семантические метрики;
- лингвистические метрики;
- графические метрики;
- агрегирование результатов в итоговый процент сходства.

**Основные термины**:
- `Phonetic Similarity`
- `Semantic Similarity`
- `Graphic Similarity`
- `Linguistic Metric`
- `Composite Metric`
- `Similarity Percentage`

---

## 6. Source Aggregation Context

**Назначение**: Сбор, объединение и унификация кандидатов из нескольких каналов данных.

**Внутри контекста**:
- внутренняя база товарных знаков;
- маркетплейсы мобильных приложений;
- интернет-источники и открытые реестры;
- merge + rerank результатов.

**Основные термины**:
- `Search Channel`
- `Source Aggregation`
- `Search Result`
- `Relevance`

---

## 7. Model and Index Management Context

**Назначение**: Управление локальными моделями, эмбеддингами и индексами.

**Внутри контекста**:
- локально встроенная модель эмбеддингов;
- предвычисленные эмбеддинги;
- векторные, фонетические и лингвистические индексы;
- офлайн-развертывание внутри сети МТС.

**Основные термины**:
- `Embedding`
- `Embedding Model`
- `Embedding Dataset`
- `Indexing`
- `Offline Deployment`

---

## 8. Предварительная карта контекстов

```text
Naming Preprocessing Context
    -> подготавливает данные для
Similarity Engine Context
    -> используется в
Naming Registration Check Context
    -> и в
Trademark Infringement Check Context

Source Aggregation Context
    -> поставляет кандидатов в
Naming Registration Check Context

Model and Index Management Context
    -> поддерживает
Similarity Engine Context

Logo Comparison Context
    -> может быть выделен в отдельный сервис
```

---

## 9. Предварительные выводы

- Наиболее естественный core domain сейчас: `Naming Registration Check Context` + `Similarity Engine Context`.
- `Naming Preprocessing Context` выглядит как критически важный supporting context.
- `Source Aggregation Context` и `Model and Index Management Context` можно рассматривать как infrastructural/supporting contexts.
- `Logo Comparison Context` лучше сразу проектировать как отдельно выделяемый контур, даже если он появится позже.

---

## 10. Связь с доменной моделью

Для реализации в backend используется единая DDD-модель с общими объектами:

- aggregate root `CheckRequest`;
- результат `ConflictResultSet`;
- async-единица `Stage2Job`;
- value objects: `NamingText`, `MktuClassSet`, `LogoAssetRef`, `SimilarityScore`.

Подробная спецификация: [domain_model.md](domain_model.md).


