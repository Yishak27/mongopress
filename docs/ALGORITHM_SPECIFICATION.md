# Stateful Anchor-Based Pagination (SABP) Algorithm Specification

## Version

v1.0

---

# Algorithm Name

Stateful Anchor-Based Pagination (SABP)

---

# Objective

Provide efficient, scalable, and consistent pagination over large MongoDB datasets while eliminating the performance limitations of skip-based pagination.

---

# Design Principles

1. No large skip operations
2. Anchor-based navigation
3. Snapshot consistency
4. Deep navigation support
5. Stateful session recovery
6. Minimal storage overhead

---

# Definitions

## Anchor

A navigation reference representing a position within a dataset.

Example:

```json
{
  "documentId": "665f...",
  "sortValue": "2026-01-01T10:00:00Z"
}
```

---

## Checkpoint

A sparse navigation marker.

Example:

```text
Position 1000
Position 2000
Position 3000
```

Checkpoints enable deep navigation.

---

## Snapshot

A stable boundary created when pagination begins.

Purpose:

* Prevent duplicates
* Prevent missing records
* Ensure consistency

---

## Session

A stateful navigation context.

Contains:

```json
{
  "sessionId": "...",
  "currentAnchor": "...",
  "snapshotBoundary": "...",
  "direction": "forward"
}
```

---

# Algorithm Inputs

```json
{
  "collection": "...",
  "limit": 20,
  "anchor": "...",
  "direction": "forward"
}
```

---

# Algorithm Outputs

```json
{
  "data": [],
  "nextAnchor": "...",
  "previousAnchor": "...",
  "hasMore": true
}
```

---

# Initial Pagination Algorithm

Step 1

Create snapshot boundary.

```text
snapshotBoundary = latestDocumentId
```

Step 2

Execute retrieval query.

```javascript
find()
.sort({
  createdAt: -1,
  _id: -1
})
.limit(limit)
```

Step 3

Generate anchor token.

Step 4

Return response.

---

# Forward Navigation Algorithm

Input:

```json
{
  "anchor": "...",
  "limit": 20
}
```

Process:

```javascript
find({
  createdAt: {
    $lt: anchor.sortValue
  },
  _id: {
    $lte: snapshotBoundary
  }
})
.limit(limit)
```

Generate new anchor.

Return results.

---

# Backward Navigation Algorithm

Input:

```json
{
  "anchor": "...",
  "limit": 20
}
```

Process:

```javascript
find({
  createdAt: {
    $gt: anchor.sortValue
  }
})
.limit(limit)
```

Reverse result order.

Generate previous anchor.

Return results.

---

# Deep Navigation Algorithm

Input:

```json
{
  "position": 2300560,
  "limit": 20
}
```

Step 1

Locate nearest checkpoint.

Example:

```text
Checkpoint Position = 2300000
```

Step 2

Calculate relative offset.

```text
560
```

Step 3

Execute localized retrieval.

Step 4

Generate anchor.

Step 5

Return results.

---

# Checkpoint Generation Algorithm

Input:

```text
Collection Dataset
```

Interval:

```text
1000 records
```

Process:

For every interval:

1. Retrieve checkpoint document.
2. Store metadata.
3. Save checkpoint.

Output:

```json
{
  "position": "...",
  "documentId": "...",
  "sortValue": "..."
}
```

---

# Session Lifecycle

Session Creation

```text
User Starts Pagination
```

Session Update

```text
User Requests Next Page
```

Session Expiration

```text
TTL Reached
```

Session Deletion

```text
Automatic Cleanup
```

---

# Anchor Token Structure

Version 1

```json
{
  "version": 1,
  "documentId": "...",
  "sortValue": "...",
  "snapshotBoundary": "...",
  "collection": "..."
}
```

Encoded as:

```text
Base64 Token
```

---

# Complexity Analysis

## Initial Page

Time Complexity:

```text
O(log n)
```

---

## Forward Navigation

Time Complexity:

```text
O(k)
```

Where:

```text
k = page size
```

---

## Deep Navigation

Time Complexity:

```text
O(log n + k)
```

---

## Storage Complexity

Checkpoint Storage:

```text
O(n / interval)
```

Session Storage:

```text
O(active_sessions)
```

---

# Consistency Guarantees

The algorithm guarantees:

* No duplicate records
* No missing records
* Stable pagination window
* Snapshot isolation during navigation

---

# Benchmark Requirements

The implementation must compare:

1. Skip/Limit Pagination
2. MongoDB Cursor Pagination
3. SABP Pagination

Measured Metrics:

* Query Time
* CPU Usage
* Memory Usage
* Documents Scanned
* Throughput
* Average Latency

---

# Success Criteria

SABP is considered successful when:

* Query latency remains stable for large datasets.
* Deep navigation is significantly faster than skip/limit.
* Memory usage remains acceptable.
* Storage overhead remains minimal.
* Consistency is preserved under concurrent updates.

---

# Research Contribution

The proposed algorithm introduces:

1. Stateful anchor navigation.
2. Sparse checkpoint indexing.
3. Snapshot-based consistency.
4. Deep navigation optimization.
5. Hybrid stateful/stateless operation.

These contributions collectively form the SABP framework for scalable MongoDB pagination.
