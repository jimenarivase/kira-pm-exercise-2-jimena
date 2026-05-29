
[ex2_request_response_log.md](https://github.com/user-attachments/files/28377194/ex2_request_response_log.md)
# Request / Response Log

## Environment
- Base URL: `https://api.balampay.com/sandbox`
- Date: 2026-05-29
- Tool: Postman

---

## 1. POST /auth — Authentication
**Request:**
```
POST https://api.balampay.com/sandbox/auth
Headers:
  x-api-key: <redacted>
  Content-Type: application/json

Body:
{
  "client_id": "<redacted-uuid>",
  "password": "<redacted>"
}
```

**Response: 200 OK**
```json
{
  "message": "Auth token",
  "data": {
    "access_token": "eyJraWQi...<truncated>",
    "expires_in": 3600,
    "token_type": "Bearer"
  }
}
```
**Notes:** Auth works. Token expires in 1 hour — no refresh token mechanism documented.

---

## 2. POST /v1/users — Create User (attempt 1: wrong type)
**Request:**
```
POST https://api.balampay.com/sandbox/v1/users
Headers:
  Authorization: Bearer <token>
  x-api-key: <redacted>
  idempotency-key: 550e8400-e29b-41d4-a716-446655440001
  Content-Type: application/json

Body:
{
  "first_name": "Jimena",
  "last_name": "Rivas",
  "email": "jime.rivase@gmail.com",
  "phone": "+529999494029",
  "type": "business"
}
```

**Response: 400 Bad Request**
```json
{
  "error": "Invalid request data",
  "details": [
    {
      "path": "business_legal_name",
      "message": "Required",
      "code": "invalid_type"
    }
  ]
}
```
**Gap identified:** business type requires business_legal_name — expected, but not surfaced upfront.

---

## 3. POST /v1/users — Create User (idempotency key reuse)
**Request:** Same endpoint, same idempotency key, different body.

**Response: 400 Bad Request**
```json
{
  "code": "validation_error",
  "message": "Failed to store idempotency key"
}
```
**Gap #4 identified:** Misleading error message. Should say "idempotency_key_reused" per docs.

---

## 4. POST /v1/users — Create User (verification_link mode)
**Request:**
```json
{
  "type": "individual",
  "verification_mode": "verification_link",
  "first_name": "Jimena",
  "last_name": "Rivas",
  "email": "jimena.test@example.com"
}
```
**idempotency-key:** 550e8400-e29b-41d4-a716-446655440099

**Response: 201 Created**
```json
{
  "id": "01299930-4af7-4875-ae12-ff7f94ebaa56",
  "type": "individual",
  "email": "jimena.test@example.com",
  "status": "CREATED",
  "verification_status": "unverified",
  "verification_mode": "verification_link",
  "verification_link": "https://verify-sandbox.aiprise.com/?..."
}
```
**Gap #5 identified:** Both `status` and deprecated `verification_status` returned simultaneously.

---

## 5. POST /v1/virtual-accounts — Attempt 1: wrong type enum
**Request:**
```json
{
  "user_id": "01299930-4af7-4875-ae12-ff7f94ebaa56",
  "type": "usa-virtual-accounts-act"
}
```

**Response: 400 Bad Request**
```json
{
  "message": "type is Invalid enum value. Expected 'US_BANK' | 'MX_SPEI', received 'usa-virtual-accounts-act'"
}
```
**Gap #1 identified:** Enum mismatch between docs and API.

---

## 6. POST /v1/virtual-accounts — Attempt 2: missing bank field
**Request:**
```json
{
  "user_id": "01299930-4af7-4875-ae12-ff7f94ebaa56",
  "type": "US_BANK"
}
```

**Response: 400 Bad Request**
```json
{
  "message": "bank is bank is required for US_BANK virtual accounts"
}
```
**Gap #2 identified:** Undocumented required field. Also contains typo "bank is bank is".

---

## 7. POST /v1/virtual-accounts — Attempt 3: wrong bank enum
**Request:**
```json
{
  "user_id": "01299930-4af7-4875-ae12-ff7f94ebaa56",
  "type": "US_BANK",
  "bank": "ACT"
}
```

**Response: 400 Bad Request**
```json
{
  "message": "bank is Invalid enum value. Expected 'portage' | 'slovak_savings_bank' | 'austin_capital_trust', received 'ACT'"
}
```
**Gap #2 continued:** Valid enum values only discoverable via error message.

---

## 8. POST /v1/virtual-accounts — Attempt 4: user not verified
**Request:**
```json
{
  "user_id": "01299930-4af7-4875-ae12-ff7f94ebaa56",
  "type": "US_BANK",
  "bank": "austin_capital_trust"
}
```

**Response: 400 Bad Request**
```json
{
  "statusCode": 400,
  "message": "User must be in VERIFIED status to create a virtual account. Current status: CREATED"
}
```
**Gap #3 identified:** verification_link flow dependency not documented upfront.
