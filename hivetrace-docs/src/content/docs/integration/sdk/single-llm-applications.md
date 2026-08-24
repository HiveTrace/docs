---
title: "Подключение приложений с одной LLM (без агентов)"
sidebar:
  order: 2
---

## Одна LLM

Эта интеграция подходит для **обычного LLM‑приложения без мультиагентных фреймворков**: вы отправляете пользовательский текст в модель и получаете ответ.

HiveTrace подключается перед моделью и решает две задачи:

- **Контроль входа (`input`)**: что пользователь отправляет в LLM
- **Контроль выхода (`output`)**: что LLM возвращает пользователю

---

## Предварительные требования

Вам понадобятся:

1. **URL инстанса** HiveTrace
2. **API‑токен** (создаётся в UI)
3. **`application_id`** вашего приложения (UUID из UI)

---

## Установка

```bash
pip install hivetrace[base]
```

---

## Конфигурация

SDK читает конфигурацию из переменных окружения:

- `HIVETRACE_URL` — базовый URL инстанса (только `http://` или `https://`)
- `HIVETRACE_ACCESS_TOKEN` — API‑токен
- `HIVETRACE_APP_ID` — опционально (удобный alias для вашего `application_id`)

Пример `.env`:

```bash
HIVETRACE_URL=https://your-hivetrace-instance.com
HIVETRACE_ACCESS_TOKEN=your-access-token
HIVETRACE_APP_ID=your-application-id
```

Альтернатива — передать конфиг явно:

```python
from hivetrace import SyncHivetraceSDK

client = SyncHivetraceSDK(
    config={
        "HIVETRACE_URL": "https://your-hivetrace-instance.com",
        "HIVETRACE_ACCESS_TOKEN": "your-access-token",
    }
)
```

---

## Быстрая интеграция (рекомендованный поток)

Надёжный паттерн для enterprise‑приложений:

1. **Inspect input**: `client.input(...)`
2. **Decide**: можно ли вызывать LLM (и какой текст отправлять — raw или cleaned)
3. **Call LLM**: ваш провайдер (OpenAI/локальная модель/и т.д.)
4. **Inspect output**: `client.output(...)`
5. **Decide**: что вернуть пользователю (ответ LLM или безопасный подготовленный ответ)

```python
from hivetrace import SyncHivetraceSDK

APP_ID = "your-application-id"

def as_dict(result) -> dict:
    """SDK возвращает Pydantic-модель; преобразуем её для бизнес-логики."""
    return result.model_dump() if hasattr(result, "model_dump") else dict(result)

def call_your_llm(prompt: str) -> str:
    # TODO: your LLM call (OpenAI, local model, etc.)
    return f"Model response to: {prompt}"

client = SyncHivetraceSDK()

user_message = "Hello from my app"

# 1) Input (user prompt)
input_result = client.input(
    application_id=APP_ID,
    message=user_message,
    additional_parameters={
        "session_id": "s-123",
        "user_id": "u-456",
        "environment": "prod",
        "llm_provider": "openai",
        "llm_model": "gpt-4.1-mini",
    },
)

input_data = as_dict(input_result)
if input_data.get("errors"):
    raise RuntimeError(input_data["errors"])

# Структура guardrails/custom_policy зависит от schema_version API.
# Примените здесь правила блокировки вашего приложения.
prompt_for_llm = (input_data.get("dataclean") or {}).get("cleaned_text") or user_message

# 2) Call your LLM
assistant_message = call_your_llm(prompt_for_llm)

# 3) Output (LLM response)
output_result = client.output(
    application_id=APP_ID,
    message=assistant_message,
    additional_parameters={
        "session_id": "s-123",
        "user_id": "u-456",
        "environment": "prod",
    },
)

output_data = as_dict(output_result)
if output_data.get("errors"):
    raise RuntimeError(output_data["errors"])

# Перед возвратом ответа примените решение на основе
# output_data["guardrails"] и output_data["custom_policy"].
```

---

## Клиенты: Sync и Async

- `SyncHivetraceSDK` — для синхронных сервисов (Flask/Django, background jobs)
- `AsyncHivetraceSDK` — для async runtimes (FastAPI/async workers)

Оба клиента поддерживают контекстный менеджер для корректного закрытия соединений.

### Управление ресурсами (Sync)

```python
from hivetrace import SyncHivetraceSDK

APP_ID = "your-application-id"

with SyncHivetraceSDK() as client:
    client.input(application_id=APP_ID, message="Hello")
```

### Управление ресурсами (Async)

```python
import asyncio
from hivetrace import AsyncHivetraceSDK

APP_ID = "your-application-id"

async def main():
    async with AsyncHivetraceSDK() as client:
        await client.input(application_id=APP_ID, message="Hello")

asyncio.run(main())
```

---

## Метаданные (`additional_parameters`)

Для non‑agent приложения обычно достаточно:

- `session_id` — связывает input/output в диалог
- `user_id` — аналитика/трассировка по пользователям
- `environment` — prod/staging/dev
- `llm_provider`, `llm_model` — полезно для расследований и A/B

```python
client.input(
    application_id=APP_ID,
    message="User prompt",
    additional_parameters={
        "session_id": "s-123",
        "user_id": "u-456",
        "environment": "prod",
        "llm_provider": "openai",
        "llm_model": "gpt-4.1-mini",
    },
)
```

---

## Файлы (`files`)

Прикрепления нужны, когда пользователь отправляет документы/контекст, который важен для анализа. Формат — список кортежей:

`(filename: str, content: bytes, mime_type: str)`

```python
from pathlib import Path

files = [
    ("doc1.txt", Path("doc1.txt").read_bytes(), "text/plain"),
]

client.input(
    application_id=APP_ID,
    message="Проанализируй вложение",
    files=files,
)
```

:::note
Параметр `files` поддерживается в `input()` и `output()`. Входящие файлы прикрепляются к анализу запроса пользователя, исходящие — к анализу ответа модели.
:::

---

## API: `input()`, `output()` и `function_call()`

### `input()`

Отправляет **пользовательский запрос** в HiveTrace.

- `application_id` — UUID приложения из UI
- `message` — текст запроса
- `additional_parameters` — опциональные метаданные
- `files` — опциональные файлы

### `output()`

Отправляет **ответ модели** в HiveTrace.

- `application_id` — UUID приложения из UI
- `message` — текст ответа LLM
- `additional_parameters` — опциональные метаданные

### `function_call()`

Отправляет вызов инструмента в HiveTrace. Обязательны `application_id`, `tool_call_id`, `func_name` и `func_args` (JSON-строка). Параметры `func_result` и `additional_parameters` опциональны.

:::info
Как правило, в `output()` отправляют **тот текст, который реально показывается пользователю** (после всех пост‑обработок/фильтров).
:::

---
