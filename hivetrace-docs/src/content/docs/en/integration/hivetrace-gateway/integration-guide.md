---
title: "Integration Guide"
sidebar:
  order: 1
---

This guide describes the **Gateway 4.4 release branch** contract. Gateway accepts client requests, checks model access, sends messages to Monitoring, and calls the provider directly or through LiteLLM.

## 1. Preparation

1. Create a HiveTrace application with the **Gateway** integration.
2. Configure protection, `sync` or `async` mode, and allowed models. **An empty allowed-model list denies every model call.**
3. Issue an API token for this application.
4. Obtain the Gateway URL and an available model name. Examples use the placeholder alias `chat-model`; replace it with a configured model allowed for your application.
5. For LiteLLM routing, obtain a separate LiteLLM key. For direct routing, the Gateway administrator configures the provider and its credentials.

In these examples, `GATEWAY_URL` includes `/v1`, for example `https://gateway.example.com/v1`. The running Gateway exposes `/docs`, `/redoc`, and `/openapi.json` relative to the service root.

## 2. Authentication and application identity

The client sends a **HiveTrace key**, not a provider key:

```http
Authorization: Bearer <hivetrace-api-token>
```

Gateway validates the key against Monitoring's signed access-policy directory. The directory associates a key with an application, allowed models, and user-directory sources. The key determines the application; `X-Application-Id` and the `HIVETRACE_APPLICATION_ID` environment variable do not select an application in this contract.

For an OpenAI client, pass the HiveTrace key as `api_key`. For other clients, ensure it is sent as `Authorization: Bearer`. The legacy `X-HiveTrace-Api-Key` header does not replace `Authorization`.

| Result | Cause |
|---|---|
| HTTP 401 | Missing key or key unknown to the directory |
| HTTP 403 | Model or supplied directory source not allowed for the application |
| HTTP 503 | No usable, verified access-policy directory available |

Gateway does not call the model until authorization succeeds. The same key authenticates telemetry requests to Monitoring; the client's HiveTrace key is not forwarded as the provider's `Authorization`.

## 3. Provider routing

| Client request | Route | Upstream credential |
|---|---|---|
| Nonempty `X-LiteLLM-Api-Key` | Configured LiteLLM upstream | Header value becomes the outgoing `Authorization: Bearer` |
| No `X-LiteLLM-Api-Key` | Direct routing using `GATEWAY_MODEL_CONFIG` | Gateway configuration: Bearer for an OpenAI-compatible provider or `x-api-key` for Anthropic |

Putting a LiteLLM key in `Authorization` does not select LiteLLM routing: Gateway treats it as a HiveTrace key. If direct routing is not configured, a request without `X-LiteLLM-Api-Key` fails. In direct mode, Gateway may replace the public model alias with the provider model name and add configured model parameters.

## 4. Headers

| Header | Purpose |
|---|---|
| `Authorization: Bearer <hivetrace-api-token>` | Required HiveTrace application key |
| `X-LiteLLM-Api-Key` | Optional key selecting the LiteLLM route |
| `X-User-Id` | External user ID for analytics |
| `X-Session-Id` | External session ID; keep it stable across a conversation for consistent pseudonymization |
| `X-Source-Id` | Directory source for a user from a synchronized directory; must be allowed for the application |
| `X-HiveTrace-Skip-System-Requests` | Overrides skipping of service requests in telemetry (`true`/`false`); does not switch `sync`/`async` |
| `X-Model-Name` | Selects the model/provider for Files API when the target provider is ambiguous |
| `anthropic-version`, `anthropic-beta` | Anthropic protocol headers for the relevant operations |

Set `sync`/`async` in application settings. Do not use the client header `X-HiveTrace-Mode` to switch modes.

## 5. Endpoints and examples

| Method | Path | Processing |
|---|---|---|
| POST | `/v1/chat/completions` | OpenAI Chat Completions, checks before and after the model |
| POST | `/v1/responses` | OpenAI Responses, checks before and after the model |
| POST | `/v1/messages` | Anthropic Messages, checks before and after the model |
| POST | `/v1/embeddings` | Proxying with model authorization, without the full message-analysis pipeline |
| GET | `/v1/models` | Models from LiteLLM or Gateway's local registry, depending on the route |
| POST / GET | `/v1/files` | Upload / list files |
| GET / DELETE | `/v1/files/{id}` | Get metadata / delete a file |
| GET | `/v1/files/{id}/content` | Check and download file contents |

A model appearing in `/v1/models` does not replace authorization: calling it must still be allowed for the application. The selected provider and model must support the API format being used.

### Chat Completions through LiteLLM

```bash
curl -sS "$GATEWAY_URL/chat/completions" \
  -H "Authorization: Bearer $HIVETRACE_API_KEY" \
  -H "X-LiteLLM-Api-Key: $LITELLM_API_KEY" \
  -H "X-User-Id: user-123" \
  -H "X-Session-Id: session-456" \
  -H "Content-Type: application/json" \
  -d '{"model":"chat-model","messages":[{"role":"user","content":"Hello!"}]}'
```

For direct routing, remove the `X-LiteLLM-Api-Key` line; the model must exist in Gateway configuration.

```python
import os
from openai import OpenAI

headers = {"X-User-Id": "user-123", "X-Session-Id": "session-456"}
if os.getenv("LITELLM_API_KEY"):
    headers["X-LiteLLM-Api-Key"] = os.environ["LITELLM_API_KEY"]

with OpenAI(
    base_url=os.environ["GATEWAY_URL"].rstrip("/"),
    api_key=os.environ["HIVETRACE_API_KEY"],
    default_headers=headers,
) as client:
    response = client.chat.completions.create(
        model=os.environ["MODEL_NAME"],
        messages=[{"role": "user", "content": "Hello!"}],
    )
    print(response.choices[0].message.content)
```

### OpenAI Responses

```bash
curl -sS "$GATEWAY_URL/responses" \
  -H "Authorization: Bearer $HIVETRACE_API_KEY" \
  -H "X-LiteLLM-Api-Key: $LITELLM_API_KEY" \
  -H "X-Session-Id: session-456" \
  -H "Content-Type: application/json" \
  -d '{"model":"chat-model","input":"Briefly explain the result."}'
```

### Anthropic Messages

This direct-routing example uses `claude-model`, which must be a configured Anthropic model alias. Gateway obtains provider credentials from its configuration.

```bash
curl -sS "$GATEWAY_URL/messages" \
  -H "Authorization: Bearer $HIVETRACE_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-model","max_tokens":256,"messages":[{"role":"user","content":"Hello!"}]}'
```

For LiteLLM routing, add `X-LiteLLM-Api-Key` and use a model configured on the corresponding LiteLLM route.

## 6. Files

Send files in the standard request body. Gateway extracts an audit copy from supported file parts; a separate file header is unnecessary.

| Format | File parts |
|---|---|
| Chat Completions | `file` / `image_url` in `messages[].content` |
| Responses | `input_file` / `input_image` in `input[]` |
| Anthropic Messages | `document` / `image` in `content` |

Depending on the format, use base64/data URLs, absolute HTTP(S) URLs, or `file_id`. Auditing a `file_id` requires Gateway to have the previously uploaded bytes available through its upload cache. Whether the model can read the file depends on the provider and model.

### Multipart

Chat Completions and Responses also accept `multipart/form-data`: JSON in a `request` part and files in repeated `attached_files` parts. Gateway adds the files to the last user message and sends the provider the regular body for the selected API. Anthropic Messages does not accept this multipart transport.

```bash
curl -sS "$GATEWAY_URL/chat/completions" \
  -H "Authorization: Bearer $HIVETRACE_API_KEY" \
  -H "X-LiteLLM-Api-Key: $LITELLM_API_KEY" \
  -F 'request={"model":"chat-model","messages":[{"role":"user","content":"Summarize the document."}]};type=application/json' \
  -F 'attached_files=@contract.pdf;type=application/pdf'
```

### Audit only

For a document processed by the client that should not be sent to the model in full, use `metadata.files` in the JSON body. An item may contain `name`, `type`, and an absolute `url` or `content_base64`. This is a separate audit channel, not a model-facing file part. The legacy `X-Attached-Files` header is no longer a way to submit attachments.

For clients repeating the file list on every turn, `metadata.request_id` and `file.request_id` distinguish new attachments from previous-turn files. `metadata.attachments` and `metadata.attached_files` do not replace `metadata.files`.

### Files API

`/v1/files` supports OpenAI and Anthropic forms. For Anthropic, send the appropriate `anthropic-beta: files-api-…` header; OpenAI uploads use `file` and `purpose`. If direct routing has multiple providers with different credentials, use `X-Model-Name` to select the provider. Subsequent operations on a `file_id` must address the same provider. Content downloads pass through a file check before bytes are returned. Anthropic only permits downloading files created by code execution or skills; attempting to download a user-uploaded file returns a provider error.

Successfully uploading a file to the provider is not the same as successfully checking its contents in Monitoring. Filename blacklists and DataClean detection in extracted file contents are separate mechanisms. DataClean does not rewrite the original file contents.

## 7. Responses, modes, and blocking

Normal responses use the selected API format. Protection can modify the body: DataClean sanitizes text, pseudonymization and restoration replace values, and blocking substitutes the configured reply. Direct routing may also change the model alias and parameters according to configuration.

- **`sync`**: Gateway waits for checks. A pre-model block prevents the provider call; a post-model block replaces the response. With `stream: true`, Gateway forces a non-streaming upstream request and returns JSON so it can check the complete response. Clients must account for this behavior.
- **`async`**: with deferred ingest enabled, Monitoring returns HTTP 202 and an `accepted` receipt to Gateway. Checks run later and do not stop the model response. This does not mean the Gateway client receives HTTP 202 instead of the model response. SSE streaming is available when the selected route supports it.

A policy block can be represented by a successful HTTP response containing replacement text. HTTP 200 alone does not prove there was no violation. Authorization, transport, and limit errors have separate HTTP statuses.

Monitoring warnings may reach the client in the `X-HiveTrace-Warning` header, for example `session_history_full` when the session pseudonym dictionary is full. Handle warnings separately from the HTTP status.

## 8. Errors and limits

| Status | Example causes |
|---|---|
| 400 | Invalid body, missing multipart `request` JSON part, incompatible model route |
| 401 | Missing/unknown HiveTrace key; an upstream credential error is also possible |
| 403 | Model or directory source not allowed for the application |
| 404 | Unknown direct-route model or provider resource |
| 413 | Per-file or total multipart size limit exceeded |
| 429 | Upstream rate limiting |
| 502/503 | Transport failure, unavailable access-policy directory, or unconfigured direct routing |

File errors must not be treated as successful checks: with file protection enabled, a missing or unsuccessfully checked expected attachment can block the message. Inspect Monitoring results and diagnostics.

| Gateway setting | Default |
|---|---|
| `GATEWAY_UPSTREAM_TIMEOUT` | 120 seconds |
| `HIVETRACE_TIMEOUT` | 60 seconds |
| `HIVETRACE_SETTINGS_TIMEOUT` | 3 seconds |
| `HIVETRACE_FILES_MAX_BYTES` | 20 MiB per file |
| `HIVETRACE_FILES_MULTIPART_MAX_TOTAL_BYTES` | 32 MiB total multipart file size |
| `HIVETRACE_FILES_TIMEOUT` | 15 seconds for attachment downloads |
| `HIVETRACE_FILES_UPLOAD_TIMEOUT` | 20 seconds for upload to Monitoring |
| `HIVETRACE_FILES_VERDICT_TIMEOUT` | 60 seconds to wait for the result with files |

A message may contain up to 10 files. Deployment values may differ. Download plus upload time must fit within Monitoring's hard file-check deadline, which must be shorter than Gateway's verdict timeout. Proxy and provider limits also apply.

## 9. Gateway configuration

The administrator configures these required settings:

```bash
HIVETRACE_URL=https://hivetrace.example.com/api
ACCESS_POLICY_PUBLIC_KEY_PEM="<public-key-for-verifying-the-directory>"
HIVETRACE_SERVICE_KEY="<gateway-service-key-issued-by-the-administrator>"
```

Gateway cannot start without them. The service key is for internal communication; it does not replace the client's application key.

Direct routing additionally requires `GATEWAY_MODEL_CONFIG` to point to a YAML file containing model routes and provider credential references. Use `model_list[].gateway_params` in that file; the Gateway registry rejects the LiteLLM field name `litellm_params`. For LiteLLM, configure `UPSTREAM_URL` and have clients supply a separate LiteLLM key.

If Monitoring becomes unavailable, authorization may use the last verified directory within its allowed lifetime. When no usable copy is available, Gateway returns 503 without calling the model. A cached directory does not guarantee that current protection settings or check results are available; diagnose those failures separately. Before production use, verify Monitoring connectivity, application settings, and blocking, file, and streaming scenarios.
