# Azure AI Gateway Plan: OpenAI-Compatible Frontend to Anthropic Messages Backend

This solution enables GitHub Copilot (OpenAI-compatible client) to call Claude models hosted in Microsoft Foundry by using Azure API Management (APIM) as an AI Gateway that transforms request/response payloads.

## 1) Target architecture

1. GitHub Copilot / client calls APIM using OpenAI-style endpoint:
   - `POST /v1/chat/completions`
2. APIM inbound policy:
   - validate auth
   - enforce/add `X-Correlation-Id`
   - transform OpenAI payload -> Anthropic Messages payload
   - route to Foundry Claude backend
3. Foundry Claude backend receives Anthropic Messages request.
4. APIM outbound policy:
   - transform Anthropic response -> OpenAI Chat Completions response
   - return `X-Correlation-Id` to caller
5. App Insights / logs capture correlation ID + request ID + model + token usage.

## 2) API contract strategy

Create one APIM API that exposes OpenAI-compatible operations:

- `POST /v1/chat/completions` (phase 1 mandatory)
- `POST /v1/responses` (optional later)
- `GET /v1/models` (optional compatibility shim)

Start with non-streaming only (`"stream": false`) for first release.  
Streaming (SSE) transform is phase 2.

## 3) Policy pipeline (inbound/outbound)

### Inbound

- `set-variable` `correlationId`:
  - use incoming `X-Correlation-Id` if present
  - else generate from `context.RequestId`
- `set-header` `X-Correlation-Id` (to backend)
- `set-header` `anthropic-version` (required by backend contract)
- `set-body` transform:
  - OpenAI `messages[]` -> Anthropic `messages[]`
  - map system message(s) to Anthropic `system`
  - map `max_tokens`, `temperature`, `top_p`, `stop`, `tools` (if needed)
- `set-backend-service` to Foundry Claude endpoint

### Outbound

- `set-body` transform:
  - Anthropic `content[]` text -> OpenAI `choices[0].message.content`
  - map stop reason -> `finish_reason`
  - map usage fields (`input_tokens`, `output_tokens`) -> OpenAI usage
- `set-header` `X-Correlation-Id` back to client
- add observability dimensions (correlation, model, status, latency, tokens)

## 4) Field mapping matrix (core)

| OpenAI (frontend) | Anthropic Messages (backend) |
|---|---|
| `model` | mapped backend/deployment (policy lookup) |
| `messages[role=user/assistant]` | `messages[role=user/assistant]` |
| `messages[role=system]` | `system` |
| `max_tokens` | `max_tokens` |
| `temperature` | `temperature` |
| `top_p` | `top_p` |
| `stop` | `stop_sequences` |
| `stream` | initially reject/force false in phase 1 |

Response mapping:

| Anthropic | OpenAI |
|---|---|
| `id` | `id` |
| `content[].text` | `choices[0].message.content` |
| `stop_reason` | `choices[0].finish_reason` |
| `usage.input_tokens` | `usage.prompt_tokens` |
| `usage.output_tokens` | `usage.completion_tokens` |

## 5) Correlation-ID design (required)

Use one canonical header: `X-Correlation-Id`.

Rules:
1. Accept caller value if provided.
2. If missing, generate in APIM (fallback to APIM request ID).
3. Forward same value to backend.
4. Return same value in response.
5. Log it in APIM traces + App Insights custom dimensions.
6. Include it in error payloads for 4xx/5xx troubleshooting.

## 6) Security and config

- Keep backend credentials in APIM Named Values + Key Vault references.
- Apply APIM subscription key/OAuth on frontend as needed.
- Add rate-limit + token-limit policies to protect Claude quota.
- Optional: content safety policy before backend call.

## 7) Rollout plan

### Phase 1 (MVP)

- `/v1/chat/completions` only
- non-streaming
- text-only responses
- correlation ID end-to-end
- token usage mapping + logging

### Phase 2

- SSE streaming transform
- tool/function-call translation
- model routing (multi-Claude deployment failover)
- stronger policy governance (cache, safety, throttling tiers)

## 8) Validation checklist

1. OpenAI-format request succeeds through APIM.
2. Backend receives Anthropic-format payload.
3. Client receives valid OpenAI-format response.
4. `X-Correlation-Id` is identical in:
   - client request
   - APIM logs
   - backend request logs
   - client response
5. Error path preserves correlation ID.
6. Token usage fields are correctly populated.
