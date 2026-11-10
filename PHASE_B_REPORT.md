# Phase B — Tool System and Derived Artifacts Report

## Initial State
- Branch: `codex/phase-b-tool-system`.
- Starting commit: `b1f2af0a248a48744d4f2179b42b97d36c41a3e0`.
- Starting tag: `v0.1.1-stabilized-core`.
- Database path: `D:\NEW\backend\data\ai_manager.db`, at Alembic revision `0002_normalize_request_state`.
- Database backup: Phase B backup is created immediately before the B2 migration; the Phase A backup remains available and valid.
- Existing tool flow: `Executor` contained four tool-name branches and called handlers from registry definitions directly.
- Existing artifact support: configuration exposed an artifact root, but there was no artifact model/service/content endpoint; final results always returned an empty artifact list.

## B1 — Generic Tool Gateway and Executor
### Status
PASS
### Changes
- Introduced separate tool, provider, handler, execution-context, asset-access, and artifact-creation contracts.
- Added `ToolGateway` as the sole handler invocation point and normalized gateway failures through the Phase A error envelope.
- Added schema-driven `InputBinding` for request assets, direct inputs, previous task outputs, and previous task artifact references.
- Added capability routing while retaining `tool_id` compatibility; providers now have explicit version, execution kind, enabled, health, and locality metadata.
- Built-in tools/providers are explicitly enabled; registration defaults remain disabled/restricted.
### Removed Coupling
- Removed all `image.compare`, `audio.inspect`, `data.statistics`, and `report.compose` branches from `Executor`.
- `Executor` now performs only DAG ordering, readiness/cancellation/state handling, generic input binding, gateway invocation, and result persistence.
### Tool Routing and Provider Resolution
- `ToolRouter.resolve` accepts a tool ID, a capability, or both; it validates capability compatibility and resolves an enabled healthy provider.
- Resolved provider identity is recorded with every tool call input under routing metadata without changing the production schema in B1.
### Input Binding
- Structured references support `request_asset`, `task_output`, and `task_artifact`, including dotted output paths.
- Legacy plans containing `asset_ids` or an empty single-dependency report input are bound generically using the target input schema.
### Policy and Validation
- Gateway order is resolve tool/provider, validate input, resolve asset IDs, check policy, create tool call, invoke handler, validate output schema/contract, and persist result.
- Unknown, disabled, policy-denied, and invalid-input requests cannot invoke handlers.
### Tests
- Backend suite: 11 passed.
- Added focused tests for a fifth tool without Executor changes, disabled/unknown/denied tools, generic output binding, topological order, and cycle rejection.
- Isolated reference demo reached `COMPLETED` with `image.compare` followed by `report.compose`.
### Problems and Fixes
- The first branch-selection PowerShell condition relied on command output rather than exit status and attempted to create an existing branch. No state changed; the existing branch was selected explicitly and was already based on the required tag.
- The pre-Phase-B compatibility test expected the former registry exception text. Registry behavior retained that public compatibility while the gateway maps it to `TOOL_NOT_FOUND`.
### Commit
`2a6316c` — `feat: add generic tool gateway and executor`.

## B2 — Derived Assets and Provenance
### Status
PASS
### Changes
- Added an `Artifact` model, controlled artifact service, bound handler-facing creator, secure content endpoint, result aggregation, and compact Workspace artifact display.
- `report.compose` keeps its text result and additionally creates a Markdown report artifact labeled Mock.
### Database Migration
- Created `0003_add_artifacts` with artifact identity, request/plan/task/tool-call linkage, multiple source asset IDs, storage metadata, hashes, parameters, tool/provider versions, execution kind, and timestamps.
- Created `0004_align_artifact_asset_index` after `alembic check` detected that the named asset index needed to be unique to match the ORM model.
- Tested upgrade on a production backup copy, downgrade to `0002`, re-upgrade, then tested the index migration downgrade/re-upgrade. Integrity and all original counts remained valid.
- Verified the Phase B backup before applying both migrations to production. Production reached `0004_align_artifact_asset_index` with no missing or autogenerate operations.
### Artifact Storage
- Writes use a temporary file in the configured artifact root, flush and `fsync`, SHA-256 calculation, atomic replace, then a database transaction.
- Failed database registration rolls back and removes temporary/final orphan files.
- Artifact lookup and opening use Asset IDs and a resolved-root containment check; handlers receive only a limited bound creator.
### Provenance and Lineage
- Persisted request, plan, task, tool call, tool/version, provider/version, execution kind, parameters, source asset IDs, content hash, and timestamp.
- The report artifact records both source image Asset IDs through generic upstream-result provenance.
### API and UI
- Added `GET /api/v1/assets/{asset_id}/content` using database lookup and streamed `FileResponse`, safe download names, stored media type, and no storage-path disclosure.
- Unknown IDs and encoded traversal attempts return 404.
- Added typed artifact references and a minimal role/type/size/Mock-or-Real/open-download list to the existing Workspace without redesign.
### Tests
- 11 backend tests passed after B2.
- The focused E2E test verifies artifact creation in temporary storage, SHA-256/size/type, multiple sources, tool/provider/task provenance, content retrieval, missing/traversal rejection, final-result exposure, and production-storage isolation.
- Fresh and populated-copy migration tests include the artifact schema and preserve original rows.
- Frontend production build passed.
### Problems and Fixes
- A build command was initially invoked from `backend` and failed because no `package.json` exists there. It was rerun from `frontend` and passed; no product defect was involved.
- `alembic check` found a uniqueness mismatch on the named `ix_artifacts_asset_id` index after the first production-safe migration. A forward corrective migration was tested on the trial copy, applied to production, and `alembic check` then passed.
### Commit
`cf694bc` — `feat: add derived artifacts and provenance`.

## Final Validation
### Backend Tests
- `uv sync` passed. `uv run pytest -q` ran twice consecutively after all changes: 11 passed each time with one known non-failing upstream Starlette/httpx warning.
### Frontend Build
- `npm install` reported zero vulnerabilities; `npm run build` passed with 16 transformed modules.
### End-to-End Demo
- `uv run python scripts/demo_e2e.py --isolated` exited 0 without traceback or cleanup error. The request reached `COMPLETED`, ran `image.compare` then `report.compose`, and returned a persisted Mock report artifact reference.
### Database Verification
- Phase B backup: `D:\NEW\backend\data\backups\ai_manager-before-phase-b-20260813-023625.db`; 155648 bytes; SHA-256 `916BD1A9556B2575DA7270621782866EE9BE99ECD4E5B7A370B7941498ECC473`; integrity `ok`.
- Production is at `0004_align_artifact_asset_index`; integrity `ok`; `alembic check` reports no new operations.
- Original counts remain requests 10, assets 20, plans 10, tasks 20, tool calls 20, experiences 10, settings 2. The new artifacts table contains zero production rows because validation used isolated storage.
- Production asset inventory remains 21 files, 309178 bytes, SHA-256 `644F557B5B87D5EC05670F0335B23066EC59299C917BC3829E96C62FF098BFFF`. Production artifact storage still contains only its pre-existing `.gitkeep`.
### Git Status
- No handler invocation exists outside `ToolGateway`, and no tool-name branch exists in `Executor`.
- Runtime databases, backups, original assets, generated artifacts, dependencies, caches, and builds remain ignored and untracked.
- B1 commit `2a6316c`; B2 commit `cf694bc`; final annotated tag `v0.1.2-tool-system` is created on the final report/build commit.

## Known Limitations
- Provider selection is deterministic first-match rather than scored; advanced health polling and provider persistence remain deferred.
- Execution stays local, sequential, and in-process. Cancellation is checked between tasks and exposed to handlers, without retries or resume.
- Artifact retention/garbage collection and preview generation are not included.
- The non-failing Starlette/httpx deprecation warning remains upstream.

## Deferred Items
OpenCV, real media/data analysis, Ollama changes, retries/resume, queues/workers, external plugin SDKs, and advanced provider scoring remain out of scope.

## Final Status
PASS
