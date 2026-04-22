# Canton Network Backend Patterns

Source: https://docs.digitalasset.com/build/3.4/sdlc-howtos/system-design/daml-app-arch-design.html

---

## Read Path — PQS (Participant Query Store)

### Architecture

```
Participant Node
  └── Private Contract Store (PCS)   ← on-ledger source of truth
        └── PQS (PostgreSQL)          ← queryable replica
              └── Backend API          ← enforces access control
                    └── Frontend / External System
```

### Key Points

- PQS is the **recommended read path** for all Canton Network applications.
- Query via **SQL over JDBC** — same as any standard web service backed by a relational database.
- PQS stores both **active contracts (ACS)** and **historical events** (create/archive/exercise).
- PQS does **not** enforce end-user access control — implement that in your API layer.
- Set PQS pruning window based on business needs and expected data volume.

### PQS vs Ledger API for Reads

| Capability | Ledger API | PQS |
|-----------|-----------|-----|
| Current ACS state | ✅ | ✅ |
| Filtering | Limited | ✅ Full SQL |
| Individual contract lookup | Limited | ✅ |
| Historical events | Streaming only | ✅ SQL query |
| App-specific indices | ❌ | ✅ (add your own) |
| Access control hooks | ❌ | In backend layer |

### Avoiding ACS Bloat

```
❌ Wrong: store completed workflow steps as active contracts
   → unbounded ACS growth → degraded query performance

✅ Correct: archive contracts when work is done
           use PQS event history as the golden source for:
           - transaction logs
           - notifications
           - reporting
```

### Reference Data & Explicit Disclosure

For contracts with many observers (pricing data, counterparty directories):

```
App Provider (stakeholder of reference contract)
  → shares contract payload out-of-band with App User
  → App User includes contract in transaction submission
  → Transaction processing succeeds without on-ledger visibility
```

This avoids the performance cost of maintaining visibility for all observers.

---

## Write Path — Reliability Patterns

### Pattern 1: Retry Handler (Shared Component)

Every write path through the backend should route through a retry handler:

```
Retry handler
  ├── Submit command to Ledger API
  ├── On retryable failure (network, timeout, conflict): retry up to N times
  ├── On permanent failure (invalid command): log + alert + dead-letter
  └── On success: return result
```

Package this as a shared library used by all backend services.

### Pattern 2: Idempotent Commands — Consuming Choice

The simplest idempotency technique: make the triggering contract consumed by the action.

```daml
-- The automation task consumes the request contract
-- If submitted twice, the second attempt fails because the contract is gone
choice ProcessOnboardingRequest : ContractId OnboardingResult
  controller provider
  do
    -- validate off-ledger KYC, then create result
    create OnboardingResult with ...
    -- The OnboardingRequest contract is archived by this consuming choice
```

### Pattern 3: Command Deduplication

Use the Ledger API's built-in deduplication for non-consuming commands:

```
commandId = deterministic_hash(business_transaction_id)

Submit(
  commandId = commandId,       -- unique per logical business action
  deduplicationPeriod = "PT1H" -- deduplicate within this window
)
```

The participant node stores `commandId` and rejects re-submissions within the deduplication period.

---

## Workflow Automation Patterns

### Pattern: State-Triggered Automation

```
┌─────────────────────────────────────────────────────────┐
│ Automation worker loop                                    │
│                                                           │
│  1. Poll PQS for pending tasks                           │
│     SELECT * FROM contracts WHERE template = 'Request'   │
│     AND status = 'pending' LIMIT batch_size              │
│                                                           │
│  2. For each task:                                        │
│     a. Re-query PQS at consistent offset (latest state)  │
│     b. Call off-ledger systems if needed (KYC, AML, …)  │
│     c. Submit consuming command to Ledger API             │
│     d. On failure: retry entire block (a → c)            │
│     e. On "already done / contract gone": exit           │
└─────────────────────────────────────────────────────────┘
```

**Critical rules:**
- Re-run the full query on every retry — don't cache ledger state between retries.
- Use a **consistent ledger offset** across all PQS queries in one task run.
- Retry the **entire task block**, not just the ledger command.
- The triggering contract must be **consumed** to prevent looping.

### Pattern: Time/Schedule Triggered Automation

```daml
-- In GCL or backend scheduler:
every 1 hour:
  tasks = pqs.query("SELECT * FROM rate_fixings WHERE date = today()")
  for task in tasks:
    ledger.submit(CreateRateFixingContract(task))
```

### Pattern: External Event Triggered Automation

```
External system (message queue, webhook, HTTP call)
  → Backend receives event
  → Backend creates/exercises contract on ledger
  → Backend acknowledges event only after successful ledger commit
```

**Do not acknowledge the external event before the ledger commit succeeds** — this ensures exactly-once processing.

---

## Off-Ledger Integration Patterns

### Push to Ledger (Ingest)

```
External system / message queue
  → Backend reads message
  → Backend creates on-ledger contract (reference data, oracle, etc.)
  → Acknowledge message only after ledger commit
```

### Pull from Ledger (Export)

```
PQS (SQL)
  → Backend reads events at increasing offset
  → Backend writes to external system (accounting, reporting DB, webhook)
  → Backend stores last processed offset (crash-recoverable)
```

### Pull-Based Consumption (Offset-Aware)

```
stored_offset = load_from_persistent_store()

loop:
  events = pqs.query("SELECT * FROM events WHERE offset > ?", stored_offset)
  process(events)
  stored_offset = max(events.offset)
  save_to_persistent_store(stored_offset)
```

**Keep read and write paths separate.** Route all off-ledger ↔ ledger communication through the backend service — never let off-ledger systems call the Ledger API directly.

---

## Reference Data API Pattern

When app users need reference data to submit Daml transactions:

```
App Provider                         App User
────────────                         ────────
Holds reference contract (e.g. FX rate)
                 ──out-of-band──►   Receives contract payload
                                    Includes contract in transaction
                                    Transaction processing succeeds
                                    (via Explicit Disclosure feature)
```

**API endpoint design:**
```
GET /api/reference-data/fx-rates?date=2025-01-15
Response: { contractId: "...", payload: {...} }

// App user includes this contract in their command submission
```

---

## Daml Codegen Integration (Java)

```bash
# Generate Java classes from DAR file
daml codegen java --input-file my-model.dar --output-dir src/generated/

# Generated classes provide:
# - Type-safe contract creation: MyContract.create(field1, field2)
# - Choice encoding: myContract.id.exerciseMyChoice(args)
# - Automatic encoding/decoding of Ledger API representation
```

Use the Ledger API Java client libraries together with generated classes:

```java
// Submit a create command
var createCmd = MyContract.create(party, value);
ledgerClient.getCommandClient().submitAndWait(
    "myApp", party, commandId, List.of(createCmd));

// Query via PQS
var contracts = jdbcTemplate.query(
    "SELECT payload FROM active_contracts WHERE template_id = ?",
    new Object[]{"MyModule:MyContract"},
    (rs, row) -> MyContract.fromJson(rs.getString("payload"))
);
```
