---
title: "Connecting Applications with a Single LLM (no agents)"
sidebar:
  order: 2
---

## Single LLM

This page covers the **standard integration** for a non-agent LLM app: you send a user prompt to a model and return the model output.

HiveTrace sits at the model boundary and gives you two control points:

- **`input`**: what the user is sending to the model
- **`output`**: what the model is returning to the user

---

## Prerequisites

You’ll need:

1. Your HiveTrace **instance URL**
2. An **API access token** (created in the UI)
3. Your **`application_id`** (UUID from the UI)

---

## Installation

```bash
pip install hivetrace[base]
```

---

## Configuration

The SDK reads configuration from environment variables:

- `HIVETRACE_URL` — base URL of your HiveTrace instance (`http://` or `https://`)
- `HIVETRACE_ACCESS_TOKEN` — API token
- `HIVETRACE_APP_ID` — optional convenience alias for your `application_id`

Example `.env`:

```bash
HIVETRACE_URL=https://your-hivetrace-instance.com
HIVETRACE_ACCESS_TOKEN=your-access-token
HIVETRACE_APP_ID=your-application-id
```

Or pass config explicitly:

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

## Quick integration (recommended flow)

Enterprise-ready pattern:

1. **Inspect input**: `client.input(...)`
2. **Decide**: whether to call the LLM (and whether to use raw vs cleaned text)
3. **Call LLM**: your provider (OpenAI / local models / etc.)
4. **Inspect output**: `client.output(...)`
5. **Decide**: what to return to the end user

```python
from hivetrace import SyncHivetraceSDK

APP_ID = "your-application-id"

def as_dict(result) -> dict:
    """The SDK returns a Pydantic model; convert it for application logic."""
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

# The guardrails/custom_policy shape depends on the API schema_version.
# Apply your application's blocking rules here.
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

# Before returning the response, apply your decision based on
# output_data["guardrails"] and output_data["custom_policy"].
```

---

## Clients: sync vs async

- `SyncHivetraceSDK` — sync runtimes (Flask/Django, background jobs)
- `AsyncHivetraceSDK` — async runtimes (FastAPI/async workers)

Both support context managers for clean resource handling.

### Resource management (Sync)

```python
from hivetrace import SyncHivetraceSDK

APP_ID = "your-application-id"

with SyncHivetraceSDK() as client:
    client.input(application_id=APP_ID, message="Hello")
```

### Resource management (Async)

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

## Metadata (`additional_parameters`)

For a non-agent app, common fields are:

- `session_id` — ties `input`/`output` together
- `user_id` — traceability and analytics
- `environment` — prod/staging/dev
- `llm_provider`, `llm_model` — useful for investigations and A/Bs

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

## Files (`files`)

Use attachments when users provide documents/context you want HiveTrace to analyze. Format:

`(filename: str, content: bytes, mime_type: str)`

```python
from pathlib import Path

files = [
    ("doc1.txt", Path("doc1.txt").read_bytes(), "text/plain"),
]

client.input(
    application_id=APP_ID,
    message="Please analyze the attachment",
    files=files,
)
```

:::note
The `files` parameter is supported by both `input()` and `output()`. Input files are attached to user-prompt analysis; output files are attached to model-response analysis.
:::

---

## API: `input()`, `output()`, and `function_call()`

### `input()`

Sends a **user prompt** to HiveTrace.

- `application_id` — application UUID from the UI
- `message` — prompt text
- `additional_parameters` — optional metadata
- `files` — optional attachments

### `output()`

Sends an **LLM response** to HiveTrace.

- `application_id` — application UUID from the UI
- `message` — LLM response text
- `additional_parameters` — optional metadata

### `function_call()`

Sends a tool call to HiveTrace. `application_id`, `tool_call_id`, `func_name`, and `func_args` (a JSON string) are required. `func_result` and `additional_parameters` are optional.

:::info
Typically you should send in `output()` the **exact text that the user sees** (after post-processing/filtering).
:::

---
