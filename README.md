kira-pm-exercise-2-jimena
API Integration & Error Hunt
Google Sheet (Exercise 1)
https://docs.google.com/spreadsheets/d/1eDmHIQL-1RyyDmX2qR3GWHStDdRD9fNN6AhJPk2USB0


Integration Path Executed
POST /auth → POST /v1/users → POST /v1/virtual-accounts

All requests made against https://api.balampay.com/sandbox. Raw request/response logs in evidence/work/.


Top 5 Findings (ranked by integrator impact)
#1 — Virtual account type enum mismatch between docs and API
Why this matters to a client: An integrator reads the docs, sends "type": "usa-virtual-accounts-act" exactly as documented, and gets a cryptic 400. They have no idea the API actually expects US_BANK or MX_SPEI. This blocks go-live and requires a support ticket to resolve — avoidable with one line fix in the docs.

Evidence: POST /v1/virtual-accounts with "type": "usa-virtual-accounts-act" returns:

{

  "message": "type is Invalid enum value. Expected 'US_BANK' | 'MX_SPEI', received 'usa-virtual-accounts-act'"

}

See: features/01_virtual_account_type_enum_mismatch.feature


#2 — bank field required for US_BANK but undocumented; valid values only discoverable via error
Why this matters to a client: Even after fixing the type enum, the integrator hits another undocumented required field (bank). Its valid values (portage, slovak_savings_bank, austin_capital_trust) are nowhere in the docs — the only way to discover them is to send a wrong value and read the error. Two sequential undocumented blockers kill integrator confidence and inflate time-to-first-successful-call.

Evidence: POST /v1/virtual-accounts with "type": "US_BANK" returns:

{

  "message": "bank is bank is required for US_BANK virtual accounts"

}

Then with "bank": "ACT":

{

  "message": "bank is Invalid enum value. Expected 'portage' | 'slovak_savings_bank' | 'austin_capital_trust', received 'ACT'"

}

See: features/02_bank_field_undocumented.feature


#3 — User must be VERIFIED before creating virtual account, but verification_link flow doesn't make this dependency explicit
Why this matters to a client: The docs show verification_link as a valid onboarding path, but don't warn that the user must complete KYC before you can create a virtual account. An integrator using verification_link will successfully create a user, then immediately try to create a VA and get a 400. The error message is clear ("User must be in VERIFIED status") but the flow dependency is never stated upfront — causing wasted integration cycles.

Evidence: POST /v1/virtual-accounts with verified user returns 400:

{

  "message": "User must be in VERIFIED status to create a virtual account. Current status: CREATED"

}

See: features/03_verification_status_dependency.feature


#4 — Idempotency key error message is misleading ("Failed to store" instead of "Key already used")
Why this matters to a client: When an integrator reuses an idempotency key with a different body, the error says "Failed to store idempotency key" — which sounds like a server-side infrastructure problem, not a client mistake. The integrator may retry, escalate to support, or assume the sandbox is broken. The correct message should be "Idempotency key already used with different request data".

Evidence: Reusing idempotency-key: 550e8400-e29b-41d4-a716-446655440001 with different body returns:

{

  "code": "validation_error",

  "message": "Failed to store idempotency key"

}

See: features/04_idempotency_error_message.feature


#5 — Deprecated verification_status field still returned in all user responses
Why this matters to a client: The docs mark verification_status as deprecated and tell integrators to use status instead — but every user response returns both fields with different values (status: "CREATED" vs verification_status: "unverified"). Integrators building on the deprecated field will have silent breakage when it's eventually removed. The inconsistency also creates confusion about which field is the source of truth today.

Evidence: POST /v1/users 201 response contains both:

{

  "status": "CREATED",

  "verification_status": "unverified"

}

See: features/05_deprecated_verification_status.feature


Bonus finding (not in top 5)
Typo in error message body: "bank is bank is required for US_BANK virtual accounts" — the word "bank is" is duplicated. Minor but signals lack of error message review process.


Repo Structure
kira-pm-exercise-2-jimena/

├── README.md

├── features/

│   ├── 01_virtual_account_type_enum_mismatch.feature

│   ├── 02_bank_field_undocumented.feature

│   ├── 03_verification_status_dependency.feature

│   ├── 04_idempotency_error_message.feature

│   └── 05_deprecated_verification_status.feature

├── evidence/

│   ├── ai/

│   │   └── prompts.md

│   └── work/

│       └── request_response_log.md
