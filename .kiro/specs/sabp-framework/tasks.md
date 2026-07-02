# Implementation Plan: SABP Framework — Stateful Anchor-Based Pagination

## Overview

This plan implements the SABP Framework as a TypeScript NPM library. Tasks follow dependency order: foundational types and utilities first, then pure-logic core components, then I/O adapters, then the orchestrating Paginator, then the benchmark module, and finally the package entry point. Property-based tests (fast-check) are sub-tasks placed immediately after the component they validate, and integration tests follow as a dedicated group.

---

## Tasks

- [ ] 1. Project scaffold and TypeScript configuration
  - Initialize `package.json` with `name: "sabp-framework"`, TypeScript, Jest, fast-check, and MongoDB driver dev-dependencies
  - Create `tsconfig.json` targeting ES2020, `strict: true`, `outDir: dist`, `rootDir: src`, `declaration: true`
  - Create `jest.config.ts` with `ts-jest` preset, `testEnvironment: node`, and separate `unit` / `integration` test path groups
  - Create directory skeleton: `src/core/`, `src/adapters/`, `src/benchmark/`, `src/types/`, `src/utils/`, `tests/unit/`, `tests/integration/`, `tests/fixtures/`
  - _Requirements: 11.1, 11.2, 11.3_

- [ ] 2. Core TypeScript types (`src/types/index.ts`)
  - Define and export all interfaces: `AnchorPayloadV1`, `PaginationRequest`, `DeepNavigationRequest`, `PaginationResponse<T>`, `PaginatorConfig`, `SABPConfig`
  - Define and export: `Checkpoint`, `CheckpointLookupResult`, `Session`, `StorageAdapter`
  - Define and export: `MongoQuery`, `ForwardQueryParams`, `BackwardQueryParams`, `DeepQueryParams`
  - Define and export: `BenchmarkConfig`, `BenchmarkMetrics`, `BenchmarkResult`
  - Define and export: `SABPError` with `error: true`, `code`, `message`, `component`, `operation` fields
  - Define string literal union for all error codes from the design (`ANCHOR_DECODE_FAILED`, `SESSION_NOT_FOUND`, etc.)
  - _Requirements: 11.1, 11.2_

- [ ] 3. Logger utility (`src/utils/logger.ts`)
  - Implement `Logger` class with `log(level, context)` method accepting `{ component, operation, message }`
  - Support log levels: `silent`, `error`, `info`, `debug` driven by `SABPConfig.logLevel`
  - Ensure no sensitive data (connection strings, passwords, raw documents) is ever included in output
  - Export a `createLogger(level)` factory function
  - _Requirements: 12.4_

- [ ] 4. AnchorEngine — encode, decode, validate, reEncode (`src/core/anchor-engine.ts`)
  - Implement `encode(payload: AnchorPayloadV1): string` using `Buffer.from(JSON.stringify(payload)).toString('base64')`
  - Implement `decode(token: string): AnchorPayloadV1 | SABPError` — catches Base64/JSON parse errors and returns `ANCHOR_DECODE_FAILED`
  - Implement `validate(token: string, activeCollection: string): true | SABPError` — checks version === 1 (`ANCHOR_VERSION_MISMATCH`) and collection match (`ANCHOR_COLLECTION_MISMATCH`)
  - Implement `reEncode(payload: AnchorPayloadV1): string` as a convenience wrapper over `encode`
  - No database I/O; no side effects
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7_

  - [ ]* 4.1 Write property test: Anchor Round-Trip (Property 1)
    - **Property 1: Anchor Round-Trip** — for any valid `AnchorPayloadV1`, `encode → decode → re-encode` produces a token equal to the original encoding and a payload structurally equal to the input
    - Use fast-check `fc.record` arbitrary for `AnchorPayloadV1` with random hex `documentId`/`snapshotBoundary`, ISO date `sortValue`, alphanumeric `collection`
    - **Validates: Requirements 1.1, 1.2, 1.3, 1.4**

  - [ ]* 4.2 Write property test: Invalid Anchor Handling (Property 2)
    - **Property 2: Invalid Anchor Handling** — for any string that is not a valid SABP anchor token, `decode()` and `validate()` SHALL return `SABPError` and SHALL NOT throw
    - Use `fc.string()`, `fc.base64String()`, truncated/corrupted variants as arbitraries
    - **Validates: Requirements 1.5, 12.3**

  - [ ]* 4.3 Write property test: Collection Mismatch Rejection (Property 3)
    - **Property 3: Collection Mismatch Rejection** — for any valid token whose `collection` ≠ `activeCollection`, `validate()` SHALL return `ANCHOR_COLLECTION_MISMATCH` error
    - **Validates: Requirements 1.6**

- [ ] 5. QueryBuilder — pure MongoDB query constructors (`src/core/query-builder.ts`)
  - Implement `buildInitialQuery(limit: number): MongoQuery` returning `{ filter: {}, sort: { createdAt: -1, _id: -1 }, limit }`
  - Implement `buildForwardQuery(params: ForwardQueryParams): MongoQuery` returning filter `{ createdAt: { $lt: sortValue }, _id: { $lte: snapshotBoundary } }` with `limit`
  - Implement `buildBackwardQuery(params: BackwardQueryParams): MongoQuery` — when `limit > 0` return filter `{ createdAt: { $gt: sortValue } }` with `limit`; when `limit === 0` return empty result immediately
  - Implement `buildDeepQuery(params: DeepQueryParams): MongoQuery` returning filter anchored to checkpoint doc with `skip: relativeOffset` and `limit`
  - No database I/O; all functions are pure
  - _Requirements: 2.2, 3.1, 4.1, 4.2, 5.3_

  - [ ]* 5.1 Write property test: Initial Query Builder Correctness (Property 4)
    - **Property 4: Initial Query Builder Correctness** — for any positive integer `limit`, `buildInitialQuery(limit)` sort SHALL equal `{ createdAt: -1, _id: -1 }` and `limit` SHALL equal the input
    - Use `fc.integer({ min: 1 })` arbitrary
    - **Validates: Requirements 2.2**

  - [ ]* 5.2 Write property test: Forward Query Filter Correctness (Property 8)
    - **Property 8: Forward Query Filter Correctness** — for any `ForwardQueryParams`, result filter SHALL contain `{ createdAt: { $lt: sortValue }, _id: { $lte: snapshotBoundary } }` and `limit` SHALL equal the input `limit`
    - **Validates: Requirements 3.1**

  - [ ]* 5.3 Write property test: Backward Query Filter Correctness (Property 11)
    - **Property 11: Backward Query Filter Correctness** — for any `BackwardQueryParams` with `limit > 0`, result filter SHALL contain `{ createdAt: { $gt: sortValue } }` and `limit` SHALL equal the input
    - **Validates: Requirements 4.1**

  - [ ]* 5.4 Write property test: Deep Query Uses Relative Offset as Skip (Property 17)
    - **Property 17: Deep Query Uses Relative Offset as Skip** — for any `DeepQueryParams`, result `skip` SHALL equal `relativeOffset` and filter SHALL be anchored to the checkpoint document
    - **Validates: Requirements 5.3**

- [ ] 6. MongoDBAdapter (`src/adapters/mongodb-adapter.ts`)
  - Implement `StorageAdapter` interface plus MongoDB-specific `find(collection, query)`, `findOne(collection, query)`, `insertOne(collection, doc)`, `updateOne(collection, filter, update)`, `deleteOne(collection, filter)` methods
  - Accept a MongoDB native `Db` instance (or Mongoose connection) in the constructor
  - Implement collection access helpers for `sabp_checkpoints` and `sabp_sessions` with their required indexes
  - Expose TTL index creation (`expiresAt: 1`) and compound index (`collectionName: 1, position: 1`) helpers
  - Wrap all driver calls in try/catch; return `SABPError` on failure, never throw
  - _Requirements: 6.2, 6.4, 9.4, 9.7_

- [ ] 7. SnapshotManager (`src/core/snapshot-manager.ts`)
  - Implement `captureSnapshot(collection: string): Promise<string | SABPError>` — executes `find().sort({ _id: -1 }).limit(1).project({ _id: 1 })` via `MongoDBAdapter` and returns the `_id` hex string
  - Implement `enforceSnapshot(query: MongoQuery, snapshotBoundary: string): MongoQuery` — merges `{ _id: { $lte: snapshotBoundary } }` into the existing filter; pure function, no I/O
  - Return `SNAPSHOT_CAPTURE_FAILED` error if the adapter call fails
  - _Requirements: 2.1, 2.3, 7.1, 7.2, 7.3_

  - [ ]* 7.1 Write property test: Snapshot Capture and Embedding (Property 5)
    - **Property 5: Snapshot Capture and Embedding** — for any non-empty mocked collection, `captureSnapshot` returns the latest `_id`, and `enforceSnapshot` merges `$lte` constraint correctly
    - Mock `MongoDBAdapter.find` to return controlled document arrays
    - **Validates: Requirements 2.1, 2.3, 7.1**

  - [ ]* 7.2 Write property test: Snapshot Boundary Exclusion (Property 9)
    - **Property 9: Snapshot Boundary Exclusion** — for any document whose `_id` exceeds the `snapshotBoundary`, it SHALL never appear in navigation results
    - Use mocked adapter returning documents above and below the boundary; verify filter excludes all above-boundary docs
    - **Validates: Requirements 3.2, 7.2, 7.3**

- [ ] 8. CheckpointManager (`src/core/checkpoint-manager.ts`)
  - Implement `generateCheckpoints(collection: string, interval?: number): Promise<void>` — iterate the collection in `interval`-step batches, saving one `Checkpoint` doc per batch to `sabp_checkpoints` via `MongoDBAdapter`
  - Implement `findNearest(collection: string, position: number): Promise<CheckpointLookupResult | SABPError>` — executes `findOne({ collectionName, position: { $lte: target } }).sort({ position: -1 })` via `MongoDBAdapter`
  - Return `{ found: false }` (not an error) when no checkpoint exists at or before `position`
  - Return `CHECKPOINT_CORRUPTED` if a retrieved document is missing required fields; log the anomaly and fall back to last valid checkpoint
  - _Requirements: 5.1, 5.7, 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

  - [ ]* 8.1 Write property test: Nearest Checkpoint Lookup Correctness (Property 15)
    - **Property 15: Nearest Checkpoint Lookup Correctness** — for any set of checkpoint records and any target position `p`, `findNearest` returns the checkpoint with the highest position ≤ `p`, or `{ found: false }` if none exists
    - Mock `MongoDBAdapter` to return controlled checkpoint arrays; use `fc.array` of checkpoint objects with `fc.integer` positions
    - **Validates: Requirements 5.1, 6.5**

  - [ ]* 8.2 Write property test: Relative Offset Invariant (Property 16)
    - **Property 16: Relative Offset Invariant** — for any valid deep navigation request, `targetPosition - checkpoint.position` SHALL satisfy `0 ≤ relativeOffset < interval`
    - Generate random `(position, checkpointPosition)` pairs where `checkpointPosition ≤ position < checkpointPosition + interval`
    - **Validates: Requirements 5.2**

  - [ ]* 8.3 Write property test: Checkpoint Storage Count and Schema Invariant (Property 18)
    - **Property 18: Checkpoint Storage Count and Schema Invariant** — for collection size `n` and interval `i`, `generateCheckpoints` writes exactly `Math.floor(n / i)` records, each with non-null `{ position, documentId, sortValue, collectionName }`
    - Mock the adapter's insert calls and count invocations
    - **Validates: Requirements 6.1, 6.2, 6.3**

- [ ] 9. Storage adapters: RedisAdapter and MemoryAdapter (`src/adapters/`)
  - Implement `redis-adapter.ts` — `StorageAdapter` wrapping `ioredis`: `get`, `set` (with Redis `EXPIRE`), `delete`; return `REDIS_CONNECTION_LOST` error on connection failure, never throw
  - Implement `memory-adapter.ts` — `StorageAdapter` using an in-process `Map<string, { value: string; expiresAt: number }>` with `setTimeout` cleanup for TTL; for dev/test only
  - Both adapters must implement the `StorageAdapter` interface from `src/types/index.ts`
  - _Requirements: 9.6, 9.8_

- [ ] 10. SessionManager (`src/core/session-manager.ts`)
  - Implement `createSession(anchor: string, snapshotBoundary: string): Promise<Session | SABPError>` — generates UUID v4 `sessionId`, builds `Session` object with `expiresAt = now + TTL`, persists via the configured `StorageAdapter`
  - Implement `getSession(sessionId: string): Promise<Session | SABPError>` — returns `SESSION_NOT_FOUND` or `SESSION_EXPIRED` as appropriate
  - Implement `updateSession(sessionId: string, newAnchor: string): Promise<Session | SABPError>` — updates `currentAnchor` and `updatedAt` in storage
  - Implement `deleteSession(sessionId: string): Promise<void>` — removes key from storage adapter
  - Accept any `StorageAdapter` implementation (MongoDB, Redis, or Memory) via constructor injection
  - In stateless mode: all session methods are no-ops that return immediately without writing state
  - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8_

  - [ ]* 10.1 Write property test: Session Lifecycle — Creation and Update (Property 21)
    - **Property 21: Session Lifecycle — Creation and Update** — for any initial `createSession` call, the persisted session SHALL have non-null `{ sessionId, currentAnchor, snapshotBoundary, expiresAt }`; after `updateSession`, `currentAnchor` SHALL match the new anchor
    - Use `MemoryAdapter` as the backing store; no mocking required
    - **Validates: Requirements 9.1, 9.2**

- [ ] 11. Paginator — orchestrator (`src/core/paginator.ts`)
  - Implement `paginate<T>(request: PaginationRequest): Promise<PaginationResponse<T> | SABPError>`:
    - **Initial** (no anchor): call `SnapshotManager.captureSnapshot`, then `QueryBuilder.buildInitialQuery`, then `MongoDBAdapter.find`, then `AnchorEngine.encode`; in stateful mode call `SessionManager.createSession`
    - **Forward**: decode anchor, call `SnapshotManager.enforceSnapshot`, build forward query, find docs, encode new anchor; update session if stateful
    - **Backward**: decode anchor, build backward query, find docs, reverse result array, encode anchor; set `hasPrevious: false` when results exhausted
  - Implement `deepNavigate<T>(request: DeepNavigationRequest): Promise<PaginationResponse<T> | SABPError>`:
    - Call `CheckpointManager.findNearest`, compute `relativeOffset`, call `QueryBuilder.buildDeepQuery`, find docs, encode anchor
    - Return `EMPTY_DATASET` if collection has zero documents; return `POSITION_OUT_OF_RANGE` if no checkpoint exists
  - In stateless mode: never call any `SessionManager` method
  - Wrap all I/O in try/catch; catch MongoDB timeout errors and return `QUERY_TIMEOUT` `SABPError`
  - Validate `SABPConfig` at initialization; throw `CONFIG_INVALID` before any DB access if config is invalid or missing required fields
  - _Requirements: 2.1–2.5, 3.1–3.5, 4.1–4.5, 5.1–5.7, 7.1–7.5, 8.1–8.4, 9.1–9.5, 11.3, 11.4, 11.5, 12.1_

  - [ ]* 11.1 Write property test: hasMore / hasPrevious Correctness (Property 6)
    - **Property 6: hasMore / hasPrevious Correctness** — when returned doc count < requested `limit`, `hasMore` SHALL be `false`; when backward navigation exhausts docs, `hasPrevious` SHALL be `false`
    - Mock adapter to return arrays of varying length relative to `limit`
    - **Validates: Requirements 2.4, 3.3, 4.4**

  - [ ]* 11.2 Write property test: Response Shape Invariant (Property 7)
    - **Property 7: Response Shape Invariant** — every `paginate()` response SHALL contain exactly `{ data, nextAnchor, previousAnchor, hasMore, hasPrevious }`
    - Generate random `PaginationRequest` inputs; assert shape for each
    - **Validates: Requirements 2.5**

  - [ ]* 11.3 Write property test: Anchor Points to Page Boundary Document (Property 10)
    - **Property 10: Anchor Points to Page Boundary Document** — decoded `nextAnchor.documentId` SHALL equal `_id` of the last doc in `data`; decoded `previousAnchor.documentId` SHALL equal `_id` of the first doc in reversed data
    - **Validates: Requirements 3.4, 4.5**

  - [ ]* 11.4 Write property test: Backward Result Ordering (Property 12)
    - **Property 12: Backward Result Ordering** — backward navigation `data` array SHALL be in descending `createdAt` order (reverse of raw MongoDB ascending result)
    - **Validates: Requirements 4.3**

  - [ ]* 11.5 Write property test: No Offset Drift Under Concurrent Deletes (Property 13)
    - **Property 13: No Offset Drift Under Concurrent Deletes** — across forward page turns during which documents are deleted, no previous-page document SHALL reappear and no document SHALL be silently skipped
    - Simulate deletes between pages by adjusting mock adapter state; verify via `_id` sets
    - **Validates: Requirements 7.4**

  - [ ]* 11.6 Write property test: No Duplicate Documents Across Page Turns (Property 14)
    - **Property 14: No Duplicate Documents Across Page Turns** — the union of all `_id` values across consecutive forward page turns SHALL contain no duplicates
    - Generate random multi-page sequences using mock adapter
    - **Validates: Requirements 7.5**

  - [ ]* 11.7 Write property test: Stateless Mode Produces No Server State (Property 19)
    - **Property 19: Stateless Mode Produces No Server State** — in stateless mode, no session write is made to any storage adapter for any `paginate()` call
    - Spy on all three adapter `set` methods; assert zero calls
    - **Validates: Requirements 8.1, 9.3**

  - [ ]* 11.8 Write property test: Stateless Mode Returns Anchor in Every Response (Property 20)
    - **Property 20: Stateless Mode Returns Anchor in Every Response** — in stateless mode with `hasMore: true`, `nextAnchor` SHALL be non-null and SHALL embed the original `snapshotBoundary`
    - **Validates: Requirements 8.3**

  - [ ]* 11.9 Write property test: Invalid Configuration Rejected Before DB Access (Property 23)
    - **Property 23: Invalid Configuration Rejected Before DB Access** — any config object missing a required field or with an invalid value SHALL cause the initializer to throw `CONFIG_INVALID` before any DB operation
    - Generate malformed configs with `fc.record` using nullable/invalid variants; assert no adapter calls
    - **Validates: Requirements 11.5**

  - [ ]* 11.10 Write property test: Query Timeout Returns SABPError Without Throwing (Property 24)
    - **Property 24: Query Timeout Returns SABPError Without Throwing** — when MongoDB raises a timeout error, `paginate()` and `deepNavigate()` SHALL return `SABPError` with `code: 'QUERY_TIMEOUT'` and SHALL NOT propagate as unhandled exception
    - Mock adapter to throw `MongoNetworkTimeoutError`; assert structured error response
    - **Validates: Requirements 1.5, 12.1**

- [ ] 12. Checkpoint — Ensure all tests pass
  - Ensure all unit tests (tasks 4–11) pass with `jest --testPathPattern=tests/unit --runInBand`
  - Fix any type errors surfaced by `tsc --noEmit`
  - Ask the user if questions arise before proceeding.

- [ ] 13. BenchmarkModule (`src/benchmark/`)
  - Implement `src/benchmark/metrics.ts` — `collectMetrics(fn: () => Promise<void>): Promise<{ cpuUsage, memoryUsage, queryExecutionTime, documentsScanned, throughput, averageLatency }>` using `process.hrtime.bigint()` and `process.cpuUsage()`
  - Implement `src/benchmark/comparison.ts` — `formatComparisonReport(results: BenchmarkMetrics[]): string` producing a side-by-side text table of all three strategies
  - Implement `src/benchmark/benchmark-runner.ts` — `BenchmarkModule.run(config: BenchmarkConfig): Promise<BenchmarkResult | SABPError>`:
    - Runs identical workloads for `skip-limit`, `cursor`, and `sabp` strategies on the same dataset
    - Records metrics per strategy; if a strategy fails, record the failure; if the failure recording itself fails, abort the entire run with `BENCHMARK_RECORD_FAILED`
    - Never returns stale results — always executes a fresh run; accessing results before `run()` returns an error or empty state
  - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7_

  - [ ]* 13.1 Write property test: Benchmark Completeness (Property 22)
    - **Property 22: Benchmark Completeness** — any completed `BenchmarkModule.run()` SHALL return metric records for all three strategies, each with non-null values for all six metric fields, plus a non-empty `comparisonReport`; accessing results without calling `run()` SHALL return error or empty state
    - Mock MongoDB calls to return controlled documents; use `fc.record` for `BenchmarkConfig` variants
    - **Validates: Requirements 10.2, 10.3, 10.4**

- [ ] 14. NPM package entry point (`src/index.ts`)
  - Re-export all public API members: `Paginator`, `AnchorEngine`, `CheckpointManager`, `SnapshotManager`, `SessionManager`, `QueryBuilder`, `BenchmarkModule`
  - Re-export all adapter classes: `MongoDBAdapter`, `RedisAdapter`, `MemoryAdapter`
  - Re-export all TypeScript interfaces and types from `src/types/index.ts`
  - Verify `tsc --noEmit` produces zero errors and zero `any` leakages in the public API surface
  - _Requirements: 11.1, 11.2_

- [ ] 15. Shared fast-check arbitraries (`tests/fixtures/generators.ts`)
  - Define and export reusable fast-check arbitraries:
    - `arbAnchorPayloadV1` — random hex `documentId`, ISO-8601 `sortValue`, hex `snapshotBoundary`, alphanumeric `collection`
    - `arbInvalidAnchorString` — arbitrary byte sequences, truncated Base64, valid JSON with wrong schema, wrong `version`
    - `arbForwardQueryParams`, `arbBackwardQueryParams`, `arbDeepQueryParams` — parameterized by the anchor arbitrary
    - `arbDocumentArray` — random arrays of `{ _id: string, createdAt: Date }` objects
    - `arbCheckpointArray` — random checkpoint sets at valid `position` values
    - `arbSABPConfig` — valid configs and intentionally malformed variants (null `db`, unsupported `adapter`, negative TTL)
  - _Requirements: All (supports 24 property tests)_

- [ ] 16. Integration tests (`tests/integration/`)
  - [ ] 16.1 Implement `pagination-lifecycle.test.ts`
    - Full lifecycle: initial → forward × N → backward × N against a live MongoDB instance with 100K seeded records
    - Verify no duplicate `_id` values across all pages; verify `hasMore` / `hasPrevious` flags match actual data boundaries
    - _Requirements: 2.1–2.5, 3.1–3.5, 4.1–4.5, 7.5_

  - [ ] 16.2 Implement `deep-navigation.test.ts`
    - `generateCheckpoints` on 100K dataset, then `deepNavigate` to positions > 10,000; verify returned docs match sequential traversal result
    - Verify relative offset never exceeds 1,000; verify `O(log n + k)` characteristic (query does not scan from position 0)
    - _Requirements: 5.1–5.5, 6.1–6.4_

  - [ ] 16.3 Implement `snapshot-consistency.test.ts`
    - Start a session, concurrently insert new documents, paginate forward across multiple pages; verify inserted docs with `_id > snapshotBoundary` never appear in results
    - _Requirements: 7.1–7.4_

  - [ ]* 16.4 Implement `session-recovery.test.ts`
    - Stateful session: create session, simulate disconnect (delete in-memory reference), re-fetch using `sessionId`; verify session is recoverable with correct `currentAnchor`
    - Test short-TTL expiry: create session with 1-second TTL; wait; verify `SESSION_EXPIRED` error
    - _Requirements: 9.1–9.5_

  - [ ]* 16.5 Implement `benchmark-runner.test.ts`
    - `BenchmarkModule.run()` against a real 100K MongoDB dataset with all three strategies; verify result contains all strategy metrics and a non-empty `comparisonReport`
    - _Requirements: 10.1–10.5_

- [ ] 17. Final checkpoint — Ensure all tests pass
  - Run full unit test suite: `jest --testPathPattern=tests/unit --runInBand`
  - Run integration tests (requires live MongoDB): `jest --testPathPattern=tests/integration --runInBand`
  - Run `tsc --noEmit` and confirm zero type errors
  - Ensure all tests pass; ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP; all 24 correctness properties have corresponding optional test sub-tasks
- Integration tests (16.4, 16.5) are also marked optional; they require a live MongoDB instance
- Checkpoints (tasks 12 and 17) are mandatory validation gates — do not skip them
- All components must satisfy `tsc --noEmit` with zero errors before the next task begins
- Property tests use a minimum of 100 fast-check iterations per property
- Mock all MongoDB and Redis I/O in unit/property tests — no live database access in the unit suite
- Each task references specific requirements for traceability; all 12 requirement groups are covered

---

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1", "2"] },
    { "id": 1, "tasks": ["3", "15"] },
    { "id": 2, "tasks": ["4.1", "4.2", "4.3", "5.1", "5.2", "5.3", "5.4"] },
    { "id": 3, "tasks": ["6"] },
    { "id": 4, "tasks": ["7.1", "7.2", "8.1", "8.2", "8.3"] },
    { "id": 5, "tasks": ["9"] },
    { "id": 6, "tasks": ["10.1"] },
    { "id": 7, "tasks": ["11.1", "11.2", "11.3", "11.4", "11.5", "11.6", "11.7", "11.8", "11.9", "11.10"] },
    { "id": 8, "tasks": ["13.1"] },
    { "id": 9, "tasks": ["14"] },
    { "id": 10, "tasks": ["16.1", "16.2", "16.3", "16.4", "16.5"] }
  ]
}
```
