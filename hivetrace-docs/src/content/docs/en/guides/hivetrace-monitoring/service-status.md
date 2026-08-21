---
title: "Service status"
sidebar:
  order: 12
---

The **Service status** page shows overall platform availability, individual components, and their dependencies. Administrators open it from the profile menu and use it for initial diagnosis.

![Platform summary and first service cards](../../../../../assets/img/service-status-summary-4-3.webp)

## Platform summary

The top card contains the overall state and update time. The overall state takes component health into account: the platform can be reported as unavailable when a required service is down even if other components remain operational.

The screenshot is an example from one check. Use the current state and update time shown in your environment.

## Service card

Each card contains:

- service name and state;
- **Type** — the component's role in the platform;
- **Response latency** — duration of the latest check in milliseconds; a dash means that no value was returned;
- **Last checked at** — time of the latest monitoring result;
- error text when a check fails;
- **Dependencies** — nested components and their individual states.

![Operational services and dependencies](../../../../../assets/img/service-status-dependencies-4-3.webp)

A dependency appears inside its parent service card with its own name, type, latency, and state. This helps distinguish a failure in the service itself from a problem in a downstream model or component.

## States

- **Operational** — the check completed successfully;
- **Degraded** — the service responds slowly or only partially;
- **Down** — the check failed;
- **Disabled** — the component is disabled by configuration and is not treated as an incident;
- **Unknown** — monitoring has not obtained a definitive result yet.

![Unavailable service, error, and unknown dependencies](../../../../../assets/img/service-status-error-4-3.webp)

For example, `HTTP 503` means the checked service could not process the request at that time. **Unknown** does not prove that a dependency failed; it means monitoring could not obtain a conclusive result.

## Responding to a failure

1. Check the latest-check time; a stale value can indicate a monitoring problem.
2. Inspect the affected service's dependencies and locate the lowest failing component.
3. Record the error text and check time.
4. Continue the investigation in that service's logs and metrics. This page does not provide restart or configuration controls.
