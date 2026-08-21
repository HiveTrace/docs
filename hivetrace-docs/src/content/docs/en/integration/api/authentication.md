---
title: "Authentication"
sidebar:
  order: 2
---

All requests require the following header:

```
Authorization: Bearer <API_TOKEN>
```

| Scenario      | Result               |
| ------------- | -------------------- |
| Missing token | **401 Unauthorized** |
| Invalid token | **401 Unauthorized** |

API tokens are generated and managed via the HiveTrace Web UI.

In HiveTrace 4.3, each API token is scoped to one application. The request `application_id` must match the application selected when the token was created.
