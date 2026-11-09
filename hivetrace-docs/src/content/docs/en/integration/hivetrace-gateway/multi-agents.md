---
title: "Multi-agent applications"
sidebar:
  order: 3
---

## Responsibilities

Route model calls through Gateway. Agent orchestration and tool execution remain in your application. Gateway checks model traffic; it does not execute local tools or automatically prevent a local tool call. Use a dedicated SDK/hook integration if tool-execution telemetry or enforcement is required.

## Gateway 4.4 connection

- Set the OpenAI client's `base_url` to the Gateway address including `/v1`.
- Pass the **HiveTrace application API token** as `api_key`.
- Supply `X-LiteLLM-Api-Key` separately for LiteLLM routing; omit it for configured direct routing.
- Allow every model used by the agents for the application. The HiveTrace token determines the application.
- Use the same `X-Session-Id` across related agent steps and a stable `X-User-Id` for the initiating user.
- Configure `sync`/`async` in application settings, not a request header.

## Agent steps and telemetry

One user task may involve several model calls. Keep their conversation identifier consistent so they can be investigated together. Do not rely on the legacy `HIVETRACE_DEDUP_AGENT_TOOL_CALLS` setting: it is not part of the Gateway 4.4 configuration.

`X-HiveTrace-Skip-System-Requests: false` disables skipping of requests identified as service requests. It does not guarantee that every system instruction is stored as a separate message. Check the records produced by the actual agent framework and avoid treating a tool result as a newly submitted user prompt.

## Python example

The example uses OpenAI Agents SDK with Chat Completions and a local tool returning demo data. Install the project's compatible `openai` and `openai-agents` packages. `MODEL_NAME` must be a configured, allowed model with tool support.

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

## Verification

Check the model requests, responses, and session correlation in HiveTrace. Test an allowed message and a configured violation. In synchronous mode, account for replacement responses and forced non-streaming JSON; do not execute local tools solely because a model request returned HTTP 200. In asynchronous mode, analysis happens after acceptance and cannot prevent that model response.

See the [integration guide](../integration-guide/) for authentication, routing, files, and errors.
