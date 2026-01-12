# Server Communication Model

## Overview

Server manages multiple EngineClients (ECs) and decides which EC should run which jobs.  
All coordination happens via a single `/engine/sync.json` endpoint:

- ECs send job status updates (including periodic result updates).
- Server stores results in database and responds with the set of jobs that should be running on that EC.
- When reassigning jobs, server checks database for existing results to enable continuation on different ECs.

## Server Process Flow

```mermaid
flowchart TD
    A[Sync Request] --> B[Update EC health<br/>Process statuses<br/>Return jobs list]
    
    C[Assignment Process<br/>most frequent] --> D{Unassigned jobs?}
    D -->|Yes| E{ECs available?}
    E -->|Yes| F[Assign by priority<br/>to least loaded EC]
    E -->|No| G[Keep pending]
    
    H[Timeout Process<br/>medium frequency] --> I{Jobs past deadline?}
    I -->|Yes| J{ECs available?}
    J -->|Yes| K[Reassign with top priority<br/>preserve result]
    J -->|No| G
    
    L[Rebalancer<br/>least frequent] --> M{Running jobs<br/>need rebalancing?}
    M -->|Yes| N{ECs available?}
    N -->|Yes| O[Reassign running jobs<br/>to balance load]
    N -->|No| G
```

---

## Server State Model

### EngineClient (EC)

- `ec_id`: unique identifier
- `last_sync_at`: timestamp of last successful sync
- **Connected**: `now - last_sync_at < 30s` (eligible for assignments)
- **Disconnected**: `now - last_sync_at >= 30s` (jobs may be reassigned)

### Job

- `job_id`: unique identifier
- `assigned_ec_id` (nullable): current EC assignment
- `state`: `pending` | `assigned` | `running` | `error` | `finished`
- `assigned_at`: last assignment timestamp
- `start_confirm_deadline`: deadline for EC to confirm start (`status: "completed"`)
- `last_result` (nullable): last result from any EC (preserved on reassignment)
- `created_at`: creation timestamp
- `priority`: higher number = higher priority (reassignments get top priority)

**Assignment Order**: Priority (descending) → `created_at` (ascending)

**Queue Position**: Count of `pending` jobs with higher priority, or same priority but earlier `created_at` (calculated on-demand)

---

## Sync Endpoint

`POST /engine/sync.json?engineId={ec_id}`  
Headers: `Authorization: Bearer {auth_token}`

**Request** (from EC):
```json
{
  "job_statuses": [
    {
      "job_id": "...",
      "status": "running" | "completed" | "error",
      "result": "base64_compressed_text",
      "error_message": "..."
    }
  ]
}
```

**Response**: List of jobs that should be running on this EC (server determines "new" jobs by checking client's `job_statuses`).

---

## Periodic Processes

### 1. Assignment Process (most frequent)

Handles jobs not yet assigned to an EngineClient:

- Find `pending` jobs sorted by priority → `created_at`
- Assign to least loaded connected EC (preserve `last_result` if exists)
- Jobs remain `pending` if no connected ECs available

### 2. Timeout Process (medium frequency)

Checks if assigned jobs have been received by their EC:

- `state = "assigned"` + `now > start_confirm_deadline`: Reassign with top priority (preserve `last_result` if exists)
- `state = "running"` + disconnected EC: Reassign with top priority (preserve `last_result`)

### 3. Rebalancer Process (least frequent)

Optimizes distribution of running jobs:

- Recompute load for all connected ECs (running job count per EC)
- Reassign `running` jobs from overloaded to underloaded ECs (preserve `last_result`)
- Reassignments get top priority

All processes use **atomic state updates** to avoid conflicts with normal request handling.

---

## Job Failure Handling

```mermaid
sequenceDiagram
    participant Server
    participant EC as EngineClient

    Server->>EC: Job in sync response
    alt Timeout (not received/started)
        EC--xServer: No 'completed' before deadline
        Server->>Server: Reassign with top priority
    else Invalid parameters
        EC->>Server: status="error"
        Server->>Server: Mark as error (no reassign)
    end
```

**Failure modes**:
- **Timeout**: No `status: "completed"` before deadline → reassigned
- **Error**: EC reports `status: "error"` → marked failed, not reassigned
