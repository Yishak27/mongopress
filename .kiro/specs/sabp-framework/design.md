# Design Document

## SABP Framework — Stateful Anchor-Based Pagination

---

## Overview

The SABP Framework is a TypeScript NPM library that replaces MongoDB's `skip()`/`limit()` pagination with an anchor-based, stateful navigation model. It sits as a middleware layer between application code and the MongoDB driver, exposing a clean, typed API that handles all cursor management, snapshot consistency, checkpoint resolution, and session persistence internally.

### Core Idea

Instead of saying "give me page 230,000", callers say "give me the next page after this anchor token". The anchor encodes exactly where the caller is in the sorted dataset. Combined with a sparse checkpoint index, the framework can jump to any deep position in O(log n + k) time rather than O(n) time required by skip-based approaches.

### Key Design Goals

- No full-collection scans during normal pagination
- Stable results under concurrent inserts and deletes (snapshot isolation)
- Pluggable storage adapters (MongoDB, Redis, in-memory)
- Both stateless (client holds token) and stateful (server holds session) operation modes
- Built-in benchmark module for academic comparison against skip/limit and cursor strategies
- Distributed as a single NPM package with full TypeScript types

---

## Architecture

### Component Topology

```
Application Code
      │
      ▼
┌─────────────────────────────────────────────────────┐
│                   Paginator                         │  ← Orchestrator / Public API entry point
│  ┌────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │AnchorEngine│  │SnapshotMgr   │  │SessionMgr   │ │
│  └────────────┘  └──────────────┘  └─────────────┘ │
│  ┌────────────┐  ┌──────────────┐                   │
│  │QueryBuilder│  │CheckpointMgr │                   │
│  └────────────┘  └──────────────┘                   │
└───────────────────────┬─────────────────────────────┘
                        │
              ┌─────────▼──────────┐
              │   Storage Adapter  │  ← MongoDBAdapter | RedisAdapter | MemoryAdapter
              └─────────┬──────────┘
                        │
                        ▼
              MongoDB / Redis / Memory
```

### Data Flow — Initial Request

```
1. Paginator.paginate({ collection, limit })
2.   SnapshotManager.captureSnapshot()  → snapshotBoundary (_id of latest doc)
3.   QueryBuilder.buildInitialQuery()   → { sort: { createdAt: -1, _id: -1 }, limit }
4.   MongoDBAdapter.find(query)         → documents[]
5.   AnchorEngine.encode(lastDoc, snapshotBoundary) → nextAnchorToken
6.   [Stateful] SessionManager.createSession()
7.   Return { data, nextAnchor, previousAnchor: null, hasMore }
```

### Data Flow — Forward Navigation

```
1. Paginator.paginate({ anchor: token, direction: 'forward', limit })
2.   AnchorEngine.decode(token)         → { documentId, sortValue, snapshotBoundary, collection }
3.   QueryBuilder.buildForwardQuery()   → { createdAt: { $lt: sortValue }, _id: { $lte: snapshotBoundary } }
4.   MongoDBAdapter.find(query, limit)  → documents[]
5.   AnchorEngine.encode(lastDoc, snapshotBoundary) → nextAnchorToken
6.   [Stateful] SessionManager.updateSession()
7.   Return { data, nextAnchor, previousAnchor, hasMore }
```

### Data Flow — Backward Navigation

```
1. Paginator.paginate({ anchor: token, direction: 'backward', limit })
2.   AnchorEngine.decode(token)         → { documentId, sortValue, ... }
3.   QueryBuilder.buildBackwardQuery()  → { createdAt: { $gt: sortValue } }
4.   MongoDBAdapter.find(query, limit)  → documents[] (ascending by createdAt)
5.   results.reverse()                  → descending order
6.   AnchorEngine.encode(...)           → updatedAnchorToken
7.   Return { data, nextAnchor, previousAnchor, hasPrevious }
```

### Data Flow — Deep Navigation

```
1. Paginator.deepNavigate({ collection, position, limit })
2.   CheckpointManager.findNearest(collection, position)  → checkpoint
3.   relativeOffset = position - checkpoint.position      (≤ 1,000)
4.   QueryBuilder.buildDeepQuery(checkpoint, relativeOffset, limit)
5.   MongoDBAdapter.find(query).skip(relativeOffset).limit(limit) → documents[]
6.   AnchorEngine.encode(lastDoc, snapshotBoundary)        → anchorToken
7.   Return { data, nextAnchor, hasMore }
```

### Project Structure

```
src/
├── core/
│   ├── paginator.ts          ← Public orchestrator; wires all components
│   ├── anchor-engine.ts      ← Encode / decode / validate anchor tokens
│   ├── checkpoint-manager.ts ← Sparse checkpoint CRUD and lookup
│   ├── query-builder.ts      ← Pure functions: build MongoDB query objects
│   ├── snapshot-manager.ts   ← Snapshot boundary capture and enforcement
│   └── session-manager.ts    ← Session CRUD with adapter abstraction
├── adapters/
│   ├── mongodb-adapter.ts    ← Wraps Mongoose / native driver
│   ├── redis-adapter.ts      ← Wraps ioredis client
│   └── memory-adapter.ts     ← In-process Map for dev/test
├── benchmark/
│   ├── benchmark-runner.ts   ← Orchestrates benchmark runs
│   ├── metrics.ts            ← Metric collection helpers (cpu, mem, latency)
│   └── comparison.ts         ← Formats side-by-side comparison reports
├── types/
│   └── index.ts              ← All exported TypeScript interfaces and types
├── utils/
│   └── logger.ts             ← Structured logger (component, operation, message)
└── index.ts                  ← Single public entry point — re-exports all public API
```

---

## Components and Interfaces

### Paginator

The top-level orchestrator. All pagination requests enter through this component.

```typescript
interface PaginationRequest {
  collection: string;
  limit: number;
  anchor?: string;          // Base64 anchor token (absent on first request)
  direction?: 'forward' | 'backward';
  sessionId?: string;       // Required in stateful mode
}

interface DeepNavigationRequest {
  collection: string;
  position: number;         // Absolute zero-based position
  limit: number;
  sessionId?: string;
}

interface PaginationResponse<T = unknown> {
  data: T[];
  nextAnchor: string | null;
  previousAnchor: string | null;
  hasMore: boolean;
  hasPrevious: boolean;
}

interface PaginatorConfig {
  mode: 'stateless' | 'stateful';
  sessionTTLSeconds?: number;   // Defaults to 3600
  checkpointInterval?: number;  // Defaults to 1000
}

interface Paginator {
  paginate<T>(request: PaginationRequest): Promise<PaginationResponse<T> | SABPError>;
  deepNavigate<T>(request: DeepNavigationRequest): Promise<PaginationResponse<T> | SABPError>;
}
```

### AnchorEngine

Responsible for all token lifecycle operations. Pure logic with no database I/O.

```typescript
// V1 anchor payload — stored as Base64(JSON.stringify(payload))
interface AnchorPayloadV1 {
  version: 1;
  documentId: string;       // _id of the last document on the page
  sortValue: string;        // ISO-8601 createdAt of that document
  snapshotBoundary: string; // _id of the latest document at session start
  collection: string;       // Target collection name
}

interface AnchorEngine {
  encode(payload: AnchorPayloadV1): string;
  decode(token: string): AnchorPayloadV1 | SABPError;
  validate(token: string, activeCollection: string): true | SABPError;
  reEncode(payload: AnchorPayloadV1): string; // re-encode after mutation
}
```

Encoding is deterministic: `Buffer.from(JSON.stringify(payload)).toString('base64')`.  
Decoding reverses the operation and validates the `version` field and `collection` match.

### CheckpointManager

Creates and resolves sparse checkpoints. Performs all I/O through the `MongoDBAdapter`.

```typescript
interface Checkpoint {
  _id?: string;
  collectionName: string;
  position: number;          // Absolute 0-based index in sorted dataset
  documentId: string;
  sortValue: string;
  createdAt: Date;
}

interface CheckpointLookupResult {
  found: true;
  checkpoint: Checkpoint;
} | {
  found: false;
}

interface CheckpointManager {
  generateCheckpoints(collection: string, interval?: number): Promise<void>;
  findNearest(collection: string, position: number): Promise<CheckpointLookupResult | SABPError>;
}
```

Lookup uses: `sabp_checkpoints.findOne({ collectionName, position: { $lte: target } }).sort({ position: -1 })` — O(log n) with compound index.

### QueryBuilder

Pure functions, no side effects, no I/O. Returns MongoDB filter/option objects.

```typescript
interface ForwardQueryParams {
  sortValue: string;          // anchor.sortValue
  snapshotBoundary: string;   // anchor.snapshotBoundary
  limit: number;
}

interface BackwardQueryParams {
  sortValue: string;
  limit: number;
}

interface DeepQueryParams {
  checkpointSortValue: string;
  checkpointDocumentId: string;
  relativeOffset: number;
  limit: number;
}

interface QueryBuilder {
  buildInitialQuery(limit: number): MongoQuery;
  buildForwardQuery(params: ForwardQueryParams): MongoQuery;
  buildBackwardQuery(params: BackwardQueryParams): MongoQuery;
  buildDeepQuery(params: DeepQueryParams): MongoQuery;
}

interface MongoQuery {
  filter: Record<string, unknown>;
  sort?: Record<string, 1 | -1>;
  limit?: number;
  skip?: number;
}
```

### SnapshotManager

Captures and enforces snapshot boundaries. Single responsibility: snapshot isolation.

```typescript
interface SnapshotManager {
  captureSnapshot(collection: string): Promise<string | SABPError>; // returns _id string
  enforceSnapshot(query: MongoQuery, snapshotBoundary: string): MongoQuery;
}
```

`captureSnapshot` executes `find().sort({ _id: -1 }).limit(1).project({ _id: 1 })` to get the latest document ID.  
`enforceSnapshot` merges `{ _id: { $lte: snapshotBoundary } }` into an existing filter object.

### SessionManager

Abstracts session CRUD over any storage adapter.

```typescript
interface Session {
  sessionId: string;
  currentAnchor: string;
  snapshotBoundary: string;
  createdAt: Date;
  updatedAt: Date;
  expiresAt: Date;
}

interface StorageAdapter {
  get(key: string): Promise<string | null>;
  set(key: string, value: string, ttlSeconds: number): Promise<void>;
  delete(key: string): Promise<void>;
}

interface SessionManager {
  createSession(anchor: string, snapshotBoundary: string): Promise<Session | SABPError>;
  getSession(sessionId: string): Promise<Session | SABPError>;
  updateSession(sessionId: string, newAnchor: string): Promise<Session | SABPError>;
  deleteSession(sessionId: string): Promise<void>;
}
```

### BenchmarkModule

Orchestrates benchmark runs and emits comparison reports.

```typescript
interface BenchmarkConfig {
  datasetSize: 100_000 | 1_000_000 | 5_000_000 | 10_000_000;
  collection: string;
  pageSize: number;
  deepPosition?: number;      // For deep navigation benchmarks
}

interface BenchmarkMetrics {
  strategy: 'skip-limit' | 'cursor' | 'sabp';
  queryExecutionTime: number; // ms
  cpuUsage: number;           // percentage
  memoryUsage: number;        // MB
  documentsScanned: number;
  throughput: number;         // pages/second
  averageLatency: number;     // ms
}

interface BenchmarkResult {
  config: BenchmarkConfig;
  results: BenchmarkMetrics[];
  comparisonReport: string;
  completedAt: Date;
}

interface BenchmarkModule {
  run(config: BenchmarkConfig): Promise<BenchmarkResult | SABPError>;
}
```

### Error Type

```typescript
interface SABPError {
  error: true;
  code: string;   // e.g. 'ANCHOR_DECODE_FAILED', 'SESSION_NOT_FOUND'
  message: string;
  component: string;
  operation: string;
}
```

---

## Data Models

### Anchor Token (Wire Format)

The anchor token is a Base64-encoded JSON string passed opaquely between client and server.

```
Base64 → JSON.stringify(AnchorPayloadV1)
```

**AnchorPayloadV1 fields:**

| Field             | Type   | Description                                              |
|-------------------|--------|----------------------------------------------------------|
| version           | number | Always `1` for v1 tokens                                |
| documentId        | string | `_id` (hex string) of the last document on the page     |
| sortValue         | string | ISO-8601 `createdAt` of that document                   |
| snapshotBoundary  | string | `_id` (hex string) of the latest doc at session start   |
| collection        | string | MongoDB collection name this anchor is scoped to        |

Example decoded token:
```json
{
  "version": 1,
  "documentId": "665f1a2b3c4d5e6f7a8b9c0d",
  "sortValue": "2026-01-15T10:30:00.000Z",
  "snapshotBoundary": "665f1a2b3c4d5e6f7a8b9c10",
  "collection": "users"
}
```

---

### `sabp_checkpoints` Collection

Persists sparse navigation markers. Shared globally across all sessions for the same collection.

```typescript
// Document schema
{
  _id:            ObjectId,       // MongoDB auto-generated
  collectionName: string,         // e.g. "users"
  position:       number,         // Absolute 0-based position in dataset
  documentId:     string,         // _id of the document at this position
  sortValue:      string,         // ISO-8601 createdAt of that document
  createdAt:      Date            // When this checkpoint was created
}
```

**Index:**
```javascript
{ collectionName: 1, position: 1 }  // Compound ascending — supports O(log n) lookup
```

Checkpoints are written once during an explicit `generateCheckpoints()` call and updated when the dataset grows substantially. For a 10M-record dataset at interval 1,000: at most 10,000 checkpoint records.

---

### `sabp_sessions` Collection

Holds stateful session state when the MongoDB adapter is used.

```typescript
// Document schema
{
  _id:              ObjectId,
  sessionId:        string,         // UUID v4
  currentAnchor:    string,         // Current Base64 anchor token
  snapshotBoundary: string,         // _id hex string
  createdAt:        Date,
  updatedAt:        Date,
  expiresAt:        Date            // TTL index target field
}
```

**Index:**
```javascript
{ expiresAt: 1 }     // TTL index — MongoDB auto-deletes expired documents
{ sessionId: 1 }     // Unique index — fast session lookup
```

When using the Redis adapter, session data is stored as a JSON-serialized string under key `sabp:session:{sessionId}` with a Redis `EXPIRE` TTL. When using the in-memory adapter, sessions are stored in a `Map<string, Session>` with a `setTimeout` cleanup.

---

### Framework Configuration Object

```typescript
interface SABPConfig {
  db: Db;                            // MongoDB native Db instance or Mongoose connection
  mode: 'stateless' | 'stateful';
  adapter: 'mongodb' | 'redis' | 'memory';
  redisClient?: RedisClient;         // Required when adapter is 'redis'
  sessionTTLSeconds?: number;        // Default: 3600
  checkpointInterval?: number;       // Default: 1000
  logLevel?: 'silent' | 'error' | 'info' | 'debug';
}
```


---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

---

### Property 1: Anchor Round-Trip

*For any* valid `AnchorPayloadV1` object, encoding it to a Base64 token and then decoding that token back to a payload object, and then re-encoding the decoded payload, SHALL produce a token equal to the original encoding — and the decoded payload SHALL be structurally equal to the original input.

**Validates: Requirements 1.1, 1.2, 1.3, 1.4**

---

### Property 2: Invalid Anchor Handling

*For any* string that is not a valid SABP anchor token (arbitrary bytes, truncated Base64, corrupted JSON, or an otherwise valid token whose version field is not `1`), calling `AnchorEngine.decode()` or `AnchorEngine.validate()` SHALL return an `SABPError` object and SHALL NOT throw an unhandled exception. When the version is present but unsupported, the error SHALL contain both the expected version (`1`) and the received version.

**Validates: Requirements 1.5, 12.3**

---

### Property 3: Collection Mismatch Rejection

*For any* valid anchor token whose `collection` field is not equal to the `activeCollection` argument passed to `AnchorEngine.validate()`, the method SHALL return an `SABPError` describing the collection mismatch.

**Validates: Requirements 1.6**

---

### Property 4: Initial Query Builder Correctness

*For any* positive integer `limit`, `QueryBuilder.buildInitialQuery(limit)` SHALL return a `MongoQuery` with `sort` equal to `{ createdAt: -1, _id: -1 }` and `limit` equal to the input value.

**Validates: Requirements 2.2**

---

### Property 5: Snapshot Capture and Embedding

*For any* non-empty collection, after calling `Paginator.paginate()` with no anchor, the `snapshotBoundary` embedded in the decoded `nextAnchor` token SHALL equal the `_id` of the most recently inserted document in the collection at the time the request was processed, and that value SHALL have been captured before the retrieval query was executed.

**Validates: Requirements 2.1, 2.3, 7.1**

---

### Property 6: hasMore / hasPrevious Correctness

*For any* `Paginator.paginate()` or `Paginator.deepNavigate()` call where the number of documents returned is strictly less than the requested `limit`, the response SHALL set `hasMore: false`. When backward navigation exhausts all preceding documents, the response SHALL set `hasPrevious: false`.

**Validates: Requirements 2.4, 3.3, 4.4**

---

### Property 7: Response Shape Invariant

*For any* call to `Paginator.paginate()`, the response SHALL contain exactly the fields `{ data, nextAnchor, previousAnchor, hasMore, hasPrevious }` regardless of direction, mode, or dataset size.

**Validates: Requirements 2.5**

---

### Property 8: Forward Query Filter Correctness

*For any* valid `ForwardQueryParams` `{ sortValue, snapshotBoundary, limit }`, `QueryBuilder.buildForwardQuery()` SHALL return a `MongoQuery` whose filter contains `{ createdAt: { $lt: sortValue }, _id: { $lte: snapshotBoundary } }` and whose `limit` equals the input `limit`.

**Validates: Requirements 3.1**

---

### Property 9: Snapshot Boundary Exclusion

*For any* document whose `_id` is greater than the `snapshotBoundary` established at session start, that document SHALL never appear in the `data` array of any forward or backward navigation response within that session, regardless of how many page turns occur.

**Validates: Requirements 3.2, 7.2, 7.3**

---

### Property 10: Anchor Points to Page Boundary Document

*For any* successful `paginate()` call in the forward direction, decoding the returned `nextAnchor` SHALL yield a `documentId` equal to the `_id` of the last document in the returned `data` array. For any successful backward navigation, decoding the returned `previousAnchor` SHALL yield a `documentId` equal to the `_id` of the first document in the reversed `data` array.

**Validates: Requirements 3.4, 4.5**

---

### Property 11: Backward Query Filter Correctness

*For any* valid `BackwardQueryParams` `{ sortValue, limit }` where `limit > 0`, `QueryBuilder.buildBackwardQuery()` SHALL return a `MongoQuery` whose filter contains `{ createdAt: { $gt: sortValue } }` and whose `limit` equals the input `limit`.

**Validates: Requirements 4.1**

---

### Property 12: Backward Result Ordering

*For any* successful backward navigation, the `data` array in the response SHALL be ordered in descending `createdAt` order — that is, it SHALL be the reverse of the raw ascending-order result returned by MongoDB.

**Validates: Requirements 4.3**

---

### Property 13: No Offset Drift Under Concurrent Deletes

*For any* sequence of forward page turns in a session during which one or more documents are deleted from the collection, no document that was present in a previous page SHALL be re-returned in a subsequent page, and no document that would have appeared SHALL be silently skipped, because queries are anchor-relative and not position-relative.

**Validates: Requirements 7.4**

---

### Property 14: No Duplicate Documents Across Page Turns

*For any* sequence of consecutive forward page turns within a single session, the union of all `_id` values across all returned pages SHALL contain no duplicates.

**Validates: Requirements 7.5**

---

### Property 15: Nearest Checkpoint Lookup Correctness

*For any* set of checkpoint records for a collection and any target position `p`, `CheckpointManager.findNearest(collection, p)` SHALL return the checkpoint record with the highest `position` value that is less than or equal to `p` — or return `{ found: false }` if no such checkpoint exists.

**Validates: Requirements 5.1, 6.5**

---

### Property 16: Relative Offset Invariant

*For any* valid deep navigation request where checkpoints exist at the configured interval (default 1,000 records), the relative offset computed as `targetPosition - checkpoint.position` SHALL satisfy `0 ≤ relativeOffset < interval` (i.e., SHALL NOT exceed 1,000 documents).

**Validates: Requirements 5.2**

---

### Property 17: Deep Query Uses Relative Offset as Skip

*For any* `DeepQueryParams` `{ checkpointSortValue, checkpointDocumentId, relativeOffset, limit }`, `QueryBuilder.buildDeepQuery()` SHALL return a `MongoQuery` whose `skip` value equals `relativeOffset` and whose filter is anchored to the checkpoint document.

**Validates: Requirements 5.3**

---

### Property 18: Checkpoint Storage Count and Schema Invariant

*For any* collection of size `n` where `generateCheckpoints()` is called with interval `interval`, the number of records written to `sabp_checkpoints` SHALL equal `Math.floor(n / interval)`. Each persisted checkpoint document SHALL contain non-null values for all of `{ position, documentId, sortValue, collectionName }`.

**Validates: Requirements 6.1, 6.2, 6.3**

---

### Property 19: Stateless Mode Produces No Server State

*For any* `Paginator.paginate()` call when the framework is configured in `stateless` mode, no session document SHALL be written to `sabp_sessions`, no key SHALL be written to Redis, and no entry SHALL be added to the in-memory session store.

**Validates: Requirements 8.1, 9.3**

---

### Property 20: Stateless Mode Returns Anchor in Every Response

*For any* `Paginator.paginate()` call in stateless mode where `hasMore: true`, the response SHALL include a non-null `nextAnchor` token containing the `snapshotBoundary` from the original session, so the client can supply it on the next request.

**Validates: Requirements 8.3**

---

### Property 21: Session Lifecycle — Creation and Update

*For any* initial `paginate()` call in stateful mode, `SessionManager.createSession()` SHALL persist a session document containing non-null values for all of `{ sessionId, currentAnchor, snapshotBoundary, expiresAt }`. For any subsequent page-turn request on that session, `SessionManager.updateSession()` SHALL update `currentAnchor` to match the `nextAnchor` returned in the response.

**Validates: Requirements 9.1, 9.2**

---

### Property 22: Benchmark Completeness

*For any* completed benchmark run initiated via `BenchmarkModule.run()`, the resulting `BenchmarkResult` SHALL contain metric records for all three strategies (`skip-limit`, `cursor`, `sabp`). Each metric record SHALL include non-null values for all six fields: `{ queryExecutionTime, cpuUsage, memoryUsage, documentsScanned, throughput, averageLatency }`. The result SHALL also include a non-empty `comparisonReport` string. Accessing benchmark results without first calling `run()` SHALL return an error or empty state — never stale results from a previous run.

**Validates: Requirements 10.2, 10.3, 10.4**

---

### Property 23: Invalid Configuration Rejected Before DB Access

*For any* configuration object that is missing a required field or contains an invalid value (e.g., null `db`, unsupported `adapter` string, negative `sessionTTLSeconds`), calling the framework initializer SHALL throw a descriptive configuration `SABPError` before any MongoDB or Redis operation is attempted.

**Validates: Requirements 11.5**

---

### Property 24: Query Timeout Returns SABPError Without Throwing

*For any* pagination operation where the underlying MongoDB query raises a timeout error (or any other driver-level error), `Paginator.paginate()` and `Paginator.deepNavigate()` SHALL catch the error and return a structured `SABPError` `{ error, code, message, component, operation }` — the error SHALL NOT propagate as an unhandled exception.

**Validates: Requirements 1.5, 12.1**

---

## Error Handling

### Strategy

All errors in the SABP Framework follow the **structured error response** pattern — no component throws unhandled exceptions to the caller. Every public method returns either a successful result or an `SABPError` object.

```typescript
interface SABPError {
  error: true;
  code: string;        // Machine-readable error code
  message: string;     // Human-readable description
  component: string;   // e.g., 'AnchorEngine', 'SessionManager'
  operation: string;   // e.g., 'decode', 'createSession'
}
```

### Error Codes

| Code                       | Component          | Condition                                               |
|----------------------------|--------------------|---------------------------------------------------------|
| `ANCHOR_DECODE_FAILED`     | AnchorEngine       | Base64 or JSON parse failure                            |
| `ANCHOR_VERSION_MISMATCH`  | AnchorEngine       | Token version ≠ 1; includes expected/received versions |
| `ANCHOR_COLLECTION_MISMATCH` | AnchorEngine     | Token collection ≠ active collection                   |
| `SNAPSHOT_CAPTURE_FAILED`  | SnapshotManager    | Cannot read latest document _id                        |
| `CHECKPOINT_NOT_FOUND`     | CheckpointManager  | No checkpoint exists at or before requested position   |
| `CHECKPOINT_CORRUPTED`     | CheckpointManager  | Stored checkpoint document is missing required fields  |
| `EMPTY_DATASET`            | Paginator          | Deep navigation requested on zero-document collection  |
| `POSITION_OUT_OF_RANGE`    | Paginator          | Requested position exceeds dataset size                |
| `SESSION_NOT_FOUND`        | SessionManager     | Session ID does not exist in storage                   |
| `SESSION_EXPIRED`          | SessionManager     | Session found but past TTL (race condition guard)      |
| `REDIS_CONNECTION_LOST`    | RedisAdapter       | Redis connection unavailable during operation          |
| `QUERY_TIMEOUT`            | Paginator          | MongoDB query exceeded timeout threshold               |
| `CONFIG_INVALID`           | Framework init     | Missing or invalid configuration field                 |
| `INIT_FAILED`              | Framework init     | Network/version error during MongoDB connection        |
| `BENCHMARK_STRATEGY_FAILED`| BenchmarkModule    | One strategy errored during benchmark run              |
| `BENCHMARK_RECORD_FAILED`  | BenchmarkModule    | Failure recording itself failed; run is aborted        |

### Logging

All internal errors are logged with structured context using the `Logger` utility:

```typescript
logger.error({
  component: 'AnchorEngine',
  operation: 'decode',
  message: 'Base64 decode failed: invalid token',
  // Never logs: connection strings, passwords, or raw document data
});
```

Sensitive data (connection strings, credentials, raw document content) is never included in logs or error responses.

### Fallback Behaviour

- **Corrupted checkpoint**: `CheckpointManager` logs the anomaly and falls back to sequential traversal from the last valid checkpoint.
- **Redis adapter failure**: `SessionManager` returns `REDIS_CONNECTION_LOST` error; session state is not silently corrupted.
- **Benchmark strategy failure**: the failure is recorded; if recording itself fails, the entire benchmark run is aborted.

---

## Testing Strategy

### Dual Approach

The SABP Framework uses both **property-based tests** and **example-based unit tests** for comprehensive coverage. These are complementary — unit tests catch concrete edge cases, property tests verify universal correctness across the full input space.

### Property-Based Testing Library

- **Library**: [fast-check](https://fast-check.dev/) (TypeScript-native, no runtime overhead)
- **Minimum iterations per property**: 100 runs
- **Tag format** for each property test:

  ```typescript
  // Feature: sabp-framework, Property N: <property_text>
  ```

### Property Test Coverage

Each of the 24 correctness properties above maps to one property-based test. Arbitraries (generators) are defined for:

- `AnchorPayloadV1` — random documentId (hex), sortValue (ISO dates), snapshotBoundary (hex), collection (alphanumeric string)
- Invalid anchor strings — arbitrary byte sequences, truncated strings, valid JSON with wrong schema
- `ForwardQueryParams`, `BackwardQueryParams`, `DeepQueryParams` — parameterized by the above
- Document arrays — random arrays of MongoDB-like documents with `_id` and `createdAt`
- Checkpoint arrays — random checkpoint sets at valid positions
- Config objects — including intentionally malformed variants

MongoDB and Redis calls are **mocked** in all property tests to ensure:
- Tests run in O(ms) not O(seconds)
- 100+ iterations are practical
- No external infrastructure required for unit tests

### Unit Test Coverage

Unit tests (example-based) cover:

- Specific error paths: empty dataset, session not found, limit = 0
- Adapter selection routing (MongoDB vs Redis vs memory)
- TTL expiry simulation
- Benchmark report format
- Cascading benchmark failure (strategy fail → record fail → abort)
- TypeScript type compilation (no `any` leakage in public API)

### Integration Tests

Integration tests (separate suite, require live MongoDB):

- Full pagination lifecycle: initial → forward × N → backward × N
- Stateful session recovery after simulated disconnect
- Checkpoint generation and deep navigation on 100K dataset
- Snapshot isolation under concurrent inserts
- TTL index expiry on `sabp_sessions` (short TTL test)
- Benchmark module against real 100K dataset

### Test File Structure

```
tests/
├── unit/
│   ├── anchor-engine.test.ts        ← Properties 1–3, unit examples
│   ├── query-builder.test.ts        ← Properties 4, 8, 11, 17
│   ├── snapshot-manager.test.ts     ← Properties 5, 9
│   ├── checkpoint-manager.test.ts   ← Properties 15, 16, 18
│   ├── paginator.test.ts            ← Properties 6, 7, 10, 12–14, 19, 20, 23, 24
│   ├── session-manager.test.ts      ← Properties 21, 22 (partial)
│   └── benchmark-module.test.ts     ← Property 22
├── integration/
│   ├── pagination-lifecycle.test.ts
│   ├── deep-navigation.test.ts
│   ├── snapshot-consistency.test.ts
│   ├── session-recovery.test.ts
│   └── benchmark-runner.test.ts
└── fixtures/
    └── generators.ts                ← Shared fast-check arbitraries
```

### Performance Benchmarks (Non-Test)

The `BenchmarkModule` is used to empirically validate complexity targets:

| Operation          | Target Complexity | Datasets                          |
|--------------------|-------------------|-----------------------------------|
| Initial page       | O(log n)          | 100K, 1M, 5M, 10M records         |
| Forward navigation | O(k)              | All sizes; k = page size          |
| Backward navigation| O(k)              | All sizes                         |
| Deep navigation    | O(log n + k)      | 1M, 5M, 10M; position > 1M       |
| Checkpoint storage | O(n / interval)   | 10M dataset → ≤ 10,000 records    |
