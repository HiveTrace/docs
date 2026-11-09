---
title: "Override API"
sidebar:
  order: 4
---

## Override Endpoints

### `POST /process_request/override`

For validating **incoming user messages** before they are sent to an LLM, when validation rules must be defined inline within the request instead of relying on Web UI settings.

---

### `POST /process_response/override`

For validating **outgoing LLM responses** before they are returned to the end user, with validation configured directly in the request.

Override endpoints allow validation behavior to be defined inline, bypassing Web UI configuration.

**Important:**

- Web UI validation settings are **not applied**
- Validation is fully controlled via `validation_config`
- The corresponding services (**dataclean, guardrails, custom, language check**) must still be enabled at the deployment level

---

## Request Body

### Required Fields

| Field               | Type            | Description                                             |
| ------------------- | --------------- | ------------------------------------------------------- |
| `application_id`    | `string` (UUID) | Application identifier. Must exist in HiveTrace.        |
| `message`           | `string`        | User message or LLM response depending on the endpoint. |
| `validation_config` | `object`        | Inline validation configuration.                        |

---

### Optional Fields

| Field                   | Type     | Description                             |
| ----------------------- | -------- | --------------------------------------- |
| `additional_parameters` | `object` | Metadata for session and user tracking. |

> **Note:** pseudonymization and depseudonymization require `additional_parameters.session_id`.

---

## Inline Validation (`validation_config`)

### Request and response directions

The two endpoints use different schemas:

| Setting | `/process_request/override` | `/process_response/override` |
|---|---|---|
| Model-based custom policies: `custom.policies`, `custom.message.model` | Supported | Not accepted |
| Message and filename blacklists | Supported | Supported |
| `dataclean.enable_depseudonymization` | Not accepted | Supported |

Sending a field to the wrong endpoint causes a validation error (HTTP 422). The first example below is for **`/process_request/override`**; the depseudonymization example is for **`/process_response/override`**.

Override changes the validation rules, but not API-token scope, platform-wide module switches, or the application's `sync`/`async` mode. Deferred requests return HTTP 202 with `status: "accepted"`, as described in [Base API](../base-api/).



Defines validation rules directly inside the request.

### Example

```json
{
  "application_id": "{{application_id}}",
  "message": "My name is John, my CVC is 222, and I own a car.",
  "additional_parameters": {
    "session_id": "{{client_session_id}}",
    "user_id": "{{client_user_id}}"
  },
  "validation_config": {
    "source": "inline",
    "inline": {
      "guardrails": {
        "enabled": true
      },
      "custom": {
        "enabled": true,
        "policies": [
          {
            "name": "medicine",
            "prompt": "Do not provide medical advice or discuss medical topics."
          }
        ]
      },
      "dataclean": {
        "enabled": true,
        "clean_type": "pseudonymization",
        "patterns": [
          "CVC",
          {
            "name": "AUTOMOBILE",
            "regex": "\\bcar\\b",
            "pseudonym": "car"
          }
        ]
      }
    }
  }
}
```
---

### `dataclean.patterns`: allowed values

`patterns` is a list of dataclean patterns. Each item can be:

- a string with a predefined pattern name
- an object with a custom pattern `{ "name": "...", "regex": "..." }`

> **Note:** If pseudonymization is selected as the processing type, `pseudonym` must be passed for objects with a custom pattern.

Allowed values for **predefined** patterns:

| Pattern            | Description                          |
| ------------------ | ------------------------------------ |
| `ADDRESS`          | Address                              |
| `BANK_ACCOUNT_NUMBER` | Bank account number               |
| `BANK_CARD_NUMBER` | Bank card number                     |
| `BIC`              | BIC                                  |
| `CORRESPONDENT_ACCOUNT` | Correspondent account           |
| `CVC`              | Card CVC/CVV                         |
| `DOMAIN`           | Domain                               |
| `DRIVER_LICENSE`   | Driver's license number              |
| `EMAIL`            | Email addresses                      |
| `INN`              | INN (taxpayer identification number) |
| `IP`               | IP address                           |
| `KPP`              | KPP                                  |
| `LICENSE_PLATE`    | Vehicle registration plate           |
| `NAME`             | Person name                          |
| `OGRN`             | OGRN                                 |
| `OGRNIP`           | OGRNIP                               |
| `PASSPORT_NUMBER`  | Passport number                      |
| `PHONE_NUMBER`     | Phone number                         |
| `SNILS`            | SNILS                                |
| `TOKEN`            | Access token                         |
| `URL`              | URL                                  |
| `VIN`              | Vehicle identification number        |

---

### `dataclean.whitelist`: exceptions

`patterns` — exceptions for the personal data cleaning module. They should be applied when a string resembles sensitive data but is valid in your workflow.

Each item should be an object with a custom pattern `{ "name": "...", "value": "..." }`

### Example

```json
{
  "application_id": "{{application_id}}",
  "message": "Unable to navigate to дети.рф",
  "additional_parameters": {
    "user_id": "{{client_user_id}}",
    "session_id": "{{client_session_id}}"
  },
  "validation_config": {
    "source": "inline",
    "inline": {
      "dataclean": {
        "enabled": true,
        "clean_type": "masking",
        "patterns": [
          "DOMAIN"
        ],
        "whitelist": {
          "patterns": [
            {
              "name": "cyrillic domain",
              "value": "(?i)\\b(?:[а-яёА-ЯЁ0-9-]+\\.)+(?:рф|бел|сайт|онлайн|дети)(?![а-яёА-ЯЁ0-9])"
            }
          ]
        }
      },
      "guardrails": {
        "enabled": false
      },
      "custom": {
        "enabled": false
      }
    }
  }
}
```

---

### `dataclean.clean_type` (processing type): allowed values

| Processing type    | Description                                           |
| -------------------| ----------------------------------------------------- |
| `MASKING`          | Masking: replaces matches with a mask                 |
| `DELETION`         | Deletion: removes matches from the text               |
| `DETECTION`        | Detection: detects matches without modifying the text |
| `PSEUDONYMIZATION` | Pseudonymization: replaces matches with a stable label such as `NAME_1`|

---

### `dataclean.enable_depseudonymization`: depseudonymization

Data can be depseudonymized within a session.


The following example uses `/process_response/override`. First pseudonymize a request with the same `application_id` and `additional_parameters.session_id` so that the session dictionary contains the values to restore.

### Example

```json
{
  "application_id": "{{application_id}}",
  "message": "My KPP is 771301001. KPP_1",
  "additional_parameters": {
    "session_id": "{{client_session_id}}",
    "user_id": "{{client_user_id}}"
  },
  "validation_config": {
    "source": "inline",
    "inline": {
      "dataclean": {
        "enabled": true,
        "clean_type": "pseudonymization",
        "patterns": [
          "KPP"
        ],
        "enable_depseudonymization": true
      },
      "guardrails": {
        "enabled": false
      },
      "custom": {
        "enabled": false
      }
    }
  }
}
```
---

### `custom.message.blacklist`: blacklist message patterns

`patterns` — patterns from this list are applied to message text.

Each item should be an object with a custom pattern `{ "name": "...", "pattern": "..." }`

### Example

```json
{
  "application_id": "{{application_id}}",
  "message": "How can I pay with an expired card?",
  "additional_parameters": {
    "user_id": "{{client_user_id}}",
    "session_id": "{{client_session_id}}"
  },
  "validation_config": {
    "source": "inline",
    "inline": {
      "custom": {
        "enabled": true,
        "message": {
          "blacklist": {
            "patterns": [
              {
                "name": "card",
                "pattern": "\\b[Cc]ard\\w*"
              }
            ]
          }
        }
      },
      "guardrails": {
        "enabled": false
      }
    }
  }
}
```

### `custom.file.blacklist`: blacklist file patterns

`patterns` — patterns from this list are matched against the **names** of attached files, not their contents. DataClean file processing checks extracted file contents.

Each item should be an object with a custom pattern `{ "name": "...", "pattern": "..." }`

### Example

```json
{
  "application_id": "{{application_id}}",
  "message": "How to use cashback?",
  "additional_parameters": {
    "user_id": "{{client_user_id}}",
    "session_id": "{{client_session_id}}"
  },
  "validation_config": {
    "source": "inline",
    "inline": {
      "custom": {
        "enabled": true,
        "file": {
          "blacklist": {
            "patterns": [
              {
                "name": "FILE_PATTERN",
                "pattern": "\\b[Cc]ard\\w*"
              }
            ]
          }
        }
      },
      "guardrails": {
        "enabled": false
      }
    }
  }
}
```

### `language`: language check

`allowed_langs` — the list of allowed languages defined by a set of codes: `en`, `ar`, `hy`, `be`, `ka`, `he`, `kk`, `ru`, `tr`, `uk`. You can choose one language or several — up to all of them. Text in a language not included in the list may be flagged as a violation, depending on the threshold of permissible letters from other writing systems.

`enforce` configures whether a detected language mismatch should be enforced (`true`) or only recorded (`false`). It is an input setting; the outcome is reported as `language.enforced`.

`tolerance` - sets the permissible share of letters from other writing systems at which the text is not yet considered a violation. The threshold can be set from 0 to 1.

### Example

```json
{
  "application_id": "{{application_id}}",
  "message": "За что списание по карте за неделю, p.s. за последние 3 дня",
  "additional_parameters": {
    "user_id": "{{client_user_id}}",
    "session_id": "{{client_session_id}}"
  },
  "validation_config": {
    "source": "inline",
    "inline": {
      "language": {
        "enabled": true,
        "allowed_langs": [
          "ru",
          "en"
        ],
        "enforce": false,
        "tolerance": 0.01
      },
      "guardrails": {
        "enabled": false
      },
      "custom": {
        "enabled": false
      }
    }
  }
}
```

## Response Schema

Override endpoints return the same response structure as the Base API.
