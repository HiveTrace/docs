---
title: "Одна LLM"
sidebar:
  order: 2
---

Используйте Gateway для чат-приложения, которое обращается к модели через OpenAI-совместимый клиент. Пример использует контракт авторизации **Gateway 4.4**.

## Подготовка

- `GATEWAY_URL` — адрес Gateway с `/v1`.
- `HIVETRACE_API_KEY` — API-токен выбранного приложения HiveTrace.
- `MODEL_NAME` — настроенный алиас модели, разрешённый этому приложению.
- `LITELLM_API_KEY` — необязательный ключ для маршрута через LiteLLM. Для прямого маршрута, настроенного администратором Gateway, не задавайте его.
- `USER_ID` и `SESSION_ID` — внешние идентификаторы пользователя и диалога для аналитики. Сохраняйте один ID сессии на протяжении диалога.

Создайте приложение, настройте защиту и режим `sync`/`async`, разрешите модель и выпустите API-токен. Пустой список разрешённых моделей запрещает все вызовы. Приложение определяется токеном; заголовок с ID приложения не требуется.

## Пример Python

```python
import os
from openai import OpenAI

headers = {
    "X-User-Id": os.environ["USER_ID"],
    "X-Session-Id": os.environ["SESSION_ID"],
}
if os.getenv("LITELLM_API_KEY"):
    headers["X-LiteLLM-Api-Key"] = os.environ["LITELLM_API_KEY"]

with OpenAI(
    base_url=os.environ["GATEWAY_URL"].rstrip("/"),
    api_key=os.environ["HIVETRACE_API_KEY"],
    default_headers=headers,
) as client:
    response = client.chat.completions.create(
        model=os.environ["MODEL_NAME"],
        messages=[{"role": "user", "content": "Briefly explain HiveTrace."}],
    )
    print(response.choices[0].message.content)
```

## Пример curl

В примере используется условный алиас `chat-model`; замените его на настроенную модель. Для прямого маршрута уберите строку `X-LiteLLM-Api-Key`.

```bash
curl -sS "$GATEWAY_URL/chat/completions" \
  -H "Authorization: Bearer $HIVETRACE_API_KEY" \
  -H "X-LiteLLM-Api-Key: $LITELLM_API_KEY" \
  -H "X-User-Id: $USER_ID" \
  -H "X-Session-Id: $SESSION_ID" \
  -H "Content-Type: application/json" \
  -d '{"model":"chat-model","messages":[{"role":"user","content":"Hello from my app"}]}'
```

## Передача файла

Chat Completions и Responses принимают multipart-запрос с JSON-частью `request` и повторяемыми частями `attached_files`. Gateway добавляет файлы в запрос модели и извлекает копию для аудита. Провайдер и модель должны поддерживать выбранный формат файла.

```bash
curl -sS "$GATEWAY_URL/chat/completions" \
  -H "Authorization: Bearer $HIVETRACE_API_KEY" \
  -H "X-LiteLLM-Api-Key: $LITELLM_API_KEY" \
  -H "X-User-Id: $USER_ID" \
  -H "X-Session-Id: $SESSION_ID" \
  -F 'request={"model":"chat-model","messages":[{"role":"user","content":"Summarize the document."}]};type=application/json' \
  -F 'attached_files=@contract.pdf;type=application/pdf'
```

Также поддерживаются стандартные файловые части JSON. Для документов, предназначенных только для аудита, используйте `metadata.files`; этот канал не передаёт документ модели. Форматы файлов, Files API и лимиты описаны в [руководстве по интеграции](../integration-guide/).

## Проверка результата

Найдите сообщения в аналитике сессий по приложению, пользователю и сессии. В режиме `sync` защита может заблокировать запрос или заменить ответ. В режиме `async` отложенные проверки не блокируют ответ модели. HTTP 200 сам по себе не означает отсутствия нарушений. При запросе `stream: true` для синхронного приложения Gateway возвращает проверенный непотоковый JSON-ответ; учитывайте это в клиенте.
