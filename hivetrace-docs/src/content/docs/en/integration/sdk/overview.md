---
title: Overview
---

The HiveTrace SDK adds monitoring for prompts, model responses, files, and tool calls to Python applications. It provides `SyncHivetraceSDK` and `AsyncHivetraceSDK`; both clients support context-manager resource handling.

## Capabilities

- `input()` — analyze a user prompt and optional files;
- `output()` — analyze a model response and optional files;
- `function_call()` — record a tool call and its result;
- `additional_parameters` — attach arbitrary JSON metadata for sessions, users, or agents;

Clients return Pydantic response models. Use `result.model_dump()` when application logic needs a dictionary.

## Configuration

Base clients read `HIVETRACE_URL` and `HIVETRACE_ACCESS_TOKEN`; pass `application_id` to each call.

## Where to start

- For single-model apps, see [**Single LLM**](./single-llm-applications/)
This documentation is verified against `origin/main` of `HiveTrace/hivetrace-sdk`, commit `37a846c`.
