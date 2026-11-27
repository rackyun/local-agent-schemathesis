---
description: "Common Schemathesis hook and custom check patterns"
connections: [ci-examples]
---

# Hooks & Custom Checks Examples

## Basic custom check

```python
# hooks.py
import schemathesis

@schemathesis.check
def check_user_isolation(ctx, response, case):
    if case.method == "GET" and "/users/" in case.path and response.status_code == 200:
        body = response.json()
        requested_id = case.path_parameters.get("user_id")
        if body.get("id") != requested_id:
            raise AssertionError(f"ID mismatch: requested {requested_id}, got {body.get('id')}")
```

## Load hooks

```bash
export SCHEMATHESIS_HOOKS=hooks
schemathesis run openapi.yaml --url http://localhost:8080
```

## More patterns

- Auth token refresh in `before_call`
- Filter flaky 503s with `filter_failure`
- Stateful setup/teardown
