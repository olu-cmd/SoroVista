# Pull Request: Per-contract ingestion cursors, pool config, and CLI validate

## Summary

This PR implements three issues from the SoroVista backlog and defers a fourth:

- **Closes #55**: Per-contract ingestion cursors
- **Closes #46**: Make the database connection pool configurable
- **Closes #45**: CLI validate configuration without starting the service
- **Deferred**: #39 SQLite backend for single-binary deployments

## Changes

### feat(ingest): per-contract ingestion cursors

- **Store interface** (`internal/store/store.go`): Added `ContractIngestState` struct and `GetContractIngestState`/`SaveContractIngestState` methods to the `Store` interface
- **Postgres store** (`internal/store/postgres.go`): Implemented per-contract state tracking with a new `contract_ingest_state` table using UpsertEvents-style idempotency (ON CONFLICT DO NOTHING)
- **Migration** `0002_contract_ingest_state.up/down.sql`: New migration creating the per-contract cursor table with `contract_id TEXT PRIMARY KEY`, `last_ledger BIGINT`, `last_cursor TEXT`, `updated_at TIMESTAMPTZ`
- **Ingester** (`internal/ingest/ingest.go`): 
  - Added `PerContractResumers` option to `Options` struct
  - Modified `resumeLedger` to compute per-contract resume ledgers from store state
  - Modified `pollOnce` to save per-contract ingestion state after each poll
  - Added `SeedContractCursors()` function called at startup to seed per-contract cursors from the global ingest state, so upgrades do not re-read the retention window
- **Main binary** (`cmd/sorovista/main.go`): Wired `PerContractResumers` from store into the ingester and added startup seeding

**Impact**: Operators can now add contracts to `WATCHED_CONTRACTS` and have their full historical events backfilled from the retention window, rather than only seeing events emitted after the contract was added. Per-contract failures no longer block other contracts. Restart resumption is preserved from stored positions.

### feat(store): make the database connection pool configurable

- **Config** (`internal/config/config.go`): Added `PoolMaxOpenConns`, `PoolMaxIdleConns`, `PoolMaxConnLifetime`, `PoolMaxConnIdleTime` fields to `Config` with default values (0 = use pgxpool defaults). Added env var parsing for `POOL_MAX_OPEN_CONNS`, `POOL_MAX_IDLE_CONNS`, `POOL_MAX_CONN_LIFETIME`, `POOL_MAX_CONN_IDLE_TIME`.
- **Validation** (`internal/config/config.go`): Added `ValidatePoolSettings()` method that validates pool settings at config time (e.g., idle must not exceed max).
- **Postgres store** (`internal/store/postgres.go`): Modified `NewPostgres` to accept `*config.Config` and apply pool settings via `pgxpool.ParseConfig`/`NewWithConfig` instead of raw `pgxpool.New`. This ensures zero values fall back to pgxpool's defaults rather than overriding them with 0.
- **Main binary** (`cmd/sorovista/main.go`): Passes `&cfg` to `store.NewPostgres`.

**Impact**: Operators can now tune the pgx connection pool size relative to their Postgres `max_connections` setting, preventing pool starvation under read-heavy load or connection limit exhaustion. Defaults maintain backward compatibility.

### feat(cli): validate configuration without starting the service

- **`validate` subcommand** (`cmd/sorovista/main.go`): Added a new subcommand that:
  - Loads and validates configuration from environment variables
  - Reports **all** validation errors (not just the first), naming both the missing variable and the mode that requires it
  - Prints the resolved configuration (with `DATABASE_URL` password redacted)
  - Supports an optional connectivity check via `VALIDATE_CHECK_CONNECTIVITY=1` env var for RPC passphrase verification
  - Preserves default behavior: the binary still starts the server when no arguments are given

**Impact**: Operators can validate their configuration in CI/CD pipelines or before deployment without mutating the database or starting a server instance.

### #39: SQLite backend — Deferred

- This issue was deemed out of scope for this PR due to its complexity (marked 200 points, "Complex"). It requires:
  - A cgo-free SQLite driver to keep `CGO_ENABLED=0` builds working
  - Migration set at exact filename parity with Postgres migrations
  - WAL mode configuration for concurrent read/write safety
  - Topic filter design decision (SQLite has no jsonb containment operator)
  - Conformance tests running the same test body against both backends
  - Documentation updates
- **Decision**: Create a separate issue/PR for the SQLite backend work.

## Testing

All existing tests pass:
- `go test ./...` — all packages green
- `go vet ./...` — no issues
- `go build ./...` — compiles successfully

**New behavior tested**:
- `sorovista validate` reports all validation errors and exits 1
- `sorovista validate` prints resolved config with redacted DB URL and exits 0
- `sorovista` (no args) starts the server as before
- Per-contract cursor seeding from global state at startup
- Pool settings applied via `POOL_MAX_OPEN_CONNS` etc.

## Review Checklist

- [x] Build passes: `go build ./...`
- [x] Vet passes: `go vet ./...`
- [x] Tests pass: `go test ./...`
- [x] Issue #55: Per-contract cursors implemented and tested
- [x] Issue #46: Pool config implemented and tested
- [x] Issue #45: CLI validate subcommand implemented and tested
- [x] SQLite backend deferred (not in this PR)
- [x] Backward compatible: defaults maintain existing behavior
- [x] Migration 0002 tested against fresh database

## Review Notes

- The per-contract cursor seeding logic at startup is the most delicate part: it only seeds when the global position exists and no per-contract state exists yet, ensuring upgrades don't lose position but also don't re-read the retention window.
- Pool settings use `pgxpool.ParseConfig`/`NewWithConfig` to properly merge operator-provided settings with pgxpool defaults.
- The `validate` subcommand collects *all* errors before reporting, consistent with the rest of SoroVista's config validation pattern.
- Test mocks were updated to satisfy the new `Store` interface methods (`GetContractIngestState`, `SaveContractIngestState`).