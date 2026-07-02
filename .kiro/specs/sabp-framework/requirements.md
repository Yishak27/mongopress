# Requirements Document

## Introduction

The Stateful Anchor-Based Pagination (SABP) Framework is a production-ready Node.js library for efficient, scalable MongoDB pagination. It replaces the conventional `skip()`/`limit()` approach with an anchor-based, stateful navigation model capable of handling datasets of 100K, 1M, 5M, and 10M records. The framework is distributed as an NPM package and provides forward, backward, and deep navigation with snapshot-based consistency, sparse checkpoint indexing, and both stateless and stateful session modes. A benchmark module enables direct comparison against skip/limit and cursor-based pagination.

---

## Glossary

- **SABP_Framework**: The Stateful Anchor-Based Pagination Node.js library as a whole.
- **Anchor_Engine**: The core component responsible for generating, encoding, decoding, and validating anchor tokens.
- **Anchor**: A navigation reference encoding a document's position within a sorted dataset, serialized as a Base64 token.
- **Anchor_Token**: A Base64-encoded JSON structure `{ version, documentId, sortValue, snapshotBoundary, collection }` representing a stable navigation pointer.
- **Checkpoint_Manager**: The component that creates, stores, and resolves sparse checkpoint records for deep navigation.
- **Checkpoint**: A sparse navigation marker stored in the `sabp_checkpoints` collection, recording a document's absolute position, documentId, sortValue, and collectionName at every 1,000-record interval.
- **Snapshot_Boundary**: The `_id` of the latest document captured at the moment pagination begins, used to prevent duplicates or missing records under concurrent writes.
- **Snapshot_Manager**: The component that creates and enforces snapshot boundaries for pagination consistency.
- **Query_Builder**: The component that generates optimized MongoDB query objects for forward, backward, and deep navigation.
- **Session_Manager**: The component that creates, updates, retrieves, and expires stateful navigation sessions.
- **Session**: A server-side navigation context stored in Redis, MongoDB (`sabp_sessions`), or an in-memory adapter, containing the current anchor, snapshot boundary, and expiration timestamp.
- **Paginator**: The top-level orchestrating component that coordinates all other core components to fulfill a pagination request.
- **Benchmark_Module**: The component that measures and compares query performance metrics across skip/limit, cursor, and SABP pagination strategies.
- **MongoDB_Adapter**: The adapter that wraps MongoDB driver operations for use by core components.
- **Redis_Adapter**: The adapter that wraps Redis client operations for stateful session storage.
- **Memory_Adapter**: The adapter that stores session state in process memory for development and testing.
- **TTL_Index**: A MongoDB index on `expiresAt` that automatically removes expired session documents.
- **Deep_Navigation**: Direct navigation to a distant dataset position without sequential traversal, using the nearest checkpoint and a localized offset.
- **Forward_Navigation**: Navigation to the next page relative to the current anchor.
- **Backward_Navigation**: Navigation to the previous page relative to the current anchor.
- **Stateless_Mode**: An operation mode where the client stores the anchor token and snapshot boundary; no server-side session is maintained.
- **Stateful_Mode**: An operation mode where session state is persisted server-side in Redis or MongoDB.

---

## Requirements

### Requirement 1: Anchor Token Generation and Decoding

**User Story:** As a developer, I want the framework to produce and consume opaque anchor tokens, so that I can pass them between client and server without managing raw cursor state.

#### Acceptance Criteria

1. WHEN a pagination result is returned, THE Anchor_Engine SHALL encode `{ version, documentId, sortValue, snapshotBoundary, collection }` into a Base64 anchor token.
2. WHEN an anchor token is received, THE Anchor_Engine SHALL decode it into the original `{ version, documentId, sortValue, snapshotBoundary, collection }` structure.
3. THE Anchor_Engine SHALL format a valid Configuration object back into a valid anchor token (pretty-print / re-encode).
4. FOR ALL valid anchor objects, encoding then decoding then re-encoding SHALL produce a token equal to the first encoding (round-trip property).
5. IF an error occurs during anchor token processing (including malformed tokens, tampered tokens, network timeouts, or database unavailability), THEN THE Anchor_Engine SHALL return a descriptive error without throwing an unhandled exception.
6. IF an anchor token references a `collection` value that does not match the active query collection, THEN THE Anchor_Engine SHALL reject the token with a descriptive error.
7. THE Anchor_Engine SHALL support anchor token version 1 as defined in the algorithm specification.

---

### Requirement 2: Initial Pagination

**User Story:** As a developer, I want to request the first page of results, so that users can begin browsing a large dataset without providing an anchor.

#### Acceptance Criteria

1. WHEN an initial pagination request is received with no anchor, THE Paginator SHALL capture the `_id` of the most recently inserted document as the Snapshot_Boundary before executing the retrieval query.
2. WHEN an initial pagination request is executed, THE Query_Builder SHALL generate a query sorted by `{ createdAt: -1, _id: -1 }` with a `limit` equal to the requested page size.
3. WHEN results are returned, THE Anchor_Engine SHALL generate a next-page anchor token embedding the Snapshot_Boundary.
4. WHEN the result set contains fewer documents than the requested page size OR equals the requested page size, THE Paginator SHALL set `hasMore: false` in the response.
5. THE Paginator SHALL return a response containing `{ data, nextAnchor, previousAnchor, hasMore }`.

---

### Requirement 3: Forward Navigation

**User Story:** As a developer, I want to navigate forward through paginated results using an anchor token, so that users can scroll to subsequent pages efficiently.

#### Acceptance Criteria

1. WHEN a forward navigation request is received with a valid anchor, THE Query_Builder SHALL generate a query `find({ createdAt: { $lt: anchor.sortValue }, _id: { $lte: snapshotBoundary } }).limit(limit)`.
2. WHEN forward navigation is executed, THE Paginator SHALL apply the Snapshot_Boundary from the anchor token to exclude documents inserted after the snapshot was taken.
3. WHEN forward navigation returns fewer documents than the page size OR exactly the page size, THE Paginator SHALL set `hasMore: false`.
4. WHEN forward navigation succeeds, THE Anchor_Engine SHALL generate a new anchor token pointing to the last document in the returned page.
5. THE Paginator SHALL achieve forward navigation in O(k) time where k equals the page size, with no full-collection scan.

---

### Requirement 4: Backward Navigation

**User Story:** As a developer, I want to navigate backward through paginated results using an anchor token, so that users can return to previous pages.

#### Acceptance Criteria

1. WHEN a backward navigation request is received with a valid anchor and a `limit` greater than zero, THE Query_Builder SHALL generate a query `find({ createdAt: { $gt: anchor.sortValue } }).limit(limit)`.
2. WHEN a backward navigation request is received with a `limit` of zero, THE Query_Builder SHALL skip query generation and return an empty result set immediately.
3. WHEN backward navigation results are retrieved, THE Paginator SHALL reverse the result array before returning it to the caller.
4. WHEN backward navigation reaches the beginning of the dataset, THE Paginator SHALL set `hasPrevious: false` in the response.
5. WHEN backward navigation succeeds, THE Anchor_Engine SHALL generate an updated anchor reflecting the new position.

---

### Requirement 5: Deep Navigation

**User Story:** As a developer, I want to navigate directly to a distant position in the dataset by page or position number, so that users can jump to specific points without sequential traversal.

#### Acceptance Criteria

1. WHEN a deep navigation request is received with a target absolute position, THE Checkpoint_Manager SHALL locate the nearest checkpoint whose position is less than or equal to the target position.
2. WHEN the nearest checkpoint is identified, THE Paginator SHALL calculate the relative offset as `targetPosition - checkpointPosition`, where the offset SHALL NOT exceed 1,000 documents.
3. WHEN the relative offset is calculated, THE Query_Builder SHALL generate a localized retrieval query starting from the checkpoint anchor and skipping only the relative offset.
4. WHEN deep navigation completes, THE Anchor_Engine SHALL generate a new anchor token for the retrieved position.
5. THE Paginator SHALL perform deep navigation in O(log n + k) time where n is the total dataset size and k is the page size.
6. IF the target dataset contains zero documents, THEN THE Paginator SHALL return a distinct empty-dataset error message indicating no records are available.
7. IF no checkpoint exists at or before the requested position in a non-empty dataset, THEN THE Paginator SHALL return a descriptive error indicating the position is out of range.

---

### Requirement 6: Checkpoint Creation and Management

**User Story:** As a developer, I want the framework to automatically maintain sparse checkpoints over the dataset, so that deep navigation remains efficient regardless of dataset size.

#### Acceptance Criteria

1. WHEN checkpoint generation is triggered for a collection, THE Checkpoint_Manager SHALL create one checkpoint record for every 1,000 documents in the dataset.
2. WHEN a checkpoint is stored, THE Checkpoint_Manager SHALL persist `{ position, documentId, sortValue, collectionName }` to the `sabp_checkpoints` collection.
3. WHEN the Checkpoint_Manager stores checkpoints for a 10,000,000-record dataset, THE Checkpoint_Manager SHALL create no more than 10,000 checkpoint records (O(n / interval) storage complexity).
4. WHEN checkpoint lookup is performed, THE Checkpoint_Manager SHALL return the nearest checkpoint in O(log n) time using an indexed query on `{ collectionName, position }`.
5. WHEN checkpoint lookup is performed on a collection with no checkpoints, THE Checkpoint_Manager SHALL return a distinct 'no checkpoint found' result in O(log n) time without error.
6. IF a checkpoint record is missing or corrupted, THEN THE Checkpoint_Manager SHALL log the anomaly and fall back to sequential traversal from the last valid checkpoint.

---

### Requirement 7: Snapshot Consistency

**User Story:** As a developer, I want paginated results to remain consistent across page turns even when documents are inserted or deleted concurrently, so that users do not see duplicates or gaps.

#### Acceptance Criteria

1. WHEN an initial pagination request is executed, THE Snapshot_Manager SHALL capture the current latest document `_id` as the Snapshot_Boundary before any query is issued.
2. WHILE a pagination session is active, THE Snapshot_Manager SHALL apply the Snapshot_Boundary to all subsequent forward and backward queries for that session.
3. WHEN a new document is inserted into the collection during an active session, THE Paginator SHALL exclude that document from results because its `_id` exceeds the Snapshot_Boundary.
4. WHEN a document is deleted during an active session, THE Paginator SHALL not re-fetch or skip unrelated documents due to offset drift, because queries are anchor-relative.
5. THE Paginator SHALL guarantee that no document appears more than once across consecutive forward page turns within a single session.

---

### Requirement 8: Stateless Mode

**User Story:** As a developer, I want to use the framework without server-side session storage, so that I can integrate it into stateless APIs and horizontally scaled services.

#### Acceptance Criteria

1. WHERE Stateless_Mode is configured, THE SABP_Framework SHALL store no session state on the server between requests.
2. WHERE Stateless_Mode is configured, THE Paginator SHALL read the Anchor_Token and Snapshot_Boundary exclusively from the client-supplied request parameters.
3. WHERE Stateless_Mode is configured, THE Paginator SHALL return the next Anchor_Token and Snapshot_Boundary to the client in every response so the client can supply them on the next request.
4. WHERE Stateless_Mode is configured, THE SABP_Framework SHALL support backward and forward navigation using only the anchor token provided by the client.

---

### Requirement 9: Stateful Mode — Session Management

**User Story:** As a developer, I want the framework to persist navigation sessions server-side, so that users can resume long browsing sessions and recover from disconnections.

#### Acceptance Criteria

1. WHERE Stateful_Mode is configured, THE Session_Manager SHALL create a new session record containing `{ sessionId, currentAnchor, snapshotBoundary, expiresAt }` when pagination begins.
2. WHEN a page turn request is received in Stateful_Mode, THE Session_Manager SHALL retrieve the existing session and update `currentAnchor` to reflect the new position.
3. WHEN a page turn request is received outside of Stateful_Mode, THE Session_Manager SHALL ignore any session update and SHALL NOT attempt to write session state.
4. WHEN a session TTL expires, THE Session_Manager SHALL allow the MongoDB TTL index on `expiresAt` to automatically delete the session document from `sabp_sessions`.
5. IF a session is not found during a page turn request, THEN THE Session_Manager SHALL return a descriptive session-not-found error with a recommendation to restart pagination.
6. WHERE Stateful_Mode is enabled AND the Redis adapter is configured, THE Session_Manager SHALL read and write session state using the Redis_Adapter.
7. WHERE the MongoDB adapter is configured, THE Session_Manager SHALL read and write session state in the `sabp_sessions` collection using a TTL index, regardless of whether Stateful_Mode is explicitly set.
8. WHERE both Stateful_Mode is configured AND the in-memory adapter is selected, THE Session_Manager SHALL store session state in process memory for development and testing purposes only.

---

### Requirement 10: Benchmark Module

**User Story:** As a researcher, I want to benchmark SABP against skip/limit and cursor pagination strategies across multiple dataset sizes, so that I can measure and document performance improvements.

#### Acceptance Criteria

1. WHEN a benchmark run is initiated, THE Benchmark_Module SHALL execute identical workloads against skip/limit, MongoDB cursor, and SABP pagination strategies on the same dataset.
2. WHEN benchmark results are produced, THE Benchmark_Module SHALL only record and report results for a run that was properly initiated via the benchmark start operation.
3. WHEN a benchmark completes, THE Benchmark_Module SHALL record `{ queryExecutionTime, cpuUsage, memoryUsage, documentsScanned, throughput, averageLatency }` for each strategy.
4. WHEN a benchmark completes, THE Benchmark_Module SHALL produce a comparison report summarizing the results of all three strategies side by side.
5. THE Benchmark_Module SHALL support benchmark runs against datasets of 100,000, 1,000,000, 5,000,000, and 10,000,000 records.
6. WHEN deep navigation is benchmarked, THE Benchmark_Module SHALL measure navigation to positions exceeding 1,000,000 records for both SABP and skip/limit strategies.
7. IF a benchmark strategy fails, THEN THE Benchmark_Module SHALL record the failure with a descriptive error message, and IF the failure recording itself fails, THEN THE Benchmark_Module SHALL abort the entire benchmark run.

---

### Requirement 11: NPM Package Distribution

**User Story:** As a developer, I want to install the framework via NPM, so that I can integrate it into any Node.js project with minimal setup.

#### Acceptance Criteria

1. THE SABP_Framework SHALL expose a public API via a single `index.ts` entry point that exports `Paginator`, `AnchorEngine`, `CheckpointManager`, `SnapshotManager`, `SessionManager`, `QueryBuilder`, `BenchmarkModule`, and all adapter classes.
2. THE SABP_Framework SHALL include TypeScript type definitions for all public API members.
3. WHEN the package is installed and imported, THE SABP_Framework SHALL initialize without errors when provided a valid MongoDB connection and configuration object.
4. IF initialization fails due to network connectivity issues or MongoDB version incompatibility, THEN THE SABP_Framework SHALL throw a descriptive non-configuration error and prevent all database operations until initialization succeeds.
5. IF an invalid or missing configuration is provided at initialization, THEN THE SABP_Framework SHALL throw a descriptive configuration error before any database operation is attempted, and SHALL prevent all database operations until valid configuration is supplied.

---

### Requirement 12: Error Handling and Resilience

**User Story:** As a developer, I want the framework to handle errors gracefully, so that pagination failures do not crash the application.

#### Acceptance Criteria

1. IF a MongoDB query times out during pagination, THEN THE Paginator SHALL catch the error and return a structured error response `{ error, code, message }` without throwing an unhandled exception.
2. IF the Redis adapter connection is lost during a stateful session operation, THEN THE Session_Manager SHALL return a structured connection-error response and SHALL NOT silently corrupt session state.
3. IF an anchor token version is not supported by the current SABP_Framework version, THEN THE Anchor_Engine SHALL return a version-mismatch error with the expected and received versions.
4. THE SABP_Framework SHALL never expose sensitive data in any output, log, or error response, and SHALL log all internal errors with sufficient context (component name, operation, error message) to enable debugging.
