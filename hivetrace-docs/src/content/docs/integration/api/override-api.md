---
title: "Override API"
sidebar:
  order: 4
---

## Override-эндпоинты

### `POST /process_request/override/`

Для валидации **входящих сообщений пользователя** до их передачи в LLM, когда правила проверки необходимо задать прямо в запросе, а не через настройки Web UI.

---

### `POST /process_response/override/`

Для валидации **исходящих ответов LLM** перед их возвратом конечному пользователю, с конфигурацией проверки непосредственно в запросе.

Override-эндпоинты позволяют задавать параметры валидации inline, игнорируя настройки Web UI.

**Важно:**

- Настройки валидации из Web UI **не применяются**
- Валидация полностью управляется через `validation_config`
- Соответствующие сервисы (**dataclean, guardrails, custom, language check**) должны быть включены на уровне развертывания

---

## Тело запроса

### Обязательные поля

| Поле                | Тип             | Описание                                                         |
| ------------------- | --------------- | ---------------------------------------------------------------- |
| `application_id`    | `string` (UUID) | Идентификатор приложения. Должен существовать в HiveTrace.       |
| `message`           | `string`        | Сообщение пользователя или ответ LLM в зависимости от эндпоинта. |
| `validation_config` | `object`        | Inline-конфигурация валидации.                                   |

---

### Опциональные поля

| Поле                    | Тип      | Описание                                           |
| ----------------------- | -------- | -------------------------------------------------- |
| `additional_parameters` | `object` | Метаданные для отслеживания сессии и пользователя. |

> **Примечание:** для псевдонимизации и обратной замены требуется `additional_parameters.session_id`.

---

## Inline-валидация (`validation_config`)

Определяет правила проверки непосредственно внутри запроса.

### Пример

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

### `dataclean.patterns`: разрешенные значения

`patterns` — список паттернов модуля очистки персональных данных. Элемент списка может быть:

- строкой с именем предопределённого паттерна
- объектом с кастомным паттерном `{ "name": "...", "regex": "..." }`

> **Примечание:** Если выбрана псевдонимизация в качестве типа обработки, то для объектов с кастомным паттерном необходимо указать `pseudonym`.

Разрешённые значения для **предопределенных** паттернов:

| Паттерн            | Описание                     |
| ------------------ | ---------------------------- |
| `ADDRESS`          | Адрес                        |
| `BANK_ACCOUNT_NUMBER`| Номер расчётного счёта     |
| `BANK_CARD_NUMBER` | Номер банковской карты       |
| `BIC`              | БИК                          |
| `CORRESPONDENT_ACCOUNT` | Номер корреспондентского счёта |
| `CVC`              | CVC/CVV код банковской карты |
| `DOMAIN`           | Домен                        |
| `DRIVER_LICENSE`   | Номер водительского удостоверения |
| `EMAIL`            | Email-адрес                  |
| `INN`              | ИНН                          |
| `IP`               | IP-адрес                     |
| `KPP`              | КПП                          |
| `LICENSE_PLATE`    | Госномер авто                |
| `NAME`             | Имя (ФИО)                    |
| `OGRN`             | ОГРН                         |
| `OGRNIP`           | ОГРНИП                       |
| `PASSPORT_NUMBER`  | Номер паспорта               |
| `PHONE_NUMBER`     | Номер телефона               |
| `SNILS`            | СНИЛС                        |
| `TOKEN`            | Токен доступа                |
| `URL`              | Ссылка                       |
| `VIN`              | Идентификационный номер транспортных средств |

---

### `dataclean.whitelist`: исключения

`patterns` — список исключений модуля очистки персональных данных. Их следует применять, когда строка похожа на конфиденциальные данные, но допустима в вашем сценарии.

Элемент списка должен быть объект с кастомным паттерном `{ "name": "...", "value": "..." }`

---

### `dataclean.clean_type` (тип обработки): разрешённые значения

| Тип обработки | Описание                                                       |
| --------------| -------------------------------------------------------------- |
| `MASKING`     | Маскирование: заменяет найденные значения маской               |
| `DELETION`    | Удаление: удаляет найденные значения из текста                 |
| `DETECTION`   | Детекция: только обнаруживает совпадения, без изменения текста |
| `PSEUDONYMIZATION` | Псевдонимизация: заменяет значение устойчивой меткой типа `NAME_1` |

---

### `dataclean.enable_depseudonymization`: депсевдонимизация

Можно депсевдонимизировать данные в рамках сессии.

```json
{
    "application_id": "{{application_id}}",
    "message": "Мой КПП - 771301001. KPP_1",
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

## Схема ответа

Override-эндпоинты возвращают ту же структуру ответа, что и Base API.
