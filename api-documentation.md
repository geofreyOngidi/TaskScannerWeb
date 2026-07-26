# TaxScanner — API Contract for the Spring Boot Backend

This documents every HTTP call the React front end makes, exactly as written
in `src/api/client.js`. It's generated from the actual request/response code
the UI runs — not aspirational — so if you implement each endpoint below to
spec and flip `VITE_USE_MOCK=false`, the UI should work with no frontend
changes.

---

## 1. Conventions

| | |
|---|---|
| **Base URL** | `VITE_API_BASE_URL` (defaults to `/api`, proxied to `http://localhost:8080` in dev — see `vite.config.js`) |
| **Content type** | `application/json` for all request/response bodies, except the three export/download endpoints, which return a raw file |
| **Auth** | Either a session cookie (`withCredentials: true` is already set) **or** a JWT bearer token — see §2 |
| **Date format — receipts** | `invoice_date` is `YYYY-MM-DD` (ISO, no time). The UI parses it with `new Date(iso)`, so it must be ISO, not `DD-MM-YYYY`. |
| **Date format — admin** | `Admin.recentPins` dates are **pre-formatted display strings** (e.g. `"25 Jul 2026"`), not ISO. The UI prints them as-is. |
| **Money fields** | Plain JSON numbers (e.g. `4310.00`), not strings, not KES-prefixed. The UI formats currency client-side. |
| **`filter` values** | One of the strings `"7" | "30" | "60" | "90" | "365" | "all"` — mirrors the WhatsApp bot's date windows. Filtering happens **server-side**; the UI does not re-filter what it receives. |
| **`role` values (per account)** | `"owner"` \| `"manager"` |
| **`role` values (on the user object)** | `"owner"` \| `"manager"` \| `"admin"` — see the note in §3 |
| **eTIMS / verification** | KRA eTIMS integration is not live. Every receipt should be treated as pending. The UI currently **hardcodes** the "Pending Verification" label rather than reading a field — see §5.4 — but send `verification_status: "not_verified"` anyway so the API is forward-compatible. Do not introduce a `"verified"` value until eTIMS actually ships. |
| **401 handling** | Any response with HTTP `401` triggers an automatic client-side logout (see the axios response interceptor). You don't need a specific body shape for this — the status code alone is enough. |
| **Error body** | Not currently parsed by the UI beyond status codes (401, 404). Recommended shape below in §1.1 so you have one consistent convention across endpoints. |

### 1.1 Recommended error shape

Not required by the current UI, but recommended for consistency as you build out the API:

```json
{
  "error": "INVALID_PIN",
  "message": "No taxpayer found for that KRA PIN or National ID."
}
```

---

## 2. Authentication

The frontend is written to support **either** of these — pick one:

- **Session cookie** (simplest with Spring Security): axios already sends `withCredentials: true`. Configure CORS on the backend to allow credentials from your dev origin (`http://localhost:5173`), and set the session cookie on the `POST /auth/otp/verify` response.
- **JWT bearer token**: return `{ token, user }` from `POST /auth/otp/verify`. The client stores `token` in `localStorage` and attaches `Authorization: Bearer <token>` to every subsequent request automatically.

There is no separate "password" concept anywhere in this app — identity is entirely phone-number + OTP, matching the WhatsApp bot's phone-first design.

### 2.1 `POST /api/auth/otp/send`

Sends a one-time code to the given WhatsApp number (reuse whatever OTP mechanism the WhatsApp bot / Twilio integration already uses).

**Request body**
```json
{ "phone": "254712345678" }
```
`phone` is digits only, always prefixed `254`, no `+`.

**Response — `204 No Content`**
(No body expected. The UI just checks the call didn't throw.)

**Error cases**
- `400` — malformed phone number
- `404` — number not registered (only relevant if you decide unregistered numbers can't request a code; the WhatsApp bot itself allows self-registration, so consider whether the web login should too, or should tell the user to text "Hi" to the WhatsApp number first)

---

### 2.2 `POST /api/auth/otp/verify`

**Request body**
```json
{ "phone": "254712345678", "code": "482913" }
```

**Response — `200 OK`**
```json
{
  "token": "eyJhbGciOi...",
  "user": {
    "name": "Alena Kimutai",
    "phone": "254712345678",
    "role": "owner",
    "accounts": [
      { "id": 1, "kra_pin": "P051234567X", "name": "Alena Kimutai Consultancy Ltd", "role": "owner" },
      { "id": 2, "kra_pin": "A009876543Z", "name": "Riverside Traders Ltd", "role": "owner" },
      { "id": 3, "kra_pin": "P004455667Y", "name": "Coastline Imports Ltd", "role": "manager" }
    ]
  }
}
```

`token` is **optional** — omit it entirely if you're using session cookies instead.

**Response shape reference: `User`**

| Field | Type | Notes |
|---|---|---|
| `name` | string | Full display name, or company name if no personal name is on file |
| `phone` | string | Digits only, `254`-prefixed |
| `role` | `"owner" \| "manager" \| "admin"` | See note below |
| `accounts` | `Account[]` | Every account (PIN) this user can act on — see §3.1 for `Account` shape |

> **On the top-level `user.role` field:** this is *not* simply "the role on their first account." It controls exactly two things in the UI: (1) whether the Admin tab is shown at all (`role === "admin"`), and (2) the label shown next to their name in the sidebar. Per-account permissions (who can invite delegates on which PIN, Owner vs Account Manager badges) come entirely from the `role` field *inside each entry of `accounts[]]`, not this top-level field. In practice: set `role: "admin"` for platform admins; for everyone else, `"owner"` if they own at least one account, otherwise `"manager"` — but don't derive UI permissions from it beyond the two uses above.

**Error cases**
- `400` — code missing/malformed
- `401` — wrong code, or code expired

---

### 2.3 `POST /api/auth/logout`

**Request body** — none

**Response — `204 No Content`**

Invalidate the session/token server-side. The client also clears its local token and redirects to `/login` regardless of the response, so this call is best-effort from the UI's perspective — but do actually invalidate it server-side.

---

## 3. Accounts

### 3.1 `GET /api/accounts`

Returns every account (KRA PIN) the **authenticated caller** can act on — as owner or as delegated Account Manager. This is the same list returned inside `user.accounts` at login; expose it as its own endpoint too so the UI can refresh it independently (e.g. after adding a PIN) without forcing a full re-login.

**Auth required:** yes

**Response — `200 OK`**
```json
[
  { "id": 1, "kra_pin": "P051234567X", "name": "Alena Kimutai Consultancy Ltd", "role": "owner" },
  { "id": 2, "kra_pin": "A009876543Z", "name": "Riverside Traders Ltd", "role": "owner" },
  { "id": 3, "kra_pin": "P004455667Y", "name": "Coastline Imports Ltd", "role": "manager" }
]
```

**`Account` shape reference**

| Field | Type | Notes |
|---|---|---|
| `id` | number | Internal account id (matches `ts_accounts.id` / whatever your Account entity's PK is) |
| `kra_pin` | string | The KRA PIN itself |
| `name` | string | Display name — company name if company, else the taxpayer's full name |
| `role` | `"owner" \| "manager"` | This caller's relationship to *this specific account* |

---

### 3.2 `GET /api/accounts/lookup?pin={pin}`

Powers "My Accounts → Look up a PIN." Any authenticated user can look up *any* PIN — this is deliberately not scoped to the caller's own accounts, matching the "search any PIN" feature.

**Query params:** `pin` — required, the KRA PIN to search (uppercase, exact match)

**Response — `200 OK`** (PIN found)
```json
{
  "name": "Alena Kimutai Consultancy Ltd",
  "type": "Individual",
  "status": "Active",
  "owner": "Alena Kimutai (254712●●●341)",
  "managers": [
    "Brian Otieno — delegated 12 May 2026"
  ]
}
```

**Response — `404 Not Found`** (PIN not found)

The UI treats *any* non-2xx response as "not found" and shows "No account found for `{pin}`." — you don't need a specific error body, just the `404` status.

**Field notes**

| Field | Type | Notes |
|---|---|---|
| `name` | string | Account/company/taxpayer name |
| `type` | `"Individual" \| "Company"` | |
| `status` | string | e.g. `"Active"` — free text, displayed as-is in a badge |
| `owner` | string | **Pre-formatted display string** — name + masked phone, e.g. `"Alena Kimutai (254712●●●341)"`. The UI renders this verbatim; do the masking and formatting server-side. |
| `managers` | string[] | **Pre-formatted display strings**, one per delegate, e.g. `"Brian Otieno — delegated 12 May 2026"`. Same deal — format server-side, UI just lists them. |

> If you'd rather return structured data (separate `ownerName`, `ownerPhone`, `managers: [{name, phone, delegatedAt}]`, etc.) that's a perfectly reasonable API design — just flag it, since the current UI expects pre-formatted strings and would need a small update to consume structured objects instead.

---

### 3.3 `POST /api/accounts`

Powers "My Accounts → Add Another PIN." Verifies a KRA PIN or National ID (same lookup the WhatsApp bot uses — `taxscannerRequests->verify_pin_by_pin` / `verify_pin_by_id`) and links it to the caller as **owner**.

**Auth required:** yes

**Request body**
```json
{ "pinOrId": "P051234567X" }
```
`pinOrId` may be a KRA PIN (alphanumeric) or a National ID (digits only) — same dual-input behaviour as the WhatsApp registration flow. Detect which one you were given the same way the PHP backend does (`ctype_digit`).

**Response — `200 OK`**
```json
{ "id": 5, "kra_pin": "P099887766X", "name": "New Verified Taxpayer", "role": "owner" }
```
Same `Account` shape as §3.1. `role` should always be `"owner"` here — this endpoint is specifically "add a PIN I own," matching the WhatsApp `add_pin_confirm` step.

**Error cases**
- `400` — empty/malformed input
- `422` — KRA/ID verification failed (invalid PIN, PIN not found, etc.) — surface whatever message KRA's lookup service returned, same as the WhatsApp bot does today

---

### 3.4 `POST /api/accounts/invites`

Powers "Add Manager." Invites a colleague as an Account Manager on one or more accounts the caller **owns**.

**Auth required:** yes

**Request body**
```json
{
  "phone": "254733456902",
  "firstName": "Brian",
  "lastName": "Otieno",
  "email": null,
  "accountIds": ["P051234567X", "A009876543Z"]
}
```

**Important:** despite the name, `accountIds` currently holds **KRA PIN strings**, not numeric account `id`s — that's what the checkboxes in the Add Manager form are keyed on (see `src/pages/AddManager.jsx`). Either:
- accept an array of PIN strings server-side (simplest — no frontend change needed), or
- ask for the field to be renamed/changed to numeric ids and update `AddManager.jsx`'s checkbox `value` to `a.id` instead of `a.kra_pin` before you build against it.

Pick one and document it back to whoever owns the frontend.

**Response — `204 No Content`**

Server-side, this must:
1. Reject any PIN in `accountIds` that the caller does not own (mirrors the PHP backend's rule: delegates can only be invited onto accounts where the inviter is `owner`, never accounts where the inviter is themselves only a `manager`).
2. Create the colleague's user record if `phone` isn't already registered (no separate registration step for them — same as the WhatsApp bot's `create_delegate_user`).
3. Link the colleague to each account with role `manager`.
4. Send them a WhatsApp notification (reuse the existing Twilio send used by the bot).

**Error cases**
- `400` — missing phone/first name, or empty `accountIds`
- `403` — one or more of the given accounts is not owned by the caller

---

## 4. Receipts

### 4.1 `GET /api/receipts?pin={pin}&filter={filter}`

Powers "Receipts → Fetch Receipts" and the Dashboard's totals (the dashboard calls this once per account the user manages, with `filter=all`).

**Query params**

| Param | Type | Required | Notes |
|---|---|---|---|
| `pin` | string | yes | KRA PIN to fetch receipts for |
| `filter` | string | yes | `"7" \| "30" \| "60" \| "90" \| "365" \| "all"` |

**Auth required:** yes — and the caller must own or be delegated on `pin`. (The UI currently lets a user type *any* PIN into the free-text box, so enforce this server-side rather than relying on the UI only offering the user's own accounts.)

**Response — `200 OK`**
```json
[
  {
    "id": 48213,
    "supplier_name": "Naivas Supermarket Ltd",
    "supplier_pin": "P051876234K",
    "invoice_number": "INV-88213",
    "total_excl_tax": 4310.00,
    "tax_amount": 689.60,
    "invoice_amount": 4999.60,
    "invoice_date": "2026-07-24",
    "extraction_method": "ai_vision",
    "verification_status": "not_verified"
  }
]
```
Sorted newest-first. Already filtered to the requested date window server-side — the frontend does not filter again.

**`Receipt` shape reference**

| Field | Type | Notes |
|---|---|---|
| `id` | number | Receipt id — used as `Ref #R{id}` and for the single-download endpoint |
| `supplier_name` | string | |
| `supplier_pin` | string | Supplier's KRA PIN (not the account's own PIN) |
| `invoice_number` | string | |
| `total_excl_tax` | number | |
| `tax_amount` | number | |
| `invoice_amount` | number | `total_excl_tax + tax_amount` |
| `invoice_date` | string | **ISO `YYYY-MM-DD`** — see §1 |
| `extraction_method` | `"ai_vision" \| "manual"` | Shown as an "AI Scanned" / "Manual" badge |
| `verification_status` | string | Send `"not_verified"` — see the eTIMS note in §1. Not currently read by the UI (see §5.4) but include it anyway. |

> The richer PHP/WhatsApp receipt schema also has `buyer_pin`, `customer_name`, `cu_invoice_number`, `cu_serial_number`, and line items. The current React UI doesn't render any of these yet — feel free to include them in the response regardless (extra fields are simply ignored), which keeps this endpoint reusable if the UI grows a receipt-detail view later.

---

### 4.2 `GET /api/receipts/export/csv?pin={pin}&filter={filter}`

Bulk CSV export — "Download All (Excel/CSV)." Same query params as §4.1.

**Response — `200 OK`**, `Content-Type: text/csv`, raw file body (not JSON)

**Column order** (must match exactly — this is what the frontend's own client-side CSV fallback produces, so keep them identical):

```
Invoice Number, Supplier PIN, Supplier Name, Amount Excl. Tax, Tax on Invoice, Invoice Amount, Date, eTIMS Compliance
```

`eTIMS Compliance` should currently always be `"Pending Verification"`.

Suggested filename: `receipts_{pin}_{filter}.csv` (the frontend sets this as the download filename either way, so the header on your response doesn't strictly need to match, but `Content-Disposition: attachment; filename="..."` is good practice).

---

### 4.3 `GET /api/receipts/export/pdf?pin={pin}&filter={filter}`

Bulk PDF export — "Download All (PDF)." Same query params as §4.1.

**Response — `200 OK`**, `Content-Type: application/pdf`, raw file body

Layout is up to you server-side, but for consistency with both the WhatsApp bot's PDF export and the frontend's own client-side fallback (which runs today while `VITE_USE_MOCK=true`, via `jspdf`), use the same column order as the CSV in §4.2, paginated, landscape.

---

### 4.4 `GET /api/receipts/{id}/download?format={format}`

Single-receipt download — the "PDF" button on each receipt row.

**Path param:** `id` — the receipt id

**Query param:** `format` — `"image" | "pdf"`. The current UI only ever sends `"pdf"`, but the client function accepts `"image"` too, so implement both:
- `format=pdf` → a one-page receipt-styled PDF (mirrors the PHP backend's `_image_to_pdf` / single-receipt PDF)
- `format=image` → the original uploaded receipt photo (jpg/png), served as-is

**Response — `200 OK`**, `Content-Type: application/pdf` or `image/jpeg`, raw file body

**Error cases**
- `404` — receipt not found, or caller isn't entitled to it (not on an account they manage)

---

## 5. Admin

### 5.1 `GET /api/admin/stats`

Powers the Admin tab. **Admin-role only** — return `403` for any non-admin caller (the frontend also hides the tab client-side via `RequireAdmin`, but that's not a substitute for a server-side check).

**Auth required:** yes, `role: "admin"`

**Response — `200 OK`**
```json
{
  "pinsRegistered": 184,
  "pinsRegisteredDelta": 9,
  "activeOwners": 151,
  "accountManagers": 63,
  "receiptsPlatformWide": 2946,
  "recentPins": [
    ["P051987654L", "Individual", "254712●●●341", "25 Jul 2026"],
    ["A008812234C", "Company", "254798●●●210", "24 Jul 2026"]
  ],
  "monthlyRegistrations": {
    "counts": [11, 14, 9, 18, 22, 9],
    "labels": ["Feb", "Mar", "Apr", "May", "Jun", "Jul"]
  }
}
```

**Field notes**

| Field | Type | Notes |
|---|---|---|
| `pinsRegistered` | number | Total PINs registered, platform-wide |
| `pinsRegisteredDelta` | number | Change over the last 30 days (shown as `"+{n} in the last 30 days"`) |
| `activeOwners` | number | Users who own ≥1 account |
| `accountManagers` | number | Users delegated on ≥1 account |
| `receiptsPlatformWide` | number | All-time receipt count, all accounts |
| `recentPins` | `[string, string, string, string][]` | **Array of 4-element arrays**, not objects: `[kra_pin, type, maskedOwnerPhone, registeredDateDisplay]`. The last element is a pre-formatted display date (`"25 Jul 2026"`), not ISO — see §1. |
| `monthlyRegistrations.counts` | number[] | 6 values, oldest → newest |
| `monthlyRegistrations.labels` | string[] | 6 short month labels matching `counts`, e.g. `"Feb"` |

> `recentPins` being an array-of-arrays instead of an array of `{kra_pin, type, ownerPhone, registeredAt}` objects is a quirk of the current frontend (`Admin.jsx` does `stats.recentPins.map(r => ...)` and indexes `r[0]`, `r[1]`, etc.). Worth cleaning up to a proper object shape if you're touching this endpoint anyway — flag it and update `Admin.jsx` alongside the backend change.

---

## 6. Endpoint summary

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `POST` | `/api/auth/otp/send` | none | Send WhatsApp OTP |
| `POST` | `/api/auth/otp/verify` | none | Verify OTP, log in |
| `POST` | `/api/auth/logout` | yes | Log out |
| `GET` | `/api/accounts` | yes | List my accounts |
| `GET` | `/api/accounts/lookup?pin=` | yes | Look up any PIN |
| `POST` | `/api/accounts` | yes | Add/verify a PIN, link as owner |
| `POST` | `/api/accounts/invites` | yes | Invite an Account Manager |
| `GET` | `/api/receipts?pin=&filter=` | yes | Fetch receipts for a PIN |
| `GET` | `/api/receipts/export/csv?pin=&filter=` | yes | Bulk CSV export |
| `GET` | `/api/receipts/export/pdf?pin=&filter=` | yes | Bulk PDF export |
| `GET` | `/api/receipts/{id}/download?format=` | yes | Single receipt download |
| `GET` | `/api/admin/stats` | yes, admin | Platform-wide stats |

---

## 7. What to build first

If you want to bring this up incrementally rather than all at once, this order unblocks the most UI at each step:

1. **Auth** (§2) — nothing else works without a logged-in user.
2. **`GET /api/accounts`** (§3.1) — the Dashboard, Accounts, and Receipts tabs all read `user.accounts`, which login already returns, so this can come slightly later.
3. **`GET /api/receipts`** (§4.1) — unlocks Dashboard totals and the Receipts tab, the two most-used screens.
4. **CSV/PDF exports** (§4.2–4.4) — the frontend already has a genuine client-side fallback for these (Blob CSV + `jspdf`), so they're safe to leave for last without blocking a demo.
5. **PIN lookup + Add PIN + Invites** (§3.2–3.4) — the Accounts and Add Manager tabs.
6. **Admin stats** (§5.1) — only reachable by admin-role users.
