# API и контракты интеграции

Документ фиксирует прикладные HTTP-контракты для backend API сервиса проверки неймингов.
Он дополняет архитектурные материалы и описывает payload-модели, статусы обработки и ошибки
в терминах интеграции.

## Общий поток

```mermaid
flowchart TD
  Client[Client] -->|POST Stage1| Stage1Api[Stage1Api]
  Stage1Api -->|200 OK + internalResults| Client
  Stage1Api -->|enqueue Stage2| AsyncStage2[AsyncStage2]
  AsyncStage2 -->|webhook partialOrFinal| Stage2Webhook[Stage2Webhook]
  Stage2Webhook -->|202 Accepted| AsyncStage2
```

## Эндпоинты

| Method | Path | Назначение |
| --- | --- | --- |
| `POST` | `/api/v1/registration-check` | Синхронный Stage 1 для проверки возможности регистрации |
| `POST` | `/api/v1/text-infringement` | Синхронный Stage 1 для попарной текстовой проверки нарушения |
| `POST` | `/api/v1/logo-comparison` | Синхронный Stage 1 для сравнения логотипов по placeholder-контракту |
| `POST` | `/api/v1/webhooks/stage2-results` | Приём частичных или финальных результатов Stage 2 |

## Общие статусы и каналы

### `ProcessingStatus`

| Значение | Смысл |
| --- | --- |
| `accepted` | запрос или batch принят, дальнейшая обработка продолжается |
| `completed` | синхронный Stage 1 завершён и внутренний результат готов |
| `partial` | Stage 2 прислал только часть внешних результатов |
| `failed` | обработка завершилась ошибкой и требует анализа |

### `DeliveryChannel`

| Значение | Смысл |
| --- | --- |
| `webhook` | доставка Stage 2 выполняется только через webhook |

## Общая модель ошибки

Все бизнес-ответы об ошибках выравниваются по единому envelope:

```json
{
  "status": "error",
  "error": {
    "code": "invalid_input",
    "message": "At least one Nice class is required.",
    "context": {
      "field": "mktu_codes",
      "details": {
        "reason": "empty_list"
      }
    }
  }
}
```

### `ErrorCode`

| Значение | Когда использовать |
| --- | --- |
| `invalid_input` | бизнес-валидация payload не пройдена |
| `validation_error` | схема запроса не прошла API-валидацию |
| `conflict` | конфликт текущего состояния обработки или дедупликации |
| `internal_error` | непредвиденный сбой backend |

### HTTP-статусы ошибок

| Status | Значение |
| --- | --- |
| `400` | payload синтаксически корректен, но нарушает прикладные правила |
| `409` | конфликт по состоянию обработки |
| `422` | schema validation error на уровне API |
| `500` | внутренняя ошибка сервиса |

## `POST /api/v1/registration-check`

Возвращает внутренний результат Stage 1 синхронно и сообщает, что внешний Stage 2 будет
доставлен отдельно в webhook-контур.

### Request

```json
{
  "naming": "PROBIMAX",
  "mktu_codes": [5, 25]
}
```

### Response `200 OK`

```json
{
  "request_id": "reg-probimax-001",
  "flow": "registration_check",
  "status": "completed",
  "naming": "PROBIMAX",
  "mktu_codes": [5, 25],
  "internal_results": [
    {
      "candidate_id": "tm-001",
      "candidate_name": "PROBI MAX",
      "source": "trademark_db",
      "mktu_codes": [5, 25],
      "similarity": 91.4,
      "summary": "High phonetic overlap in the same Nice classes.",
      "similarity_breakdown": {
        "semantic": 84.0,
        "phonetic": 96.0,
        "graphic": 88.0,
        "legal": 90.0,
        "visual": null
      }
    }
  ],
  "stage2": {
    "status": "accepted",
    "delivery": "webhook",
    "correlation_id": "reg-probimax-001",
    "partial_results_allowed": true,
    "webhook_path": "/api/v1/webhooks/stage2-results"
  },
  "meta": {
    "internal_result_count": 1,
    "result_limit": 200,
    "stage2_enabled": true
  }
}
```

## `POST /api/v1/text-infringement`

Сравнивает защищаемый нейминг и спорное обозначение, возвращает pairwise similarity и
внутренний shortlist.

### Request

```json
{
  "protected_naming": "PROBIMAX",
  "suspicious_naming": "PROBI MAX",
  "mktu_codes": [5]
}
```

### Response `200 OK`

```json
{
  "request_id": "txt-probi-max-001",
  "flow": "text_infringement",
  "status": "completed",
  "protected_naming": "PROBIMAX",
  "suspicious_naming": "PROBI MAX",
  "mktu_codes": [5],
  "pair_similarity": 94.2,
  "internal_results": [
    {
      "candidate_id": "tm-002",
      "candidate_name": "PROBI MAX",
      "source": "trademark_db",
      "mktu_codes": [5],
      "similarity": 94.2,
      "summary": "Pairwise comparison found the same dominant verbal core.",
      "similarity_breakdown": {
        "semantic": 82.0,
        "phonetic": 98.0,
        "graphic": 93.0,
        "legal": 95.0,
        "visual": null
      }
    }
  ],
  "stage2": {
    "status": "accepted",
    "delivery": "webhook",
    "correlation_id": "txt-probi-max-001",
    "partial_results_allowed": true,
    "webhook_path": "/api/v1/webhooks/stage2-results"
  },
  "meta": {
    "internal_result_count": 1,
    "result_limit": 200,
    "stage2_enabled": true
  }
}
```

## `POST /api/v1/logo-comparison`

Текущий контракт фиксирует только placeholder-структуру для передачи логотипов. Окончательный
формат бинарных данных будет уточнён отдельно.

### Request

```json
{
  "reference_logo": {
    "asset_ref": "logo://protected/probimax-main",
    "media_type": "image/png",
    "filename": "probimax.png"
  },
  "suspicious_logo": {
    "asset_ref": "logo://suspicious/probi-market",
    "media_type": "image/png",
    "filename": "probi-market.png"
  },
  "mktu_codes": [35],
  "notes": "Placeholder contract until final binary transport is approved."
}
```

### Response `200 OK`

```json
{
  "request_id": "logo-probimax-main-001",
  "flow": "logo_comparison",
  "status": "completed",
  "reference_logo": {
    "asset_ref": "logo://protected/probimax-main",
    "media_type": "image/png",
    "filename": "probimax.png"
  },
  "suspicious_logo": {
    "asset_ref": "logo://suspicious/probi-market",
    "media_type": "image/png",
    "filename": "probi-market.png"
  },
  "mktu_codes": [35],
  "internal_results": [
    {
      "candidate_id": "logo-001",
      "candidate_name": "Internal similar visual mark",
      "source": "trademark_db",
      "mktu_codes": [35],
      "similarity": 88.6,
      "summary": "Similar visual silhouette and retained text element.",
      "similarity_breakdown": {
        "semantic": null,
        "phonetic": null,
        "graphic": null,
        "legal": 85.0,
        "visual": 88.6
      }
    }
  ],
  "comparison_summary": "Placeholder Stage 1 response with internal logo matches.",
  "stage2": {
    "status": "accepted",
    "delivery": "webhook",
    "correlation_id": "logo-probimax-main-001",
    "partial_results_allowed": true,
    "webhook_path": "/api/v1/webhooks/stage2-results"
  },
  "meta": {
    "internal_result_count": 1,
    "result_limit": 200,
    "stage2_enabled": true
  }
}
```

## `POST /api/v1/webhooks/stage2-results`

Принимает частичные или финальные batch-обновления асинхронного Stage 2. Контракт допускает
unordered delivery и несколько batch-ов по одному `correlation_id`.

### Request

```json
{
  "correlation_id": "reg-probimax-001",
  "flow": "registration_check",
  "status": "partial",
  "naming": "PROBIMAX",
  "mktu_codes": [5, 25],
  "partial": true,
  "matches": [
    {
      "candidate_id": "ext-001",
      "candidate_name": "Probimax Mobile",
      "source": "google_play",
      "mktu_codes": [9],
      "similarity": 81.0,
      "summary": "Found in external store search.",
      "similarity_breakdown": {
        "semantic": 77.0,
        "phonetic": 86.0,
        "graphic": null,
        "legal": null,
        "visual": null
      }
    }
  ],
  "source_batch": ["google_play"]
}
```

### Response `202 Accepted`

```json
{
  "status": "accepted",
  "delivery": "webhook",
  "processing_status": "accepted",
  "correlation_id": "reg-probimax-001",
  "partial": true,
  "use_case": "WebhookCallbackProcessingUseCase"
}
```

## Поля результата

### `MatchCandidate`

| Поле | Тип | Смысл |
| --- | --- | --- |
| `candidate_id` | `string` | стабильный идентификатор кандидата |
| `candidate_name` | `string` | отображаемое имя найденного обозначения |
| `source` | `string` | источник кандидата, например `trademark_db` или внешний каталог |
| `mktu_codes` | `integer[]` | классы МКТУ, связанные с кандидатом |
| `similarity` | `number` | итоговый score `0..100` |
| `summary` | `string?` | короткое объяснение совпадения |
| `similarity_breakdown` | `object?` | детализация score по типам сходства |

## Замечания по совместимости

- Stage 1 контракты описывают синхронный внутренний результат, а не финальный merged output.
- Stage 2 доставляется только через webhook и может быть частичным.
- Для `logo comparison` `asset_ref` считается временным placeholder-полем до утверждения
  окончательного file transport.
- Доменные объекты backend описаны отдельно и маппятся в контракт через адаптерный слой:
  [domain_model.md](domain_model.md).
