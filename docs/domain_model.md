# Domain Model

Документ фиксирует DDD-first модель для трех Stage 1 сценариев:

- `registration_check`
- `text_infringement`
- `logo_comparison`

Модель ориентирована на единый жизненный цикл запроса (`CheckRequest`) и общий формат результата (`ConflictResultSet`) с поддержкой async Stage 2.

## 1. Aggregates и Entities

### 1.1 `CheckRequest` (Aggregate Root)

Отвечает за жизненный цикл проверки и связывает:

- входные данные сценария;
- статус Stage 1;
- статус Stage 2;
- `correlation_id` для webhook.

Поля:

- `request_id: str`
- `flow: FlowType` (`registration_check`, `text_infringement`, `logo_comparison`)
- `status: ProcessingStatus`
- `mktu_codes: MktuClassSet`
- `naming_payload: NamingPair | SingleNaming | LogoPair`
- `created_at`, `updated_at` (для backend state tracking)

Инварианты:

- `request_id` не пустой и стабилен после создания.
- `flow` определяет обязательный тип `naming_payload`.
- `status` может переходить только по разрешенным переходам.

### 1.2 `ConflictResultSet`

Результат Stage 1 или merged Stage 1/2.

Поля:

- `request_id: str`
- `candidates: list[MatchCandidate]`
- `result_limit: int` (по умолчанию 200)
- `partial: bool`

Инварианты:

- `len(candidates) <= result_limit`
- `candidates` отсортированы по `similarity.total` по убыванию.

### 1.3 `Stage2Job`

Представление async-задачи для внешнего обогащения.

Поля:

- `correlation_id: str`
- `dedup_key: str`
- `delivery: DeliveryChannel` (`webhook`)
- `partial_results_allowed: bool`

Инварианты:

- `dedup_key` строится по нормализованной схеме `naming + sorted(MKTU)`.
- `delivery` фиксирован как `webhook` в текущем контракте.

## 2. Value Objects

### 2.1 `NamingText`

- `raw: str`
- `canonical: str`

Правила:

- `canonical` = whitespace-normalized + casefold.
- пустые строки запрещены.

### 2.2 `MktuClassSet`

- уникальный, отсортированный список `int`.

Правила:

- допускаются только положительные целые;
- при нормализации удаляются дубликаты.

### 2.3 `LogoAssetRef`

- `asset_ref: str`
- `media_type: str | None`
- `filename: str | None`

Правила:

- `asset_ref` обязателен;
- доменная логика не привязана к конкретному transport protocol.

### 2.4 `SimilarityBreakdown` и `SimilarityScore`

- `SimilarityBreakdown`: `semantic`, `phonetic`, `graphic`, `legal`, `visual` (опционально).
- `SimilarityScore.total`: итоговый score `0..100`.

Правила:

- каждая метрика в диапазоне `0..100`;
- `total` обязателен для ранжирования.

### 2.5 `CandidateRef`

- `candidate_id`
- `candidate_name`
- `source`

Правила:

- все поля обязательны;
- `candidate_id` стабилен в пределах источника.

## 3. Flow-специализация

### 3.1 `registration_check`

Payload:

- `naming: NamingText`
- `mktu_codes: MktuClassSet`

Результат:

- `ConflictResultSet` с `top-N` совпадениями.

### 3.2 `text_infringement`

Payload:

- `protected_naming: NamingText`
- `suspicious_naming: NamingText`
- `mktu_codes: MktuClassSet`

Результат:

- `pair_similarity` для пары + shortlist кандидатов.

### 3.3 `logo_comparison`

Payload:

- `reference_logo: LogoAssetRef`
- `suspicious_logo: LogoAssetRef`
- `mktu_codes: MktuClassSet`

Результат:

- visual-oriented `ConflictResultSet`;
- `comparison_summary` как explanation.

## 4. Domain Policies

### 4.1 `DeduplicationPolicy`

Строит `Stage2Job.dedup_key`:

- naming нормализуется;
- MKTU сортируются и дедуплицируются;
- формат ключа: `canonical_naming|code1,code2`.

### 4.2 `RankingPolicy`

- сортировка кандидатов по `SimilarityScore.total` по убыванию;
- ограничение `result_limit`;
- стабильный tie-break по `candidate_id`.

### 4.3 `SimilarityPolicy`

- проверка диапазонов breakdown-метрик;
- расчет или валидация `total` для единообразного ранжирования.

### 4.4 `PreprocessingPolicy`

- нормализация whitespace;
- casefold;
- подготовка к dedup/ranking.

## 5. Разрешенные переходы статусов

```mermaid
stateDiagram-v2
  [*] --> accepted
  accepted --> completed
  accepted --> partial
  partial --> completed
  accepted --> failed
  partial --> failed
```

## 6. Mapping: Domain Model -> API Contract

- `CheckRequest.request_id` -> `request_id`
- `CheckRequest.flow` -> `flow`
- `CheckRequest.status` -> `status`
- `ConflictResultSet.candidates` -> `internal_results`
- `SimilarityScore.total` -> `MatchCandidate.similarity`
- `SimilarityBreakdown` -> `MatchCandidate.similarity_breakdown`
- `Stage2Job.correlation_id` -> `stage2.correlation_id`
- `Stage2Job.partial_results_allowed` -> `stage2.partial_results_allowed`

Для текущего API допускаются транспортные адаптеры (`DTO <-> Domain`), если доменная модель богаче контракта.
