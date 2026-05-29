[ex2_ai_prompts.md](https://github.com/user-attachments/files/28377154/ex2_ai_prompts.md)
# AI Prompts & Reasoning Log — Exercise 2

## Tool used: Claude (claude.ai)

---

## Prompt 1 — Documentation analysis before integration
**Goal:** Read the Kira API docs and identify the integration path and potential gaps before making any calls.

> "Read the Kira API documentation at kira-financial-ai.readme.io. What is the minimum happy path to: create a user → KYB → create virtual account → simulate deposit → payout? What fields are required at each step? What could go wrong?"

**Claude output:** Identified the 5-step integration path. Flagged upfront that:
- Virtual account creation requires user to be VERIFIED first
- Idempotency keys are required on all mutating endpoints
- The `X-Api-Version` header is recommended but not enforced
- Two separate auth mechanisms exist (Bearer token + x-api-key) which is unusual and worth testing

---

## Prompt 2 — Gap hunting strategy
**Goal:** Define what types of gaps to look for beyond obvious validation errors.

> "What are the categories of API gaps that hurt integrators the most? I want to prioritize findings by real impact, not just surface-level errors."

**Claude output — gap taxonomy used:**
1. **Enum mismatches** — documented values that don't work in the API
2. **Undocumented required fields** — fields not in docs that cause 400s
3. **Misleading error messages** — errors that point to wrong cause
4. **Silent deprecations** — deprecated fields still returned
5. **Missing flow documentation** — steps that are implied but not stated
6. **Inconsistent error formats** — different error shapes across endpoints

---

## Prompt 3 — Interpreting the enum mismatch finding
**Goal:** Understand why the type enum mismatch (docs say `usa-virtual-accounts-act`, API expects `US_BANK`) is the highest-impact finding.

> "I got this error: 'type is Invalid enum value. Expected US_BANK | MX_SPEI, received usa-virtual-accounts-act'. The docs say usa-virtual-accounts-act is valid. Why is this the most critical finding?"

**Claude reasoning:**
- This is the very first field an integrator sets after auth — it blocks all downstream work
- The docs use product codes (user-facing) while the API uses infrastructure codes (internal) — the abstraction layer is broken
- An integrator has no way to discover the correct value without hitting the error or contacting support
- It inflates time-to-first-successful-call, the metric Kira explicitly cares about

---

## Prompt 4 — BDD scenario writing
**Goal:** Write Gherkin scenarios precise enough for an engineer to build failing tests without asking questions.

> "Write Gherkin BDD scenarios for this finding: POST /v1/virtual-accounts with type='usa-virtual-accounts-act' returns 400 with enum mismatch error. The scenario should be concrete enough that an engineer could turn it into a failing test without asking me a question."

**Claude output:** Structured scenarios with concrete request bodies, exact expected response bodies, and a scenario showing both the broken state and the expected fixed state.

---

## Prompt 5 — Ranking findings by impact
**Goal:** Rank all 7 gaps found by integrator impact, not just severity.

> "I found 7 gaps in the Kira API. Help me rank them by 'integrator impact' — meaning which ones would cost a real client the most time, cause silent breakage, or block go-live."

**Claude reasoning framework used:**
- **Time cost:** How long does it take to discover and resolve this gap without help?
- **Silent vs loud:** Silent errors (deprecated field still returned) are worse than loud ones (clear 400)
- **Go-live blocking:** Does this prevent the integration from working at all?
- **Systemic vs one-off:** Does this affect every integrator or only some?

Final ranking: enum mismatch (#1) > undocumented bank field (#2) > verification flow dependency (#3) > idempotency message (#4) > deprecated field (#5)
