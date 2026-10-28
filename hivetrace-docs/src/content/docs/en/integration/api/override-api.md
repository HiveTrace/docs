---
title: "Override API"
sidebar:
  order: 4
---

## Override Endpoints

### `POST /process_request/override/`

For validating **incoming user messages** before they are sent to an LLM, when validation rules must be defined inline within the request instead of relying on Web UI settings.

---

### `POST /process_response/override/`

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
			"guardrails": { "enabled": true },
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
            }
        }
    }
}
```
---

## Response Schema

Override endpoints return the same response structure as the Base API.
