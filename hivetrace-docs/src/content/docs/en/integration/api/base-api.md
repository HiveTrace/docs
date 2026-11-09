---
title: "Base API"
sidebar:
  order: 3
---

## Base Endpoints

### `POST /process_request/`

Use this endpoint to analyze **user messages before they are sent to an LLM** using validation rules configured in the Web UI.

When enabled, HiveTrace performs:

- sensitive data detection, deletion, masking and pseudonymization (dataclean)
- built-in policy validation (guardrails)
- custom policy validation
- language check
- token usage calculation
- session/user association
- monitoring persistence

---

### `POST /process_response/`

Use this endpoint to analyze **LLM responses before they are returned to the end user** using Web UI validation settings.

When enabled, HiveTrace performs:

- sensitive data detection, deletion, masking and pseudonymization (dataclean)
- built-in policy validation (guardrails)
- language check
- token usage calculation
- session/user association
- monitoring persistence

> **Note:** Model-based custom policies (prompts) apply only to incoming requests. Message and filename blacklists can check both directions, according to their settings.

---

## Request Body

### Required Fields

| Field            | Type            | Description                                             |
| ---------------- | --------------- | ------------------------------------------------------- |
| `application_id` | `string` (UUID) | Application identifier. Must exist in HiveTrace.        |

---

### Optional Fields

| Field                   | Type     | Description                             |
| ----------------------- | -------- | --------------------------------------- |
| `additional_parameters` | `object` | Metadata for session and user tracking. |

> **Note:** pseudonymization and depseudonymization require `additional_parameters.session_id`.

---

## additional_parameters Structure

| Field        | Type     | Description                  |
| ------------ | -------- | ---------------------------- |
| `user_id`    | `string` | External user identifier.    |
| `session_id` | `string` | External session identifier. |

---

## Response Schema

This page describes HiveTrace **4.4.0**. These analysis endpoints return results without top-level `decision`, `message`, `block_reasons`, or `user_block` fields. Use the separate `/check` endpoints below when your integration needs an allow/deny verdict.

Both endpoints return a JSON object.

| Field                    | Type             | Description                                                       |
| ------------------------ | ---------------- | ------------------------------------------------------------------|
| `request_id`             | `string`         | Analysis record UUID.                                             |
| `schema_version`         | `string`         | Response schema version.                                          |
| `status`                 | `string`         | `success \| partial_success \| error \| accepted`                             |
| `errors`                 | `string[]`       | List of processing errors (may be empty).                         |
| `warnings`               | `object[]`       | List of warnings (may be empty).                                  |
| `tokens.count`           | `integer`        | Number of tokens processed.                                       |
| `tokens.usage_severity`  | `string \| null` | `low \| high \| critical \| null`                                 |
| `guardrails.flagged`     | `boolean`        | Indicates built-in Guardrail violations.                          |
| `guardrails.truncated`   | `boolean`        | Indicates whether Guardrail processing was truncated.             |
| `custom_policy.message.model.flagged`| `boolean`| Indicates custom policy violations.                           |
| `custom_policy.message.blacklist.flagged`| `boolean`| Indicates message blacklist violations.                   |
| `custom_policy.file.blacklist.flagged`| `boolean`| Indicates file blacklist violations.                         |
| `dataclean.flagged`      | `boolean`        | Indicates sensitive data was detected in the message or in an attached file. |
| `dataclean.message.flagged`| `boolean`      | Indicates sensitive data was detected in the message.             |
| `dataclean.message.text_modified`|`boolean` | Indicates text was modified during processing.                    |
| `dataclean.message.cleaned_text`|`string \| null`   | Sanitized version of the message after processing. With the `Detection` processing type, the text remains unchanged. |
| `dataclean.message.types[]`| `array`   | Detected entity types with counts (may be empty).                      |
| `dataclean.message.entities[]` | `array` | Detected entity types, their position in the original and processed text, as well as their value (may be empty).                                                                      |
| `dataclean.file.flagged` | `boolean`         | Indicates sensitive data was detected in the attached file.      |
| `dataclean.file.save_failed`| `boolean`      | Indicates whether a file save error occurred.                    |
| `dataclean.file.check_failed` | `boolean` | Indicates whether a file check error occurred.                      |
| `dataclean.file.files[]` | `array` | List with detailed results for each file (may be empty).                   |
| `language.flagged` | `boolean \| null`         | Indicates language check violations.                                   |
| `language.enforced`| `boolean \| null`      | Indicates the block was set for language mismatch.                        |
| `language.status` | `string` | `disabled \| checked \| error`                                                   |
| `language.error` | `string \| null` | Error cause (may be empty).                                               |

Each `warnings[]` item contains three strings: `code`, `message`, and `message_ru`. For example, `session_history_full` reports that the session pseudonym dictionary is full.

`dataclean.message.cleaned_text` is `null` when no processed text is available. In detection mode, available text is unchanged. When `language.status` is `error`, `flagged` and `enforced` can be `null`; do not interpret this as a successful language check. Inspect `errors`, `warnings`, and each module status before using the result.

### dataclean.message.types structure

```json
{
	"type": "NAME",
	"count": 1
}
```

---

## Example Requests

### Process Request

```bash
curl -sS -X POST "$BASE_URL/process_request/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "application_id":"<uuid>",
    "message":"Contact me at alex@example.com.",
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

## Example Response

Illustrative synchronous response to the request above when only EMAIL masking is enabled. The token count is illustrative and depends on tokenization.

```json
{
  "request_id": "ad1472eb-e7f5-4d5c-ad33-33a9a07ca870",
  "schema_version": "2.4.0",
  "status": "success",
  "errors": [],
  "warnings": [],
  "tokens": {
    "count": 8,
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
      "cleaned_text": "Contact me at XXXXXXXXXXXXXXXX.",
      "types": [
        {
          "type": "EMAIL",
          "count": 1
        }
      ],
      "entities": [
        {
          "type": "EMAIL",
          "start": 14,
          "end": 30,
          "processed_start": 14,
          "processed_end": 30,
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

## Asynchronous processing

For an application in `async` mode with deferred ingest enabled (`ENABLE_ASYNC_INGEST=true`, the default), the server returns **HTTP 202** and `status: "accepted"`. It stores and queues the message; classification happens later. The response retains the same structure with neutral values, such as zero tokens and `flagged: false`. These values are placeholders, not completed checks.

Use `request_id` to find the record in session analytics. The original HTTP response is not updated with the later verdict. With deferred ingest disabled, classification runs in the request even for an `async` application; this does not enable synchronous blocking.

## Allow/deny checks

`POST /process_request/check` and `POST /process_response/check` accept the same message body and API token as the corresponding base endpoint. They use the application's protection settings and return a separate, compact response:

| Field | Type | Meaning |
|---|---|---|
| `decision` | `string` | `allow`, `deny`, or `accepted` |
| `reason` | `string \| null` | Explanation of the verdict |
| `request_id` | `string` | Analysis record UUID |
| `errors` | `string[] \| null` | Processing errors |
| `language` | `object \| null` | Language-check result |

In synchronous mode, the client must act on `decision`; calling the endpoint does not itself stop the client's LLM call. Check `errors` as well: an `allow` result with a failed module does not mean all checks succeeded. In deferred mode, HTTP 202 with `decision: "accepted"` is a receipt, not an allow verdict.
