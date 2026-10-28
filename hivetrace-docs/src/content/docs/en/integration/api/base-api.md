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

> **Note:** Custom policies are applied **only** to `/process_request/`.

---

## Request Body

### Required Fields

| Field            | Type            | Description                                             |
| ---------------- | --------------- | ------------------------------------------------------- |
| `application_id` | `string` (UUID) | Application identifier. Must exist in HiveTrace.        |
| `message`        | `string`        | User message or LLM response depending on the endpoint. |

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

Both endpoints return a JSON object.

| Field                    | Type             | Description                                                       |
| ------------------------ | ---------------- | ------------------------------------------------------------------|
| `decision`               | `string`         | `allow \| deny \| accepted`                                       |
| `message`                | `string \| null` | Response upon violation (may be empty).                           |
| `block_reasons[]`        | `array`          | List of block codes and sources (may be empty).                   |
| `user_block.blocked`     | `boolean`        | Indicates if the user has been blocked.                           |
| `user_block.blocked_until`| `string \| null`| The date until which the user is blocked (may be empty).          |
| `request_id`             | `string`         | Analysis record UUID.                                             |
| `schema_version`         | `string`         | Response schema version.                                          |
| `status`                 | `string`         | `success \| partial_success \| error`                             |
| `errors`                 | `string[]`       | List of processing errors (may be empty).                         |
| `warnings`               | `string[]`       | List of warnings (may be empty).                                  |
| `tokens.count`           | `integer`        | Number of tokens processed.                                       |
| `tokens.usage_severity`  | `string \| null` | `low \| high \| critical \| null`                                 |
| `guardrails.flagged`     | `boolean`        | Indicates built-in Guardrail violations.                          |
| `guardrails.truncated`   | `boolean`        | Indicates whether Guardrail processing was truncated.             |
| `custom_policy.message.model.flagged`| `boolean`| Indicates custom policy violations.                           |
| `custom_policy.message.blacklist.flagged`| `boolean`| Indicates message blacklist violations.                   |
| `custom_policy.file.blacklist.flagged`| `boolean`| Indicates file blacklist violations.                         |
| `dataclean.flagged`      | `boolean`        | Indicates sensitive data was detected in the message or in an 
attached file.                                                                                                    |
| `dataclean.message.flagged`| `boolean`      | Indicates sensitive data was detected in the message.             |
| `dataclean.message.text_modified`|`boolean` | Indicates text was modified during processing.                    |
| `dataclean.message.cleaned_text`|`string`   | Sanitized version of the message after processing. With the 
`Detection` processing type, the text remains unchanged.  |
| `dataclean.message.types[]`| `array`   | Detected entity types with counts (may be empty).                      |
| `dataclean.message.entities[]` | `array` | Detected entity types, their position in the original and processed text, as well as their value (may be empty).                                                                      |
| `dataclean.file.flagged` | `boolean`         | Indicates sensitive data was detected in the attached file.      |
| `dataclean.file.save_failed`| `boolean`      | Indicates the file was saved.                                    |
| `dataclean.file.check_failed` | `boolean` | Indicates the file was checked.                                     |
| `dataclean.file.files[]` | `array` | List with detailed results for each file (may be empty).                   |
| `language.flagged` | `boolean`         | Indicates language check violations.                                   |
| `language.enforced`| `boolean`      | Indicates the block was set for language mismatch.                        |
| `language.status` | `string` | `disabled \| checked \| error`                                                   |
| `language.error` | `string \| null` | Error cause (may be empty).                                               |

### dataclean.types structure

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

## Example Response

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
