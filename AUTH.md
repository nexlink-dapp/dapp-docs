# Nexlink dApp Login & Registration

Users open a dApp inside the Nexlink native app (or scan a QR code from an external browser), and the platform provides a cryptographically signed identity payload — no username/password form required.

---

## 1. Authentication Architecture

### Two access contexts

| Context | How user arrives | Auth method |
|---|---|---|
| **dApp Browser** (in-app) | Opens dApp inside Nexlink mobile app | Automatic `initData` injection |
| **General Browser** (external) | Visits dApp URL in Chrome/Safari/etc. | QR code scan with Nexlink app |

### Key concepts

| Concept | Value |
|---|---|
| Global JS object | `window.NexlinkApp` |
| Signed payload | `initData` (URL-encoded string) |
| HMAC key label | `"NexlinkData"` |
| Secret source | `dapps.secret_key` (per-dApp) |
| Signature algo | HMAC-SHA256 |
| Remote verify | `POST /dapp/verify` |

### initData fields

| Field | Required | Description |
|---|---|---|
| `user` | yes | JSON-stringified user object (`id`, `uid`, `nickname`, `avatar`, `language_code`) |
| `user_id` | yes | Numeric Nexlink user ID |
| `auth_date` | yes | Unix seconds — server rejects if older than 24h |
| `dapp_id` | yes | Numeric ID from `dapps` table |
| `query_id` | yes | Unique per WebView session |
| `start_param` | no | Deep-link param from opening URL |
| `hash` | yes | HMAC-SHA256 hex signature of all other fields |

---

## 2. Login Flow: dApp Browser (In-App)

When a user opens a dApp inside the Nexlink app, login is **fully automatic** — zero user interaction required.

### Detailed injection pipeline

```
┌──────────────────────────────────────────────────────────────┐
│  Nexlink App                                                  │
│                                                               │
│  1. User taps dApp in browser/explorer                        │
│                                                               │
│  2. DappSdkInjector.kickOff() starts two parallel tasks:      │
│     a. Preload per-chain wallet addresses                     │
│     b. Fetch signed initData (cache or network)               │
│          │                                                    │
│          ▼                                                    │
│  3. Cache check (DappInitDataCache):                          │
│     - Fresh (<30min): serve immediately, no network           │
│     - Usable (30min-24h): serve stale, refresh in background  │
│     - Unconfigured (errCode 1001): skip, dApp has no secret   │
│     - Miss: call POST /browser/init_data (3s timeout cap)     │
│          │                                                    │
│  4. WebView navigation starts (parallel with step 3)          │
│          │                                                    │
│  5. onPageStarted fires → inject in order:                    │
│     a. Viewport compat shim                                   │
│     b. window.__NEXLINK_APP_VERSION__ = "<real version>"      │
│     c. window.__nexlink_addresses = { chainId: "0x..." }      │
│     d. Stub SDK (JsBridge.stubScript)                         │
│        - Installs window.NexlinkApp immediately               │
│        - Queues all method calls into __pending.calls          │
│        - Queues onReady callbacks into __pending.readyCbs      │
│        - Getter-backed initData reads from __NEXLINK_STATE__  │
│          │                                                    │
│  6. When initData resolves (from cache or network):           │
│     → Inject real SDK (JsBridge.buildScript)                  │
│     → Updates __NEXLINK_STATE__ with signed payload            │
│     → Replays all queued calls in order                       │
│     → Fires onReady callbacks                                 │
│          │                                                    │
│  7. dApp JS reads NexlinkApp.initData                         │
│     Sends to its own backend for verification                 │
│          │                                                    │
│  8. dApp backend verifies signature (Mode A or B)             │
│     Creates session → user is logged in                       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**If initData fetch times out (3s) or fails:** The real SDK is still injected with `initData = ""`. EIP-1193 wallet bridge (`window.ethereum`) still works. The dApp operates as a plain web page without signed identity.

### Relevant code locations

| Step | File |
|---|---|
| init_data API call | `nexlink/lib/pages/dapp/dapp_apis.dart` → `DappApis.initData()` |
| init_data cache | `nexlink/lib/pages/dapp/dapp_browser/dapp_init_data_cache.dart` |
| Dart-side HMAC signer | `nexlink/lib/pages/dapp/dapp_browser/init_data.dart` → `InitDataSigner` |
| Go-side HMAC signer | `nexlink-api/internal/api/nexlinkdapp/handler_verify.go` |
| Go-side initData generator | `nexlink-api/internal/api/nexlinkdapp/handler_browser.go` |
| SDK injector pipeline | `nexlink/lib/pages/dapp/dapp_browser/dapp_sdk_injector.dart` |
| JS SDK (stub + real) | `nexlink/lib/pages/dapp/dapp_browser/js_bridge.dart` |
| Bridge modules | `nexlink/lib/pages/dapp/dapp_browser/bridge/modules/` |
| Browser page | `nexlink/lib/pages/dapp/dapp_browser/dapp_browser_logic.dart` |

### dApp frontend code (in-app)

```html
<script src="https://nexlink-api/static/nexlink-sdk.js"></script>
<script>
  // Wait for SDK to be ready (initData may still be loading)
  NexlinkApp.onReady(async function () {
    const initData = NexlinkApp.initData;
    if (!initData) {
      // Not running inside Nexlink app — show QR login instead
      showQrLogin();
      return;
    }

    // Send signed payload to your backend
    const res = await fetch('/api/auth/nexlink-login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ initData }),
    });
    const { token } = await res.json();
    // Store session token, proceed to app
    localStorage.setItem('session', token);
    window.location.href = '/dashboard';
  });
</script>
```

**Caveat:** Code that captures `initData` as a primitive at module load (`const id = NexlinkApp.initData`) before the real SDK arrives will see an empty string. Always use `NexlinkApp.onReady(cb)` to wait for the signed payload.

---

## 3. Login Flow: General Browser (QR Code Scan)

When a user visits a dApp URL in a regular browser (not inside the Nexlink app), there is no `window.NexlinkApp` object. The dApp must fall back to a **QR code login flow**.

```
┌─────────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│  General Browser     │    │  Nexlink Backend  │    │  Nexlink Mobile App  │
│  (Chrome/Safari)     │    │                  │    │                     │
│                      │    │                  │    │                     │
│  1. dApp backend     │    │                  │    │                     │
│     creates QR       │    │                  │    │                     │
│     session via      │    │                  │    │                     │
│     Nexlink API ─────┼───►│ POST /dapp/qr/   │    │                     │
│                      │    │ create           │    │                     │
│  2. Show QR code     │    │                  │    │                     │
│         ┌────┐       │    │                  │    │                     │
│         │ QR │ ◄─────┼────┼──────────────────┼────┼── 3. User scans QR  │
│         └────┘       │    │                  │    │                     │
│                      │    │                  │    │  4. App confirms     │
│                      │    │  5. Backend signs │◄───┼── POST /dapp/qr/    │
│                      │    │     initData,     │    │    confirm          │
│                      │    │     stores with   │    │    { token, dappId }│
│  6. dApp backend     │    │     qrToken      │    │                     │
│     polls for  ──────┼───►│                  │    │                     │
│     result           │    │  GET /dapp/qr/   │    │                     │
│                      │    │  status          │    │                     │
│  7. Receives         │◄───┼── { initData }   │    │                     │
│     initData,        │    │                  │    │                     │
│     verifies,        │    │                  │    │                     │
│     creates session  │    │                  │    │                     │
│                      │    │                  │    │                     │
└─────────────────────┘    └──────────────────┘    └─────────────────────┘
```

**The Nexlink app never touches any external URL.** It only talks to the Nexlink backend. The QR code contains no callback URL — only a token and dApp ID. This eliminates the `callback=https://evil.com` attack vector entirely.

### 3.1. QR Code Login Protocol (To Be Implemented)

> **Status:** This entire QR login flow is a design proposal. No code exists yet in the Nexlink app or backend. The existing QR scanner in the app (`home_logic.dart` → `MobileScannerController`) can decode QR images but has no deep link handler for `nexlink://auth/qr`. The existing QR code display in the dApp browser (`dapp_browser_dialogs.dart` → `showQrCode`) only shows the dApp's domain URL, not an auth token.

#### Step 1: dApp backend creates QR session via Nexlink API

The dApp backend (not the browser) requests a QR login session from the **Nexlink backend**:

```
POST https://nexlink-api/dapp/qr/create
Authorization: Bearer <dapp_api_key>
{ "clientId": 42 }

→ { "qrToken": "abc123...", "expiresAt": 1718700000 }
```

- `qrToken` is a one-time random token (UUID), valid for 2-5 minutes.
- Stored in the Nexlink backend's database, scoped to `clientId` (dApp ID).
- The dApp backend forwards the token to the browser.

#### Step 2: Browser displays QR code and polls dApp backend

```js
// Detect environment
if (window.NexlinkApp && NexlinkApp.initData) {
  // In-app: use initData directly
  loginWithInitData(NexlinkApp.initData);
} else {
  // External browser: show QR code
  showQrLogin();
}

async function showQrLogin() {
  // dApp backend proxied the /dapp/qr/create call
  const { qrToken, expiresAt } = await fetch('/api/auth/qr/create', {
    method: 'POST',
  }).then(r => r.json());

  // QR code contains ONLY token + dApp ID — no callback URL
  renderQrCode(`nexlink://auth/qr?token=${qrToken}&dapp=42`);

  // Poll dApp backend (which polls Nexlink backend)
  pollQrStatus(qrToken);
}

async function pollQrStatus(qrToken) {
  // Long polling: each request blocks for up to ~25s on the server.
  // When the server responds, immediately reconnect for the next cycle.
  while (true) {
    try {
      const res = await fetch(`/api/auth/qr/status?token=${qrToken}`);
      const data = await res.json();

      if (data.status === 'confirmed') {
        localStorage.setItem('session', data.sessionToken);
        window.location.href = '/dashboard';
        return;
      }
      if (data.status === 'expired') {
        showExpiredMessage();
        return;
      }
      // status === 'pending' → server held ~25s, loop immediately
    } catch (e) {
      // Network error — wait 2s then retry
      await new Promise(r => setTimeout(r, 2000));
    }
  }
}
```

#### Step 3: User scans with Nexlink app

The Nexlink mobile app:

1. Opens the built-in QR scanner or camera.
2. Parses the deep link: `nexlink://auth/qr?token=abc123&dapp=42`.
3. Fetches dApp info from the Nexlink backend (name, icon) for the confirmation dialog.
4. Shows a confirmation dialog: _"Log in to [dApp Name] as [Your Name]?"_
5. On confirm, the app calls the **Nexlink backend** (not any external URL):

```
POST https://nexlink-api/dapp/qr/confirm
Authorization: Bearer <user's nexlinkToken>
{
  "qrToken": "abc123",
  "dappId": 42
}
```

6. The Nexlink backend generates signed `initData` (using the dApp's `secret_key` from the `dapps` table) and stores it alongside the `qrToken`.

#### Step 4: dApp backend polls for result

The dApp backend polls the Nexlink backend for the confirmed initData:

```
GET https://nexlink-api/dapp/qr/status?token=abc123&clientId=42
Authorization: Bearer <dapp_api_key>

→ pending:   { "status": "pending" }
→ confirmed: { "status": "confirmed", "initData": "user=%7B...%7D&hash=..." }
→ expired:   { "status": "expired" }
```

On `confirmed`, the dApp backend:

1. **Verifies initData** signature (Mode A with local secret, or trust it since it came from Nexlink API over authenticated channel).
2. **Finds or creates user** (`FindOrCreateByNexlinkID`).
3. **Issues its own session token** (OAuth2 token, JWT, etc.).
4. Returns the session token to the browser (via the browser's poll to `/api/auth/qr/status`).

The Nexlink backend's `GET /dapp/qr/status` endpoint supports **long polling**: it holds the connection for up to 25 seconds, returning immediately only when the status changes (confirmed/expired) or the hold times out (returns pending). This means the dApp backend's proxy handler is a simple pass-through — no polling loop needed on the dApp side.

```js
// dApp backend (Node.js example) — proxies Nexlink long-poll
app.get('/api/auth/qr/status', async (req, res) => {
  // Single request to Nexlink backend — it holds for up to 25s
  const nexlink = await fetch(
    `${NEXLINK_API}/dapp/qr/status?token=${req.query.token}&clientId=${MY_CLIENT_ID}`,
    { headers: { 'Authorization': `Bearer ${DAPP_API_KEY}` } }
  ).then(r => r.json());

  if (nexlink.status === 'confirmed') {
    // Verify and create session
    const result = verifyNexlinkInitData(nexlink.initData, process.env.NEXLINK_DAPP_SECRET);
    if (!result.valid) return res.status(401).end();

    const user = await db.upsertUser({ nexlinkId: result.user.id });
    const sessionToken = createJWT({ sub: user.id });

    return res.json({ status: 'confirmed', sessionToken });
  }

  // 'pending' (hold timed out) or 'expired' — browser will reconnect
  res.json({ status: nexlink.status });
});
```

**Nexlink backend long-poll implementation (Go):**

```go
// GET /dapp/qr/status?token=abc123&clientId=42
func (h *Handler) QrStatus(w http.ResponseWriter, r *http.Request) {
    token := r.URL.Query().Get("token")
    clientID := r.URL.Query().Get("clientId")

    // Validate dApp API key + clientId ownership (omitted for brevity)

    deadline := time.Now().Add(25 * time.Second)
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        session, err := h.qrStore.Get(r.Context(), token)
        if err != nil || session.ClientID != clientID {
            json.NewEncoder(w).Encode(map[string]string{"status": "expired"})
            return
        }

        if session.Status != "pending" {
            resp := map[string]any{"status": session.Status}
            if session.Status == "confirmed" {
                resp["initData"] = session.InitData
                // One-time read: delete after delivery
                _ = h.qrStore.Delete(r.Context(), token)
            }
            json.NewEncoder(w).Encode(resp)
            return
        }

        if time.Now().After(deadline) {
            // Hold timed out — return pending, client will reconnect
            json.NewEncoder(w).Encode(map[string]string{"status": "pending"})
            return
        }

        select {
        case <-ticker.C:
            continue // check again
        case <-r.Context().Done():
            return // client disconnected
        }
    }
}
```

---

## 4. Danbao Integration (Existing OAuth2 + Nexlink initData)

Danbao (`danbao/`) has its **own independent auth system** that predates the Nexlink dApp browser. To work as a Nexlink dApp, danbao must support both auth paths.

### 4.1. Danbao's current auth (standalone)

Danbao uses a full OAuth2 server (`danbao-api/pkg/oauth2server/`):

- **Grant types:** `password`, `refresh_token`, `authorization_code`
- **Token type:** Opaque 256-bit bearer tokens (not JWT), stored in PostgreSQL
- **TTLs:** Access tokens 2h, refresh tokens 7d
- **User login:** `POST /api/v1/user/login` with `{ username, password }` → returns `{ access_token, refresh_token }`
- **Admin login:** `POST /api/v1/admin/login` — same flow but validates `is_admin` flag and scope `"admin"`
- **Middleware:** Bearer token validated on every request, user ID threaded into request context
- **Frontend (danbao-web):** Stores token in localStorage, `Authorization: Bearer <token>` on every request
- **Frontend (danbao-admin):** Same pattern, additionally re-checks `is_admin` on each `check()` call

### 4.2. Database migration: add Nexlink binding column

The current `users` table has **no column to store a Nexlink user ID**, and `password_hash` / `email` are `NOT NULL` — blocking auto-creation from initData.

**Current schema constraints that block initData users:**

| Column | Constraint | Problem |
|---|---|---|
| `password_hash` | `VARCHAR(255) NOT NULL` | initData users have no password |
| `email` | `VARCHAR(255) NOT NULL` | initData doesn't include email |
| `username` | `VARCHAR(64) NOT NULL UNIQUE` | Need to generate one from Nexlink identity |
| (missing) | — | No column to store `nexlink_user_id` |

**Required migration:**

```sql
-- File: danbao-api/internal/db/migrations/000018_nexlink_user_binding.sql

-- 1. Add Nexlink binding column (nullable — existing users won't have it)
ALTER TABLE users
  ADD COLUMN nexlink_user_id BIGINT;

-- Unique constraint: one Nexlink user → one danbao user
CREATE UNIQUE INDEX users_nexlink_user_id_uniq
  ON users(nexlink_user_id) WHERE nexlink_user_id IS NOT NULL;

-- 2. Allow passwordless users (initData-created accounts)
ALTER TABLE users
  ALTER COLUMN password_hash DROP NOT NULL;

-- 3. Allow email-less users (initData doesn't provide email)
ALTER TABLE users
  ALTER COLUMN email DROP NOT NULL;
```

**Updated user table after migration:**

| Column | Type | Constraint | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | danbao internal ID |
| `username` | VARCHAR(64) | NOT NULL UNIQUE | Generated as `"nx_{nexlink_user_id}"` for initData users |
| `password_hash` | VARCHAR(255) | **NULLABLE** | NULL for initData-only users |
| `email` | VARCHAR(255) | **NULLABLE** | NULL for initData-only users |
| `nexlink_user_id` | BIGINT | UNIQUE (nullable) | **NEW** — Nexlink numeric user ID |
| `display_name` | VARCHAR(64) | nullable | Synced from initData `nickname` |
| `avatar_url` | VARCHAR(512) | nullable | Synced from initData `avatar` |
| ... | | | (other columns unchanged) |

### 4.3. Four login methods — one account

Every danbao user ends up with **one account** regardless of how they first arrive. The four entry methods are:

| # | Method | Context | Nexlink account required? |
|---|---|---|---|
| 1 | **Form registration** | General browser | No |
| 2 | **QR code scan** | General browser + Nexlink app | Yes |
| 3 | **dApp browser login** | Inside Nexlink app | Yes (automatic) |
| 4 | **Browser authorization popup** | General browser (after initData arrives) | Yes |

All Nexlink-based methods (2, 3, 4) deliver the same `initData` payload to the dApp backend. The backend handles them identically: verify signature → look up `nexlink_user_id` → respond.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Method 1: Form registration (no Nexlink needed)                 │
│  ───────────────────────────────────────────────                 │
│  User visits danbao-web → fills register form                    │
│  → account created with username, password, email                │
│  → nexlink_user_id = NULL                                        │
│  → user can later bind Nexlink via Method 4                      │
│                                                                  │
│  Method 2: QR code scan login (general browser)                  │
│  ──────────────────────────────────────────────                  │
│  User visits danbao-web in Chrome/Safari                         │
│  → site shows QR code (or Nexlink Login Widget)                  │
│  → user scans with Nexlink app → confirms                        │
│  → Nexlink backend delivers initData to dApp backend             │
│  → backend verifies → checks nexlink_user_id binding             │
│                                                                  │
│  Method 3: dApp browser login (inside Nexlink app)               │
│  ─────────────────────────────────────────────────               │
│  User opens danbao inside Nexlink app                            │
│  → initData auto-injected by SDK                                 │
│  → frontend sends initData to dApp backend                       │
│  → backend verifies → checks nexlink_user_id binding             │
│                                                                  │
│  Method 4: Browser authorization popup                           │
│  ────────────────────────────────────────                        │
│  After Method 2 or 3 delivers initData for an UNBOUND user:     │
│  → backend returns { status: "unbound", nexlinkUser }            │
│  → browser shows popup: "Bind existing account or create new?"   │
│  → user chooses to bind (enter password) or create fresh         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Method 1: Form registration

Standard registration — no Nexlink identity involved.

```
POST /api/v1/user/register
{ "username": "john_doe", "password": "...", "email": "john@mail.com" }
    │
    ▼
INSERT INTO users (
  username,        -- "john_doe"
  password_hash,   -- bcrypt("password")
  email,           -- "john@mail.com"
  nexlink_user_id  -- NULL (not bound)
)
    │
    ▼
Issue OAuth2 token → user is logged in
```

The user logs in later with `POST /user/login { username, password }` as usual.

#### Methods 2 & 3: Nexlink login (QR scan or in-app)

Both methods deliver `initData` to the same backend endpoint. The backend does **not** auto-create an account when the `nexlink_user_id` is unknown — it returns `"unbound"` status so the frontend can show the authorization popup (Method 4).

```
initData arrives (via QR poll or in-app SDK)
    │
    ▼
POST /api/v1/user/login-nexlink
{ "initData": "user=%7B...%7D&hash=..." }
    │
    ▼
Backend:
  1. Verify initData signature → extract nexlink_user_id
  2. SELECT * FROM users WHERE nexlink_user_id = 12345
    │
    ├── FOUND → already bound
    │   → sync profile (display_name, avatar_url)
    │   → issue OAuth2 token
    │   → return { status: "ok", token, user }
    │
    └── NOT FOUND → unbound
        → return { status: "unbound", nexlinkUser: { id, nickname, avatar } }
```

When the frontend receives `"ok"`, the user is logged in. When it receives `"unbound"`, it triggers the authorization popup (Method 4).

#### Method 4: Browser authorization popup

When initData arrives for an unknown `nexlink_user_id`, the frontend shows a popup:

```
┌──────────────────────────────────────────────┐
│                                              │
│  Welcome, John!                              │
│  Your Nexlink account is not yet linked.     │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  I have an existing account            │  │
│  │  (enter username + password to bind)   │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  Create a new account                  │  │
│  │  (auto-create from Nexlink identity)   │  │
│  └────────────────────────────────────────┘  │
│                                              │
└──────────────────────────────────────────────┘
```

**Option A — Bind existing account:** User enters their danbao username and password.

```
POST /api/v1/user/bind-nexlink
{
  "username": "john_doe",
  "password": "their_password",
  "initData": "user=%7B...%7D&..."
}
    │
    ▼
Backend:
  1. Verify initData signature → extract nexlink_user_id = 12345
  2. Authenticate username/password → find user id=1
  3. Check user.nexlink_user_id IS NULL (not already bound)
  4. UPDATE users SET nexlink_user_id = 12345 WHERE id = 1
  5. Issue OAuth2 token for the bound account
```

| Column | Before bind | After bind |
|---|---|---|
| `username` | `john_doe` | `john_doe` |
| `password_hash` | `$2a$10$...` | `$2a$10$...` |
| `email` | `john@mail.com` | `john@mail.com` |
| `nexlink_user_id` | **NULL** | **12345** |

**Option B — Create new account:** One-click, no form needed.

```
POST /api/v1/user/create-nexlink
{
  "initData": "user=%7B...%7D&..."
}
    │
    ▼
Backend:
  1. Verify initData signature → extract nexlink_user_id = 12345
  2. INSERT INTO users (
       username,          -- "nx_12345" (generated)
       password_hash,     -- NULL
       email,             -- NULL
       nexlink_user_id,   -- 12345
       display_name,      -- "John"
       avatar_url         -- from initData
     )
  3. Issue OAuth2 token → return to frontend
```

The user can later set a password via `POST /user/set-password` to enable form login.

#### Returning user (already bound)

Once an account has `nexlink_user_id` set, all subsequent Nexlink logins (QR or in-app) skip the popup:

```
initData.user.id = 12345
    │
    ▼
SELECT * FROM users WHERE nexlink_user_id = 12345
    → found: danbao user id=1
    │
    ▼
UPDATE users SET
  display_name = initData.nickname,   -- sync profile
  avatar_url = initData.avatar,
  last_login_at = NOW()
WHERE nexlink_user_id = 12345
    │
    ▼
Issue OAuth2 token → return { status: "ok", token, user }
```

#### Account completion

| Entry path | What's missing | How to complete |
|---|---|---|
| Method 1 (form) | `nexlink_user_id` | Log in via QR or in-app → bind (Method 4, Option A) |
| Method 2/3 → Option B (create new) | password, email, custom username | Settings → "Set password" (`POST /user/set-password`) |
| Method 2/3 → Option A (bind existing) | Nothing — already complete | — |

#### Why duplicates are impossible

| Guard | How |
|---|---|
| `nexlink_user_id UNIQUE` constraint | DB rejects a second row with the same Nexlink user ID |
| Option A bind checks `nexlink_user_id IS NULL` | Won't overwrite an already-bound account |
| Option B checks `SELECT` before `INSERT` | Only creates if no match exists |
| `"unbound"` response prevents silent auto-create | User must explicitly choose bind or create |

### 4.4. Backend service methods

```go
// FindByNexlinkUserID looks up a danbao user by Nexlink ID.
// Returns ErrNotFound if no account is bound.
func (s *Service) FindByNexlinkUserID(ctx context.Context, nexlinkUserID int64) (UserResp, error) {
    e, err := s.repo.FindByNexlinkUserID(ctx, nexlinkUserID)
    if err != nil {
        return UserResp{}, err
    }
    return toResp(e), nil
}

// CreateNexlinkUser creates a new passwordless user bound to a Nexlink identity.
// Called when user chooses "Create new account" in the authorization popup.
func (s *Service) CreateNexlinkUser(
    ctx context.Context,
    nexlinkUserID int64,
    nickname string,
    avatar string,
) (UserResp, error) {
    e, err := s.repo.CreateNexlinkUser(ctx, Entity{
        Username:      fmt.Sprintf("nx_%d", nexlinkUserID),
        NexlinkUserID: &nexlinkUserID,
        DisplayName:   nickname,
        AvatarURL:     avatar,
        // password_hash = NULL, email = NULL
    })
    if err != nil {
        return UserResp{}, err
    }
    return toResp(e), nil
}

// SetNexlinkUserID binds a Nexlink identity to an existing account.
// Called when user chooses "Bind existing account" in the authorization popup.
func (s *Service) SetNexlinkUserID(ctx context.Context, userID int64, nexlinkUserID int64) error {
    return s.repo.SetNexlinkUserID(ctx, userID, nexlinkUserID)
}

// UpdateProfile syncs display name and avatar from initData.
func (s *Service) UpdateProfile(ctx context.Context, userID int64, nickname, avatar string) error {
    return s.repo.UpdateProfile(ctx, userID, nickname, avatar)
}
```

```sql
-- name: FindByNexlinkUserID :one
SELECT * FROM users WHERE nexlink_user_id = $1;

-- name: CreateNexlinkUser :one
INSERT INTO users (
  username, nexlink_user_id,
  display_name, avatar_url
) VALUES ($1, $2, $3, $4)
RETURNING *;

-- name: SetNexlinkUserID :exec
UPDATE users SET nexlink_user_id = $1 WHERE id = $2;
```

### 4.5. Login endpoint — returns `"unbound"` for unknown users

```go
// POST /api/v1/user/login-nexlink
func (h *Handler) LoginNexlink(w http.ResponseWriter, r *http.Request) {
    var req struct {
        InitData string `json:"initData"`
    }
    json.NewDecoder(r.Body).Decode(&req)

    // 1. Verify initData signature
    result, err := VerifyNexlinkInitData(req.InitData, h.dappSecret)
    if err != nil {
        http.Error(w, "unauthorized", 401)
        return
    }

    // 2. Look up existing bound user
    user, err := h.userService.FindByNexlinkUserID(r.Context(), result.User.ID)
    if errors.Is(err, ErrNotFound) {
        // No account bound to this nexlink_user_id → tell frontend
        json.NewEncoder(w).Encode(map[string]any{
            "status": "unbound",
            "nexlinkUser": map[string]any{
                "id":       result.User.ID,
                "nickname": result.User.Nickname,
                "avatar":   result.User.Avatar,
            },
        })
        return
    }
    if err != nil {
        http.Error(w, "internal error", 500)
        return
    }

    // 3. Sync profile from initData
    _ = h.userService.UpdateProfile(r.Context(), user.ID, result.User.Nickname, result.User.Avatar)

    // 4. Issue danbao OAuth2 token
    token, err := h.oauth2.PasswordGrantInternal(user.ID, user.Username, "user")
    if err != nil {
        http.Error(w, "internal error", 500)
        return
    }

    // 5. Return same response shape as /user/login
    json.NewEncoder(w).Encode(map[string]any{
        "status":        "ok",
        "access_token":  token.AccessToken,
        "refresh_token": token.RefreshToken,
        "token_type":    "Bearer",
        "expires_in":    token.ExpiresIn,
        "user":          user,
    })
}
```

### 4.6. Bind endpoint — link Nexlink identity to existing account

```go
// POST /api/v1/user/bind-nexlink
func (h *Handler) BindNexlink(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Username string `json:"username"`
        Password string `json:"password"`
        InitData string `json:"initData"`
    }
    json.NewDecoder(r.Body).Decode(&req)

    // 1. Verify initData
    result, err := VerifyNexlinkInitData(req.InitData, h.dappSecret)
    if err != nil {
        http.Error(w, "unauthorized", 401)
        return
    }

    // 2. Authenticate with username/password
    user, err := h.userService.Authenticate(r.Context(), req.Username, req.Password)
    if err != nil {
        http.Error(w, "invalid credentials", 401)
        return
    }

    // 3. Check not already bound
    if user.NexlinkUserID != nil {
        http.Error(w, "account already bound to a Nexlink user", 409)
        return
    }

    // 4. Bind
    err = h.userService.SetNexlinkUserID(r.Context(), user.ID, result.User.ID)
    if err != nil {
        // UNIQUE constraint violation → this Nexlink user is bound elsewhere
        http.Error(w, "nexlink user already bound to another account", 409)
        return
    }

    // 5. Issue token
    token, err := h.oauth2.PasswordGrantInternal(user.ID, user.Username, "user")
    if err != nil {
        http.Error(w, "internal error", 500)
        return
    }

    json.NewEncoder(w).Encode(map[string]any{
        "status":        "ok",
        "access_token":  token.AccessToken,
        "refresh_token": token.RefreshToken,
        "token_type":    "Bearer",
        "expires_in":    token.ExpiresIn,
        "user":          user,
    })
}
```

### 4.7. Create endpoint — new account from Nexlink identity

```go
// POST /api/v1/user/create-nexlink
func (h *Handler) CreateNexlink(w http.ResponseWriter, r *http.Request) {
    var req struct {
        InitData string `json:"initData"`
    }
    json.NewDecoder(r.Body).Decode(&req)

    // 1. Verify initData
    result, err := VerifyNexlinkInitData(req.InitData, h.dappSecret)
    if err != nil {
        http.Error(w, "unauthorized", 401)
        return
    }

    // 2. Create passwordless user
    user, err := h.userService.CreateNexlinkUser(r.Context(), result.User.ID, result.User.Nickname, result.User.Avatar)
    if err != nil {
        http.Error(w, "nexlink user already exists", 409)
        return
    }

    // 3. Issue token
    token, err := h.oauth2.PasswordGrantInternal(user.ID, user.Username, "user")
    if err != nil {
        http.Error(w, "internal error", 500)
        return
    }

    json.NewEncoder(w).Encode(map[string]any{
        "status":        "ok",
        "access_token":  token.AccessToken,
        "refresh_token": token.RefreshToken,
        "token_type":    "Bearer",
        "expires_in":    token.ExpiresIn,
        "user":          user,
    })
}
```

**Key point:** After initData verification, danbao issues its own OAuth2 token. The rest of the danbao system (middleware, refresh, revocation) works unchanged. The initData is only the **entry point** — it replaces the username/password step but everything downstream stays the same.

### 4.8. Danbao-web frontend changes

```js
// danbao-web login page — handles all 4 methods
async function login() {
  if (window.NexlinkApp && NexlinkApp.initData) {
    // Method 3: Inside Nexlink app — use initData
    await loginWithInitData(NexlinkApp.initData);
  } else {
    // Method 1: form login, or Method 2: QR code scan
    // QR scan also calls loginWithInitData after poll completes
    const res = await api.post('/user/login', { username, password });
    authStore.setToken(res.access_token);
    router.push('/modules/profile');
  }
}

// Shared handler for Method 2 (QR) and Method 3 (in-app)
async function loginWithInitData(initData) {
  const res = await api.post('/user/login-nexlink', { initData });

  if (res.status === 'ok') {
    // Already bound — logged in
    authStore.setToken(res.access_token);
    router.push('/modules/profile');
    return;
  }

  if (res.status === 'unbound') {
    // Method 4: Show authorization popup
    showBindingPopup(res.nexlinkUser, initData);
  }
}

// Method 4: Browser authorization popup
function showBindingPopup(nexlinkUser, initData) {
  // Show dialog: "Welcome, {nexlinkUser.nickname}!"
  // Option A: "I have an existing account" → show username/password form
  // Option B: "Create a new account" → one-click create
}

async function bindExistingAccount(username, password, initData) {
  const res = await api.post('/user/bind-nexlink', {
    username, password, initData,
  });
  authStore.setToken(res.access_token);
  router.push('/modules/profile');
}

async function createNexlinkAccount(initData) {
  const res = await api.post('/user/create-nexlink', { initData });
  authStore.setToken(res.access_token);
  router.push('/modules/profile');
}
```

### 4.9. Existing password login must handle nullable password_hash

After the migration, `password_hash` is nullable. The existing `Authenticate` method must reject users with no password:

```go
func (s *Service) Authenticate(ctx context.Context, username, password string) (Entity, error) {
    e, err := s.repo.FindByUsername(ctx, username)
    if err != nil { return Entity{}, ErrInvalidCredentials }

    // Reject passwordless users (initData-only accounts)
    if e.PasswordHash == "" {
        return Entity{}, ErrInvalidCredentials
    }

    if err := bcrypt.CompareHashAndPassword(
        []byte(e.PasswordHash), []byte(password),
    ); err != nil {
        return Entity{}, ErrInvalidCredentials
    }
    return e, nil
}
```

### 4.10. Relevant danbao code locations

| Component | File |
|---|---|
| OAuth2 service | `danbao/danbao-api/pkg/oauth2server/service.go` |
| Token storage (PostgreSQL) | `danbao/danbao-api/pkg/oauth2server/store.go` |
| User login handler | `danbao/danbao-api/internal/modules/user/handle.go` |
| Admin login handler | `danbao/danbao-api/internal/modules/user/admin_handle.go` |
| Admin auth middleware | `danbao/danbao-api/pkg/adminauth/adminauth.go` |
| OAuth2 routes | `danbao/danbao-api/internal/router/oauth2.go` |
| Request middleware | `danbao/danbao-api/internal/router/middleware.go` |
| DB migration | `danbao/danbao-api/internal/db/migrations/000003_oauth2_tokens.sql` |
| Web login page | `danbao/danbao-web/app/(auth)/login/page.tsx` |
| Web auth hook | `danbao/danbao-web/hooks/use-auth.ts` |
| Web auth store | `danbao/danbao-web/store/use-auth-store.ts` |
| Admin auth provider | `danbao/danbao-admin/src/provider/authProvider.ts` |

---

## 5. Registration Flow

Registration is **not a separate flow** — it happens during the first login attempt, depending on which method the user chooses.

### 5.1. Method 1: Form registration (general browser)

User visits danbao-web in a regular browser (no `window.NexlinkApp`) and fills a traditional register form:

```
┌─────────────────────────────┐
│ Username:  [           ]    │
│ Email:     [           ]    │
│ Password:  [           ]    │
│ Confirm:   [           ]    │
│                             │
│       [  Register  ]        │
└─────────────────────────────┘
```

The account is created with `nexlink_user_id = NULL`. The user can later bind their Nexlink identity when they log in via QR code or in-app — the authorization popup (Section 4.3, Method 4) handles binding.

### 5.2. Methods 2 & 3: QR code scan or dApp browser login

When initData arrives for a `nexlink_user_id` that doesn't exist in the database, the `POST /user/login-nexlink` endpoint returns `{ status: "unbound" }`. The frontend shows the authorization popup (Section 4.3, Method 4).

If the user chooses **"Create a new account"** → `POST /user/create-nexlink` creates a passwordless account with `nexlink_user_id` set. This is the registration step — no form required.

If the user chooses **"I have an existing account"** → `POST /user/bind-nexlink` links the Nexlink identity to the existing account. No new registration occurs.

### 5.3. Account completion

| Entry path | What's missing | How to complete |
|---|---|---|
| Method 1 (form) | `nexlink_user_id` | Log in via QR or in-app → authorization popup → bind |
| Methods 2/3 → create new | password, email, custom username | Settings → "Set password" (`POST /user/set-password`) |
| Methods 2/3 → bind existing | Nothing — already complete | — |

Once both sides are filled in, the account has full access from both contexts (Nexlink app + general browser).

### 5.4. Nexlink Platform Registration (prerequisite)

Users must have a Nexlink account before using any dApp. The native app registration flow:

```
Phone/Email input → Verification code → Set password → Set profile → Done
```

| Step | Implementation |
|---|---|
| Phone/Email validation | `nexlink/lib/pages/register/register_logic.dart` |
| Verification code | `nexlink/lib/pages/register/verify_phone/` |
| Password setup | `nexlink/lib/pages/register/set_password/` |
| Profile setup | `nexlink/lib/pages/register/set_self_info/` |

---

## 6. initData Signature Verification

### 6.1. Algorithm

Two-step HMAC-SHA256 derivation:

```
Step 1: secret_key = HMAC_SHA256(key="NexlinkData", message=<dapp_secret_key>)

Step 2: data_check_string = sort(all_fields_except_hash)
                              .map(k => k + "=" + v)
                              .join("\n")

Step 3: computed_hash = HEX(HMAC_SHA256(key=secret_key, message=data_check_string))

Step 4: if (computed_hash === received_hash) → VALID
```

**Important:** In step 1, `"NexlinkData"` is the **HMAC key** and `dapp_secret_key` is the **message**. This matches how the Dart `Hmac(sha256, utf8.encode(hmacKeyLabel))` constructor works (second arg = key) and how Go `hmac.New(sha256.New, []byte(hmacKeyLabel))` works (second arg = key). Both implementations are verified consistent.

### 6.2. Mode A — Local Verification (Recommended)

The dApp backend holds the `secret_key` and verifies locally. No network call to Nexlink.

**Node.js:**

```js
import { createHmac } from 'crypto';

function verifyNexlinkInitData(initData, dappSecret) {
  const params = new URLSearchParams(initData);
  const hash = params.get('hash');
  if (!hash) return { valid: false, reason: 'missing hash' };
  params.delete('hash');

  const dataCheckString = [...params.entries()]
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([k, v]) => `${k}=${v}`)
    .join('\n');

  const secretKey = createHmac('sha256', 'NexlinkData')
    .update(dappSecret).digest();
  const computed = createHmac('sha256', secretKey)
    .update(dataCheckString).digest('hex');

  if (computed !== hash) return { valid: false, reason: 'bad signature' };

  const authDate = parseInt(params.get('auth_date'), 10);
  if (Date.now() / 1000 - authDate > 86400) {
    return { valid: false, reason: 'expired' };
  }

  return {
    valid: true,
    user: JSON.parse(params.get('user')),
    authDate,
    dappId: parseInt(params.get('dapp_id'), 10),
  };
}
```

**Go:**

```go
func VerifyNexlinkInitData(initData, dappSecret string) (*VerifyResult, error) {
    v, _ := url.ParseQuery(initData)
    hash := v.Get("hash")
    v.Del("hash")

    keys := make([]string, 0, len(v))
    for k := range v { keys = append(keys, k) }
    sort.Strings(keys)
    parts := make([]string, len(keys))
    for i, k := range keys { parts[i] = k + "=" + v.Get(k) }
    dataCheckString := strings.Join(parts, "\n")

    mac1 := hmac.New(sha256.New, []byte("NexlinkData"))
    mac1.Write([]byte(dappSecret))
    secretKey := mac1.Sum(nil)

    mac2 := hmac.New(sha256.New, secretKey)
    mac2.Write([]byte(dataCheckString))
    computed := hex.EncodeToString(mac2.Sum(nil))

    if !hmac.Equal([]byte(computed), []byte(hash)) {
        return nil, errors.New("bad signature")
    }
    authDate, _ := strconv.ParseInt(v.Get("auth_date"), 10, 64)
    if time.Now().Unix()-authDate > 86400 {
        return nil, errors.New("expired")
    }
    return &VerifyResult{ /* ... */ }, nil
}
```

**Note on Node.js `createHmac`:** `createHmac('sha256', 'NexlinkData')` — the second argument is the key. `.update(dappSecret)` provides the message. This matches the algorithm.

### 6.3. Mode B — Remote Verification

For dApps that prefer not to store the secret. Calls the Nexlink API to verify:

```
POST /dapp/verify
{ "initData": "user=...&hash=...", "clientId": 42 }

→ 200: { "errCode": 0, "data": { "valid": true, "user": {...} } }
→ 401: { "errCode": 40002, "errMsg": "bad signature" }
```

---

## 7. Security Model

### Trust boundaries

```
TRUSTED:     Nexlink native app, Nexlink API, dApp backend
             (share dapp.secret_key)

UNTRUSTED:   dApp frontend (JS in WebView or browser), network
             (receives signed initData it cannot forge)
```

### Threat mitigations

| Threat | Mitigation |
|---|---|
| dApp JS steals user's master token | Master token never enters WebView |
| dApp JS forges user identity | Cannot generate HMAC hash without secret |
| Replay attack (reuse old initData) | `auth_date` expiry (24h server-enforced), `query_id` uniqueness |
| Cross-dApp impersonation | Each dApp has its own HMAC secret |
| Stale cached initData | Cache TTLs: 30min fresh, 24h max usable, background refresh |
| QR code replay | One-time token, expires in 2-5 minutes |
| QR code interception | Token is useless without Nexlink app confirmation |
| Man-in-the-middle | All endpoints require HTTPS/TLS |

### QR login specific security

- QR tokens are **single-use** — consumed on first successful confirmation.
- QR tokens **expire** after a short window (2-5 minutes).
- The mobile app must show a **confirmation dialog** before sending initData.
- The browser poll endpoint returns the session token **only once**, then deletes it.
- The `initData` sent during QR confirmation is the same HMAC-signed payload used in-app — same verification, same security guarantees.

---

## 8. Deep Link Format for QR Codes

```
nexlink://auth/qr?token=<qrToken>&dapp=<dappId>
```

| Parameter | Required | Description |
|---|---|---|
| `token` | yes | One-time QR session token (from Nexlink backend) |
| `dapp` | yes | dApp ID (for display and initData scoping) |

**No callback URL.** The QR code never contains any external URL. The Nexlink app only communicates with the Nexlink backend.

The Nexlink app must register a handler for the `nexlink://auth/qr` scheme that:

1. Parses the parameters
2. Fetches dApp info from the Nexlink backend using the `dapp` ID (name, icon for confirmation dialog)
3. Shows: "Log in to **[dApp Name]** as **[Your Name]**?"
4. On confirm: calls `POST /dapp/qr/confirm { qrToken, dappId }` on the Nexlink backend — the backend signs initData and stores it with the token

---

## 9. Dual-Mode dApp Template

A complete pattern for dApps that work both inside Nexlink and in external browsers:

```js
class NexlinkAuth {
  constructor({ dappId, apiBase }) {
    this.dappId = dappId;
    this.apiBase = apiBase;
  }

  async login() {
    // Check if running inside Nexlink app
    if (window.NexlinkApp) {
      return this.loginInApp();
    } else {
      return this.loginWithQr();
    }
  }

  // --- In-app login (automatic) ---
  async loginInApp() {
    return new Promise((resolve, reject) => {
      const tryLogin = async () => {
        const initData = window.NexlinkApp.initData;
        if (!initData) {
          reject(new Error('No initData available'));
          return;
        }

        const res = await fetch(`${this.apiBase}/auth/nexlink-login`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ initData }),
        });

        if (!res.ok) reject(new Error('Login failed'));
        resolve(await res.json());
      };

      if (NexlinkApp.initData) {
        tryLogin();
      } else {
        NexlinkApp.onReady(tryLogin);
      }
    });
  }

  // --- QR code login (external browser) ---
  async loginWithQr() {
    // 1. Create QR session (dApp backend proxies to Nexlink API)
    const { qrToken, expiresAt } = await fetch(
      `${this.apiBase}/auth/qr/create`,
      { method: 'POST' }
    ).then(r => r.json());

    // 2. Build deep link and render QR — NO callback URL
    const deepLink = `nexlink://auth/qr?token=${qrToken}&dapp=${this.dappId}`;

    this.renderQrCode(deepLink);

    // 3. Poll dApp backend for confirmation
    return this.pollStatus(qrToken, expiresAt);
  }

  renderQrCode(data) {
    // Use qrcode.js, qr-code-styling, or similar
    const container = document.getElementById('qr-container');
    // ... render QR code with `data` ...
  }

  async pollStatus(qrToken, expiresAt) {
    // Long polling: each request blocks for up to ~25s on the server.
    // Loop immediately on 'pending' (server hold timed out).
    while (Date.now() < expiresAt * 1000) {
      try {
        const res = await fetch(
          `${this.apiBase}/auth/qr/status?token=${qrToken}`
        );
        const data = await res.json();

        if (data.status === 'confirmed') {
          return { token: data.sessionToken };
        }
        if (data.status === 'expired') {
          throw new Error('QR code expired');
        }
        // 'pending' → reconnect immediately
      } catch (e) {
        if (e.message === 'QR code expired') throw e;
        // Network error — wait 2s then retry
        await new Promise(r => setTimeout(r, 2000));
      }
    }
    throw new Error('QR code expired');
  }
}
```

Usage:

```js
const auth = new NexlinkAuth({
  dappId: 42,
  apiBase: 'https://api.my-dapp.com',
});

try {
  const { token } = await auth.login();
  localStorage.setItem('session', token);
  // Proceed to app
} catch (e) {
  console.error('Login failed:', e);
}
```

---

## 10. Nexlink Login Widget (Hosted Embeddable Script)

The Dual-Mode Template (Section 9) requires each dApp developer to implement QR rendering, long-poll logic, and environment detection themselves. A **hosted login widget** bundles all of this into a single `<script>` tag — similar to Telegram's Login Widget.

### 10.1. dApp developer usage

```html
<!-- Drop-in: one script tag + one callback -->
<div id="nexlink-login"></div>
<script src="https://nexlink-api/static/nexlink-login-widget.js"
  data-dapp-id="42"
  data-size="large"
  data-radius="8"
  data-onauth="onNexlinkAuth">
</script>
<script>
  function onNexlinkAuth(result) {
    // result = {
    //   initData: "user=%7B...%7D&hash=...",   // signed payload
    //   user: { id, uid, nickname, avatar },    // parsed (display only)
    //   authDate: 1718700000
    // }

    // Send to your backend for verification + session creation
    fetch('/api/auth/nexlink-login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ initData: result.initData }),
    })
    .then(r => r.json())
    .then(({ token }) => {
      localStorage.setItem('session', token);
      window.location.href = '/dashboard';
    });
  }
</script>
```

### 10.2. What the widget does internally

```
┌─────────────────────────────────────────────────────────┐
│  nexlink-login-widget.js (hosted on Nexlink CDN)        │
│                                                         │
│  1. Read data-* attributes from <script> tag            │
│                                                         │
│  2. Detect environment:                                 │
│     ├─ window.NexlinkApp exists?                        │
│     │  YES → In-app mode (auto-login)                   │
│     │        • NexlinkApp.onReady(cb)                   │
│     │        • Read NexlinkApp.initData                 │
│     │        • Call data-onauth callback immediately     │
│     │                                                   │
│     │  NO → Browser mode (QR login)                     │
│     │       • POST /dapp/qr/create (Nexlink API)        │
│     │       • Render QR code into container element      │
│     │       • Start long-poll loop                      │
│     │       • On confirmed → call data-onauth callback  │
│     │       • On expired → show refresh button          │
│                                                         │
│  3. Widget provides:                                    │
│     • "Log in with Nexlink" branded button              │
│     • QR code display with countdown timer              │
│     • Loading spinner during poll                       │
│     • Auto-refresh on expiry                            │
│     • Mobile-responsive layout                          │
│     • Dark/light theme (auto-detect or data-theme)      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 10.3. Widget `<script>` attributes

| Attribute | Required | Default | Description |
|---|---|---|---|
| `data-dapp-id` | yes | — | Your dApp's numeric ID from the Nexlink backend |
| `data-onauth` | yes | — | Global callback function name, called with `{ initData, user, authDate }` |
| `data-size` | no | `"large"` | Button size: `"small"`, `"medium"`, `"large"` |
| `data-radius` | no | `"8"` | Border radius in pixels |
| `data-theme` | no | `"auto"` | `"light"`, `"dark"`, or `"auto"` (follows `prefers-color-scheme`) |
| `data-qr-size` | no | `"200"` | QR code dimensions in pixels |
| `data-container` | no | auto-insert | CSS selector for custom container element |

### 10.4. Widget vs manual integration

| Aspect | Widget (`<script>` tag) | Manual (Section 9 NexlinkAuth class) |
|---|---|---|
| **Setup effort** | 5 lines of HTML | ~100 lines of JS + QR library |
| **QR rendering** | Built-in | Developer brings own QR library |
| **Long polling** | Built-in | Developer implements |
| **Environment detection** | Built-in | Developer implements |
| **UI/styling** | Branded, configurable via `data-*` | Fully custom |
| **Updates** | Auto-updated from CDN | Developer must update manually |
| **Customization** | Limited to `data-*` attributes | Full control |
| **Best for** | Quick integration, standard login pages | Custom UIs, SPAs, non-standard flows |

### 10.5. Widget implementation (Nexlink-side)

The widget JS file is served from the Nexlink backend's static assets. Core structure:

```js
// nexlink-login-widget.js (served from Nexlink CDN)
(function () {
  'use strict';

  // 1. Find the <script> tag that loaded us
  const script = document.currentScript;
  const dappId = parseInt(script.getAttribute('data-dapp-id'), 10);
  const onAuthName = script.getAttribute('data-onauth');
  const theme = script.getAttribute('data-theme') || 'auto';
  const qrSize = parseInt(script.getAttribute('data-qr-size') || '200', 10);
  const NEXLINK_API = script.src.replace('/static/nexlink-login-widget.js', '');

  // 2. Inject minimal CSS (scoped to .nexlink-login-widget)
  const style = document.createElement('style');
  style.textContent = `
    .nexlink-login-widget { /* ... branded styles ... */ }
    .nexlink-login-widget .qr-container { /* ... */ }
    .nexlink-login-widget .status-text { /* ... */ }
  `;
  document.head.appendChild(style);

  // 3. Detect environment
  if (window.NexlinkApp) {
    // In-app: auto-login
    const ready = () => {
      const initData = window.NexlinkApp.initData;
      if (initData && window[onAuthName]) {
        const params = new URLSearchParams(initData);
        window[onAuthName]({
          initData,
          user: JSON.parse(params.get('user') || '{}'),
          authDate: parseInt(params.get('auth_date'), 10),
        });
      }
    };
    if (NexlinkApp.initData) ready();
    else NexlinkApp.onReady(ready);
    return;
  }

  // 4. Browser mode: render QR login UI
  const container = script.getAttribute('data-container')
    ? document.querySelector(script.getAttribute('data-container'))
    : script.parentElement;

  const widget = document.createElement('div');
  widget.className = 'nexlink-login-widget';
  container.appendChild(widget);

  async function startQrLogin() {
    widget.innerHTML = '<div class="status-text">Loading...</div>';

    // Create QR session directly with Nexlink API
    const { qrToken, expiresAt } = await fetch(
      `${NEXLINK_API}/dapp/qr/create`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ clientId: dappId }),
      }
    ).then(r => r.json());

    // Render QR code (uses inline SVG — no external QR library needed)
    const deepLink = `nexlink://auth/qr?token=${qrToken}&dapp=${dappId}`;
    widget.innerHTML = `
      <div class="qr-container">${generateQrSvg(deepLink, qrSize)}</div>
      <div class="status-text">Scan with Nexlink app</div>
    `;

    // Long-poll for result
    while (Date.now() < expiresAt * 1000) {
      try {
        const res = await fetch(
          `${NEXLINK_API}/dapp/qr/status?token=${qrToken}&clientId=${dappId}`
        );
        const data = await res.json();

        if (data.status === 'confirmed') {
          widget.innerHTML = '<div class="status-text">Login successful</div>';
          if (window[onAuthName]) {
            const params = new URLSearchParams(data.initData);
            window[onAuthName]({
              initData: data.initData,
              user: JSON.parse(params.get('user') || '{}'),
              authDate: parseInt(params.get('auth_date'), 10),
            });
          }
          return;
        }
        if (data.status === 'expired') break;
        // 'pending' → loop immediately (server held ~25s)
      } catch (e) {
        await new Promise(r => setTimeout(r, 2000));
      }
    }

    // Expired — show refresh button
    widget.innerHTML = `
      <div class="status-text">QR code expired</div>
      <button class="refresh-btn" onclick="this.remove()">Refresh</button>
    `;
    widget.querySelector('.refresh-btn').addEventListener('click', startQrLogin);
  }

  startQrLogin();

  // Inline QR code generator (SVG-based, no external dependency)
  function generateQrSvg(data, size) {
    // Minimal QR encoder — or embed a lightweight library like qr-creator
    // Returns an SVG string: <svg ...>...</svg>
    // Implementation omitted for brevity
  }
})();
```

### 10.6. Security considerations for hosted widget

| Concern | Mitigation |
|---|---|
| Widget JS served over CDN — could be tampered | Serve with `integrity` hash; host on same domain as API |
| Widget calls Nexlink API directly from browser | `POST /dapp/qr/create` requires `clientId` (public) — no secret exposed. QR token is useless without Nexlink app confirmation |
| `data-onauth` callback receives initData in browser | Same as in-app flow — browser gets initData but cannot forge it. Backend must still verify signature |
| Cross-origin requests | Nexlink API sets CORS headers for registered dApp domains |

---

## 11. Implementation Checklist

### Already implemented

- [x] initData generation on Nexlink backend (`POST /browser/init_data`)
- [x] Go-side HMAC signing and verification (`handler_verify.go`, `handler_browser.go`)
- [x] Dart-side HMAC signing (`init_data.dart` → `InitDataSigner`)
- [x] JS SDK injection into WebView (`JsBridge.buildScript()`)
- [x] Stub SDK with queue-and-replay (`JsBridge.stubScript`)
- [x] initData cache with stale-while-revalidate (`DappInitDataCache`)
- [x] Per-chain address injection (`window.__nexlink_addresses`)
- [x] Remote verify endpoint (`POST /dapp/verify`)
- [x] Bridge module architecture (registry + per-module handlers)
- [x] QR scanner hardware access (`home_logic.dart` → `MobileScannerController`)
- [x] Danbao OAuth2 server (password, refresh_token, authorization_code grants)
- [x] Danbao user/admin login endpoints

### To be implemented for danbao initData integration

- [ ] **danbao-api: DB migration `000018_nexlink_user_binding.sql`** — add `nexlink_user_id` column; make `password_hash` and `email` nullable
- [ ] **danbao-api: Update `Entity` struct** — add `NexlinkUserID` field
- [ ] **danbao-api: `FindByNexlinkUserID` query** — lookup by `nexlink_user_id`
- [ ] **danbao-api: `CreateNexlinkUser` query** — insert without password/email
- [ ] **danbao-api: `SetNexlinkUserID` repo method** — `UPDATE users SET nexlink_user_id = $1 WHERE id = $2`
- [ ] **danbao-api: `POST /user/login-nexlink` endpoint** — verify initData, return `"ok"` (bound) or `"unbound"` (not found)
- [ ] **danbao-api: `POST /user/bind-nexlink` endpoint** — verify initData + password, bind `nexlink_user_id` to existing account (Method 4, Option A)
- [ ] **danbao-api: `POST /user/create-nexlink` endpoint** — verify initData, create passwordless account (Method 4, Option B)
- [ ] **danbao-api: `POST /user/set-password` endpoint** — let initData-created users set username/password/email
- [ ] **danbao-api: Guard `Authenticate` for NULL password_hash** — reject passwordless users on password login
- [ ] **danbao-web: Dual-mode login page** — detect `window.NexlinkApp`, auto-login with initData or show form/QR
- [ ] **danbao-web: Authorization popup UI** — show "Bind existing account" / "Create new account" when `login-nexlink` returns `"unbound"`
- [ ] **danbao-web: Set password UI** — settings page for initData-created users to set password/email

### To be implemented for QR login

- [ ] **Nexlink backend: `POST /dapp/qr/create`** — create QR session (scoped to clientId, 2-5min TTL)
- [ ] **Nexlink backend: `POST /dapp/qr/confirm`** — user confirms login, backend signs initData and stores with token
- [ ] **Nexlink backend: `GET /dapp/qr/status`** — dApp backend polls for confirmed initData
- [ ] **Nexlink backend: QR session storage** — table/cache for token → status + initData
- [ ] **Nexlink app: Deep link handler** — `nexlink://auth/qr?token=&dapp=` scheme parsing
- [ ] **Nexlink app: QR confirmation dialog** — show dApp name, user name, confirm/cancel
- [ ] **Nexlink app: Confirm action** — call `POST /dapp/qr/confirm` on Nexlink backend (no external URL)

### To be implemented for Login Widget

- [ ] **Nexlink backend: `nexlink-login-widget.js`** — hosted embeddable script with environment detection, QR rendering, long-poll loop
- [ ] **Nexlink backend: CORS configuration** — allow registered dApp domains to call `/dapp/qr/create` and `/dapp/qr/status` from browser
- [ ] **Nexlink backend: Inline QR SVG generator** — embed lightweight QR encoder in widget (no external dependency)
- [ ] **Nexlink backend: Widget CSS** — branded, scoped styles with light/dark theme support
- [ ] **Nexlink backend: Static asset serving** — serve widget JS from `/static/nexlink-login-widget.js` with cache headers and SRI hash

### To be implemented for general improvements

- [ ] `NexlinkApp.version` — currently hardcoded as `'0.0.0'`, should reflect real app version
- [ ] `NexlinkApp.checkVersion()` / `NexlinkApp.requestUpgrade()` — stub only
- [ ] Session invalidation (`POST /dapp/session/invalidate`) — endpoint spec exists, needs implementation
- [ ] Secret rotation (`POST /dapp/rotate-secret`) — endpoint spec exists, needs implementation

---

## 12. References

- [Nexlink Mini App Spec](../docs/NEXLINK_MINIAPP_SPEC.md) — full JS SDK and API specification
- [Nexlink Architecture](../docs/architecture-README.md) — app versioning and upgrade system
