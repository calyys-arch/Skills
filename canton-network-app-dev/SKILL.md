---
name: canton-network-app-dev
description: "Canton Network (Digital Asset / Daml) application development reference. Covers app architecture design (frontend, Daml model, backend), read/write path patterns, PQS (Participant Query Store) integration, automation of on-ledger workflows, off-ledger integration, JWT/OAuth2 authorization, ACS size management, latency/throughput optimization, and Daml smart contract design patterns. Use when developing, designing, or reviewing Canton Network or Daml applications, when the user mentions Daml, Canton, Ledger API, PQS, participant node, DAR files, or on-ledger workflows."
---

# Canton Network Application Development

[Official docs](https://docs.digitalasset.com/build/3.4/) — Digital Asset SDK 3.4

## Key Concepts

| Term | Definition |
|------|-----------|
| **App Provider** | Organization that builds and operates the Canton Network application |
| **App User** | Organization that uses the application; operates their own participant node |
| **End-user** | Human acting on behalf of an App User (employee, trader, etc.) |
| **Participant Node** | Stores and validates the ledger shard visible to a party (= Validator Node) |
| **Synchronizer Node** | Coordinates commits across participants (no double-spends) |
| **Daml Party** | Identifier for cryptographic keys; unit of access control on the ledger |
| **DAR file** | Compiled Daml package — must be uploaded to all relevant participant nodes |
| **ACS** | Active Contract Set — all created but not yet archived contracts |
| **PQS** | Participant Query Store — SQL-queryable operational store of ledger data |
| **Ledger API** | gRPC API exposed by participant nodes; also available as JSON over HTTP |

## Application Components

Every Canton Network application has three logical components:

```
Frontend  →  Backend  →  Ledger API  →  Participant Node
                ↕
           Daml Model (DAR files on each participant node)
```

1. **Frontend** — UI for end-users; submits commands to the *backend*, not directly to the ledger. Each organization should host its own frontend (non-repudiation). Login via that org's IAM.

2. **Daml Model** — defines cross-org workflows and serves as the API contract. Compiled to DAR files, uploaded to all participant nodes.

3. **Backend** — sole direct consumer of the Ledger API. Three responsibilities:
   - Provision higher-level APIs
   - Automate on-ledger workflows
   - Integrate with off-ledger systems

## Read Path — Use PQS

```
Participant Node (PCS) → PQS (PostgreSQL) → Backend API → Frontend
```

- **Always use PQS for reads** — query via SQL over JDBC.
- PQS gives access to both ACS (current state) and ledger history (within pruning window).
- PQS does **not** enforce access control — implement in your backend API layer.
- For very high-scale specialized reads, build a custom ODS on top of PQS.
- Avoid storing completed workflow data as active contracts — use PQS history instead.

Full PQS patterns → [references/backend-patterns.md](references/backend-patterns.md)

## Write Path — Reliability First

Two non-negotiable requirements on the write path:

1. **Retry on failure** — package retry logic into a shared component used by all writers.
2. **Idempotency** — commands must be safe to resubmit. Two techniques:
   - Make commands **consuming** (exercise a consuming choice on the triggering contract).
   - Use **command deduplication** (participant stores command ID, de-dupes re-submissions).

```
Backend → submit command with unique commandId → Ledger API
        ← success | retryable failure (retry) | permanent failure (log + alert)
```

## Workflow Automation

Daml contracts are **passive** — workflows only advance when external components act.

**State-triggered automation pattern**:
```
PQS poll for new tasks
  → build command from most-recent PQS snapshot
  → submit to Ledger API (with retry + deduplication)
  → consuming choice on trigger contract avoids looping
```

Key rules:
- Re-query PQS on every retry (state may have changed).
- Use a consistent ledger offset across multiple queries in one task run.
- Retry the **entire task block**, not just the ledger command.
- Exit retry loop when the task is already completed or no longer valid.

Full automation & off-ledger integration → [references/backend-patterns.md](references/backend-patterns.md)

## Architecture Options

Three patterns (choose based on app-user requirements):

| Pattern | Who runs backend | Non-repudiation | App-user integration | Engineering cost |
|---------|-----------------|-----------------|---------------------|-----------------|
| **A** Provider-only backend | App Provider | ✅ | ❌ | Lowest |
| **B** Provider builds, user operates | App User | ✅ | ✅ | Medium |
| **C** Each org builds own | App User | ✅ | ✅ (custom) | Highest |

**Recommendation**: start with Pattern A; add complexity only when business requirements demand it.

Full architecture trade-offs → [references/architecture.md](references/architecture.md)

## Tech Stack

- **Any language** that supports HTTP can use the JSON Ledger API.
- Use **Daml Codegen** (Java / JS) for typed contract encoding/decoding.
- Use standard enterprise patterns for IAM integration (OAuth 2.0 / JWT).
- Background processing: standard job queue / scheduler.
- Database: PostgreSQL (PQS runs on Postgres).

## Authorization Quick Reference

```
IAM (Keycloak / etc.)
  → issues JWT access token
  → app attaches token to every Ledger API request

Participant node verifies:
  - issued by trusted token issuer
  - not tampered with
  - not expired
  - carries sufficient rights (canActAs / canReadAs / participant_admin)
```

Prefer **audience-based JWT** (`aud` = participant ID). Use short-lived tokens (5–15 min).

Full JWT format + rights table → [references/security-performance.md](references/security-performance.md)

## Performance Quick Reference

| Problem | Solution |
|---------|---------|
| Large ACS slows queries | Archive auxiliary contracts promptly; schedule pruning |
| High transaction latency | Minimize `create`/`fetch`/`exercise` actions; batch commands |
| Contention on shared contracts | Batch multiple updates in one transaction; implement queuing backend |
| Memory / CPU in Daml code | Use Daml Profiler; skip intermediary contract states |

Full optimization patterns → [references/security-performance.md](references/security-performance.md)

## Common Pitfalls

| Wrong | Correct |
|-------|---------|
| Frontend talks directly to Ledger API | Frontend → backend → Ledger API |
| Storing completed workflow steps as ACS | Use PQS history as source of truth |
| Retrying only the ledger command | Retry the entire task block |
| Using `!!` non-null assert after null guard | Control flow narrowing removes need |
| Reading ACS for every query | Query PQS with application-specific index |
| Long-lived JWTs (hours/days) | Issue short-lived tokens (5–15 min) |
| Accessing PCS / Ledger API directly from off-ledger systems | Route through backend service |
