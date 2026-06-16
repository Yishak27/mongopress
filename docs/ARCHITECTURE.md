# SABP Framework Architecture

## Overview

Stateful Anchor-Based Pagination (SABP) is a middleware framework that provides scalable pagination and deep navigation for MongoDB datasets.

The framework sits between the application layer and MongoDB, providing efficient data navigation while avoiding expensive offset-based pagination.

---

# High-Level Architecture

```text
Application Layer
       │
       ▼
┌──────────────────────────┐
│ SABP Framework           │
├──────────────────────────┤
│ Anchor Engine            │
│ Checkpoint Manager       │
│ Query Builder            │
│ Snapshot Manager         │
│ Session Manager          │
│ Benchmark Module         │
└──────────────────────────┘
       │
       ▼
MongoDB Driver
       │
       ▼
MongoDB Database
```

---

# Component Architecture

## Anchor Engine

Purpose:

* Generate anchor tokens
* Validate anchors
* Encode anchor metadata
* Decode anchor metadata

Responsibilities:

* Anchor creation
* Anchor navigation
* Boundary validation

Inputs:

```json
{
  "documentId": "...",
  "sortValue": "...",
  "snapshotId": "..."
}
```

Outputs:

```json
{
  "anchorToken": "..."
}
```

---

## Checkpoint Manager

Purpose:

Provide deep navigation support.

Responsibilities:

* Create sparse checkpoints
* Lookup nearest checkpoint
* Maintain checkpoint metadata

Example:

```text
Checkpoint_1 -> Position 1,000
Checkpoint_2 -> Position 2,000
Checkpoint_3 -> Position 3,000
```

Storage Collection:

```text
sabp_checkpoints
```

---

## Query Builder

Purpose:

Generate optimized MongoDB queries.

Responsibilities:

* Forward pagination query generation
* Backward pagination query generation
* Deep navigation query generation

Example:

```javascript
{
  createdAt: {
    $lt: anchor.createdAt
  }
}
```

---

## Snapshot Manager

Purpose:

Ensure pagination consistency.

Responsibilities:

* Generate snapshot boundaries
* Prevent duplicate records
* Prevent missing records
* Handle concurrent insertions

Example:

```json
{
  "snapshotBoundary": "ObjectId(...)"
}
```

---

## Session Manager

Purpose:

Maintain stateful navigation.

Responsibilities:

* Session creation
* Session update
* Session recovery
* Session expiration

Session storage options:

* Redis
* MongoDB TTL Collection
* In-memory adapter

---

## Benchmark Module

Purpose:

Measure algorithm performance.

Metrics:

* Query execution time
* CPU usage
* Memory usage
* Throughput
* Documents scanned
* Average latency

---

# Data Flow

## Initial Request

```text
Client Request
      │
      ▼
Anchor Engine
      │
      ▼
Query Builder
      │
      ▼
MongoDB
      │
      ▼
Results
      │
      ▼
Anchor Generation
      │
      ▼
Client Response
```

---

## Deep Navigation Request

```text
Requested Position
        │
        ▼
Checkpoint Lookup
        │
        ▼
Nearest Checkpoint
        │
        ▼
Relative Offset Calculation
        │
        ▼
Optimized Retrieval
        │
        ▼
Response
```

---

# Storage Architecture

## Persistent Storage

### Checkpoints Collection

```json
{
  "_id": "...",
  "collectionName": "users",
  "position": 2300000,
  "documentId": "...",
  "sortValue": "...",
  "createdAt": "..."
}
```

Purpose:

* Deep navigation
* Anchor lookup

---

## Temporary Storage

### Sessions Collection

```json
{
  "_id": "...",
  "sessionId": "...",
  "currentAnchor": "...",
  "snapshotBoundary": "...",
  "expiresAt": "..."
}
```

TTL Index:

```javascript
{
  expiresAt: 1
}
```

---

# Framework Modes

## Stateless Mode

State stored on client.

Advantages:

* No server storage
* Horizontal scalability

---

## Stateful Mode

State stored in Redis or MongoDB.

Advantages:

* Session recovery
* Strong consistency
* Navigation persistence

---

# Benchmark Architecture

```text
┌───────────────────┐
│ Benchmark Runner  │
└─────────┬─────────┘
          │
          ▼
 ┌─────────────────┐
 │ Test Controller │
 └───────┬─────────┘
         │
 ┌───────┴────────┐
 ▼                ▼

Skip/Limit     SABP

 ▼                ▼

MongoDB Dataset
```

---

# Scalability Design

Target Dataset Sizes:

* 100K Records
* 1M Records
* 5M Records
* 10M Records

Target Concurrent Users:

* 1,000+
* 10,000+
* 100,000+

---

# Future Extensions

* Adaptive checkpoint density
* Multi-node synchronization
* Distributed cache support
* Predictive prefetching
* AI-assisted checkpoint optimization
