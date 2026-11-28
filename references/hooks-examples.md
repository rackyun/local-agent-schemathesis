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

Set the top-level `hooks` key in the configuration file:

```toml
# schemathesis.toml
hooks = ["./hooks.py"]
```

```bash
uvx schemathesis@4.34.3 --config-file schemathesis.toml run openapi.yaml --url http://localhost:8080
```

For a one-off run from the directory containing `hooks.py`, use the environment variable:

```bash
SCHEMATHESIS_HOOKS=hooks uvx schemathesis@4.34.3 run openapi.yaml --url http://localhost:8080
```

`run --help` lists CLI options. Both the config key and the environment variable load the module and register its `@schemathesis.check` functions in 4.34.3. Use one loading method per run.

## Local verification with 4.34.3

Used a local OpenAPI fixture with a deliberately failing `check_demo` assertion and `BASE_URL` set to its address.

```bash
SCHEMATHESIS_HOOKS=hooks uvx schemathesis@4.34.3 run openapi.json \
  --url "$BASE_URL" -H "Authorization: Bearer local-demo" --mode=positive \
  --phases=examples --report=html,pytest \
  --report-html-path=schemathesis-report/html \
  --report-pytest-path=schemathesis-report/test_regression.py

uvx schemathesis@4.34.3 --config-file hooks.toml run openapi.json \
  --url "$BASE_URL" -H "Authorization: Bearer local-demo" --mode=positive \
  --phases=examples
```

Both runs reported `ST901 Custom check failed: check_demo` with the expected assertion. The first run wrote `schemathesis-report/html/index.html` and `schemathesis-report/test_regression.py`. The generated Python used `requests`, masked the token as `[Filtered]`, and preserved the observed status code. Add a business assertion to capture the custom check's intended behavior after the fix.

## More patterns

- Auth token refresh in `before_call`
- Filter flaky 503s with `filter_failure`
- Stateful setup/teardown
