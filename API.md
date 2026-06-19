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
