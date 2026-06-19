# Nexlink dApp API Reference

This document describes the API interface between the Nexlink platform and dApps. All requests and responses use JSON. All endpoints require HTTPS.

---

## Types

### InitData

A URL-encoded query string containing a cryptographically signed user identity. Delivered to dApps via the Nexlink SDK (`window.NexlinkApp.initData`) or the QR login flow.

**Format:** URL-encoded query string (not JSON)

**Example:**

```
user=%7B%22id%22%3A12345%2C%22uid%22%3A%22u_abc%22%2C%22nickname%22%3A%22John%22%2C%22avatar%22%3A%22https%3A%2F%2Fcdn.nexlink.app%2Favatar%2F12345.jpg%22%2C%22language_code%22%3A%22en%22%7D&user_id=12345&auth_date=1718700000&dapp_id=42&query_id=550e8400-e29b-41d4-a716-446655440000&hash=a1b2c3d4e5f6...
```

| Field | Type | Required | Description |
|---|---|---|---|
| `user` | String | Yes | JSON-encoded [User](#user) object |
| `user_id` | Integer | Yes | Nexlink user ID (same as `user.id`) |
| `auth_date` | Integer | Yes | Unix timestamp (seconds) when this payload was generated |
| `dapp_id` | Integer | Yes | Numeric dApp ID |
| `query_id` | String | Yes | Unique session identifier (UUID), used for replay prevention |
| `start_param` | String | No | Deep-link parameter from opening URL |
| `hash` | String | Yes | HMAC-SHA256 hex signature (64 characters) of all other fields |

---

### User

Represents a Nexlink user identity. Embedded as a JSON string inside [InitData](#initdata).

| Field | Type | Description |
|---|---|---|
| `id` | Integer | Unique Nexlink user ID |
| `uid` | String | Nexlink string user identifier |
| `nickname` | String | Display name |
| `avatar` | String | Profile picture URL |
| `language_code` | String | ISO 639-1 language code (e.g., `"en"`, `"zh"`) |

---

### QrSession

Represents a QR login session created by [POST /dapp/qr/create](#post-dappqrcreate).

| Field | Type | Description |
|---|---|---|
| `qrToken` | String | One-time session token (UUID) |
| `expiresAt` | Integer | Unix timestamp when this session expires (2-5 minutes from creation) |

---

### QrStatus

Response from [GET /dapp/qr/status](#get-dappqrstatus).

| Field | Type | Description |
|---|---|---|
| `status` | String | `"pending"`, `"confirmed"`, or `"expired"` |
| `initData` | String | _Optional._ Signed [InitData](#initdata) string. Present only when `status` is `"confirmed"`. |

---

### TokenResponse

OAuth2 token response returned by danbao login/bind/create endpoints.

| Field | Type | Description |
|---|---|---|
| `status` | String | `"ok"` |
| `access_token` | String | Bearer access token (2h TTL) |
| `refresh_token` | String | Refresh token (7d TTL) |
| `token_type` | String | Always `"Bearer"` |
| `expires_in` | Integer | Access token lifetime in seconds |
| `user` | Object | User profile object |

---

### UnboundResponse

Returned by [POST /user/login-nexlink](#post-userlogin-nexlink) when the Nexlink user has no linked danbao account.

| Field | Type | Description |
|---|---|---|
| `status` | String | `"unbound"` |
| `nexlinkUser` | Object | `{ id, nickname, avatar }` — display info for the authorization popup |

---

### PaymentOrder

Represents a payment order created by [POST /dapp/order/create](#post-dappordercreate).

| Field | Type | Description |
|---|---|---|
| `orderId` | String | Nexlink-assigned order UUID |
| `merchantOrderId` | String | dApp-assigned order identifier |
| `amount` | Integer | Payment amount in smallest unit (5 decimals: `100000` = `1.00`) |
| `symbol` | String | Token symbol: `"USDK"` or `"CNYT"` |
| `status` | Integer | `1` = pending, `2` = paid, `3` = cancelled, `4` = expired |
| `txHash` | String | _Optional._ On-chain transaction hash. Present when `status` is `2`. |
| `callbackUrl` | String | Webhook delivery URL |
| `expireAt` | Integer | Unix timestamp when this order expires |
| `paidAt` | Integer | _Optional._ Unix timestamp when payment was confirmed |

---

### PaymentSession

Represents a QR payment session created by [POST /dapp/payment/create](#post-dapppaymentcreate). Used for browser-based payments via QR code.

| Field | Type | Description |
|---|---|---|
| `payToken` | String | One-time payment session token (UUID) |
| `expiresAt` | Integer | Unix timestamp when this session expires (5 minutes from creation) |

---

### PaymentStatus

Response from [GET /dapp/payment/status](#get-dapppaymentstatus).

| Field | Type | Description |
|---|---|---|
| `status` | String | `"pending"`, `"paid"`, or `"expired"` |
| `orderId` | String | _Optional._ Order UUID. Present when `status` is `"paid"`. |
| `txHash` | String | _Optional._ Transaction hash. Present when `status` is `"paid"`. |

---

### PaymentResult

Return value from `NexlinkApp.payment.pay()` (in-app JS SDK).

| Field | Type | Description |
|---|---|---|
| `status` | String | `"paid"`, `"cancelled"`, or `"failed"` |
| `txHash` | String | _Optional._ Transaction hash. Present when `status` is `"paid"`. |
| `orderId` | String | The order UUID |
| `error` | String | _Optional._ Error message. Present when `status` is `"failed"`. |

---

### TransferResult

Return value from `NexlinkApp.payment.transfer()` (in-app JS SDK, direct transfer mode).

| Field | Type | Description |
|---|---|---|
| `status` | String | `"sent"`, `"cancelled"`, or `"failed"` |
| `txHash` | String | _Optional._ Transaction hash. Present when `status` is `"sent"`. |
| `error` | String | _Optional._ Error message. Present when `status` is `"failed"`. |

---

### WebhookPayload

Payload delivered to the dApp's `callbackUrl` when an order is paid.

| Field | Type | Description |
|---|---|---|
| `orderId` | String | Nexlink-assigned order UUID |
| `merchantOrderId` | String | dApp-assigned order identifier |
| `status` | Integer | Always `2` (paid) |
| `amount` | Integer | Payment amount in smallest unit |
| `symbol` | String | Token symbol (`"USDK"` or `"CNYT"`) |
| `txHash` | String | On-chain transaction hash |
| `paidAt` | Integer | Unix timestamp when payment was confirmed |

**Headers:**

| Header | Description |
|---|---|
| `X-Nexlink-Timestamp` | Unix timestamp of webhook delivery |
| `X-Nexlink-Signature` | HMAC-SHA256 hex signature for verification |

> For webhook signature verification, see [PAYMENT.md Section 6](PAYMENT.md#6-webhook-callbacks).

---

### ContractCallResult

Return value from `NexlinkApp.contract.call()` (in-app JS SDK).

| Field | Type | Description |
|---|---|---|
| `status` | String | `"sent"`, `"cancelled"`, or `"failed"` |
| `txHash` | String | _Optional._ Transaction hash. Present when `status` is `"sent"`. |
| `error` | String | _Optional._ Error message. Present when `status` is `"failed"`. |

---

### ContractSession

Represents a QR contract call session created by [POST /dapp/contract/create](#post-dappcontractcreate). Used for browser-based contract interaction via QR code.

| Field | Type | Description |
|---|---|---|
| `sessionToken` | String | One-time contract session token (UUID) |
| `expiresAt` | Integer | Unix timestamp when this session expires (5 minutes from creation) |

---

### ContractStatus

Response from [GET /dapp/contract/status](#get-dappcontractstatus).

| Field | Type | Description |
|---|---|---|
| `status` | String | `"pending"`, `"sent"`, or `"expired"` |
| `txHash` | String | _Optional._ Transaction hash. Present when `status` is `"sent"`. |

---

## Signature Verification

dApp backends must verify the `hash` field in [InitData](#initdata) before trusting the payload.

### Algorithm

```
Step 1:  secret_key    = HMAC-SHA256(key = "NexlinkData", message = <your dapp_secret_key>)
Step 2:  check_string  = sort all fields except "hash" alphabetically
                         → join as "key1=value1\nkey2=value2\n..."
Step 3:  computed_hash = HEX( HMAC-SHA256(key = secret_key, message = check_string) )
Step 4:  compare computed_hash with received hash using constant-time comparison
```

### Validation Rules

| Rule | Detail |
|---|---|
| Signature | `computed_hash` must match `hash` (use `hmac.Equal` in Go, `timingSafeEqual` in Node.js) |
| Expiry | `now() - auth_date` must be less than max age (default: 86400 seconds / 24 hours) |
| Replay | `query_id` should be tracked to prevent reuse (recommended: Redis `SET NX` with 24h TTL) |

> For full verification code examples (Go and Node.js), see [AUTH.md Section 4](AUTH.md#4-initdata-signature-verification).

---

## Nexlink Platform API

These endpoints are hosted on the **Nexlink backend**. dApp backends call them to support QR login and remote verification.

**Base URL:** `https://nexlink-api` (replace with your actual Nexlink API domain)

---

### POST /browser/init\_data

Generates a signed [InitData](#initdata) payload for a user opening a dApp inside the Nexlink mobile app. This is an internal endpoint called by the Nexlink app, not by dApp backends directly.

**Authorization:** Bearer token (Nexlink mobile app internal)

| Parameter | Type | Required | Description |
|---|---|---|---|
| `dappId` | Integer | Yes | dApp numeric ID |

**Returns:** [InitData](#initdata) string inside response envelope.

```json
{
  "errCode": 0,
  "data": {
    "initData": "user=%7B...%7D&auth_date=...&hash=..."
  }
}
```

**Errors:**

| errCode | Description |
|---|---|
| 1001 | dApp has no `secret_key` configured — initData cannot be generated |

---

### POST /dapp/qr/create

Creates a new QR login session. The dApp backend calls this when a user visits the dApp in a general browser. Returns a [QrSession](#qrsession) on success.

**Authorization:** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `clientId` | Integer | Yes | Your dApp's numeric ID |

**Returns:** [QrSession](#qrsession)

```json
{
  "qrToken": "550e8400-e29b-41d4-a716-446655440000",
  "expiresAt": 1718700300
}
```

The dApp backend encodes the token into a deep link for the QR code:

```
nexlink://auth/qr?token=<qrToken>&dapp=<clientId>
```

> The QR code contains **no callback URL** — only the token and dApp ID.

**Errors:**

| HTTP Status | Description |
|---|---|
| 401 | Invalid or missing API key |
| 404 | dApp not found for given `clientId` |

---

### POST /dapp/qr/confirm

Called by the **Nexlink mobile app** after the user scans the QR code and confirms login. The backend generates signed [InitData](#initdata) and stores it with the `qrToken` for retrieval via [GET /dapp/qr/status](#get-dappqrstatus).

**Authorization:** `Bearer <user's nexlinkToken>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `qrToken` | String | Yes | Token from the scanned QR code |
| `dappId` | Integer | Yes | dApp numeric ID from the QR deep link |

**Returns:**

```json
{ "errCode": 0 }
```

**Errors:**

| HTTP Status | Description |
|---|---|
| 400 | Token expired or invalid |
| 401 | User not authenticated |

**Side effect:** Generates signed [InitData](#initdata) using the dApp's `secret_key` and stores it with `qrToken`.

---

### GET /dapp/qr/status

Polls for QR login result. Supports **long polling** — the server holds the connection for up to 25 seconds, returning immediately when the status changes. Returns a [QrStatus](#qrstatus) object.

**Authorization:** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `token` | String | Yes | QR session token (query parameter) |
| `clientId` | Integer | Yes | dApp numeric ID (query parameter) |

**Returns:** [QrStatus](#qrstatus)

Pending (server held ~25s, then returned):

```json
{ "status": "pending" }
```

Confirmed (includes signed initData):

```json
{
  "status": "confirmed",
  "initData": "user=%7B...%7D&auth_date=...&hash=..."
}
```

Expired:

```json
{ "status": "expired" }
```

| Status | Meaning | dApp Action |
|---|---|---|
| `pending` | User has not scanned yet | Reconnect immediately |
| `confirmed` | User confirmed — [InitData](#initdata) included | Verify initData, create session |
| `expired` | Token TTL exceeded | Show "QR expired", offer refresh |

> **One-time read:** After returning `confirmed`, the server deletes the stored initData. A second request returns `expired`.

---

### POST /dapp/verify

Remote [InitData](#initdata) signature verification. For dApps that prefer not to store the `secret_key` locally. The server looks up the secret by `clientId` and verifies server-side.

**Authorization:** None

| Parameter | Type | Required | Description |
|---|---|---|---|
| `initData` | String | Yes | Full [InitData](#initdata) query string |
| `clientId` | Integer | Yes | Your dApp's numeric ID |

**Returns:**

```json
{
  "errCode": 0,
  "data": {
    "valid": true,
    "user": {
      "id": 12345,
      "uid": "u_abc",
      "nickname": "John",
      "avatar": "https://cdn.nexlink.app/avatar/12345.jpg"
    }
  }
}
```

**Errors:**

| errCode | Description |
|---|---|
| 40002 | Bad signature — initData is invalid or tampered |

---

## Payment API

These endpoints handle payment order management and QR payment sessions. For architecture details, see [Payment Integration](PAYMENT.md).

**Base URL:** `https://nexlink-api` (same as Nexlink Platform API)

---

### POST /dapp/order/create

Creates a new payment order. The dApp backend calls this before initiating payment. Returns a [PaymentOrder](#paymentorder) on success.

**Authorization:** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `merchantOrderId` | String | Yes | dApp-assigned order identifier (unique per dApp) |
| `amount` | Integer | Yes | Amount in smallest unit (5 decimals: `100000` = `1.00`) |
| `symbol` | String | Yes | Token symbol: `"USDK"` or `"CNYT"` |
| `to` | String | Yes | Recipient wallet address (hex, checksummed) |
| `callbackUrl` | String | No | Webhook URL for payment notification |
| `expireSeconds` | Integer | No | Order TTL in seconds (default: 3600) |

**Returns:** [PaymentOrder](#paymentorder)

```json
{
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "merchantOrderId": "shop-001",
  "amount": 10000000,
  "symbol": "USDK",
  "status": 1,
  "expireAt": 1718703600
}
```

> **Idempotent:** Creating an order with an existing `merchantOrderId` returns the existing order.

**Errors:**

| HTTP Status | Description |
|---|---|
| 401 | Invalid or missing API key |
| 400 | Invalid amount, symbol, or address |

---

### POST /dapp/order/query

Queries the current status of a payment order.

**Authorization:** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | Yes | Nexlink order UUID |

**Returns:** [PaymentOrder](#paymentorder)

```json
{
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "merchantOrderId": "shop-001",
  "amount": 10000000,
  "symbol": "USDK",
  "status": 2,
  "txHash": "0xabc123...",
  "paidAt": 1718700100
}
```

**Errors:**

| HTTP Status | Description |
|---|---|
| 401 | Invalid or missing API key |
| 404 | Order not found |

---

### POST /dapp/order/cancel

Cancels a pending payment order. Only orders with status `1` (pending) can be cancelled.

**Authorization:** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | Yes | Nexlink order UUID |

**Returns:**

```json
{ "errCode": 0 }
```

**Errors:**

| HTTP Status | Description |
|---|---|
| 401 | Invalid or missing API key |
| 400 | Order is not in pending status |
| 404 | Order not found |

---

### POST /dapp/order/pay

Marks an order as paid. Called by the **NexLink mobile app** after the user confirms an in-app payment and the on-chain transaction is broadcast.

**Authorization:** `Bearer <user's nexlinkToken>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | Yes | Nexlink order UUID |
| `txHash` | String | Yes | On-chain transaction hash |

**Returns:**

```json
{ "errCode": 0 }
```

**Errors:**

| HTTP Status | Description |
|---|---|
| 400 | Order expired or already paid |
| 401 | User not authenticated |
| 404 | Order not found |

**Side effects:**
- Transitions order to `paid` (status `2`)
- Records `txHash` and `paidAt`
- Enqueues webhook delivery to `callbackUrl`

---

### POST /dapp/payment/create

Creates a QR payment session for browser-based payments. Linked to an existing order. Returns a [PaymentSession](#paymentsession) on success.

**Authorization:** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | Yes | Nexlink order UUID (must be pending) |
| `clientId` | Integer | Yes | dApp numeric ID |

**Returns:** [PaymentSession](#paymentsession)

```json
{
  "payToken": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "expiresAt": 1718700300
}
```

The dApp backend encodes the token into a deep link for the QR code:

```
nexlink://pay?token=<payToken>&dapp=<clientId>
```

> The QR code contains **no amount, no address, no callback URL** — only the token and dApp ID.

**Errors:**

| HTTP Status | Description |
|---|---|
| 401 | Invalid or missing API key |
| 400 | Order is not in pending status |
| 404 | Order not found |

---

### GET /dapp/payment/info

Fetches payment details for a scanned QR code. Called by the **NexLink mobile app** after the user scans a payment QR.

**Authorization:** `Bearer <user's nexlinkToken>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `token` | String | Yes | Payment session token (query parameter) |

**Returns:**

```json
{
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 10000000,
  "symbol": "USDK",
  "to": "0x1234...abcd",
  "dappName": "MyShop",
  "dappIcon": "https://cdn.nexlink.app/dapp/myshop.png",
  "label": "Order #shop-001",
  "expiresAt": 1718700300
}
```

**Errors:**

| HTTP Status | Description |
|---|---|
| 400 | Token expired or invalid |
| 401 | User not authenticated |
| 404 | Payment session not found |

---

### POST /dapp/payment/confirm

Called by the **NexLink mobile app** after the user confirms a QR payment and the on-chain transaction is broadcast. The backend marks the linked order as paid and stores the result for polling via [GET /dapp/payment/status](#get-dapppaymentstatus).

**Authorization:** `Bearer <user's nexlinkToken>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `payToken` | String | Yes | Payment session token |
| `txHash` | String | Yes | On-chain transaction hash |

**Returns:**

```json
{ "errCode": 0 }
```

**Errors:**

| HTTP Status | Description |
|---|---|
| 400 | Token expired or already used |
| 401 | User not authenticated |

**Side effects:**
- Transitions linked order to `paid` (status `2`)
- Records `txHash` and `paidAt`
- Stores result with `payToken` for status polling
- Enqueues webhook delivery to `callbackUrl`

---

### GET /dapp/payment/status

Polls for QR payment result. Supports **long polling** — the server holds the connection for up to 25 seconds, returning immediately when the status changes. Returns a [PaymentStatus](#paymentstatus) object.

**Authorization:** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `token` | String | Yes | Payment session token (query parameter) |
| `clientId` | Integer | Yes | dApp numeric ID (query parameter) |

**Returns:** [PaymentStatus](#paymentstatus)

Pending (server held ~25s, then returned):

```json
{ "status": "pending" }
```

Paid (includes order and transaction details):

```json
{
  "status": "paid",
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "txHash": "0xabc123..."
}
```

Expired:

```json
{ "status": "expired" }
```

| Status | Meaning | dApp Action |
|---|---|---|
| `pending` | User has not scanned or confirmed yet | Reconnect immediately |
| `paid` | User confirmed — transaction complete | Update UI, fulfill order |
| `expired` | Session TTL exceeded | Show "QR expired", offer refresh |

> **One-time read:** After returning `paid`, the server deletes the stored result. A second request returns `expired`.

---

## Contract API

These endpoints handle QR-based contract call sessions for external browsers. For architecture details, see [Contract Interaction](CONTRACT.md).

**Base URL:** `https://nexlink-api` (same as Nexlink Platform API)

---

### POST /dapp/contract/create

Creates a new contract call session for QR-based interaction. The dApp backend calls this when a user needs to sign a contract call from an external browser. Returns a [ContractSession](#contractsession) on success.

**Authorization:** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `contract` | String | Yes | Target contract address (hex, checksummed) |
| `data` | String | Yes | ABI-encoded calldata (hex, `0x`-prefixed) |
| `value` | String | No | Native token value in wei (default `"0"`) |
| `methodName` | String | No | Human-readable method name for display (e.g., `"freeze"`) |
| `clientId` | Integer | Yes | dApp numeric ID |

**Returns:** [ContractSession](#contractsession)

```json
{
  "sessionToken": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "expiresAt": 1718700300
}
```

The dApp backend encodes the token into a deep link for the QR code:

```
nexlink://contract?token=<sessionToken>&dapp=<clientId>
```

> The QR code contains **no contract address, no calldata, no value** — only the token and dApp ID.

**Errors:**

| HTTP Status | Description |
|---|---|
| 401 | Invalid or missing API key |
| 400 | Invalid contract address or calldata |

---

### GET /dapp/contract/info

Fetches contract call details for a scanned QR code. Called by the **NexLink mobile app** after the user scans a contract call QR.

**Authorization:** `Bearer <user's nexlinkToken>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `token` | String | Yes | Contract session token (query parameter) |

**Returns:**

```json
{
  "contract": "0x3d8b4425...",
  "data": "0x57e871e7...",
  "value": "0",
  "methodName": "freeze",
  "dappName": "Danbao",
  "dappIcon": "https://cdn.nexlink.app/dapp/danbao.png",
  "expiresAt": 1718700300
}
```

**Errors:**

| HTTP Status | Description |
|---|---|
| 400 | Token expired or invalid |
| 401 | User not authenticated |
| 404 | Contract session not found |

---

### POST /dapp/contract/confirm

Called by the **NexLink mobile app** after the user confirms a QR contract call and the on-chain transaction is broadcast. The backend stores the result for polling via [GET /dapp/contract/status](#get-dappcontractstatus).

**Authorization:** `Bearer <user's nexlinkToken>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `sessionToken` | String | Yes | Contract session token |
| `txHash` | String | Yes | On-chain transaction hash |

**Returns:**

```json
{ "errCode": 0 }
```

**Errors:**

| HTTP Status | Description |
|---|---|
| 400 | Token expired or already used |
| 401 | User not authenticated |

**Side effects:**
- Stores `txHash` with session for status polling
- Transitions session to `sent` status

---

### GET /dapp/contract/status

Polls for QR contract call result. Supports **long polling** — the server holds the connection for up to 25 seconds, returning immediately when the status changes. Returns a [ContractStatus](#contractstatus) object.

**Authorization:** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `token` | String | Yes | Contract session token (query parameter) |
| `clientId` | Integer | Yes | dApp numeric ID (query parameter) |

**Returns:** [ContractStatus](#contractstatus)

Pending (server held ~25s, then returned):

```json
{ "status": "pending" }
```

Sent (includes transaction hash):

```json
{
  "status": "sent",
  "txHash": "0xabc123..."
}
```

Expired:

```json
{ "status": "expired" }
```

| Status | Meaning | dApp Action |
|---|---|---|
| `pending` | User has not scanned or confirmed yet | Reconnect immediately |
| `sent` | User confirmed — transaction broadcast | Update UI, process result |
| `expired` | Session TTL exceeded | Show "QR expired", offer refresh |

> **One-time read:** After returning `sent`, the server deletes the stored result. A second request returns `expired`.

---

## JS SDK — Payment Methods

These methods are available on `window.NexlinkApp.payment` inside the NexLink dApp browser. They are **not available** in external browsers.

For detection:

```javascript
if (window.NexlinkApp && NexlinkApp.payment) {
  // In-app: payment methods available
} else {
  // External browser: use QR payment flow instead
}
```

---

### NexlinkApp.payment.pay

Triggers payment for a pre-created order. Shows a native confirmation UI. Returns a [PaymentResult](#paymentresult).

```javascript
const result = await NexlinkApp.payment.pay({
  orderId: "550e8400-e29b-41d4-a716-446655440000"
});
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | Yes | Order UUID from [POST /dapp/order/create](#post-dappordercreate) |

**Returns:** [PaymentResult](#paymentresult)

---

### NexlinkApp.payment.transfer

Requests a direct token transfer without a backend order. Shows a native confirmation UI. Returns a [TransferResult](#transferresult). **In-app only.**

```javascript
const result = await NexlinkApp.payment.transfer({
  to: "0x1234...abcd",
  amount: "100.00",
  token: "USDK"
});
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `to` | String | Yes | Recipient wallet address (hex, checksummed) |
| `amount` | String | Yes | Human-readable amount (e.g., `"100.00"`) |
| `token` | String | Yes | Token symbol: `"USDK"` or `"CNYT"` |

**Returns:** [TransferResult](#transferresult)

---

### NexlinkApp.payment.getOrderStatus

Queries an order's current status from the NexLink app without a backend round-trip.

```javascript
const status = await NexlinkApp.payment.getOrderStatus({
  orderId: "550e8400-e29b-41d4-a716-446655440000"
});
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | Yes | Order UUID |

**Returns:**

| Field | Type | Description |
|---|---|---|
| `status` | String | `"pending"`, `"paid"`, `"cancelled"`, or `"expired"` |
| `txHash` | String | _Optional._ Transaction hash. Present when `status` is `"paid"`. |

---

## JS SDK — Contract Methods

These methods are available on `window.NexlinkApp.contract` inside the NexLink dApp browser. They are **not available** in external browsers — use the [QR contract flow](#contract-api) instead.

For detection:

```javascript
if (window.NexlinkApp && NexlinkApp.contract) {
  // In-app: contract SDK available
} else if (window.ethereum) {
  // EIP-1193 fallback (Layer 1)
} else {
  // External browser: use QR contract flow
}
```

---

### NexlinkApp.contract.call

Sends a state-changing transaction to a smart contract. The SDK encodes calldata from the ABI, and the NexLink app shows a decoded confirmation UI. Returns a [ContractCallResult](#contractcallresult).

```javascript
const result = await NexlinkApp.contract.call({
  contract: "0x3d8b4425...",
  abi: ESCROW_ABI,
  method: "freeze",
  args: [orderId, 10000000, USDK_ADDRESS],
  value: "0"
});
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `contract` | String | Yes | Contract address (hex, checksummed) |
| `abi` | Array | Yes | ABI array (standard Solidity ABI JSON or human-readable) |
| `method` | String | Yes | Function name (e.g., `"freeze"`) |
| `args` | Array | Yes | Function arguments in order |
| `value` | String | No | Native token value in wei (default `"0"`) |

**Returns:** [ContractCallResult](#contractcallresult)

---

### NexlinkApp.contract.read

Calls a `view` or `pure` function on a smart contract. No signing required, no confirmation UI, no gas cost. Returns the decoded return value.

```javascript
const status = await NexlinkApp.contract.read({
  contract: "0x3d8b4425...",
  abi: ESCROW_ABI,
  method: "getOrderStatus",
  args: [orderId]
});
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `contract` | String | Yes | Contract address (hex, checksummed) |
| `abi` | Array | Yes | ABI array |
| `method` | String | Yes | Function name (must be `view` or `pure`) |
| `args` | Array | Yes | Function arguments in order |

**Returns:** Decoded value(s) according to the ABI output specification. Single return values are returned directly; multiple values are returned as an array.

---

### NexlinkApp.contract.encode

Encodes ABI calldata without sending a transaction. Useful for building calldata manually for use with `NexlinkApp.wallet.sendTransaction()` or for debugging.

```javascript
const calldata = NexlinkApp.contract.encode({
  abi: ESCROW_ABI,
  method: "freeze",
  args: [orderId, 10000000, USDK_ADDRESS]
});
// "0x57e871e7000000000000000000000000..."
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `abi` | Array | Yes | ABI array |
| `method` | String | Yes | Function name |
| `args` | Array | Yes | Function arguments in order |

**Returns:** `String` — hex-encoded calldata prefixed with `0x`.

---

## Danbao Backend API

These endpoints are implemented in the **danbao backend** (`danbao-api`). They handle Nexlink-based login, account binding, and account creation for the danbao application.

**Base URL:** `/api/v1`

> Other dApp backends can use these as a reference implementation. The pattern (verify initData → look up user → issue token) is the same for any dApp.

---

### POST /user/login-nexlink

Verifies [InitData](#initdata) and logs in. If the Nexlink user has a linked danbao account, returns a [TokenResponse](#tokenresponse). If not, returns an [UnboundResponse](#unboundresponse) so the frontend can show the authorization popup.

**Authorization:** None

| Parameter | Type | Required | Description |
|---|---|---|---|
| `initData` | String | Yes | Signed [InitData](#initdata) from Nexlink |

**Returns on success:** [TokenResponse](#tokenresponse)

```json
{
  "status": "ok",
  "access_token": "abc123...",
  "refresh_token": "def456...",
  "token_type": "Bearer",
  "expires_in": 7200,
  "user": { "id": 1, "username": "john_doe", "nexlink_user_id": 12345 }
}
```

**Returns when unbound:** [UnboundResponse](#unboundresponse)

```json
{
  "status": "unbound",
  "nexlinkUser": { "id": 12345, "nickname": "John", "avatar": "https://..." }
}
```

**Errors:**

| HTTP Status | Body | Description |
|---|---|---|
| 401 | `"unauthorized"` | InitData signature invalid |
| 403 | `"initData already used"` | Replay detected (`query_id` reuse) |

---

### POST /user/bind-nexlink

Links a Nexlink identity to an existing danbao account. The user must provide their danbao credentials to prove account ownership. Returns a [TokenResponse](#tokenresponse) on success.

**Authorization:** None

| Parameter | Type | Required | Description |
|---|---|---|---|
| `username` | String | Yes | Existing danbao username |
| `password` | String | Yes | Existing danbao password |
| `initData` | String | Yes | Signed [InitData](#initdata) from Nexlink |

**Returns:** [TokenResponse](#tokenresponse)

```json
{
  "status": "ok",
  "access_token": "abc123...",
  "refresh_token": "def456...",
  "token_type": "Bearer",
  "expires_in": 7200,
  "user": { "id": 1, "username": "john_doe", "nexlink_user_id": 12345 }
}
```

**Errors:**

| HTTP Status | Body | Description |
|---|---|---|
| 401 | `"unauthorized"` | InitData signature invalid |
| 401 | `"invalid credentials"` | Wrong username or password |
| 403 | `"initData already used"` | Replay detected |
| 409 | `"account already bound to a Nexlink user"` | This danbao account already has a Nexlink binding |
| 409 | `"nexlink user already bound to another account"` | This Nexlink user is linked to a different danbao account |
| 429 | `"too many attempts, try again later"` | Rate limit exceeded (5 attempts per 10 minutes) |

---

### POST /user/create-nexlink

Creates a new passwordless danbao account from a Nexlink identity. The username is auto-generated as `nx_{id}`. Returns a [TokenResponse](#tokenresponse) on success.

**Authorization:** None

| Parameter | Type | Required | Description |
|---|---|---|---|
| `initData` | String | Yes | Signed [InitData](#initdata) from Nexlink |

**Returns:** [TokenResponse](#tokenresponse)

```json
{
  "status": "ok",
  "access_token": "abc123...",
  "refresh_token": "def456...",
  "token_type": "Bearer",
  "expires_in": 7200,
  "user": { "id": 7, "username": "nx_7", "display_name": "John", "nexlink_user_id": 12345 }
}
```

**Errors:**

| HTTP Status | Body | Description |
|---|---|---|
| 401 | `"unauthorized"` | InitData signature invalid |
| 403 | `"initData already used"` | Replay detected |
| 429 | `"too many attempts"` | Rate limit exceeded (3 attempts per 10 minutes) |

> If the Nexlink user already has an account (concurrent request / double-click), the endpoint falls back to login instead of returning an error.

---

### POST /user/unbind-nexlink

Disconnects a Nexlink identity from a danbao account. Requires the user to be logged in and confirm their password. Revokes all existing sessions after unbinding.

**Authorization:** `Bearer <session_token>` (user must be logged in)

| Parameter | Type | Required | Description |
|---|---|---|---|
| `password` | String | Yes | Current account password for confirmation |

**Returns:**

```json
{ "status": "ok" }
```

**Errors:**

| HTTP Status | Body | Description |
|---|---|---|
| 400 | `"set a password before unbinding"` | Passwordless accounts cannot unbind (would lose all access) |
| 401 | `"invalid password"` | Wrong password |
| 404 | `"not found"` | User not found |

**Side effects:**
- Sets `nexlink_user_id = NULL` on the user record
- Revokes all existing sessions (all tokens invalidated immediately)

> After unbinding, the user can re-bind via the authorization popup on their next Nexlink login.
