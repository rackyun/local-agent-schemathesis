# Changelog

## 2026-11-28
- Expanded the order scenario example from one to four cases: normal order lookup, quantity at the upper limit, invalid quantity rejection, and lookup after deletion. YAML, pytest, and reports share the same case IDs.

## 2026-11-27
- Initial skill creation for Local Agent + Schemathesis self-hosted API testing workflow.
- Optimized directory structure: added `references/` with INDEX.md, hooks-examples.md, and ci-examples.md.
- Added ready-to-use agent prompts in `references/agent-prompts.md`.
- Added case generation volume recommendations in `references/case-volume.md`.
- Added required post-run test analysis report template in `references/test-report.md`.
- Clarified that max-examples is per operation (not total for the API) in case-volume.md.
