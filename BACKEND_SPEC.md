# PowerSync Server Architecture & Protocol Specification

This document provides a reverse-engineered specification of the generic **PowerSync Sync Server** protocol, extracted directly from the Dart client SDK (`packages/powersync/lib/`).

The purpose of this document is to serve as a blueprint for implementing a compatible PowerSync server instance in Rust, completely independent of any specific demo application or user schema.

---

## 1. Core Architecture

The PowerSync Server is responsible for mediating between the client's local SQLite database and the authoritative backend database (typically PostgreSQL via logical replication).

The server must implement two primary sync mechanisms for the client SDK:
1.  **Streaming Sync (Read):** A persistent, unidirectional HTTP stream sending BSON or NDJSON encoded events from the server to the client.
2.  **Write Checkpoints (Consistency):** A standard REST endpoint allowing the client to verify that its offline queued writes have been processed by the backend database and will be reflected in the sync stream.

*Note: The client SDK relies entirely on external user-defined REST/GraphQL APIs to perform mutations (inserts, updates, deletes). The PowerSync server itself is predominantly read-only from the perspective of the sync protocol.*

---

## 2. API Contract Extraction

Based on the SDK `StreamingSyncImplementation`, the server MUST expose the following endpoints.

### 2.1 The Sync Stream Endpoint

This is the core of the PowerSync protocol. The client opens a long-lived HTTP POST request to receive database updates.

**Endpoint:** `POST /sync/stream`

**Headers Required by Client:**
*   `Authorization`: `Token <token>` (Provided by the client's backend connector)
*   `Accept`: `application/vnd.powersync.bson-stream;q=0.9,application/x-ndjson;q=0.8`
*   `Content-Type`: `application/json`
*   `User-Agent` / `X-User-Agent`: (e.g., `powersync-dart-core/1.0.0 Dart/3.1.0 ...`)

**Request Body (JSON):**
When the client initiates the stream, it sends its current state and sync requirements:

```json
{
  "app_metadata": { "app_version": "1.0.1" },
  "parameters": { "client_params": "..." },
  "schema": {
    "tables": [
      {
        "name": "lists",
        "columns": [{"name": "name", "type": "TEXT"}],
        "indexes": [],
        "local_only": false,
        "insert_only": false
      }
    ]
  },
  "include_defaults": true,
  "active_streams": [
    {
      "name": "lists",
      "params": {}
    }
  ]
}
```

**Server Response:**
*   **Status:** `200 OK`
*   **Content-Type:** `application/vnd.powersync.bson-stream` (or `application/x-ndjson`)
*   **Body:** A continuous, chunked stream of BSON or NDJSON encoded instructions.

**BSON Stream Format (`_BsonSplittingSink`):**
If responding with BSON, each message sent by the server MUST be prefixed with a 4-byte little-endian integer representing the length of the BSON document (including the 4-byte length header itself), followed by the binary BSON payload.

### 2.2 The Write Checkpoint Endpoint

When a client successfully uploads local mutations to their backend API, it pings the PowerSync server to retrieve a `write_checkpoint`. This tells the client "Wait until the sync stream reaches this checkpoint before assuming the local mutation has been fully replicated."

**Endpoint:** `GET /write-checkpoint2.json?client_id=<client_id>`

**Headers Required by Client:**
*   `Authorization`: `Token <token>`

**Response:**
*   **Status:** `200 OK`
*   **Content-Type:** `application/json`
*   **Body:**
```json
{
  "data": {
    "write_checkpoint": "9223372036854775807"
  }
}
```
*(The checkpoint is a string representation of an integer sequence number or transaction ID).*

---

## 3. Server-to-Client Instruction Protocol

The data sent over the `/sync/stream` endpoint consists of discrete instructions. The client's SQLite extension parses these. Based on the Dart SDK's `Instruction.fromJson` parser, the Rust server must emit JSON/BSON objects with exactly one of the following root keys.

### 3.1 `UpdateSyncStatus`
Notifies the client of the current download progress across sync buckets.

```json
{
  "UpdateSyncStatus": {
    "status": {
      "connected": true,
      "connecting": false,
      "priority_status": [
        {
          "priority": 0,
          "has_synced": true,
          "last_synced_at": 1680000000
        }
      ],
      "downloading": {
        "buckets": {
          "bucket_name": {
            "priority": 0,
            "at_last": 100,
            "since_last": 10,
            "target_count": 150
          }
        }
      },
      "streams": [
        {
          "name": "lists",
          "parameters": null,
          "priority": 0,
          "progress": { "total": 100, "downloaded": 100 },
          "active": true,
          "is_default": true,
          "has_explicit_subscription": true,
          "expires_at": null,
          "last_synced_at": 1680000000
        }
      ]
    }
  }
}
```

### 3.2 `LogLine`
Sends a debug or info string to the client's internal logger.

```json
{
  "LogLine": {
    "severity": "INFO",
    "line": "Sync stream connected successfully"
  }
}
```
*(Supported severities: `DEBUG`, `INFO`, `WARNING`, `ERROR`)*

### 3.3 `FetchCredentials`
Forces the client to refresh its JWT token. This is typically sent if the server detects the token is nearing expiration during a long-lived streaming connection.

```json
{
  "FetchCredentials": {
    "did_expire": false
  }
}
```

### 3.4 `CloseSyncStream`
Instructs the client to gracefully terminate the stream. If `hide_disconnect` is true, the client immediately attempts to reconnect without throwing errors to the UI.

```json
{
  "CloseSyncStream": {
    "hide_disconnect": true
  }
}
```

### 3.5 `EstablishSyncStream`
Sent as a handshake confirmation or to instruct the SQLite extension internally.
```json
{
  "EstablishSyncStream": {
    "request": {}
  }
}
```

### 3.6 Sync Operations (`FlushFileSystem`, `DidCompleteSync`)
Signals that a synchronization cycle has concluded.
```json
{ "FlushFileSystem": {} }
```
```json
{ "DidCompleteSync": {} }
```

### 3.7 Database Mutation Payloads (Sync Data)
While `Instruction.fromJson` handles control messages, the core extension also processes lines containing actual database row data.
The Rust server streams Postgres logical replication deltas translated into row operations (`PUT`, `PATCH`, `DELETE`). The core extension intercepts these via the `powersync_control('line_binary', ...)` or `powersync_control('line_text', ...)` functions.

*(Note: The exact schema of the row mutation payload is handled internally by the PowerSync `sqlite3` C extension. The server streams operations targeting `ps_data__<tablename>`).*

---

## 4. Sync Engine Mechanics

### Authentication and Reconnection
1.  **JWT Validation:** The server must validate the JWT token sent in the `Authorization: Token <jwt>` header on every connection to `/sync/stream` or `/write-checkpoint2.json`.
2.  **Connection Errors:** If the server returns HTTP `401 Unauthorized`, the client automatically invalidates its token cache, fetches a new token via `SupabaseConnector`, and retries.
3.  **Backoff:** If the stream drops or returns `5xx`, the client SDK (`StreamingSyncImplementation`) retries automatically with a configurable `retryDelay` (default 5 seconds).

### Client Upload Queue (CRUD)
The server **does not** handle client uploads directly through the sync stream.
1.  Clients record mutations offline in the `ps_crud` SQLite table.
2.  When online, the client executes an HTTP `POST / PATCH / DELETE` to the user's standard backend API.
3.  The client then calls `/write-checkpoint2.json`.
4.  The server's response (`write_checkpoint`) allows the client to locally verify when the sync stream has caught up to the mutation.

---

## 5. Rust Service Design Architecture

Based on these extracted requirements, the Rust backend must be structured to maintain massive numbers of long-lived, concurrent streaming connections while monitoring a Postgres logical replication slot.

### Axum + Tokio Architecture

```text
src/
├── main.rs                 # Entry point, Tokio runtime, Axum router
├── config.rs               # Environment and JWT secrets
├── server/
│   ├── mod.rs
│   ├── auth.rs             # JWT parsing and validation middleware
│   ├── checkpoints.rs      # GET /write-checkpoint2.json handler
│   └── sync_stream.rs      # POST /sync/stream handler (Long-lived BSON streams)
├── replication/
│   ├── mod.rs              # Connects to PostgreSQL logical replication slot
│   ├── pg_decoder.rs       # decodes WAL messages into logical operations
│   └── wal2bson.rs         # Translates Postgres operations into PowerSync BSON
├── buckets/
│   ├── mod.rs              # Manages Sync Rules and Buckets
│   └── filtering.rs        # Filters replication events based on Client JWT and Sync Rules
└── state/
    └── client_manager.rs   # Tracks active streaming connections and pushes events to channels
```

### Core Responsibilities of the Rust Server
1.  **Postgres Logical Replication:** Maintain a connection to PostgreSQL using `pgoutput` plugin. Decode INSERT/UPDATE/DELETE events.
2.  **Sync Rules Evaluation:** Evaluate the `active_streams` and `schema` defined by the client in the `/sync/stream` payload against the decoded replication events.
3.  **BSON Streaming:** Serialize the filtered database operations into BSON. Prefix each BSON document with its 4-byte `i32` length.
4.  **Instruction Injection:** Inject `UpdateSyncStatus` and `DidCompleteSync` BSON documents into the stream to keep the client's progress UI updated.
5.  **Write Checkpoints:** Track the current replication LSN (Log Sequence Number) and serve it via `/write-checkpoint2.json`.