# Plan: Enforce Per-User Dollar Budgets with Azure API Management

## 1) Purpose and relationship to usage monitoring

Extend [usage-in-dollars.md](./usage-in-dollars.md) so Azure API Management (APIM) allows an incoming model request only when the authenticated user has enough remaining budget.

Reuse that plan's identity, deployment catalog, versioned price card, normalized usage meters, Event Hub, ADX / Fabric Eventhouse, capacity allocation, and billed reconciliation. Add a **synchronous budget authorization path**; do not query ADX, Log Analytics, or Cost Management for each incoming request. Their ingestion and billing delays make them unsuitable for enforcing a spending limit.

This is an implementation plan, not a deployed solution. No Azure resources or executable policies are created by this document.

### Recommended decisions

| Concern | Recommendation |
|---|---|
| Enforcement point | APIM inbound policy, before forwarding to Foundry |
| Fast data service | **Azure Cosmos DB for NoSQL**, using point reads and atomic, conditional transactions within one logical partition |
| Budget service | Private HTTPS Azure Function on an always-ready hosting configuration, colocated with APIM and Cosmos DB |
| Budget scope | One USD budget per authenticated individual across all models and sessions |
| Initial period | Calendar month in UTC; configurable by an administrator before rollout |
| Admission accounting | Settled enforcement charges plus outstanding request reservations |
| Failure behavior | Fail closed: no budget decision means no model invocation |
| Initial coverage | The existing non-streaming, text-only Claude transformation route |

These are proposed defaults to approve before implementation. Other APIs, streaming, batch, and fixed-capacity billing require the extensions in section 8.

## 2) Architecture

```text
Client
  |
  v
APIM: authenticate, resolve actual deployment, validate/transform request
  |
  +-- managed-identity HTTPS --> Budget Function
  |                               |
  |                               +--> Cosmos DB: budget + reservations
  |                               |    + settlement ledger + outbox
  |                               |
  |                               +--> Local versioned catalog/price snapshot
  |
  +-- deny: return error without invoking Foundry
  |
  +-- allow: forward the bounded request --> Foundry
                                             |
APIM outbound: capture raw usage <-------------+
  |
  +-- synchronous settlement --> Budget Function --> Cosmos DB transaction
  |
  +-- existing usage event --> Event Hub --> ADX / Eventhouse
  |
  +-- existing Anthropic-to-OpenAI response transform --> Client

Cosmos DB outbox/change-feed publisher --> Event Hub --> Budget audit in ADX
Hourly allocation / daily reconciliation --> Adjustment worker --> Budget Function
```

### Why Cosmos DB rather than a cache-only gate?

Cosmos DB provides low-latency point operations and durable transactional updates. Keep documents small, supply the item ID and partition key, and avoid cross-partition queries on the request path. Its database-operation latency is not the end-to-end APIM authorization latency; measure that separately.

Azure Managed Redis is a possible later optimization for display balances or immutable configuration snapshots. It is **not required for the initial design**, and a cached balance must never independently authorize a request. A read-compare-write cache gate has concurrency races; even an atomic Redis script needs an explicit durability and recovery design to avoid reopening spent budget after data loss.

Use one authoritative Cosmos DB write region for a user's budget. APIM gateways in other regions call that authority rather than spending independent regional copies of the same balance. Do not enable multi-region writes with last-writer-wins merging for financial counters. Pause authorization during uncertain recovery rather than restoring an older balance and resuming spending.

## 3) Identity, budget definition, and storage

### 3.1 Trusted individual identity

Reuse `usage-identity` from [usage-in-dollars.md](./usage-in-dollars.md), with stricter enforcement:

- Validate the Entra token and use `tenantId + oid` as the principal key. `oid` alone is not globally unique across tenants.
- For API-key-only clients, require a dedicated APIM subscription mapped to one individual. Resolve it to that same canonical principal where possible.
- Reject missing, shared, unmapped, or `unknown` identities. Do not silently use a subscription ID as an individual without an approved mapping.
- Never trust client-supplied user IDs, dollar amounts, rates, budget versions, or reservation IDs.
- Keep `sessionId` for reporting, not as the budget key. Starting a new session or switching models must not reset the budget.

### 3.2 Budget configuration

An administrator provisions the budget before the user can call a billable model:

| Field | Meaning |
|---|---|
| `principalKey` | Canonical tenant/user identity or approved subscription-to-user mapping |
| `periodId` | For example `2026-10`; derived from trusted request time in UTC |
| `periodStartUtc`, `periodEndUtc` | Explicit inclusive start and exclusive end |
| `currency` | `USD` only for the initial release |
| `limitMicrousd` | Predefined limit; $100 = `100000000` micro-USD |
| `chargedMicrousd` | Settled enforcement charges, including explicit adjustments |
| `reservedMicrousd` | Total held for requests not yet settled or safely released |
| `enabled`, `budgetVersion` | Administrative control and audit version |
| `accountingPolicyVersion` | Which price/capacity/reconciliation rules apply |

Missing budget configuration is a denial, not an unlimited allowance. A period reset creates a new budget document; it does not zero the old document. Late settlements and adjustments belong to the original period.

Partition the container by `principalPeriodKey`, constructed from canonical principal and period. Place the budget document, its reservation documents, ledger entries, and outbox records in that **same logical partition**. Retain the immutable trusted request timestamp across internal retries so a request crossing midnight still uses the original period.

Use integer micro-USD for enforcement and decimal arithmetic for pricing, not binary floating point. Round each reservation upward to cover the computed upper bound; round each final request charge upward to a micro-USD and record the rounding difference. Preserve unrounded decimal amounts for invoice reconciliation. Keep integer values within the exact range of every chosen runtime and serialization format.

### 3.3 Reservation records

Each reservation records:

- `reservationId` derived from APIM's server-generated `context.RequestId`, scoped to the principal and period.
- `apimRequestId`, `correlationId`, `sessionId`, deployment and model/version.
- Request fingerprint, enforced output/size limits, rate-card version, and reserved micro-USD.
- State: `reserved`, `settled`, `released`, or `unknown`.
- Creation time, execution deadline, actual usage if available, and settlement/adjustment versions.

Do not use the caller-controlled correlation or session ID as an idempotency key. Retain settlement and idempotency records longer than the permitted replay/recovery window; archive and purge only under an explicit retention policy. Monitor partition growth and per-user contention.

## 4) The allow/deny rule: atomically reserve before execution

Define:

```text
available = limitMicrousd - chargedMicrousd - reservedMicrousd
allow     = enabled AND completePricing AND requestedReservation <= available
```

An equality match is allowed. A positive-cost request is denied when no budget remains. A disabled budget is denied even for a zero-cost operation; exempt nonbillable endpoints explicitly rather than calling missing prices zero.

**Reserve the maximum supported cost of the incoming request**, not just the current spend. Otherwise a user with $0.01 left could start a $5 request, and many concurrent requests could all pass the same balance check.

### 4.1 Compute a defensible reservation

The Budget Function resolves prices from the same effective-dated `DeploymentCatalog` and `PriceCard` semantics used by the monitoring plan. It prices the **actual backend request after transformation**, not the client-facing model alias.

For the first text-only Claude route:

1. Validate payload size, supported fields, model/context limits, and maximum output tokens.
2. Resolve and enforce the final `max_tokens`, including the existing default of 1024 in [apim-policy.xml](../policy/apim-policy.xml). Reserve against the value actually forwarded.
3. Determine a supported upper bound for billable input, including system text, messages, and provider framing. Prefer provider-supported counting where available, including its latency and limitations.
4. If exact counting is unavailable, use a validated conservative bound up to the deployment's permitted input/context ceiling. Do not treat a characters-divided-by-four estimate as a guaranteed bound.
5. Price input at the highest applicable uncached/cache-write rate unless cheaper cache behavior is guaranteed. Add maximum output cost and every other supported billable meter.
6. Pin the deployment, rate-card version, and all execution limits to the reservation.

If a model's cost cannot be bounded, deny that operation in hard-enforcement mode until a supported adapter exists. A percentage safety margin alone does not establish a hard bound.

Missing or stale mandatory pricing blocks new reservations and alerts operators. Publish catalog and rate snapshots to the Budget Function ahead of time; never fetch retail pricing or ARM deployment metadata for every user request.

### 4.2 Atomic transaction

The Budget Function:

1. Point-reads the budget and any existing reservation for this APIM request.
2. Returns the existing result for an identical internal retry; rejects a fingerprint mismatch.
3. Computes availability and checks the rule above.
4. In one Cosmos DB transactional batch, conditionally replaces the budget using its `_etag`, creates the reservation, and creates its audit/outbox record.
5. Returns `allow=true` **only after commit succeeds**.

An ETag conflict means another request changed the budget. Re-read and retry within a bounded authorization deadline. Do not interpret a conflict, database throttle, timeout, or malformed response as approval.

All budget writers, including administrators and adjustment workers, use the same conditional transaction protocol. Stale reads may cause retries or conservative denials, but cannot authorize a conflicting update without a successful conditional commit.

Example: with $1.00 available and simultaneous $0.60 reservations, only one can commit. The other re-reads $0.40 and is denied.

## 5) Budget API and APIM policy integration

### 5.1 Private Budget API

| Operation | Purpose |
|---|---|
| `POST /internal/budget/reserve` | Validate request bounds, compute cost, atomically reserve, return decision |
| `POST /internal/budget/settle` | Apply authenticated usage once and release the unused reservation |
| `POST /internal/budget/release` | Release only a reservation proven not to have incurred cost |
| `POST /internal/budget/adjust` | Worker-only, versioned reconciliation/capacity adjustment |
| `GET /budget/me` | Authenticated read-only balance with period, accounting basis, and freshness |
| Administrative budget API | Create/change limits, suspend a user, and audit changes |

Reserve requests carry trusted principal/period identity, APIM request ID, deployment, fingerprint, and the transformed request or a validated complete metering description. Avoid persisting prompts; counting may process them transiently.

A successful reserve response contains `allow`, `reason`, `reservationId` when allowed, `periodId`, `currency`, `limitMicrousd`, `chargedMicrousd`, `reservedMicrousd`, `availableMicrousd`, `requestReservationMicrousd`, `priceCardVersion`, and `asOfUtc`. Validate response types and values in APIM. Balances are snapshots, not promises that later requests will succeed.

### 5.2 Inbound order

Integrate into [apim-policy.xml](../policy/apim-policy.xml) without changing its external OpenAI-compatible contract:

1. Initialize correlation/session IDs and authenticate the caller.
2. Apply cheap rate/size limits; reject unsupported streaming and malformed requests before reserving.
3. Resolve the real Foundry deployment and finish the OpenAI-to-Anthropic transform. Preserve the body when reading it for pricing.
4. Call the private Budget Function using `send-request mode="new"` and `authentication-managed-identity` with the Budget API's Entra audience.
5. Use `choose` to inspect the HTTP status and validated decision; store reservation metadata in request context on approval.
6. On denial, use `return-response`. No `forward-request` may run.
7. Only on approval, apply the existing Foundry authentication, backend selection, and forwarding.

Start with a two-second `send-request` timeout and tune from measurements. With `ignore-error="true"`, explicitly detect a null response and return a dependency error; this setting must never mean fail open. With `ignore-error="false"`, route the failure through a budget-aware `on-error` branch. Choose one approach and test all failure paths.

Do not expose the internal Budget API through the same billable policy chain; call its private backend directly to avoid recursion.

| Condition | Client result |
|---|---|
| Missing/invalid caller credentials | `401`, preserving the authentication challenge where applicable |
| Missing/disabled budget or disallowed operation | `403`, explicit error code |
| Insufficient available budget | `429`, code `budget_exceeded` |
| Budget service/database/pricing unavailable, invalid decision, or deadline exceeded | `503`, code `budget_check_unavailable`; no Foundry invocation |
| Normal model execution | Existing backend status and response shape |

Return JSON errors using the existing `error` envelope and correlation ID. Include the period end for exhausted budgets, but omit a misleading `Retry-After` when active reservations or administrative changes make recovery time unknown. Expose optional `X-Budget-Remaining-USD` and `X-Budget-Period-End` headers without leaking other users' data.

### 5.3 Policy scope and inheritance

The monitoring plan puts `usage-capture` at All APIs outbound scope, reached through API-level `<base />` before the response transformation. Keep that ordering so settlement reads native Anthropic meters before they are lost in the OpenAI conversion.

Put reusable reservation/settlement logic in policy fragments, but include reservation **after deployment resolution and transformation** at each applicable API. An All APIs inbound fragment invoked before those variables exist cannot price this route correctly.

Require every billable route to include the gate; explicitly classify health/model-list/usage/budget endpoints. Audit effective policies and `<base />` inheritance so operation-level overrides cannot bypass enforcement.

## 6) Settlement, failures, and durable recovery

### 6.1 Normal completion

Before transforming the response, reuse the monitoring plan's non-overlapping normalized usage meters, including Claude cache creation/read and output tokens. Call `/settle` with the trusted reservation ID and raw normalized usage.

In one conditional Cosmos DB transaction:

```text
reservedMicrousd -= reservation.amount
chargedMicrousd  += actualEnforcementCharge
reservation.state = settled
create unique settlement ledger entry and outbox record
```

Settle exactly once using the reservation ID and settlement version. Duplicate identical settlements are no-ops; contradictory usage requires an explicit correction entry, not an overwrite.

If actual cost exceeds the reserved bound, record the full charge even if availability becomes negative, block further spending, and alert on a bound violation. Never cap recorded spend to the reservation to hide overspend.

### 6.2 Known failure versus uncertain execution

- **Rejected before dispatch:** release the reservation if execution is definitively known not to have occurred.
- **Provider confirms zero billable usage:** settle to zero or release under the documented provider rule.
- **Provider reports usage on an error:** settle that usage. Do not assume every 4xx/5xx is free.
- **Backend timeout, client disconnect, missing usage, APIM failure, or ambiguous dispatch:** retain the hold and mark it `unknown`. A timeout does not prove that Foundry did not execute.
- **Settlement service unavailable after successful inference:** preserve the reservation, log/alert explicitly, and return the normal model response with `X-Budget-Settlement: pending`. Do not automatically replay the model call to repair accounting.

Extend the existing `on-error` handling to distinguish budget failures from transformation/backend failures and attempt safe recovery only when it can establish execution state. APIM cleanup is best effort; durable reservation state is the recovery anchor.

A periodic recovery worker scans indexed unresolved reservations. Set an initial investigation deadline beyond the route's 120-second backend timeout, for example 10 minutes, and tune for retries and telemetry delays. At the deadline, **do not simply expire the hold and return money**. Recover trusted usage, retain the hold, or conservatively convert it to a separately labeled maximum enforcement charge under an approved policy. Later evidence can correct that charge.

Disable automatic model retries initially. If retries or model failover are added, each potentially billable attempt needs its own reservation and usage identity; a shared reservation cannot silently cover multiple executions.

## 7) Connect the enforcement ledger to the existing monitoring plan

Retain the existing `llm.usage` events and ADX reporting. Add principal/period identity, reservation ID, attempt ID, accounting policy, price-card version, settlement state, and enforcement charge so operators can join accounting to usage.

Use Cosmos DB settlement ledger/outbox records as the durable source for budget changes. A change-feed publisher forwards them to Event Hub with deterministic event IDs and retry handling. Consumers deduplicate; alert on publisher lag and poison records.

The original APIM `log-to-eventhub` path remains useful telemetry, but its best-effort delivery must not be the only mechanism updating financial counters. An Event Hub usage consumer can repair a missing settlement through the same idempotent API; it must not add another charge for an already settled request.

Avoid double counting across sources:

- Live pay-per-unit settlement uses the monitoring plan's meter normalization and effective-dated rates.
- Hourly capacity allocations **replace** provisional shadow charges through delta adjustments, rather than adding both.
- Daily billed reconciliation posts `new recognized amount - previously recognized amount` with a unique source/version ID.
- A repeated reconciliation run applies only a new delta; it does not charge the entire invoice again.
- Every late adjustment targets the original period and retains its lineage. An upward correction can exhaust that period; it cannot undo already completed requests.

Display enforcement spend, outstanding holds, estimated USD, allocated USD, and reconciled USD separately. The existing usage API answers historical reporting questions; `/budget/me` reads the transactional balance for current availability.

### Hard-limit guarantee and its boundary

For supported operations with valid upper bounds, atomic reservations prevent concurrent requests from committing more **enforcement-priced** spend than the configured budget. They do not guarantee that the final Azure invoice can never exceed that number.

Billing corrections, price changes not yet published, missing meters, and retroactive capacity allocation can change final attributed dollars. Approve an accounting policy before rollout: the default is to admit at pinned current rates and apply later adjustments to the original period, recording debt and blocking new requests when necessary. Do not promise invoice-level hard caps based on delayed billing data.

## 8) Coverage beyond the initial Claude route

| Workload | Required extension before hard enforcement |
|---|---|
| OpenAI / other per-token models | Model-specific input bounds, output limits, cached/audio/image/tool meters, and a tested normalizer |
| Streaming | Reserve before opening the stream; use a trusted backend/stream-aware metering component for completion and disconnect recovery. Client-reported usage alone cannot release funds |
| Batch | Reserve the entire bounded job at submission, persist submitter identity, settle individual results exactly once, and release only verified unexecuted work |
| Images, audio, tools, agent loops | Enforce image count/quality, media duration, tool-call counts, or per-step budgets. Reject unbounded combinations |
| PTU / managed compute / fine-tune hosting | Use approved shadow-cost admission and delta adjustments to hourly allocations, or fund capacity separately. Future allocation depends on other users, so it is not a request-time invoice hard cap |
| Fine-tuning training | Separate job-level reservation and settlement; inference authorization does not cover training spend |

Do not enable a new model solely because it appears in the deployment catalog. Pricing, bounded execution, usage capture, and settlement must all be supported. Until then, deny it explicitly rather than charging zero.

## 9) Security, reliability, and operations

- Use managed identities from APIM to the Budget API and from the Budget Function to Cosmos DB. Give the APIM identity only reserve/settle permissions; restrict adjustment and administrative operations to separate roles.
- Require the Budget API to validate token issuer, audience, tenant, caller identity, and application roles. Network isolation alone is not authorization.
- Use private connectivity and disable public access where supported by the selected APIM/Functions tiers. Verify outbound VNet integration, private DNS, and endpoint reachability during deployment design.
- Keep Foundry inaccessible to end users directly, following the backend-lockdown requirements in [usage-in-dollars.md](./usage-in-dollars.md).
- Avoid prompt/response logging in the budget ledger. Restrict identity and financial data access; audit administrative changes.
- Keep Budget Function instances warm, reuse Cosmos SDK clients/connections, provision RU/s for measured peak load, and monitor hot user-period partitions.
- Proposed performance acceptance target: incremental inbound authorization latency **p95 <= 50 ms and p99 <= 100 ms**, at an agreed peak requests/second and per-user concurrency. These are targets to validate, not service guarantees; report counting/network/database components separately.
- Monitor reservation/settlement latency, Cosmos throttles and ETag retries, denials by reason, missing prices, bound violations, unresolved holds, negative balances, outbox lag, and drift between ledger totals and reporting.
- Configure backups and a recovery runbook. Before reopening admission after restore/failover, reconcile ledger high-water marks and outstanding requests. Never initialize an empty budget as fully available after data loss.
- Budget changes use audited conditional updates. Lowering a limit below committed spend blocks new requests but does not cancel already reserved work. Cancellation requires a separate execution-control mechanism.

## 10) Implementation sequence

| Step | Deliverable and completion condition |
|---|---|
| 1 | Approve identity mapping, per-user budget, period, accounting/late-adjustment rules, supported operations, expected throughput, and latency target |
| 2 | Define schemas and build Cosmos DB conditional transactions, reservation state machine, idempotency, ledger, and outbox |
| 3 | Publish shared versioned catalog/pricing snapshots; implement and validate the initial Claude cost bound against the transformed payload |
| 4 | Build the private Budget Function, Entra roles, administrator provisioning, and read-only balance endpoint |
| 5 | Add APIM authentication and budget fragments in the required order; keep the existing request/response contract and non-streaming guard |
| 6 | Add trusted raw-usage settlement, recovery worker, outbox publisher, and idempotent repair from existing usage events |
| 7 | Connect hourly allocation and daily billed reconciliation through versioned delta adjustments |
| 8 | Add IaC, private connectivity, dashboards, alerts, backup/recovery procedures, and operational documentation |
| 9 | Run the acceptance tests below in a production-like environment; tune capacity and deadlines |
| 10 | Run shadow decisions without a blocking promise, compare expected/actual charges, then enable enforcement for a pilot and expand |

Shadow mode must be clearly labeled: it measures decisions but does not provide hard-budget protection. Review unresolved holds, false denials, and bound violations before expanding enforcement. This plan stops before resource provisioning; implementation requires approved Azure subscription, region, SKUs, and operational cost estimates.

## 11) Acceptance tests

1. **Basic decision:** a configured $10 budget permits a bounded $1 request; a request exceeding available funds returns `429` without invoking Foundry.
2. **Boundary:** exact remaining reservation succeeds; one micro-USD above it fails. Negative, malformed, missing, and disabled budgets never fail open.
3. **Concurrency:** with $1 remaining, launch 100 simultaneous $0.60 reservations across multiple APIM/Function instances. Exactly one commits; no successful committed total exceeds $1.
4. **Idempotency:** retry reserve/settle after a lost response. Counters change once, and a fingerprint mismatch is rejected. Test duplicate Event Hub and outbox deliveries.
5. **Settlement:** reserving $1 and settling $0.25 releases $0.75. Validate Claude input, output, cache-write, and cache-read charges against the monitoring calculation.
6. **Bound correctness:** exercise maximum accepted input, framing, Unicode, cache modes, maximum output, and all supported meters. Measured enforcement cost must never exceed the bound; reject unsupported payloads.
7. **Dependency failure:** inject Budget API timeout, invalid response, Cosmos `429`, ETag conflicts, unavailable pricing, and private DNS failure. No model invocation occurs without a committed approval.
8. **Unknown execution:** disconnect the client and kill the processing instance after reservation or dispatch. Recovery retains funds until execution/usage is established; no TTL silently restores budget.
9. **Settlement outage:** successful model output remains compatible, the pending header/alert appears, and later repair settles once without replaying inference.
10. **Identity/bypass:** forged identity/cost headers, shared keys, a second session, another model, and operation-level policy overrides cannot escape the same user's budget.
11. **Period/admin changes:** test a request crossing UTC month-end, late settlement to the old period, suspension, and a limit reduction racing with a reservation.
12. **Reconciliation:** hourly allocation and repeated daily true-ups apply only deltas. Enforcement and reporting amounts remain distinguishable and traceable.
13. **Performance:** measure the stated p95/p99 thresholds at the approved peak load, including per-user contention and dependency throttling; report inbound and settlement overhead separately.
14. **Recovery:** rehearse failover/restore, reconcile ambiguous reservations, and demonstrate that stale data cannot reopen already spent funds.
15. **Compatibility:** existing OpenAI-to-Anthropic mapping, usage fields, backend errors, correlation/session headers, and `stream=true` rejection remain intact.

## 12) Microsoft documentation

- [APIM send-request policy](https://learn.microsoft.com/azure/api-management/send-request-policy): synchronous callout, timeout, response variable, and error behavior.
- [APIM managed-identity authentication](https://learn.microsoft.com/azure/api-management/authentication-managed-identity-policy): authenticate the gateway to the Budget API.
- [APIM choose policy](https://learn.microsoft.com/azure/api-management/choose-policy) and [return-response policy](https://learn.microsoft.com/azure/api-management/return-response-policy): implement the gateway's allow/deny branch.
- [Cosmos DB overview](https://learn.microsoft.com/azure/cosmos-db/overview): low-latency operational database.
- [Cosmos DB transactional batches](https://learn.microsoft.com/azure/cosmos-db/transactional-batch): all-or-nothing operations sharing one partition key.
- [Cosmos DB transactions and optimistic concurrency](https://learn.microsoft.com/azure/cosmos-db/database-transactions-optimistic-concurrency): ETags, conditional updates, and single-region versus multi-region write behavior.