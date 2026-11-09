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

> **Примечание:** Пользовательские политики с проверкой моделью (промпты) применяются только к входящим запросам. Чёрные списки сообщений и имён файлов могут проверять оба направления, согласно настройкам.

---

## Тело запроса

### Обязательные поля

| Поле             | Тип             | Описание                                                         |
| ---------------- | --------------- | ---------------------------------------------------------------- |
| `application_id` | `string` (UUID) | Идентификатор приложения. Должен существовать в HiveTrace.       |

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

На этой странице описан HiveTrace **4.4.0**. Эндпоинты анализа возвращают результаты без верхнеуровневых полей `decision`, `message`, `block_reasons` и `user_block`. Для решения о допуске или блокировке используйте отдельные эндпоинты `/check`, описанные ниже.

Оба эндпоинта возвращают JSON-объект.

| Поле                     | Тип              | Описание                                                          |
| ------------------------ | ---------------- | ----------------------------------------------------------------- |
| `request_id`             | `string`         | UUID записи анализа.                                              |
| `schema_version`         | `string`         | Версия схемы ответа.                                              |
| `status`                 | `string`         | `success \| partial_success \| error \| accepted`                             |
| `errors`                 | `string[]`       | Список ошибок обработки (может быть пустым).                      |
| `warnings`               | `object[]`       | Список предупреждений (может быть пустым).                        |
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
| `dataclean.message.cleaned_text`|`string \| null`  | Санитизированная версия сообщения после обработки. При типе обработки `Детекция` текст остаётся без изменений. |
| `dataclean.message.types[]`| `array`   | Типы обнаруженных сущностей и их количество (может быть пустым).       |
| `dataclean.message.entities[]` | `array` | Типы обнаруженных сущностей, их позиция в исходном и обработанном тексте, а также их значение (может быть пустым).|
| `dataclean.file.flagged` | `boolean`         | Флаг, показывающий, были ли обнаружены чувствительные данные во вложенном файле. |
| `dataclean.file.save_failed`| `boolean`      | Флаг, показывающий, ошибку сохранения файла.                     |
| `dataclean.file.check_failed` | `boolean` | Флаг, показывающий, ошибку проверки файла.                          |
| `dataclean.file.files[]` | `array` | Список с результатами детализации по каждому файлу (может быть пустым).    |
| `language.flagged` | `boolean \| null`         | Флаг нарушения по несоответствию языка.                                |
| `language.enforced`| `boolean \| null`      | Флаг, показывающий, была ли установлена блокировка при обнаружении несоответствия языка. |
| `language.status` | `string` | `disabled \| checked \| error`                                                   |
| `language.error` | `string \| null` | Причина ошибки (может быть пустым).                                       |

Каждый элемент `warnings[]` содержит три строки: `code`, `message` и `message_ru`. Например, `session_history_full` сообщает об исчерпании словаря псевдонимов сессии.

`dataclean.message.cleaned_text` равен `null`, если обработанный текст отсутствует. В режиме детекции доступный текст не изменяется. При `language.status: "error"` поля `flagged` и `enforced` могут быть `null`; это не означает успешного прохождения языковой проверки. Перед использованием результата учитывайте `errors`, `warnings` и статусы модулей.

### Структура dataclean.message.types

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
    "message":"Напишите на alex@example.com.",
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
    "message":"Привет, Иван. Я обработал твой запрос",
    "additional_parameters":{
      "user_id":"user-123",
      "session_id":"session-456"
    }
  }'
```

---

## Пример ответа

Иллюстративный синхронный ответ на запрос выше при включённом только маскировании EMAIL. Число токенов приведено для примера и зависит от токенизации.

```json
{
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
      "cleaned_text": "Напишите на XXXXXXXXXXXXXXXX.",
      "types": [
        {
          "type": "EMAIL",
          "count": 1
        }
      ],
      "entities": [
        {
          "type": "EMAIL",
          "start": 12,
          "end": 28,
          "processed_start": 12,
          "processed_end": 28,
          "text": "alex@example.com"
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

## Асинхронная обработка

Для приложения в режиме `async` при включённой отложенной обработке (`ENABLE_ASYNC_INGEST=true`, значение по умолчанию) сервер возвращает **HTTP 202** и `status: "accepted"`. Сообщение сохраняется и ставится в очередь, а проверки выполняются позже. Ответ сохраняет ту же структуру с нейтральными значениями: например, нулём токенов и `flagged: false`. Это заглушки, а не результаты завершённых проверок.

По `request_id` можно найти запись в аналитике сессий. Исходный HTTP-ответ не дополняется результатом после завершения проверки. Если отложенная обработка отключена, классификация выполняется в рамках запроса даже для приложения `async`; это не включает синхронную блокировку.

## Проверка с решением о допуске

`POST /process_request/check` и `POST /process_response/check` принимают то же тело сообщения и API-токен, что и соответствующие базовые эндпоинты. Они используют настройки защиты приложения и возвращают отдельный компактный ответ:

| Поле | Тип | Значение |
|---|---|---|
| `decision` | `string` | `allow`, `deny` или `accepted` |
| `reason` | `string \| null` | Объяснение решения |
| `request_id` | `string` | UUID записи анализа |
| `errors` | `string[] \| null` | Ошибки обработки |
| `language` | `object \| null` | Результат проверки языка |

В синхронном режиме клиент должен сам обработать `decision`: вызов эндпоинта не останавливает обращение клиента к LLM автоматически. Учитывайте также `errors`: решение `allow` при ошибке модуля не означает, что все проверки завершились успешно. При отложенной обработке HTTP 202 с `decision: "accepted"` означает приём сообщения, а не разрешение его пропустить.
