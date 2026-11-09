---
title: "Мультиагенты"
sidebar:
  order: 3
---

## Ответственность компонентов

Направляйте вызовы модели через Gateway. Оркестрация агентов и выполнение инструментов остаются в вашем приложении. Gateway проверяет трафик модели; он не исполняет локальные инструменты и не останавливает их вызовы автоматически. Для телеметрии или контроля исполнения инструментов используйте отдельную SDK/hook-интеграцию.

## Подключение к Gateway 4.4

- В `base_url` OpenAI-клиента укажите адрес Gateway с `/v1`.
- В `api_key` передайте **API-токен приложения HiveTrace**.
- Для LiteLLM передайте отдельный заголовок `X-LiteLLM-Api-Key`; для настроенного прямого маршрута не задавайте его.
- Разрешите приложению все модели, используемые агентами. Приложение определяется токеном HiveTrace.
- Используйте один `X-Session-Id` для связанных шагов агентов и устойчивый `X-User-Id` инициатора задачи.
- Настройте режим `sync`/`async` в приложении, а не в заголовке запроса.

## Шаги агента и телеметрия

Одна задача пользователя может вызывать несколько обращений к модели. Сохраняйте общий идентификатор диалога, чтобы расследовать их вместе. Не используйте старую настройку `HIVETRACE_DEDUP_AGENT_TOOL_CALLS`: она не входит в конфигурацию Gateway 4.4.

`X-HiveTrace-Skip-System-Requests: false` отключает пропуск запросов, определённых как служебные. Это не гарантирует сохранения каждой system-инструкции отдельным сообщением. Проверьте записи, создаваемые вашим агентным фреймворком, и не считайте результат инструмента новым пользовательским промптом.

## Пример Python

Пример использует OpenAI Agents SDK с Chat Completions и локальный инструмент с демонстрационными данными. Установите совместимые с проектом пакеты `openai` и `openai-agents`. `MODEL_NAME` должен быть настроенной и разрешённой моделью с поддержкой инструментов.

```python
import asyncio
import os
from typing import Any

from openai import AsyncOpenAI
from agents import Agent, RunConfig, Runner, function_tool
from agents.models.openai_chatcompletions import OpenAIChatCompletionsModel


@function_tool
def fetch_users_summary() -> dict[str, Any]:
    """Return a demo user count; replace with an authorized application query."""
    return {"total_count": 123}


async def main() -> None:
    headers = {
        "X-User-Id": os.environ["USER_ID"],
        "X-Session-Id": os.environ["SESSION_ID"],
        "X-HiveTrace-Skip-System-Requests": "false",
    }
    if os.getenv("LITELLM_API_KEY"):
        headers["X-LiteLLM-Api-Key"] = os.environ["LITELLM_API_KEY"]

    async with AsyncOpenAI(
        base_url=os.environ["GATEWAY_URL"].rstrip("/"),
        api_key=os.environ["HIVETRACE_API_KEY"],
        default_headers=headers,
    ) as client:
        agent = Agent(
            name="Demo assistant",
            instructions="Call fetch_users_summary once, then answer briefly.",
            tools=[fetch_users_summary],
            model=OpenAIChatCompletionsModel(
                model=os.environ["MODEL_NAME"], openai_client=client
            ),
        )
        result = await Runner.run(
            agent,
            input="How many users do we have?",
            max_turns=3,
            run_config=RunConfig(tracing_disabled=True),
        )
        print(result.final_output)


if __name__ == "__main__":
    asyncio.run(main())
```

## Проверка

Проверьте запросы модели, ответы и связь с сессией в HiveTrace. Отправьте допустимое сообщение и сообщение с настроенным нарушением. В синхронном режиме учитывайте подмену ответа и принудительный непотоковый JSON; не запускайте локальные инструменты только на основании HTTP 200 от запроса модели. В асинхронном режиме анализ выполняется после приёма и не может остановить этот ответ модели.

Авторизация, маршрутизация, файлы и ошибки описаны в [руководстве по интеграции](../integration-guide/).
