# Security & Performance Reference

Sources:
- Authorization: https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/secure/authorization.html
- Latency/Throughput: https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/optimise/latency-and-throughput.html
- ACS: https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/optimise/active-contract-set.html

---

## Authorization

### Flow Overview

```
1. App sends credentials to IAM (Keycloak, Auth0, etc.)
2. IAM verifies identity, returns signed JWT
3. App attaches JWT to every Ledger API request
4. Participant node verifies: issuer trusted? signature valid? not expired? rights sufficient?
```

### JWT Token Rights

| Right | Grants |
|-------|--------|
| `public` | Read publicly available info (ledger identity, packages) |
| `participant_admin` | Administer participant node (parties, users, pruning) |
| `idp_admin` | Administer users/parties for one identity provider |
| `canReadAs(p)` | Read contracts and events visible to party `p` |
| `canActAs(p)` | `canReadAs(p)` + submit commands on behalf of party `p` |

### Ledger API Service → Required Right

| Service | Endpoint | Right |
|---------|----------|-------|
| StateService | GetActiveContracts | `canReadAs(p)` for each `p` |
| CommandService | All | `canActAs(p)` for submitting `p` |
| CommandSubmissionService | Submit | `canActAs(p)` |
| UpdateService | All (except LedgerEnd) | `canReadAs(p)` |
| PackageService | All | `public` |
| PartyManagementService | All | `participant_admin` |
| UserManagementService | All | `participant_admin` or `idp_admin` |
| Health | All | No token required |

### Audience-Based JWT Format (preferred)

```json
{
  "aud": "https://daml.com/jwt/aud/participant/someParticipantId",
  "sub": "someUserId",
  "iss": "someIdpId",
  "exp": 1300819380
}
```

- `aud` — restricts token to specific participant ID
- `sub` — participant user ID
- `iss` — identity provider ID
- `exp` — expiry (seconds since epoch)

### Scope-Based JWT Format (alternative)

```json
{
  "aud": "someParticipantId",
  "sub": "someUserId",
  "iss": "someIdpId",
  "exp": 1300819380,
  "scope": "daml_ledger_api"
}
```

### Security Best Practices

- **Short-lived tokens** (5–15 min) — JWT cannot be revoked; short lifetime limits theft window.
- **Rotate signing keys** if a token is compromised (invalidates all outstanding tokens via JWKS).
- **User IDs** — alphanumeric ASCII + `@^$.!\`-#+'~_|:()`, max 128 chars.
- **Mutual TLS** — client certificate + JWT; note: certificate CN does not prove application identity.
- **Dynamic rights** — use `UserManagementService` to change user rights without re-issuing tokens.

### Identity Provider Configuration

```
Default IDP: id = "" (empty string)
Custom IDPs: configure via IdentityProviderConfigService
  - supply non-empty IDP id
  - supply JWKS URL for token verification

Access tokens for non-default IDPs must include iss field = IDP id
```

---

## Performance: Latency & Throughput

### Transaction Processing Steps (Bottleneck Map)

```
Command submitted by client
  → Interpretation (submitting node: CPU, DB reads, computation)
  → Blinding (submitting node: CPU/memory, number of views)
  → Submission to sequencer (serialization, transaction size)
  → Sequencing (sequencer storage, network bandwidth)
  → Validation (receiving nodes: network, CPU, DB reads)
  → Confirmation (validating nodes + sequencer: network, number of confirming parties)
  → Commit (mediator + sequencer: CPU, DB, network)
```

**Bottlenecks in order of likelihood:**
1. Large transaction size → high serialization/deserialization cost
2. Large transaction → sequencer backend overload (especially blockchains)
3. High computation complexity or large memory use in Daml code
4. Large number of involved nodes → network bandwidth

**Baseline latency**: 0.5–1 second on DB sequencers; several seconds on blockchain sequencers.

### Minimize Transaction Size

Each of these Daml operations adds a node to the transaction tree (with contract payload):
`create`, `fetch`, `fetchByKey`, `lookupByKey`, `exercise`, `exerciseByKey`, `archive`

**Skip intermediary states — write only the end state:**

```daml
-- BAD: creates N-1 intermediary contract versions
choice BadIncrementMany : ContractId Incrementor
  with m : Int
  controller p
  do foldlA (\self' _ -> exercise self' Increment) self [1..m]

-- GOOD: single create with final value
choice GoodIncrementMany : ContractId Incrementor
  with m : Int
  controller p
  do create this with n = n + m
```

**Bundle multiple actions on the same contract into one:**

```daml
-- BAD: fetches each contract 3 times (validate, extract, archive)
choice BadMerge : ContractId Asset
  with otherCids : [ContractId Asset]
  -- fetches for validation, for quantity, then archives = 3x per contract

-- GOOD: consuming fetch → archive + return in one action
choice ConsumingFetch : Asset
  controller owner
  do return this

choice GoodMerge : ContractId Asset
  with otherCids : [ContractId Asset]
  controller owner
  do
    others <- forA otherCids (`exercise` ConsumingFetch)  -- 1x per contract
    ...
```

### Batching Commands

```daml
-- WITHOUT batching: 10 ledger transactions
cid1 <- submit p do createCmd T with ..
cid2 <- submit p do createCmd T with ..
-- ... 8 more submits ...

-- WITH batching: 2 ledger transactions
cids <- submit p do replicateA 5 $ createCmd T with ..
submit p do forA_ cids (`exerciseCmd` Foo)
```

Trade-off: batching slightly increases latency and command failure probability, but dramatically improves throughput.

### CPU & Memory

- Use the [Daml Profiler](https://docs.digitalasset.com/build/3.4/component-howtos/smart-contracts/profiler.html) to identify hot spots in Daml code.
- Monitor JVM heap (Java backend) and database performance.
- Scale up machine when Daml interpretation is verified not to be the bottleneck.

---

## Performance: ACS (Active Contract Set) Size

### Impact of Large ACS

| Impact | Description |
|--------|-------------|
| Slow ACS initialization | Hours to transfer full ACS via StateService |
| Index size exceeds shared buffers | Increasing query latency for contract lookups, command submission |
| Write amplification | Full-page writes grow with ACS volume |
| Checkpoint cost | Prolonged disk flushes during PostgreSQL checkpointing |

ACS issues become noticeable at hundreds of GB to TB of ACS data in PostgreSQL.

### Solutions

1. **Archive auxiliary contracts promptly** — ensure supporting/auxiliary contracts are archived as soon as they are no longer needed.

2. **Frequent pruning schedule:**
   ```
   -- Pruning only affects archived contracts
   -- If all contracts are active, pruning has limited effect
   -- Use participant node pruning documentation for schedule setup
   ```

3. **Implement ODS (Operational Data Store):**
   ```
   If ACS initialization time exceeds acceptable limits:
   → Build a custom ODS in your backend
   → Populate from PQS event stream (offset-aware, recoverable)
   → Query ODS instead of ACS for application reads
   ```

4. **Database monitoring:**
   - Monitor disk read/write patterns — sudden increase in reads = indices no longer in shared buffers.
   - Monitor query performance (use `log_min_duration_statement` in PostgreSQL).
   - Enable `autovacuum` — after pruning, large number of dead tuples need removal.

### Daml Model Design to Limit ACS Growth

```daml
-- BAD: store completed transaction records as active contracts
template CompletedTrade
  with buyer: Party; seller: Party; amount: Decimal
  where signatory [buyer, seller]
  -- These never get archived → unbounded ACS growth

-- GOOD: archive when complete; use PQS history for records
template Trade
  with buyer: Party; seller: Party; amount: Decimal; status: Status
  where
    signatory [buyer, seller]
    choice Complete : ()
      controller buyer
      do return ()
      -- archive implicit in consuming choice → removed from ACS
      -- event stored in PQS history
```

---

## Contention

### Avoid Contention

- **Identify shared resources** — contracts that many parties exercise simultaneously.
- **Split aggregates** — instead of one large shared contract, use many smaller ones per sub-entity.
- **Use non-consuming choices** where possible — they don't block concurrent access.

### Reduce Contention via Backend Batching

When many end-users need to act on the same on-ledger resource (e.g. company account):

```
Multiple trader requests → Backend queue
                         → Backend batches N requests into one Daml transaction
                         → Single exercise on the shared contract
                         → Much higher throughput than sequential submission
```

```daml
choice BatchAllocate : ContractId Account
  with allocations : [(Party, Decimal)]
  controller owner
  do
    let total = sum (map snd allocations)
    -- Single debit from account for all allocations
    create this with balance = balance - total
```
