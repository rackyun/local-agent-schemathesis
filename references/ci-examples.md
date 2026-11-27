---
description: "GitHub Actions CI templates for PR and nightly Schemathesis runs"
connections: [hooks-examples]
---

# CI Examples

Set the repository secrets `STAGING_URL` and `STAGING_API_TOKEN` for the target API. The examples use Bearer auth. The PR job skips fork PRs, where GitHub Actions withholds repository secrets. Both workflows install Schemathesis 4.34.3 and upload the report directory even when tests fail.

## PR workflow (shallow)

```yaml
# .github/workflows/api-test-pr.yml
name: API Test (PR)
on: [pull_request]
jobs:
  test:
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.13'
      - run: python -m pip install schemathesis==4.34.3
      - name: Run Schemathesis
        env:
          STAGING_URL: ${{ secrets.STAGING_URL }}
          STAGING_API_TOKEN: ${{ secrets.STAGING_API_TOKEN }}
        run: |
          : "${STAGING_URL:?Set the STAGING_URL secret}"
          : "${STAGING_API_TOKEN:?Set the STAGING_API_TOKEN secret}"
          schemathesis run openapi.yaml --url "$STAGING_URL" \
            --header "Authorization: Bearer $STAGING_API_TOKEN" \
            --max-examples=50 --phases=examples,coverage,fuzzing \
            --report=html,pytest --report-html-path=schemathesis-report/html \
            --report-pytest-path=schemathesis-report/test_regression.py
      - name: Upload Schemathesis reports
        if: ${{ !cancelled() }}
        uses: actions/upload-artifact@v4
        with:
          name: schemathesis-pr
          path: schemathesis-report/
          if-no-files-found: warn
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
      - uses: actions/setup-python@v5
        with:
          python-version: '3.13'
      - run: python -m pip install schemathesis==4.34.3
      - name: Run Schemathesis
        env:
          STAGING_URL: ${{ secrets.STAGING_URL }}
          STAGING_API_TOKEN: ${{ secrets.STAGING_API_TOKEN }}
        run: |
          : "${STAGING_URL:?Set the STAGING_URL secret}"
          : "${STAGING_API_TOKEN:?Set the STAGING_API_TOKEN secret}"
          schemathesis run openapi.yaml --url "$STAGING_URL" \
            --header "Authorization: Bearer $STAGING_API_TOKEN" \
            --max-examples=400 --phases=examples,coverage,fuzzing,stateful \
            --report=html,pytest --report-html-path=schemathesis-report/html \
            --report-pytest-path=schemathesis-report/test_regression.py
      - name: Upload Schemathesis reports
        if: ${{ !cancelled() }}
        uses: actions/upload-artifact@v4
        with:
          name: schemathesis-nightly
          path: schemathesis-report/
          if-no-files-found: warn
```

## Local verification with 4.34.3

Ran each template's `Run Schemathesis` shell block with `bash -e -c`, a local `STAGING_URL`, and a dummy `STAGING_API_TOKEN`. Both returned exit code 1 for an intentional HTTP 201 versus documented HTTP 200 mismatch and reported HTML and pytest files under `schemathesis-report/`. Artifact upload itself runs on GitHub Actions.
