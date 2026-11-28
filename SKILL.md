---
name: local-agent-schemathesis
description: "Design and run a fully self-hosted API testing workflow that combines Schemathesis property-based testing with a local AI agent (Claude Code, Cursor, etc.) for schema improvement, custom checks, and failure analysis."
type: workflow
lifecycle: active
---

# Local Agent + Schemathesis Workflow

Use this skill when the user wants a free, self-hosted alternative to paid AI API testing tools (Keploy, Karate Agent, Postman Agent). Generate comprehensive API tests from OpenAPI/GraphQL using Schemathesis, and use a local agent to handle business rules, failure analysis, and schema quality.

## Core Workflow

1. **Prepare environment**
   - Ensure Schemathesis is available: `uv tool install schemathesis` or `uvx schemathesis`.
   - Confirm the user has a local agent (Claude Code, Cursor, Continue + Ollama, etc.).

2. **Schema quality gate**
   - Ask the agent to review the OpenAPI for missing constraints, error responses, and examples.
   - Prefer storing the spec in the repo as `openapi.yaml` or generating it from code.

3. **Run Schemathesis**
   - Start with CLI for speed: `schemathesis run openapi.yaml --url <base-url> --max-examples=100`.
   - Add `schemathesis.toml` for auth, phases, and per-operation overrides. See [[case-volume]] for a validated example.
   - Select phases with `--phases`:
     - `examples` runs explicit examples from the schema.
     - `coverage` generates targeted cases for schema constraints and boundary values.
     - `fuzzing` generates cases with Hypothesis. `--max-examples` applies per operation.
     - `stateful` chains dependent operations when the schema exposes usable links.

4. **Enhance with local agent**
   - Generate custom `@schemathesis.check` functions for business rules (user isolation, audit headers, etc.).
   - Place them in `hooks.py` and load via `hooks = ["./hooks.py"]` in `schemathesis.toml`. See [[hooks-examples]] for the environment-variable form.
   - Generate pytest reproducers with `--report=pytest` and adapt their assertions into permanent regression tests.
   - Improve schema based on real failures.
   - Before drafting a new multi-operation scenario, require OpenAPI, a test objective, and a readable technical design. Check the workspace and supplied materials first. Reuse a sufficient design or ask for one if missing or unreadable. Resolve missing rules and conflicts with the user, and confirm the YAML draft before generating pytest tests. See [[scenario-patterns]] for the design checklist.

5. **CI dual-track**
   - PR: shallow run (`max-examples=50`).
   - Nightly/main: deep run (`max-examples=300+`, stateful enabled).
   - Pin Schemathesis and upload reports in both workflows. See [[ci-examples]].

## Key Files to Produce

| File | Purpose | Who generates it |
|------|---------|------------------|
| `schemathesis.toml` | Auth, rate limits, phases, filters | Agent or user |
| `hooks.py` | Custom checks and hooks | Local agent |
| `tests/test_regression.py` | Fixed cases from failures | Local agent |
| `.github/workflows/api-test-*.yml` | CI pipelines | Agent |
| `AGENTS.md` / `CLAUDE.md` | Project rules for the agent | Agent |

## Common Commands

```bash
# Quick smoke
uvx schemathesis@4.34.3 run openapi.yaml --url http://localhost:8080 --max-examples=50

# Load configured hooks and write HTML and pytest reports
uvx schemathesis@4.34.3 --config-file schemathesis.toml run openapi.yaml \
  --url http://localhost:8080 --report=html,pytest \
  --report-html-path=schemathesis-report/html \
  --report-pytest-path=schemathesis-report/test_regression.py

# Set API_TOKEN to the token used for the original run.
# Replace <schema-key> and <crash-file> with names from the crash directory.
uvx schemathesis@4.34.3 replay --keep ".schemathesis/default/cache/crashes/<schema-key>/<crash-file>.json" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

Open `schemathesis-report/html/index.html` and keep the report directory together. The pytest file reproduces observed failures. Restore redacted auth from your test environment and add business-specific assertions where needed before keeping it as a regression test.

`replay` accepts a saved crash file or a crash directory. The crash file stores the Authorization value as `[Filtered]`, so pass the original credentials again when replaying an authenticated request. If the JSON has been moved away from its crash directory and `manifest.json`, pass `--schema-location openapi.yaml` and set `--url` to the original API base URL. In 4.34.3 it also resolves a test case ID when a corresponding crash file is available in the cache. For failures without a saved crash, use the printed curl command or the generated pytest reproducer.

## Local verification with 4.34.3

The commands below use a local OpenAPI fixture. `"$BASE_URL"`, `"$CRASH_FILE"`, and `"$CASE_ID"` stand for values from that run.

| Command | Observed output |
|---------|-----------------|
| `uvx schemathesis@4.34.3 --help` | Listed `--config-file PATH` and the `replay` command. |
| `uvx schemathesis@4.34.3 replay --help` | `Usage: schemathesis replay [OPTIONS] [PATH]` |
| `uvx schemathesis@4.34.3 replay --keep "$CRASH_FILE" --url "$BASE_URL" -H "Authorization: Bearer local-demo"` | `Replaying 1 case from` the saved JSON file and `ST103 Undocumented status code` for the fixture's HTTP 201. |
| `uvx schemathesis@4.34.3 replay --keep "$CASE_ID" --url "$BASE_URL" -H "Authorization: Bearer local-demo"` | Replayed the same saved case by its test case ID. |
| `uvx schemathesis@4.34.3 run --help` | Listed all four phases, `--report-html-path DIRECTORY`, and `--report-pytest-path FILENAME`. |
| `uvx schemathesis@4.34.3 run openapi.json --url "$BASE_URL" -H "Authorization: Bearer local-demo" --mode=positive --max-examples=2 --phases=examples,coverage,fuzzing,stateful` | Examples, Coverage, and Fuzzing passed. The fixture had no links, so output reported `Stateful (not applicable)`. |

## When to Read More

- For multi-operation scenario drafts and pytest examples → see [[scenario-patterns]]
- For custom check patterns and hook examples → see [[hooks-examples]]
- For CI templates → see [[ci-examples]]
- For ready-to-use agent prompts → see [[agent-prompts]]
- For case generation volume recommendations → see [[case-volume]]
- For post-run test analysis report template → see [[test-report]]
- Full knowledge graph → `references/INDEX.md`

## Knowledge Graph

Start at `references/INDEX.md` for the full map.

## Success Criteria

- Schema is reviewed before the first deep run.
- At least one custom check exists for a business rule the schema cannot express.
- Failures produce both a curl repro and a regression test.
- PR pipeline stays under 5 minutes; nightly can run longer.
