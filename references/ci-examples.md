---
description: "GitHub Actions CI templates for PR and nightly Schemathesis runs"
connections: [hooks-examples]
---

# CI Examples

## PR workflow (shallow)

```yaml
# .github/workflows/api-test-pr.yml
name: API Test (PR)
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: schemathesis/action@v3
        with:
          schema: openapi.yaml
          base-url: ${{ secrets.STAGING_URL }}
          args: --max-examples=50 --phases=fuzzing
```

## Nightly workflow (deep)

```yaml
# .github/workflows/api-test-nightly.yml
name: API Test (Nightly)
on:
  schedule:
    - cron: '0 2 * * *'
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: schemathesis/action@v3
        with:
          schema: openapi.yaml
          base-url: ${{ secrets.STAGING_URL }}
          args: --max-examples=400 --phases=fuzzing,stateful --report=html
```
