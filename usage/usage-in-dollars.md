# Plan: Microsoft Foundry Model Usage in US Dollars, per Session and per Individual

This plan explains how to get the **USD cost of Microsoft Foundry model usage** for a given **session**, grouped by **individual**. An individual's total includes every model they used. Traffic reaches Foundry through the **Azure API Management (APIM) AI Gateway** described in [../README.md](../README.md) and [../policy/apim-policy.xml](../policy/apim-policy.xml).

It covers **every model family and billing model in Foundry**:

- Claude (Anthropic)
- OpenAI models (Azure OpenAI)
- Other models sold by Azure (for example Llama, DeepSeek, Phi, Mistral, Grok)
- Partner and community models from Azure Marketplace
- Open-source / open-weight models on managed compute
- Fine-tuned models

The goals are **accuracy** (it reconciles to the Azure invoice) and **freshness** (seconds, not days). No single data source does both, so the design has three layers plus a completeness check:

| Layer | Source | Latency | Accuracy | Purpose |
|---|---|---|---|---|
| **1. Real-time estimate** | Per-request usage events emitted by APIM × versioned USD rate card | Seconds | Same as billed for pay-per-token models when the rate card is correct | Live per-session / per-user dollars |
| **2. Capacity allocation** | Hourly cost of fixed-capacity deployments (provisioned throughput (PTU), managed compute, fine-tune hosting), split by each user's share of traffic | Hourly | Exact split of a known hourly cost | Attribute dollars that aren't charged per token |
| **3. Billed reconciliation** | Azure Cost Management Query / Cost Details APIs | ~8–24h+ (final after month close) | Matches the invoice | True-up factor applied to layers 1 and 2 |
| Completeness check | Azure Monitor metrics on Foundry / Azure ML resources | ~1–3 min | Authoritative token counts, but with no user identity | Detect traffic that bypassed APIM or events that were lost |

---

## 1) Foundry model and billing taxonomy

Each Foundry deployment falls into one **billing category**. The category decides the meters, the rate source, and where the cost lands in Cost Management.

| # | Billing category | Examples | How it's charged | Rate source | Where cost lands in Cost Management |
|---|---|---|---|---|---|
| **A** | **Azure OpenAI (sold by Azure), pay-per-token** | GPT-5 family, GPT-4.1, GPT-4o, o-series, embeddings, gpt-image, realtime/audio | Per 1M tokens by meter: input, **cached input**, output. Reasoning tokens are billed as output. Separate audio/image token meters. Price varies by **deployment type**: Global / Data Zone / Regional Standard, and **Batch** (discounted). Some legacy image/audio models are billed per image or per minute. | **Azure Retail Prices API** (`productName eq 'Azure OpenAI'` or `'Azure OpenAI GPT5'`; `skuName` encodes model + direction + deployment type). Negotiated: **Price Sheet API**. | Foundry resource (`Microsoft.CognitiveServices/accounts`), Microsoft meters |
| **B** | **Other models sold by Azure, pay-per-token** | DeepSeek, Llama, Phi, Mistral, Grok, and others, depending on the catalog | Per 1M input / output tokens. Some models offer Global / Data Zone variants. | **Azure Retail Prices API**: discover each model's meters once, then pin `meterId`. Negotiated: **Price Sheet API**. | Foundry resource, Microsoft meters |
| **C** | **Claude (Anthropic), CCU** | Claude Opus / Sonnet / Haiku | Tokens are converted to **Claude Consumption Units (CCU)** at Anthropic per-model rates for input, output, cache write (5m / 1h), and cache read, less any private-offer discount. CCU is USD-denominated and metered hourly. | Anthropic rates at https://aka.ms/ccu-pricing + private-offer discount. Per the [CCU billing doc](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/claude-models-billing), USD = tokens × rate × (1 − discount). | Foundry resource, Marketplace meter (CCU). Claude deployments created before CCU use legacy per-model meters. |
| **D** | **Partner and community models (Azure Marketplace), pay-per-token** | Third-party serverless offers not sold by Azure | Per 1M tokens (or per request) at the publisher's Marketplace price | The offer's **Pricing and terms** in the Foundry portal or Azure Marketplace. Store it manually or through the Marketplace private offer. Don't assume it's in the Retail Prices API. | **Resource group**, under the Marketplace resource created when the project subscribed to the offer (not the Foundry resource) |
| **E** | **Provisioned throughput (PTU)** | Any category A/B model deployed as Global / Data Zone / Regional **Provisioned** | **$ per PTU per hour × deployed PTUs**, prorated by minute, **regardless of tokens used**. Azure Reservations (monthly or yearly) lower the effective rate. | Hourly: **Retail Prices API** (PTU meters). Reserved: reservation price amortized (Cost Management `AmortizedCost`). | Foundry resource. Reservation purchases appear as amortized cost. |
| **F** | **Managed compute (open-source / open-weight)** | Hugging Face and other open-weight models on Azure ML online endpoints. Some catalog models add a per-hour software surcharge. | **VM hours × instance count** (+ any Marketplace software surcharge per hour), independent of tokens | **Retail Prices API** for the VM SKU (`serviceName eq 'Virtual Machines'` or Azure Machine Learning meters, `armSkuName eq '<sku>'`) + offer surcharge | Azure ML workspace / online endpoint resource |
| **G** | **Fine-tuned models** | Fine-tuned GPT / Llama / Phi, and others | **Inference** per token (same meters as A/B) **+ hosting fee per hour** while deployed. Training runs are one-time costs. | **Retail Prices API** (fine-tuning meters) | Foundry resource |
| *(opt.)* | **OpenAI direct (api.openai.com)**, if APIM also fronts it | GPT models billed by OpenAI, not Azure | Per 1M tokens, OpenAI price list | OpenAI published pricing | Not on the Azure invoice. Reconcile with the OpenAI organization **Costs API** (`GET https://api.openai.com/v1/organization/costs`). |

Two cost shapes follow from this table:

- **Variable (per-request) cost** covers categories **A, B, C, D**, G inference, and OpenAI direct. Each request has an exact dollar amount, which is units × rate.
- **Fixed-capacity (per-hour) cost** covers categories **E, F**, and G hosting. The cost is incurred whether or not anyone calls the model, so it must be **allocated** to individuals (section 6.5).

---

## 2) Why the gateway is the system of record for "who"

- **Foundry has no individual identity for this traffic.** APIM calls every backend with its **managed identity** (`authentication-managed-identity`), so Foundry logs, metrics, and Cost Management all see one caller: APIM.
- **Cost Management's finest grain is a resource, meter, or tag.** Foundry resource and deployment tags help with allocation by deployment. Claude doesn't support project-level attribution. Marketplace models land in the resource group. None of these identify a person.
- **So APIM must capture identity, session, deployment, model, and every billable usage unit together on every request.** It is the only point where all of them are present.

---

## 3) Identity and session capture (APIM inbound, all APIs)

Put these in a **policy fragment** (`usage-identity`) included at **All APIs** scope, so every model API gets them.

### 3.1 Individual (`userId`)
Pick one approach, in order of preference:

1. **Entra ID token** (preferred). Use `validate-azure-ad-token` / `validate-jwt`, then take `oid` as the stable key and `upn` / `preferred_username` for display.
2. **Per-user APIM subscription key** (when the client, such as Copilot BYOK, can only send an API key). Issue one subscription per person, scoped to an APIM *user*, and use `context.User.Id` / `context.User.Email`. Don't use shared keys; they make per-person attribution impossible.

Reject requests that have no resolvable identity (401).

### 3.2 Session (`sessionId`)
- Accept a client-provided `X-Session-Id` header. It should be one stable value per chat or agent session, even when the session calls several models.
- If the header is missing, fall back to `X-Correlation-Id` (already implemented). That makes each request its own session.
- Echo `X-Session-Id` back on the response, the same way `X-Correlation-Id` is echoed today.

```xml
<set-variable name="sessionId" value="@{
    var s = context.Request.Headers.GetValueOrDefault("X-Session-Id", "");
    return string.IsNullOrWhiteSpace(s) ? (string)context.Variables["correlationId"] : s;
}" />
<set-variable name="userId" value="@{
    // Option 1: Entra token (after validate-azure-ad-token output-token-variable-name="jwt")
    // var jwt = (Jwt)context.Variables["jwt"]; return jwt.Claims.GetValueOrDefault("oid", "");
    // Option 2: per-user APIM subscription
    return context.User?.Id ?? context.Subscription?.Id ?? "unknown";
}" />
```

### 3.3 Deployment key (`deploymentKey`), per API
Each model API sets the variable that identifies the **Foundry deployment** being called. The deployment, not the model name, determines the deployment type, the region, and the billing category.

| API style | How to get the deployment |
|---|---|
| Azure OpenAI `/openai/deployments/{deployment-id}/...` | `context.Request.MatchedParameters["deployment-id"]` |
| Azure OpenAI / Foundry `/openai/v1/...` and Foundry Models inference (`model` in body) | `model` from the request body (it's the deployment name) |
| Anthropic Messages (this repo's transform) | `{{foundry-claude-model-or-deployment}}` named value |
| Managed online endpoint (category F) | Endpoint + deployment name from the APIM backend |
| APIM unified model API (alias → backend) | The resolved backend deployment, not the client-facing alias |

Store `deploymentKey = "{foundryResourceName}/{deploymentName}"` so the same deployment name on two resources isn't confused.

---

## 4) Capture usage for every response format (APIM outbound)

### 4.1 Normalized meters
Each provider reports usage differently. Normalize every response into **billable meters** with **unit counts**, where **meter units never overlap**:

| Normalized meter | Anthropic Messages (C, D) | OpenAI Chat Completions / Foundry Models inference (A, B, D, F) | OpenAI Responses API (A) | Embeddings (A, B) |
|---|---|---|---|---|
| `input` | `input_tokens` (already **excludes** cache tokens) | `prompt_tokens − cached_tokens − audio_tokens` | `input_tokens − input_tokens_details.cached_tokens` | `prompt_tokens` |
| `cachedInput` | — | `prompt_tokens_details.cached_tokens` | `input_tokens_details.cached_tokens` | — |
| `cacheWrite5m` / `cacheWrite1h` | `cache_creation.ephemeral_5m/1h_input_tokens` (or all of `cache_creation_input_tokens` → 5m) | — | — | — |
| `cacheRead` | `cache_read_input_tokens` | — | — | — |
| `output` | `output_tokens` (includes thinking) | `completion_tokens − completion_tokens_details.audio_tokens` (**reasoning tokens included**; don't add them again) | `output_tokens` (reasoning included) | — |
| `audioInput` / `audioOutput` | — | `prompt_tokens_details.audio_tokens` / `completion_tokens_details.audio_tokens` | — | — |
| `imageInput` / `imageOutput` | — | — | gpt-image: `input_tokens_details.image_tokens` / `output_tokens` | — |
| `webSearch` (per call) | `server_tool_use.web_search_requests` | — | count of `web_search_call` items in `output` | — |
| `image` (per image, legacy DALL·E) | — | No `usage`. Count `data[]` items and add `size` + `quality` from the request to the meter key. | — | — |
| `audioSeconds` (Whisper-style transcription) | — | No `usage`. Use the response `duration` (verbose_json), or let reconciliation allocate. | — | — |

Note the semantics: **Anthropic `input_tokens` excludes cached tokens, but OpenAI `prompt_tokens` includes them.** Normalizing to non-overlapping meters keeps that difference from causing double-charging.

Also record `reasoningTokens` as an **informational** field (not a meter) for analysis.

### 4.2 Where the capture runs
Put a single **`usage-capture` policy fragment** at **All APIs** scope in `<outbound>`.

API-level outbound policies (like the Anthropic→OpenAI transform in this repo) start with `<base />`, so the All-APIs outbound runs **before** any API-level `set-body`. The fragment therefore always sees the **raw backend response**. That keeps one capture path for every model.

```xml
<!-- Fragment: usage-capture (All APIs scope, outbound) -->
<set-variable name="usageMeters" value="@{
    var meters = new JArray();
    if (context.Response.StatusCode >= 400) return meters.ToString();
    var ct = context.Response.Headers.GetValueOrDefault("Content-Type", "");
    if (!ct.Contains("application/json")) return meters.ToString();   // SSE handled in 4.4
    var b = context.Response.Body.As<JObject>(preserveContent: true);
    var u = b["usage"] as JObject;
    void add(string m, long n) { if (n > 0) meters.Add(new JObject { ["meter"] = m, ["units"] = n }); }
    long v(string path) => (long?)u?.SelectToken(path) ?? 0;

    if (u == null) {
        if (b["data"] is JArray imgs) add("image", imgs.Count);               // legacy image gen
        return meters.ToString();
    }
    if ((string)b["type"] == "message") {                                     // Anthropic Messages
        var w5 = v("cache_creation.ephemeral_5m_input_tokens");
        var w1 = v("cache_creation.ephemeral_1h_input_tokens");
        var wAll = v("cache_creation_input_tokens");
        add("input", v("input_tokens"));
        add("cacheWrite5m", (w5 + w1 == 0) ? wAll : w5);
        add("cacheWrite1h", w1);
        add("cacheRead", v("cache_read_input_tokens"));
        add("output", v("output_tokens"));
        add("webSearch", v("server_tool_use.web_search_requests"));
    } else if (u["input_tokens"] != null) {                                   // OpenAI Responses / gpt-image
        var cached = v("input_tokens_details.cached_tokens");
        var imgIn = v("input_tokens_details.image_tokens");
        add("input", v("input_tokens") - cached - imgIn);
        add("cachedInput", cached);
        add("imageInput", imgIn);
        add((string)b["object"] == "response" ? "output" : "imageOutput", v("output_tokens"));
        if (b["output"] is JArray outItems)
            add("webSearch", outItems.Count(i => (string)i["type"] == "web_search_call"));
    } else {                                                                  // Chat Completions / inference / embeddings
        var cached = v("prompt_tokens_details.cached_tokens");
        var aIn = v("prompt_tokens_details.audio_tokens");
        var aOut = v("completion_tokens_details.audio_tokens");
        add("input", v("prompt_tokens") - cached - aIn);
        add("cachedInput", cached);
        add("audioInput", aIn);
        add("output", v("completion_tokens") - aOut);
        add("audioOutput", aOut);
    }
    return meters.ToString();
}" />
<log-to-eventhub logger-id="usage-eventhub-logger">@{
    JObject b = null;
    try { b = context.Response.Body.As<JObject>(preserveContent: true); } catch { }
    return new JObject {
        ["eventType"]       = "llm.usage",
        ["timestampUtc"]    = DateTime.UtcNow.ToString("o"),
        ["apimRequestId"]   = context.RequestId.ToString(),
        ["correlationId"]   = (string)context.Variables.GetValueOrDefault("correlationId", ""),
        ["sessionId"]       = (string)context.Variables.GetValueOrDefault("sessionId", ""),
        ["userId"]          = (string)context.Variables.GetValueOrDefault("userId", "unknown"),
        ["apimSubscription"]= context.Subscription?.Id,
        ["apiId"]           = context.Api.Id,
        ["operationId"]     = context.Operation.Id,
        ["deploymentKey"]   = (string)context.Variables.GetValueOrDefault("deploymentKey", ""),
        ["responseModel"]   = (string)b?["model"] ?? "",
        ["isBatch"]         = context.Request.Url.Path.Contains("/batches"),
        ["reasoningTokens"] = (long?)b?.SelectToken("usage.completion_tokens_details.reasoning_tokens")
                              ?? (long?)b?.SelectToken("usage.output_tokens_details.reasoning_tokens") ?? 0,
        ["statusCode"]      = context.Response.StatusCode,
        ["meters"]          = JArray.Parse((string)context.Variables["usageMeters"])
    }.ToString(Newtonsoft.Json.Formatting.None);
}</log-to-eventhub>
```

In the Claude API, also map `cache_read_input_tokens` into the OpenAI response as `usage.prompt_tokens_details.cached_tokens`, so clients see consistent numbers.

### 4.3 Batch, PTU, and managed compute specifics
- **Batch (A/B):** Batch jobs bypass the synchronous path. Results arrive in an output file with `usage` per line. A small job reads each completed batch output file, normalizes it the same way, and writes events with `isBatch=true` and the `userId` / `sessionId` recorded when APIM accepted the `/batches` create call. Store `batchId → userId, sessionId` at submission time.
- **PTU (E):** Usage events are captured exactly as for pay-per-token. The **deployment catalog** (section 6.1) marks the deployment as capacity-billed, so its dollars come from allocation (section 6.5), not from token rates.
- **Managed compute (F):** The same capture applies (OpenAI-compatible or custom `usage`). If the model returns no usage, record `requests=1` and `latencyMs` as allocation weights.

### 4.4 Streaming
Usage arrives at the end of the stream: Anthropic `message_delta`, OpenAI final chunk when the request sets `stream_options.include_usage=true`, or Responses `response.completed`. Options:
1. Have APIM inject `stream_options.include_usage=true` for Chat Completions, and rely on **`llm-emit-token-metric`** plus `ApiManagementGatewayLlmLog`, which both parse streams.
2. Have the client report the final usage object to a small ingestion endpoint keyed by `apimRequestId`.

Broken streams may produce no usage. Reconciliation (section 7) and the completeness check (section 8) cover that gap.

### 4.5 Secondary gateway telemetry (cross-checks, not the source of truth)
- **APIM diagnostic setting → "Logs related to generative AI gateway"** writes to [`ApiManagementGatewayLlmLog`](https://learn.microsoft.com/azure/azure-monitor/reference/tables/apimanagementgatewayllmlog) (Prompt/Completion/Total tokens, ModelName, DeploymentName, CorrelationId). Join it to `ApiManagementGatewayLogs` on `CorrelationId` to get the user. It has **no cached, audio, or image columns**, so use it only for audits and as a fallback.
- **[`llm-emit-token-metric`](https://learn.microsoft.com/azure/api-management/llm-emit-token-metric-policy)** supports OpenAI Chat Completions / Responses and Anthropic Messages (Anthropic on APIM **v2 tiers**). Add the dimensions `User ID`, `Session ID`, and `Deployment` for near-real-time dashboards and alerts. These are pre-aggregated metrics, so don't use them for invoicing.

---

## 5) Ingestion and storage

```
APIM ──log-to-eventhub──► Event Hub ──streaming ingestion──► Azure Data Explorer / Fabric Eventhouse
Batch output reader ─────► Event Hub                         ├── UsageEvents        (1 row/request, raw)
                                                             ├── UsageMeters        (1 row/request/meter, via update policy mv-expand)
                                                             ├── DeploymentCatalog  (deployment → model, type, category, region)
                                                             ├── PriceCard          (versioned USD rates, all categories)
                                                             ├── CapacityHourly     (hourly fixed cost per capacity pool)
                                                             ├── UserDirectory      (oid → name, dept, cost center)
                                                             └── BilledDaily        (Cost Management pulls)
```

- **Streaming ingestion** makes events queryable in seconds. An **update policy** expands `meters[]` into `UsageMeters` rows (long format), so pricing is a single join for every model.
- Dedupe on `apimRequestId` with a materialized view (`arg_max` by ingestion time), because Event Hub delivers at least once.
- Lighter alternative: a Log Analytics custom table via the Logs Ingestion API (about 2–5 minutes of latency).

---

## 6) Converting usage to US dollars

### 6.1 Deployment catalog (`DeploymentCatalog`), refreshed every 15 minutes
List each deployment from ARM:

`GET https://management.azure.com/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.CognitiveServices/accounts/{account}/deployments?api-version=2024-10-01`

| Column | From |
|---|---|
| `deploymentKey` | `{account}/{deployment name}` |
| `modelFormat`, `modelName`, `modelVersion` | `properties.model.format` (for example `OpenAI`, `Meta`, `DeepSeek`, `Anthropic`), `.name`, `.version` |
| `deploymentType` | `sku.name`: `GlobalStandard`, `DataZoneStandard`, `Standard`, `GlobalBatch`, `DataZoneBatch`, `GlobalProvisionedManaged`, `DataZoneProvisionedManaged`, `ProvisionedManaged` |
| `capacity` | `sku.capacity` (PTUs for provisioned) |
| `region` | account `location` |
| `billingCategory` | Derived: A / B / C / D / E / G (section 1) |
| `costBasis` | `perUnit` (A, B, C, D, G inference) or `capacity` (E, F, G hosting) |
| `validFromUtc` / `validToUtc` | Change tracking (resizes, model version upgrades) |

Add **managed online endpoints** (category F) from `Microsoft.MachineLearningServices/workspaces/onlineEndpoints/deployments` (instance type, instance count) and **Marketplace serverless endpoints** (category D).

Price a request by **its deployment's** catalog row at request time, not by the model name the client sent.

### 6.2 Rate card (`PriceCard`), versioned and effective-dated

| Column | Example |
|---|---|
| `billingCategory` | `A` |
| `modelName` / `modelVersion` | `gpt-4.1` / `2025-04-14`, `claude-sonnet-4-5`, `DeepSeek-R1`, `Llama-3.3-70B-Instruct` |
| `deploymentType` | `GlobalStandard` / `DataZoneStandard` / `Standard` / `GlobalBatch` / `*` (where type doesn't matter) |
| `region` | `eastus2` / `*` (global prices) |
| `meter` | `input`, `cachedInput`, `output`, `cacheWrite5m`, `cacheWrite1h`, `cacheRead`, `audioInput`, `audioOutput`, `imageInput`, `imageOutput`, `image:{size}:{quality}`, `webSearch`, `audioSeconds`, `ptuHour`, `vmHour`, `hostingHour` |
| `unitSize` | `1000000` for tokens, `1` for per-request / per-image / per-hour |
| `usdPerUnitSize` | List price in USD |
| `discountPct` | Negotiated discount (EA / MCA price sheet delta, Anthropic or Marketplace private offer) |
| `azureMeterId` | Retail Prices / Cost Management `meterId`. Used to join reconciliation to the same meter. |
| `effectiveFromUtc` / `effectiveToUtc` | Price changes apply from the moment they take effect |
| `source` | `retail-prices-api` / `price-sheet-api` / `anthropic-ccu-pricing` / `marketplace-offer` / `openai-pricing` |

### 6.3 Where rates come from (daily refresh job)

| Category | API / source | Notes |
|---|---|---|
| A: Azure OpenAI | **Retail Prices API** `GET https://prices.azure.com/api/retail/prices?api-version=2023-01-01-preview&$filter=productName eq 'Azure OpenAI' and armRegionName eq '{region}'` (repeat for `'Azure OpenAI GPT5'`) | Parse `skuName` (for example `gpt 4.1 Inp Data Zone`, `gpt 4o 0513 Input global`) into model + meter + deploymentType. Filters are **case-sensitive** in this API version. Cached-input, batch, audio, and image meters are separate SKUs. Legacy models are in the legacy pricing PDF linked from the Azure OpenAI FAQ. |
| B: Other sold by Azure | **Retail Prices API**: search once per model (by `productName` / `meterName`), then pin `$filter=meterId eq '{id}'` | Pinning `meterId` keeps refreshes stable when display names change. |
| A/B negotiated (EA / MCA) | **Price Sheet API**: MCA `POST /providers/Microsoft.Billing/billingAccounts/{ba}/billingProfiles/{bp}/providers/Microsoft.CostManagement/pricesheets/default/download?api-version=2023-11-01`; EA billing-period price sheet | Use the customer price as `usdPerUnitSize` (or keep the list price plus `discountPct`). |
| C: Claude (CCU) | Anthropic per-model rates at https://aka.ms/ccu-pricing + private-offer discount per model | USD = tokens × rate × (1 − discount). Dividing by the fixed CCU price gives only the CCU count. |
| D: Partner Marketplace | The offer's **Pricing and terms** (Foundry portal / Azure Marketplace) or the private offer | Loaded manually or semi-automatically. Alert when an offer is accepted without a `PriceCard` row. |
| E: PTU | **Retail Prices API** PTU hourly meters (per deployment type / region). Reservation: amortized price from **Reservations / Cost Management** (`AmortizedCost`). | The `ptuHour` meter. The effective rate is the lower of reserved and hourly, depending on reservation coverage. |
| F: Managed compute | **Retail Prices API** for the VM `armSkuName` in the region, plus the model offer's hourly surcharge | The `vmHour` meter. |
| G: Fine-tuned | **Retail Prices API** fine-tuning hosting and inference meters | The `hostingHour` meter plus token meters. |
| OpenAI direct (optional) | OpenAI's published pricing | Reconcile with the OpenAI Costs API, not Azure. |

The job closes the previous row's `effectiveToUtc` when a price changes and alerts on:

- Any price change
- **Any deployment in `DeploymentCatalog` with no matching `PriceCard` rows**, which would leave usage unpriced

### 6.4 Variable (per-request) cost
Applies to deployments with `costBasis = perUnit`:

```
estUsd(request) = Σ over meters m [ units(m) / unitSize(m) × usdPerUnitSize(model, deploymentType, region, m) ]
                  × (1 − discountPct)
```

- Use `deploymentType = GlobalBatch` / `DataZoneBatch` rates when `isBatch = true`.
- Rate lookup order is the most specific match first: `(model, version, deploymentType, region)` → `(model, deploymentType, *)` → `(model, *, *)`.
- Requests that return 4xx or 5xx with no usage cost $0.
- Keep full decimal precision and round only for presentation.

### 6.5 Fixed-capacity cost allocation (PTU, managed compute, fine-tune hosting)
These costs accrue **per hour per capacity pool**, whether or not anyone sends requests. Allocate them every hour, for each pool `p` (a PTU deployment, a managed online deployment, or a hosted fine-tune):

1. **Hourly cost.**
   - Live: `hourCost(p,h) = capacity × minutesDeployed/60 × rate(ptuHour | vmHour | hostingHour)`. Use the reservation's effective rate when the deployment is covered.
   - Replace it with billed `AmortizedCost` from Cost Management once that arrives (section 7).
2. **Weights.** `weight(user,p,h) = Σ shadowUsd` of the user's requests on `p` in hour `h`. `shadowUsd` is the request priced at the **equivalent pay-per-token rates** for the same model (section 6.4), so output and cached tokens are weighted the way they consume capacity. If a model has no per-token rate (category F), use total tokens, or failing that `requests × latencyMs`.
3. **Allocate.** `allocatedUsd(user,p,h) = hourCost(p,h) × weight(user,p,h) / Σ_users weight(·,p,h)`.
4. **Idle capacity.** For hours with no traffic, or capacity not used, choose one policy and record it:
   - **(a) Showback:** put idle cost in an `unallocated:{pool}` bucket owned by the deployment's cost center. This is the default and the most transparent.
   - **(b) Full chargeback:** spread each day's idle cost over that day's users, proportional to their weights.

While the hour is in progress, show `shadowUsd` as the **live estimate** for capacity-billed requests, with `status = estimated-capacity`. Replace it with `allocatedUsd` when the hour closes.

Under PTU, a user's cost depends on everyone else's usage in that hour. Show both `shadowUsd` (what it would cost on pay-per-token) and `allocatedUsd` (actual share), so users and capacity owners can judge PTU efficiency.

---

## 7) Reconciliation to billed dollars (accuracy)

1. **Pull billed cost daily** with the Cost Management Query API: `POST {scope}/providers/Microsoft.CostManagement/query?api-version=2023-11-01`.
   - Daily granularity.
   - Group by `ResourceId`, `MeterId`, `Meter`, `MeterSubcategory`, and `PublisherType`.
   - Include `tags` if you tag deployments.
   - For line-level detail, use the **Cost Details API** (`generateCostDetailsReport`) or scheduled **Exports** (FOCUS format).

| Category | Scope to query | Metric |
|---|---|---|
| A, B, G | Foundry resource(s) | `ActualCost` |
| C: Claude CCU | Foundry resource(s) (Marketplace publisher) | `ActualCost` |
| D: Partner Marketplace | **Resource group(s)** that contain Foundry projects (Marketplace resources) | `ActualCost` |
| E: PTU | Foundry resource(s) | **`AmortizedCost`**, which spreads reservation purchases into the hours they cover |
| F: Managed compute | Azure ML workspace / endpoint resources | `ActualCost` (VM + surcharge) |

2. **Map billed rows to estimates** with `azureMeterId` (`PriceCard`) and `ResourceId` (`DeploymentCatalog`). For meters that don't map 1:1 to a model and meter, fall back to `(day, resource, billingCategory)`.
3. **True-up factor** per `(day, mapping key)`: `factor = billedUsd / Σ (estUsd or allocatedUsd)`. With a correct rate card, the factor should be about 1.00. Alert outside ±2%, which signals a rate card error, lost events, bypass traffic, or an unpriced deployment.
4. **Allocate to individuals:** `reconciledUsd(user) = Σ cost(user) × factor`. Every billed dollar is then attributed, plus the `unallocated:*` buckets, and the totals add up to the invoice.
5. **Finality.** Cost Management data typically arrives 8–24h after usage, and Marketplace charges can take longer. Mark days *final* after month close (around day 5 of the next month) and recompute.

Report `estimatedUsd` (live), `allocatedUsd` (capacity, hourly), and `reconciledUsd` (billed), with `status` of `estimated`, `estimated-capacity`, `provisional`, or `final`.

---

## 8) Completeness check (Foundry and Azure ML telemetry)

- **Foundry resources (A, B, C, E, G):** Call the Azure Monitor Metrics REST API (`{resourceId}/providers/Microsoft.Insights/metrics`) for the input and output token metrics at `PT1H`, split by `ModelDeploymentName`. For PTU, also pull the provisioned-managed **utilization** metric to sanity-check the allocation weights.
- **Managed online endpoints (F):** Request count and latency metrics per endpoint deployment.
- Compare hourly with `Σ UsageMeters` per deployment. A gap means traffic bypassed APIM or events were lost.
- **Enforce the gateway path** on every backend:
  - Set `disableLocalAuth=true` on Foundry resources.
  - Use key-less (Entra) auth on managed endpoints where supported.
  - Grant `Cognitive Services User` (or the equivalent) **only** to the APIM managed identity.
  - Use private endpoints or network ACLs.

---

## 9) Usage API (how consumers get the numbers)

Expose a small read API, for example an Azure Function behind the same APIM instance protected by Entra ID, that queries ADX:

- `GET /usage/sessions/{sessionId}` returns per-user totals for the session, each with a per-model breakdown across all categories.
- `GET /usage/users/{userId}?from=&to=` returns a user's total across all models and sessions.
- `GET /usage/summary?from=&to=&groupBy=user|model|category|deployment|day`

Example response, for one user across four billing categories in one session:
```json
{
  "sessionId": "abc-123",
  "currency": "USD",
  "asOfUtc": "2026-10-29T14:10:05Z",
  "users": [
    {
      "userId": "8f1c...-oid", "displayName": "Jane Doe",
      "estimatedUsd": 5.0417, "reconciledUsd": null, "status": "estimated",
      "models": [
        { "model": "claude-sonnet-4-5", "category": "C", "deploymentType": "GlobalStandard",
          "requests": 42, "units": { "input": 512340, "cacheWrite5m": 120000, "cacheRead": 2100400, "output": 48211 },
          "estimatedUsd": 3.9020 },
        { "model": "gpt-4.1", "category": "A", "deploymentType": "DataZoneStandard",
          "requests": 11, "units": { "input": 88000, "cachedInput": 40000, "output": 9100 },
          "estimatedUsd": 0.3510 },
        { "model": "DeepSeek-R1", "category": "B", "deploymentType": "GlobalStandard",
          "requests": 3, "units": { "input": 12000, "output": 21000 }, "reasoningTokens": 17500,
          "estimatedUsd": 0.1190 },
        { "model": "gpt-4o", "category": "E", "deploymentType": "GlobalProvisionedManaged",
          "requests": 20, "units": { "input": 150000, "output": 12000 },
          "shadowUsd": 0.4950, "allocatedUsd": 0.6697, "status": "estimated-capacity" }
      ]
    }
  ],
  "unallocatedCapacityUsd": 0.0
}
```

### 9.1 Core query (KQL, ADX / Eventhouse)
```kusto
let session = "abc-123";
let usage = UsageMetersDedup                                   // one row per request x meter
    | where sessionId == session and statusCode < 400
    | join kind=inner (DeploymentCatalog) on deploymentKey
    | where timestampUtc >= validFromUtc and timestampUtc < coalesce(validToUtc, datetime(2100-01-01))
    | extend priceDeploymentType = iff(isBatch, replace_string(deploymentType, "Standard", "Batch"), deploymentType);
// Variable-cost and shadow-cost pricing (most-specific PriceCard row resolved in PriceCardResolved)
let priced = usage
    | join kind=leftouter (PriceCardResolved) on modelName, $left.priceDeploymentType == $right.deploymentType, region, meter
    | where isnull(effectiveFromUtc) or (timestampUtc >= effectiveFromUtc and timestampUtc < coalesce(effectiveToUtc, datetime(2100-01-01)))
    | extend lineUsd = todouble(units) / unitSize * usdPerUnitSize * (1 - discountPct)
    | summarize perUnitUsd = sumif(lineUsd, costBasis == "perUnit"),
                shadowUsd  = sumif(lineUsd, costBasis == "capacity"),
                unpriced   = countif(isnull(usdPerUnitSize)),
                requests   = dcount(apimRequestId),
                units      = make_bag(bag_pack(meter, units))
      by userId, modelName, billingCategory, deploymentType, deploymentKey;
// Capacity allocation already computed hourly into CapacityAllocation (userId, sessionId, deploymentKey, allocatedUsd)
priced
| join kind=leftouter (CapacityAllocation | where sessionId == session
                       | summarize allocatedUsd = sum(allocatedUsd) by userId, deploymentKey) on userId, deploymentKey
| extend estimatedUsd = perUnitUsd + coalesce(allocatedUsd, shadowUsd)
| join kind=leftouter (UserDirectory) on userId
| summarize userTotalUsd = sum(estimatedUsd), unpricedLines = sum(unpriced),
            models = make_list(bag_pack("model", modelName, "category", billingCategory, "deploymentType", deploymentType,
                                        "requests", requests, "units", units, "estimatedUsd", estimatedUsd,
                                        "shadowUsd", shadowUsd, "allocatedUsd", allocatedUsd))
  by userId, displayName
| order by userTotalUsd desc
```
- `PriceCardResolved` is a function that returns, for each `(modelName, deploymentType, region, meter, window)`, the most specific matching `PriceCard` row (falling back to `*`).
- Any `unpricedLines > 0` is surfaced and alerts.
- `units` in the `summarize` above is simplified. In practice, sum units per meter before `make_bag`.

---

## 10) Implementation steps

| # | Step | Artifact |
|---|---|---|
| 1 | Policy fragment `usage-identity` (Entra / subscription identity, `userId`, `sessionId`) at All APIs scope. Set `deploymentKey` in each model API. | `policy/` |
| 2 | Policy fragment `usage-capture` (normalized meters + `log-to-eventhub`) at All APIs outbound. Map Claude cache reads into the OpenAI response. | `policy/` |
| 3 | Event Hub + APIM logger `usage-eventhub-logger` (managed identity auth) | IaC (Bicep) |
| 4 | ADX / Fabric Eventhouse: tables in section 5, streaming ingestion, `mv-expand` update policy, dedup materialized views | IaC + KQL |
| 5 | `DeploymentCatalog` sync (ARM deployments for Foundry, Azure ML online endpoints, Marketplace serverless endpoints) every 15 minutes | Function |
| 6 | `PriceCard` refresh: Retail Prices API (A, B, E, F, G), Price Sheet API (negotiated), CCU rates (C), Marketplace offers (D). Alert on unpriced deployments. | Function |
| 7 | Hourly capacity job: `CapacityHourly` + `CapacityAllocation` (section 6.5), with an agreed idle-capacity policy | Function |
| 8 | Batch output reader for `/batches` jobs (section 4.3) | Function |
| 9 | Daily Cost Management pull (all scopes in section 7) → `BilledDaily`, true-up factors, variance alerts | Function |
| 10 | Hourly completeness check (section 8). Lock every backend to APIM only. | Function + IaC |
| 11 | Enable APIM generative AI gateway logs and `llm-emit-token-metric` for cross-checks and dashboards | IaC |
| 12 | Usage API (`/usage/sessions/{id}`, `/usage/users/{id}`, `/usage/summary`) behind APIM with Entra auth | Function + APIM API |
| 13 | Dashboard: estimated vs allocated vs reconciled per user, per model, per category; PTU efficiency (`shadowUsd` vs `allocatedUsd`) | ADX dashboard / Power BI / Workbook |

### Required roles
- **APIM managed identity:** `Azure Event Hubs Data Sender` on the Event Hub; `Cognitive Services User` on Foundry resources; the scoring role on managed endpoints.
- **Catalog / pricing / reconciliation identity:**
  - `Reader` on the Foundry and Azure ML resources.
  - `Cost Management Reader` on the subscriptions / resource groups.
  - `Monitoring Reader`.
  - Billing-account `Billing Reader` (needed for the Price Sheet API).
- **Usage API identity:** `Viewer` on the ADX database.

---

## 11) Validation checklist

1. **Per category:** For one request each to Claude (C), a GPT model (A, including a cached-prompt hit), a sold-by-Azure open model (B), and an embeddings call, verify that `UsageMeters` has the expected non-overlapping meters. Hand-calculate the dollars and compare with the API output.
2. **Cache semantics:** a Chat Completions request with `cached_tokens > 0` produces `input + cachedInput == prompt_tokens`. A Claude request produces `input`, `cacheWrite*`, and `cacheRead` separately.
3. **Reasoning models:** `output` equals `completion_tokens` (reasoning is not added twice).
4. **Batch:** a batch job's usage is priced at batch rates and attributed to the submitting user and session.
5. **PTU:** in an hour with two users on one PTU deployment, `Σ allocatedUsd + unallocated == hourCost`.
6. **Multi-model session:** one user across four models returns one user entry whose model lines sum to the user total.
7. **Completeness:** hourly `Σ UsageMeters` tokens per deployment are within 0.5% of the Azure Monitor token metrics.
8. **Reconciliation:** after Cost Management data lands, the true-up factor per category is 1.00 ± 0.02.
9. **Coverage:** every deployment in `DeploymentCatalog` has `PriceCard` rows (no unpriced lines).
10. **Lockdown:** direct calls that bypass APIM are rejected on every backend. Error responses are $0, and missing identity returns 401.

## 12) Known limitations and risks

- **Foundry portal cost estimates** (Monitor tab) exclude private-offer and negotiated discounts. Use Cost Management and the invoice for billed dollars.
- **Partner Marketplace prices (D)** usually aren't in the Retail Prices API and must be maintained from the offer. Their charges land in the resource group, not the Foundry resource.
- **PTU and managed compute** costs are allocations, not per-request facts. The idle-capacity policy (section 6.5) is a business decision that must be agreed with finance.
- **Models without usage in the response** (legacy image or audio, some custom managed-compute models) rely on request-count or duration weights, and are corrected by reconciliation.
- **Streaming and broken connections** may produce no usage. Reconciliation allocates those dollars proportionally.
- **Event Hub delivery is best-effort.** Dedupe, the completeness check, and reconciliation cover loss or duplication.
- **The session is only as good as the client.** If clients don't send `X-Session-Id`, each request is its own session.
- **Prices change** (Claude rate changes apply immediately, and Azure meters change with model versions), so `PriceCard` must be effective-dated and refreshed daily.
- **The `usage-capture` fragment's response parsing** must be extended whenever a new response shape or meter is introduced. The unpriced-line alert catches gaps.
