---
title: "Single LLM"
sidebar:
  order: 2
---

Use Gateway for a chat application that calls a model through an OpenAI-compatible client. This example uses the **Gateway 4.4** authentication contract.

## Preparation

- `GATEWAY_URL` — Gateway address including `/v1`.
- `HIVETRACE_API_KEY` — API token issued for the HiveTrace application.
- `MODEL_NAME` — configured model alias allowed for that application.
- `LITELLM_API_KEY` — optional key selecting LiteLLM routing. Omit it for direct routing configured by the Gateway administrator.
- `USER_ID` and `SESSION_ID` — external user and conversation identifiers for analytics. Keep the session ID stable across the conversation.

Create the application, configure protection and `sync`/`async` mode, allow the model, and issue its API token. An empty allowed-model list denies all model calls. The token determines the application; no application-ID header is needed.

## Python example

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

## curl example

The example uses the placeholder model alias `chat-model`; replace it with your configured model. Remove `X-LiteLLM-Api-Key` for direct routing.

```bash
curl -sS "$GATEWAY_URL/chat/completions" \
  -H "Authorization: Bearer $HIVETRACE_API_KEY" \
  -H "X-LiteLLM-Api-Key: $LITELLM_API_KEY" \
  -H "X-User-Id: $USER_ID" \
  -H "X-Session-Id: $SESSION_ID" \
  -H "Content-Type: application/json" \
  -d '{"model":"chat-model","messages":[{"role":"user","content":"Hello from my app"}]}'
```

## Sending a file

Chat Completions and Responses accept a multipart request containing a JSON `request` part and repeated `attached_files` parts. Gateway inserts those files into the model request and extracts an audit copy. The provider and model must support the selected file format.

```bash
curl -sS "$GATEWAY_URL/chat/completions" \
  -H "Authorization: Bearer $HIVETRACE_API_KEY" \
  -H "X-LiteLLM-Api-Key: $LITELLM_API_KEY" \
  -H "X-User-Id: $USER_ID" \
  -H "X-Session-Id: $SESSION_ID" \
  -F 'request={"model":"chat-model","messages":[{"role":"user","content":"Summarize the document."}]};type=application/json' \
  -F 'attached_files=@contract.pdf;type=application/pdf'
```

Standard JSON file parts also work. Use `metadata.files` only for audit-only documents; it does not deliver the document to the model. See the [integration guide](../integration-guide/) for file formats, Files API, and limits.

## Checking the result

Find the messages in session analytics using the application, user, and session. In `sync` mode, protection can block a request or replace a response. In `async` mode, deferred checks do not block the model response. HTTP 200 alone does not mean there was no violation. When a synchronous application requests `stream: true`, Gateway returns a checked non-streaming JSON response; account for this in the client.
