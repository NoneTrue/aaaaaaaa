# Phase A — Core Stabilization Report

## Initial State
- Repository status: Git was not initialized. The application source, four critical tests, generated frontend files, one production SQLite database, and uploaded runtime assets were present.
- Production database path: `D:\NEW\backend\data\ai_manager.db`
- Production database backup: `D:\NEW\backend\data\backups\ai_manager-before-phase-a-20260813-014854.db`; 147456 bytes; SHA-256 `97F39CD57DE74AC20EF7FAAAD1A4E308F240E1F970C904D2E5E5DF80E0AC34B6`; SQLite integrity check `ok`.
- Existing tests: Four tests in `backend/tests/test_critical.py`; they were not run before isolation because they used production storage.
- Assumptions: The 20 database Asset rows and 20 uploaded files are user/runtime data. The additional `.gitkeep` file explains the 21-file asset-directory inventory. All must be preserved and ignored.

## A1 — Git Baseline
### Status
PASS

### Changes
- Extended `.gitignore` for SQLite variants, backups, runtime assets/artifacts, logs, TypeScript build metadata, generated Vite files, environments, dependencies, builds, caches, secrets, and superseded PDF documents.
- Created a consistent SQLite backup using the SQLite backup API before migrations.
- Scanned the intended tracked source set for credential patterns and excluded runtime/generated content.

### Validation
- Production database pre-change counts: requests 10, assets 20, plans 10, tasks 20, tool calls 20, experiences 10, settings 2.
- Production asset inventory: 21 files including `.gitkeep`, 309178 bytes, inventory SHA-256 `644F557B5B87D5EC05670F0335B23066EC59299C917BC3829E96C62FF098BFFF`.
- Sensitive scan found no credential or private-key value; the only match was the local variable name `token` used for UUID generation.
- Backup record counts equal the production counts and `PRAGMA integrity_check` returned `ok`.

### Commit and Tag
- Baseline commit: `6ecb9d53611284a240389b6ad350a9f77685b904` (`chore: establish v0.1 backbone baseline`).
- Baseline tag: annotated tag `v0.1-backbone`.
- Working branch: `codex/phase-a-stabilization`.
- Git identity was absent; repository-local identity only was configured as `Codex Local <codex-local@localhost>`.

### Problems and Fixes
- PowerShell 5.1 lacks `Convert.ToHexString`; asset inventory hashing was rerun using `BitConverter`, producing the recorded SHA-256. The backup itself was created successfully and independently verified.

## A2 — Test Isolation
### Status
PASS
### Changes
- Added injectable `asset_root` and `artifact_root` settings alongside the existing database URL.
- Updated database directory initialization and `AssetManager` to use configured roots rather than fixed production paths.
- Added `backend/tests/conftest.py` with a temporary file-backed SQLite database and temporary asset/artifact directories. Database tables and file roots reset before every test.
- Added `--isolated` to `backend/scripts/demo_e2e.py`; it configures temporary database and storage paths before importing the application.
### Validation
- Ran `uv run pytest -q` twice consecutively: 4 passed on each run.
- Ran `uv run python scripts/demo_e2e.py --isolated`: exit 0; request reached `COMPLETED` using `image.compare` then `report.compose`.
- API request and response behavior remained unchanged.
### Production Data Verification
- Before and after both test runs: production DB SHA-256 remained `00438845D512C5236344B7582408E6A1F361EFFA5AF866CCEEF8274213668FD8`.
- Counts remained requests 10, assets 20, plans 10, tasks 20, tool calls 20, experiences 10, settings 2.
- Production asset inventory remained 21 files and SHA-256 `644F557B5B87D5EC05670F0335B23066EC59299C917BC3829E96C62FF098BFFF`.
- The isolated demo also left the production database hash unchanged.
### Problems and Fixes
- The first isolated demo completed successfully but Windows reported a temporary-file cleanup `PermissionError` because SQLite still held a file handle. The demo now disposes the SQLAlchemy engine before removing its temporary directory; the rerun exited cleanly without a traceback.
- The existing upstream Starlette/httpx deprecation warning remains non-failing and out of scope.

## A3 — Contracts, States and Migrations
### Status
PASS
### Changes
- Added versioned Pydantic contracts for intent, plan, task, tool result, final result, and structured errors. Tool and final result contracts explicitly distinguish `execution_kind: mock|real`; legacy response fields and `mock: true` remain present for mock compatibility.
- Added centralized canonical request/task states, the `VALIDATING_PLAN` compatibility alias, legal transition maps, and transition guards used by orchestration, execution, and request management.
- Updated the intent analyzer, planner, verifier, result composer, and four mock tools to emit or consume the versioned contracts. The verifier and composer now accept valid real-tool output without requiring `mock: true`.
- Added Alembic as the schema authority; application initialization now upgrades to the current revision rather than calling `metadata.create_all()`.
### Migrations
- `0001_backbone_baseline`: creates the seven domain tables on a fresh database; adopts only an exact recognized legacy schema without recreating it; rejects partial or unknown schemas.
- `0002_normalize_request_state`: changes stored `VALIDATING_PLAN` values to canonical `PLAN_VALIDATION`; its downgrade restores the legacy literal.
- Upgraded an isolated fresh database, a copied pre-Alembic populated database, a disposable copy of the production backup, and finally the production database after backup verification.
- The disposable production-copy trial successfully upgraded to head, downgraded to revision `0001`, and upgraded to head again with all domain counts preserved.
### Validation
- Contract tests accept complete mock and real outputs, reject incomplete successful/failed outputs, and prove the verifier/composer accept a direct real result.
- State tests reject illegal transitions and normalize the legacy alias.
- Migration tests verify fresh upgrade/downgrade and populated legacy-database preservation with state normalization.
- `uv run alembic current` reports `0002_normalize_request_state (head)`; `uv run alembic check` reports no new upgrade operations.
### Compatibility
- Existing API v1 fields and the deterministic mock workflow are preserved. New metadata is additive.
- Production domain-row counts are unchanged. The only intentional production mutation is Alembic metadata plus canonicalization of any legacy state literal; the production database contained zero legacy literals after upgrade.
- No UI behavior, provider integration, authentication, queue, or out-of-scope feature was added.
### Problems and Fixes
- The first populated-copy test inherited the source database's `alembic_version`, so no migration was expected to run. The fixture was corrected to model a genuine pre-Alembic database by removing that metadata before upgrade.
- One PowerShell verification command interpreted `*` before SQLite received it; the count query was rerun safely as a read-only Python/SQLite check.
- A3 commit: `7d286dc` (`feat: stabilize core contracts states and migrations`).

## Final Validation
### Backend Tests
- `uv run pytest -q` ran twice consecutively after all A3 changes: 8 passed on each run. The one warning is the known upstream Starlette/httpx deprecation warning.
- The health endpoint returned HTTP 200 with `healthy` status under isolated configuration.
### Frontend Build
- `npm run build` passed; TypeScript and Vite completed and transformed 16 modules.
### End-to-End Demo
- `uv run python scripts/demo_e2e.py --isolated` exited 0 without cleanup errors. The request reached `COMPLETED`; `image.compare` and `report.compose` both reached `SUCCEEDED`; the versioned final result retained `mock: true`.
### Database Verification
- Backup remains present, passes integrity check, and retains SHA-256 `97F39CD57DE74AC20EF7FAAAD1A4E308F240E1F970C904D2E5E5DF80E0AC34B6`.
- Production is at `0002_normalize_request_state`, passes integrity check, and has zero `VALIDATING_PLAN` rows.
- Final production counts remain requests 10, assets 20, plans 10, tasks 20, tool calls 20, experiences 10, settings 2. Its post-migration SHA-256 is `6D3E917B00522E64C311D2C528A9951D55FF5068E464537E8E982D94CC6BA4DF`.
- Production asset inventory remains 21 files, 309178 bytes, and SHA-256 `644F557B5B87D5EC05670F0335B23066EC59299C917BC3829E96C62FF098BFFF` using the same filename/size/content-digest manifest algorithm recorded before A1.
### Git Status
- A1 commit/tag: `6ecb9d5`, annotated `v0.1-backbone`.
- A2 commit: `d8219b6` (`test: isolate database and asset storage`).
- A3 commit: `7d286dc` (`feat: stabilize core contracts states and migrations`).
- Runtime databases, backups, uploaded assets, environments, caches, dependencies, and frontend build output remain ignored and untracked.

## Decisions
- Use a temporary file-backed SQLite database and temporary asset/artifact roots for tests and demo; do not use SQLite `:memory:`.
- Preserve API v1 response fields and add contract metadata without removing existing fields.
- Keep migrations minimal: establish the current schema as baseline, then migrate the legacy request-state literal safely.

## Known Limitations
- Error envelopes are defined and validated at the contract layer but are not yet a universal replacement for every existing FastAPI HTTP error response.
- Transition guards cover the current synchronous MVP path; there is no external queue or distributed workflow engine by design.
- The baseline migration safely adopts only the exact known seven-table legacy schema; an unknown or partially created schema is intentionally rejected for manual investigation.
- The Starlette/httpx compatibility warning is upstream and non-failing.

## Final Status
PASS
