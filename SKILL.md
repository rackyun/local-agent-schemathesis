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
   - Add `schemathesis.toml` for auth, phases, and per-operation overrides.
   - Enable stateful and fuzzing phases for deeper coverage.

4. **Enhance with local agent**
   - Generate custom `@schemathesis.check` functions for business rules (user isolation, audit headers, etc.).
   - Place them in `hooks.py` and load with `SCHEMATHESIS_HOOKS=hooks`.
   - Convert failures into permanent regression tests.
   - Improve schema based on real failures.

5. **CI dual-track**
   - PR: shallow run (`max-examples=50`).
   - Nightly/main: deep run (`max-examples=300+`, stateful enabled).
   - Use official GitHub Action `schemathesis/action@v3` or Docker.

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
uvx schemathesis run openapi.yaml --url http://localhost:8080 --max-examples=50

# With hooks + HTML report
export SCHEMATHESIS_HOOKS=hooks
schemathesis run openapi.yaml --url http://localhost:8080 --report=html

# Replay a failure
schemathesis replay <failure-id>
```

## When to Read More

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
