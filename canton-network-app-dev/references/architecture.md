# Canton Network Application Architecture

Source: https://docs.digitalasset.com/build/3.4/sdlc-howtos/system-design/daml-app-arch-design.html

## Component Overview

```
                    ┌─────────────────────────────────────────┐
  App Provider org  │  Frontend (optional for B2B)            │
                    │  Backend (always)                        │
                    │  Daml Model (DAR on all nodes)           │
                    └──────────────┬──────────────────────────┘
                                   │ Ledger API
                    ┌──────────────▼──────────────────────────┐
                    │  Participant Node (Validator Node)        │
                    │  Private Contract Store (PCS)            │
                    │  Participant Query Store (PQS / Postgres) │
                    └──────────────┬──────────────────────────┘
                                   │ Global Synchronizer
                    ┌──────────────▼──────────────────────────┐
  App User org      │  Participant Node                        │
                    │  Backend (Pattern B/C only)              │
                    │  Frontend (always)                       │
                    └─────────────────────────────────────────┘
```

---

## Architecture Pattern A — App Provider Operates Backend

**Who runs what:**
- App Provider: Frontend + Backend
- App User: Frontend only (IAM → participant node directly)

**Suitable when:**
- App users need no off-ledger system integration
- No automated on-ledger workflows required on the user side
- Simplest to build and deploy

**Non-repudiation:** ✅ Preserved — each org's frontend authenticates via its own IAM and submits to its own participant node.

**Limitation:** Backend can only submit commands on behalf of the App Provider's Daml parties. App-user Daml parties submit exclusively through the frontend.

```
App User Frontend ──► App User Participant Node
                       (reads from Provider Backend API)

App Provider Backend ──► App Provider Participant Node
                          (automation + higher-level APIs)
```

---

## Architecture Pattern B — App User Operates Backend Built by Provider

**Who runs what:**
- App Provider: builds + operates Provider backend; packages User backend for deployment
- App User: operates the Provider-supplied User backend + their own frontend

**Additional benefits over Pattern A:**
- Self-sovereign queries over app data (feeds from user's own PQS)
- Off-ledger system integration (KYC, AML, pricing feeds, etc.)
- Batched access to contended resources (e.g. multiple traders on same account)
- Fine-grained end-user permission management

**Disadvantages:**
- App user operating costs (monitoring, maintenance)
- Multi-version deployments risk when users delay upgrades
- App Provider must function as on-prem software vendor (support, release management)

---

## Architecture Pattern C — Each Organization Builds Its Own

**Who runs what:** Every organization builds and operates its own frontend and backend.

**Additional benefits:**
- Full customization of backend and frontend
- Lower software supply chain risk (less third-party code)

**Disadvantages:**
- High development cost for app users
- Cross-organization coordination on every workflow change
- Restricted app evolution (users may not update their code)

---

## Architecture Properties Summary

| Property | Pattern A | Pattern B | Pattern C |
|----------|-----------|-----------|-----------|
| Non-repudiation | ✅ | ✅ | ✅ |
| App-user data sovereignty | ✅ | ✅ | ✅ |
| Self-sovereign queries | ❌ | ✅ | ✅ |
| Off-ledger integration (user) | ❌ | ✅ | ✅ |
| Backend customization (user) | ❌ | Limited | ✅ |
| App-user engineering effort | Low | Medium | High |
| App-user operating cost | Low | Medium | High |
| App Provider release complexity | Low | Medium (on-prem) | High (coordination) |
| Multi-version risk | None | Medium | High |

---

## Daml Model Design Principles

1. **DAR files must be uploaded to every participant node** that will process the workflow.

2. **Avoid storing completed workflow steps as active contracts** — this causes unbounded ACS growth. Use PQS history instead.

3. **Reference data with many observers is a performance anti-pattern.** Use Explicit Disclosure:
   - Contract stakeholder shares contract out-of-band with the submitter.
   - Submitter includes the contract in their transaction submission.
   - Allows transaction access without the submitter being a stakeholder.

4. **Non-consuming exercise events** can represent workflow events / notifications without creating persistent state.

5. **ABI compatibility** — never add non-nullable fields to an existing type. Make new fields nullable to avoid requiring all participants to redeploy simultaneously.

---

## Frontend Design Principles

- Frontend must be hosted by the **organization whose IAM issues the access token**.
- Frontend talks to the **backend**, never directly to the Ledger API.
- The IAM issues access tokens; backend attaches them to Ledger API calls.
- Non-repudiation requires each org to host its own frontend on a server it controls.

---

## Key Takeaways (from official docs)

> "A Canton Network application typically requires three components: an app frontend, Daml models, and app backends. Daml models need to be deployed on the app provider's and each app user's participant node."

> "Favor an architecture that minimizes software engineering and operational effort for app users while meeting their requirements. This maximizes total addressable market."

> "App architecture can evolve over time. Start with a simpler architecture that provides the minimum required set of properties."
