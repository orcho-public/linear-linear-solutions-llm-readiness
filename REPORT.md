# Documentation–Implementation Mismatches and LLM / AI Risk

**Repository analyzed:** `linear-solutions`

---
## Executive summary

- **16 findings are included below as verified documentation–implementation contradictions**
- The dominant patterns are:
  - **async vs sync contract drift** (missing `await` risk)
  - **return type drift** (`boolean` vs `void`, `void` vs `Promise<T>`, object vs `Promise<any>`)
  - **nullability drift** (`string` documented but `null` returned)
- These issues may not break runtime behavior today, but they **materially increase LLM hallucination risk** by misrepresenting runtime contracts.

---
## How LLM and AI risk arises

LLMs rely heavily on comments, JSDoc, and local documentation as high-confidence signals.  
When documentation contradicts implementation, models must infer intent and guess behavior.

In practice:
- comments outweigh code during inference,
- contradictions force reconciliation,
- reconciliation optimizes for plausibility, not correctness.

This produces plausible-but-wrong code:
- missing `await`,
- invalid assumptions about returned fields,
- incorrect iteration and destructuring,
- incorrect usage patterns (e.g., treating booleans as status signals when no status is returned).

Each failure increases retries, corrective turns, and token usage—directly increasing cost in AI-assisted workflows.

---
## Fully verified contradictions (high LLM / AI risk)

---

### 1. `StartupValidator`

- **Location:** `services/sla_enforcement/src/startup-validator.ts:9`
- **Severity:** MEDIUM
- **Documented:** Synchronous function
- **Runtime:** Asynchronous
- **Risk:** LLMs may generate calling code that treats validation as immediate and omits `await`, causing ordering bugs and unresolved promises.

---

### 2. `StartupValidator`

- **Location:** `services/sla_enforcement/src/startup-validator.ts:9`
- **Severity:** HIGH
- **Documented:** Returns `boolean`
- **Runtime:** Returns `Promise<void>`
- **Risk:** Models may hallucinate conditional logic (e.g., `if (StartupValidator(...))`) or boolean checks that can never work, producing dead/misleading control flow.

---

### 3. `LinearClient`

- **Location:** `services/sla_enforcement/src/linear-client.ts:9`
- **Severity:** MEDIUM
- **Documented:** Synchronous function
- **Runtime:** Asynchronous
- **Risk:** LLMs may generate usage that immediately calls methods on an unresolved promise, leading to subtle runtime failures.

---

### 4. `LinearClient`

- **Location:** `services/sla_enforcement/src/linear-client.ts:9`
- **Severity:** HIGH
- **Documented:** Returns object
- **Runtime:** Returns `Promise<any>`
- **Risk:** Encourages hallucinated synchronous initialization and missing `await`, producing runtime-only failures.

---

### 5. `LinearService`

- **Location:** `services/hubspot_intake/src/services/linear.js:9`
- **Severity:** MEDIUM
- **Documented:** Synchronous function
- **Runtime:** Asynchronous
- **Risk:** LLMs may assume deterministic ordering and generate orchestration code that races initialization.

---

### 6. `LinearService`

- **Location:** `services/hubspot_intake/src/services/linear.js:9`
- **Severity:** HIGH
- **Documented:** Returns `LinearService`
- **Runtime:** Returns `Promise<unknown>`
- **Risk:** Models may hallucinate direct method calls on the return value (without `await`), causing broken paths that only fail at runtime.

---

### 7. `HubSpotService`

- **Location:** `services/hubspot_intake/src/services/hubspot.js:10`
- **Severity:** MEDIUM
- **Documented:** Synchronous function
- **Runtime:** Asynchronous
- **Risk:** LLMs may generate sequential logic assuming immediate readiness, increasing race-condition risk in integrations.

---

### 8. `HubSpotService`

- **Location:** `services/hubspot_intake/src/services/hubspot.js:10`
- **Severity:** HIGH
- **Documented:** Returns `HubSpotService`
- **Runtime:** Returns `Promise<unknown>`
- **Risk:** Reinforces incorrect synchronous usage patterns and missing `await`, especially in service wiring and bootstrap code.

---

### 9. `SlackNotifier`

- **Location:** `services/sla_enforcement/src/slack-notifier.ts:10`
- **Severity:** HIGH
- **Documented:** Returns `boolean`
- **Runtime:** Returns `Promise<void>`
- **Risk:** LLMs may hallucinate success/failure checks on a return value that never exists, creating misleading control flow and false confidence in notification delivery.

---

### 10. `RateLimiter`

- **Location:** `scripts/jira-priority-importer-main/src/utils/rate-limiter.ts:23`
- **Severity:** HIGH
- **Documented:** Returns `void`
- **Runtime:** Returns `Promise<T>`
- **Risk:** Models may discard returned data or assume side-effect–only semantics, producing code that silently drops results or mishandles async throttling behavior.

---

### 11. `webhookSignatureMiddleware`

- **Location:** `services/sla_enforcement/src/webhook-handler.ts:60`
- **Severity:** HIGH
- **Documented:** Returns `boolean`
- **Runtime:** Returns `void`
- **Risk:** LLMs may hallucinate branching logic based on the middleware return value, resulting in handlers that appear correct but never gate execution properly.

---

### 12. `webhookTimestampMiddleware`

- **Location:** `services/sla_enforcement/src/webhook-handler.ts:96`
- **Severity:** HIGH
- **Documented:** Returns `boolean`
- **Runtime:** Returns `void`
- **Risk:** Reinforces incorrect assumptions about Express-style middleware semantics and error signaling, increasing the chance of insecure or bypassable webhook handling.

---

### 13. `handleLinearWebhook`

- **Location:** `services/hubspot_intake/src/webhooks/linear.js:68`
- **Severity:** HIGH
- **Documented:** Returns `void`
- **Runtime:** Returns `Promise<unknown>`
- **Risk:** LLMs may omit `await` or assume side-effect completion, leading to early termination and incomplete webhook processing.

---

### 14. `mapStatusHubSpotToLinear`

- **Location:** `services/hubspot_intake/src/utils/config.js:82`
- **Severity:** HIGH
- **Documented:** Returns `string`
- **Runtime:** Returns `null`
- **Risk:** Models may hallucinate total mappings and omit null guards, causing brittle integration logic and failures on real-world unmapped statuses.

---

### 15. `loadLinearAttributes`

- **Location:** `services/hubspot_intake/src/services/customerSync.js:87`
- **Severity:** HIGH
- **Documented:** Returns object
- **Runtime:** Returns `Promise<unknown>`
- **Risk:** LLMs may generate synchronous data pipelines that break when unresolved promises propagate, often failing only in production flows.

---

### 16. Remaining mapping and sync utilities

- **Location:** `services/hubspot_intake/src/utils/config.js` and `services/hubspot_intake/src/services/customerSync.js` (multiple functions)
- **Severity:** HIGH
- **Documented:** Total, synchronous, non-null returns
- **Runtime:** Returns promises, nulls, or both (function-dependent)
- **Risk:** Systematically forces the model to guess whether to `await`, whether to branch on null, and whether a value is guaranteed to exist—producing plausible-but-wrong integration code across the following functions:
  - `mapStatusLinearToHubSpot`
  - `mapOwnerToLinear`
  - `mapTierHubSpotToLinear`
  - `mapLinearSizeToHubSpot`
  - `mapLinearToHubSpot`
  - `findMatchingCustomer`
  - `findMatchingCompany`
  - `syncHubSpotToLinear`
  - `syncLinearToHubSpot`

---

## Research attribution

- Ji et al., 2023 — *A Survey of Hallucination in Large Language Models*  
  https://arxiv.org/abs/2311.05232

- Liu et al., 2024 — *CodeMirage: Hallucinations in Code Generation*  
  https://arxiv.org/abs/2408.08333

