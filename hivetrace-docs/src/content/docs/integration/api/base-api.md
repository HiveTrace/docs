---
title: "Base API"
sidebar:
  order: 3
---

## Базовые эндпоинты

### `POST /process_request/`

Используется для анализа **сообщений пользователя до их передачи в LLM** с применением правил валидации, настроенных в Web UI.

При включенных модулях HiveTrace выполняет:

- обнаружение, удаление, маскирование и псевдонимизацию чувствительных данных (dataclean)
- проверку встроенных политик (guardrails)
- проверку пользовательских политик
- проверку языка
- расчет потребления токенов
- связывание сессии и пользователя
- сохранение результатов мониторинга

---

### `POST /process_response/`

Используется для анализа **ответов LLM перед их возвратом конечному пользователю** с применением настроек валидации из Web UI.

При включенных модулях HiveTrace выполняет:

- обнаружение, удаление, маскирование и псевдонимизацию чувствительных данных (dataclean)
- проверку встроенных политик (guardrails)
- проверку языка
- расчет потребления токенов
- связывание сессии и пользователя
- сохранение результатов мониторинга

> **Примечание:** Пользовательские политики применяются **только** к `/process_request/`.

---

## Тело запроса

### Обязательные поля

| Поле             | Тип             | Описание                                                         |
| ---------------- | --------------- | ---------------------------------------------------------------- |
| `application_id` | `string` (UUID) | Идентификатор приложения. Должен существовать в HiveTrace.       |
| `message`        | `string`        | Сообщение пользователя или ответ LLM в зависимости от эндпоинта. |

---

### Опциональные поля

| Поле                    | Тип      | Описание                                           |
| ----------------------- | -------- | -------------------------------------------------- |
| `additional_parameters` | `object` | Метаданные для отслеживания сессии и пользователя. |

> **Примечание:** для псевдонимизации и обратной замены требуется `additional_parameters.session_id`.

---

## Структура additional_parameters

| Поле         | Тип      | Описание                            |
| ------------ | -------- | ----------------------------------- |
| `user_id`    | `string` | Внешний идентификатор пользователя. |
| `session_id` | `string` | Внешний идентификатор сессии.       |

---

## Схема ответа

Оба эндпоинта возвращают JSON-объект.

| Поле                     | Тип              | Описание                                                          |
| ------------------------ | ---------------- | ----------------------------------------------------------------- |
| `decision`               | `string`         | `allow \| deny \| accepted`                                       |
| `message`                | `string \| null` | Кастомная фраза при нарушении (может быть пустым).                |
| `block_reasons[]`        | `array`          | Список кодов и источников блокировки (может быть пустым).         |
| `user_block.blocked`     | `boolean`        | Флаг, показывающий, был ли пользователь заблокирован.             |
| `user_block.blocked_until`| `string \| null`| Момент времени, до которого действует блокировка пользователя (может быть пустым). |
| `request_id`             | `string`         | UUID записи анализа.                                              |
| `schema_version`         | `string`         | Версия схемы ответа.                                              |
| `status`                 | `string`         | `success \| partial_success \| error`                             |
| `errors`                 | `string[]`       | Список ошибок обработки (может быть пустым).                      |
| `warnings`               | `string[]`       | Список предупреждений (может быть пустым).                        |
| `tokens.count`           | `integer`        | Количество обработанных токенов.                                  |
| `tokens.usage_severity`  | `string \| null` | `low \| high \| critical \| null`                                 |
| `guardrails.flagged`     | `boolean`        | Флаг нарушения встроенных политик.                                |
| `guardrails.truncated`   | `boolean`        | Флаг, показывающий, была ли усечена обработка Guardrails          |
| `custom_policy.message.model.flagged`| `boolean`| Флаг нарушения кастомной политики.                            |
| `custom_policy.message.blacklist.flagged`| `boolean`| Флаг нарушения по чёрному списку сообщений.               |
| `custom_policy.file.blacklist.flagged`| `boolean`| Флаг нарушения по чёрному списку файлов.                     |
| `dataclean.flagged`      | `boolean`        | Флаг, показывающий, были ли обнаружены чувствительные данные в сообщении или во вложенном файле.                                                                                 |
| `dataclean.message.flagged`| `boolean`      | Флаг, показывающий, были ли обнаружены чувствительные данные в сообщении. |
| `dataclean.message.text_modified`|`boolean`| Флаг, показыващий, был ли текст изменён в ходе обработки.          |
| `dataclean.message.cleaned_text`|`string`  | Санитизированная версия сообщения после обработки. При типе обработки `Детекция` текст остаётся без изменений. |
| `dataclean.message.types[]`| `array`   | Типы обнаруженных сущностей и их количество (может быть пустым).       |
| `dataclean.message.entities[]` | `array` | Типы обнаруженных сущностей, их позиция в исходном и обработанном тексте, а также их значение (может быть пустым).|
| `dataclean.file.flagged` | `boolean`         | Флаг, показывающий, были ли обнаружены чувствительные данные во вложенном файле. |
| `dataclean.file.save_failed`| `boolean`      | Флаг, показывающий, был ли сохранён файл.                        |
| `dataclean.file.check_failed` | `boolean` | Флаг, показывающий, была ли выполнена проверка файла.               |
| `dataclean.file.files[]` | `array` | Список с результатами детализации по каждому файлу (может быть пустым).    |
| `language.flagged` | `boolean`         | Флаг нарушения по несоответствию языка.                                |
| `language.enforced`| `boolean`      | Флаг, показывающий, была ли установлена блокировка при обнаружении несоответствия языка. |
| `language.status` | `string` | `disabled \| checked \| error`                                                   |
| `language.error` | `string \| null` | Причина ошибки (может быть пустым).                                       |

### Структура dataclean.types

```json
{
	"type": "NAME",
	"count": 1
}
```

---

## Примеры запросов

### Process Request

```bash
curl -sS -X POST "$BASE_URL/process_request/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "application_id":"<uuid>",
    "message":"My name is John and my CVC is 222.",
    "additional_parameters":{
      "user_id":"user-123",
      "session_id":"session-456"
    }
  }'
```

---

### Process Response

```bash
curl -sS -X POST "$BASE_URL/process_response/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "application_id":"<uuid>",
    "message":"Hello John, I have processed your request.",
    "additional_parameters":{
      "user_id":"user-123",
      "session_id":"session-456"
    }
  }'
```

---

## Пример ответа

```json
{
    "decision": "allow",
    "message": null,
    "block_reasons": [],
    "user_block": {
        "blocked": false,
        "blocked_until": null
    },
    "request_id": "6c769c7a-f513-47e5-836a-5d59e0587c5b",
    "schema_version": "2.4.0",
    "status": "success",
    "errors": [],
    "warnings": [],
    "tokens": {
        "count": 19,
        "usage_severity": null
    },
    "guardrails": {
        "flagged": false,
        "truncated": false
    },
    "custom_policy": {
        "message": {
            "model": {
                "flagged": false
            },
            "blacklist": {
                "flagged": false
            }
        },
        "file": {
            "blacklist": {
                "flagged": false
            }
        }
    },
    "dataclean": {
        "flagged": true,
        "message": {
            "flagged": true,
            "text_modified": true,
            "cleaned_text": "Отпишитесь по моему балансу на XXXXXXXXXXXXXXXXXXXXXXXXXX",
            "types": [
                {
                    "type": "EMAIL",
                    "count": 1
                }
            ],
            "entities": [
                {
                    "type": "EMAIL",
                    "start": 31,
                    "end": 57,
                    "processed_start": 31,
                    "processed_end": 57,
                    "text": "Roxane_Stracke58@gmail.com"
                }
            ]
        },
        "file": {
            "flagged": false,
            "save_failed": false,
            "check_failed": false,
            "files": []
        }
    },
    "language": {
        "flagged": false,
        "enforced": false,
        "status": "disabled",
        "error": null
    }
}
```
