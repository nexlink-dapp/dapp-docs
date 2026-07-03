# Nexlink dApp API 参考

本文档描述 Nexlink 平台与 dApp 之间的 API 接口。所有请求和响应均使用 JSON。所有接口都要求使用 HTTPS。

---

## Types

### InitData

一个经 URL 编码的查询字符串，包含经过加密签名的用户身份。通过 Nexlink SDK（`window.NexlinkApp.initData`）或二维码登录流程交付给 dApp。

**格式：** 经 URL 编码的查询字符串（非 JSON）

**示例**（为便于阅读已解码——线上传输格式为 URL 编码）：

```
user      = {"uid":"79552634","openim_id":"8566642169","nickname":"John","avatar":"https://cdn.nexlink.app/avatar/79552634.jpg","language_code":"en"}
user_id   = 79552634
auth_date = 1718700000
dapp_id   = 42
query_id  = 550e8400-e29b-41d4-a716-446655440000
hash      = a1b2c3d4e5f6...
```

| Field | Type | Required | Description |
|---|---|---|---|
| `user` | String | 是 | JSON 编码的 [User](#user) 对象 |
| `user_id` | String | 是 | 用户的永久 `uid`（与 `user.uid` 相同）。请将其作为稳定键使用——它绝不是顺序递增的 id。 |
| `auth_date` | Integer | 是 | 生成该载荷时的 Unix 时间戳（秒） |
| `dapp_id` | Integer | 是 | 数字 dApp ID |
| `query_id` | String | 是 | 唯一会话标识符（UUID），用于防重放 |
| `start_param` | String | 否 | 来自打开 URL 的深链接参数 |
| `hash` | String | 是 | 对所有其他字段计算的 HMAC-SHA256 十六进制签名（64 个字符） |

---

### User

表示一个 Nexlink 用户身份。作为 JSON 字符串嵌入在 [InitData](#initdata) 中。

| Field | Type | Description |
|---|---|---|
| `uid` | String | **永久、非顺序递增**的用户标识符。不可变——请将其作为该用户的主键使用。 |
| `openim_id` | String | 聊天/会话身份。同样稳定；如果你集成 Nexlink 聊天会用到它。 |
| `nickname` | String | 显示名称 |
| `avatar` | String | 头像图片 URL |
| `language_code` | String | ISO 639-1 语言代码（例如 `"en"`、`"zh"`） |

> **通过 `uid` 识别用户。** 它是永久、不可变且非顺序递增的，因此可以安全地作为主键存储，且无法用于猜测其他用户或推断用户总数。Nexlink **不会**暴露任何顺序递增的内部 id。

#### Using Nexlink chat from your dApp

`uid` 用于识别一个用户，但它**无法**用来定位一个聊天会话。Nexlink 聊天运行在 OpenIM 上，会话以 **`openim_id`** 为键——因此打开会话时你传入的值是 `openim_id`。

在 Nexlink 应用内浏览器中，通过调用桥接处理器并传入目标用户的 `openim_id`，即可打开 1:1 聊天（例如客服或交易对手方）：

```js
// WebViewJavascriptBridge convention
WebViewJavascriptBridge.callHandler(
  "wvjb_contactCustomService",
  { contact_id: user.openim_id },   // the OpenIM id of the person to chat with
  function (res) { /* { ok: true } on success */ }
);
```

已登录用户自己的 `openim_id` 可从 [InitData](#initdata) 获取；要与**另一个**用户聊天，请通过 `POST /dapp/user/by_uid`（返回 `openimUserId`）从其 `uid` 解析出 `openim_id`。注意，对于尚未在聊天中开通的用户，`openim_id` 可能为空。

---

### QrSession

表示由 [POST /dapp/qr/create](#post-dappqrcreate) 创建的二维码登录会话。

| Field | Type | Description |
|---|---|---|
| `qrToken` | String | 一次性会话令牌（UUID） |
| `expiresAt` | Integer | 该会话过期时的 Unix 时间戳（自创建起 2-5 分钟） |

---

### QrStatus

来自 [GET /dapp/qr/status](#get-dappqrstatus) 的响应。

| Field | Type | Description |
|---|---|---|
| `status` | String | `"pending"`、`"confirmed"` 或 `"expired"` |
| `initData` | String | _可选。_ 已签名的 [InitData](#initdata) 字符串。仅当 `status` 为 `"confirmed"` 时存在。 |

---

### TokenResponse

由 danbao 登录/绑定/创建接口返回的 OAuth2 令牌响应。

| Field | Type | Description |
|---|---|---|
| `status` | String | `"ok"` |
| `access_token` | String | Bearer 访问令牌（2 小时 TTL） |
| `refresh_token` | String | 刷新令牌（7 天 TTL） |
| `token_type` | String | 始终为 `"Bearer"` |
| `expires_in` | Integer | 访问令牌的有效期（秒） |
| `user` | Object | 用户资料对象 |

---

### UnboundResponse

当 Nexlink 用户没有已关联的 danbao 账户时，由 [POST /user/login-nexlink](#post-userlogin-nexlink) 返回。

| Field | Type | Description |
|---|---|---|
| `status` | String | `"unbound"` |
| `nexlinkUser` | Object | `{ uid, openim_id, nickname, avatar }` —— 用于授权弹窗的展示信息 |

---

### PaymentOrder

表示由 [POST /dapp/order/create](#post-dappordercreate) 创建的支付订单。

| Field | Type | Description |
|---|---|---|
| `orderId` | String | Nexlink 分配的订单 UUID |
| `externalOrderId` | String | _可选。_ dApp 分配的订单标识符，用于关联对账 |
| `amount` | Integer | 以最小单位表示的支付金额（5 位小数：`100000` = `1.00`） |
| `symbol` | String | 代币符号：`"USDK"` 或 `"CNYT"` |
| `recipientAddress` | String | _可选。_ 收款方钱包地址（十六进制，带校验和） |
| `status` | Integer | `1` = 待支付，`2` = 已支付，`3` = 已取消，`4` = 已过期 |
| `txHash` | String | _可选。_ 链上交易哈希。当 `status` 为 `2` 时存在。 |
| `expireAt` | Integer | 该订单过期时的 Unix 时间戳 |
| `paidAt` | Integer | _可选。_ 支付确认时的 Unix 时间戳 |
| `dappName` | String | _可选。_ 创建该订单的 dApp 名称。在 `/browser/order/info` 响应中存在。 |
| `dappIcon` | String | _可选。_ 该 dApp 的图标 URL。在 `/browser/order/info` 响应中存在。 |

---

### PaymentResult

`NexlinkApp.payment.pay()`（应用内 JS SDK）的返回值。该 Promise 仅在成功时 **resolve**。在取消或失败时，该 Promise 会以一个错误 **reject**。

| Field | Type | Description |
|---|---|---|
| `status` | String | 始终为 `"paid"`（Promise 仅在成功时 resolve） |
| `txHash` | String | 链上交易哈希 |
| `orderId` | String | 订单 UUID |

**错误处理：** 取消和失败会抛出异常——请使用 `try/catch`：

```javascript
try {
  const result = await NexlinkApp.payment.pay({ orderId });
  // result.status === "paid"
} catch (e) {
  // e.message === "user_rejected" (user cancelled)
  // e.message === "order expired" / "unsupported token" / etc.
}
```

| Error message | Cause |
|---|---|
| `user_rejected` | 用户点击了取消或拒绝了生物识别 |
| `orderId is required` | 缺少 `orderId` 参数 |
| Order expired (localized) | 超过订单 TTL |
| `unsupported token: X` | 代币符号不在注册表中 |
| `already_processed` is returned as `{status: "already_processed", orderId}` | 订单已支付（不是错误） |

---

### TransferResult

`NexlinkApp.payment.transfer()`（应用内 JS SDK，直接转账模式）的返回值。该 Promise 仅在成功时 **resolve**。在取消或失败时，该 Promise 会以一个错误 **reject**。

| Field | Type | Description |
|---|---|---|
| `status` | String | 始终为 `"sent"`（Promise 仅在成功时 resolve） |
| `txHash` | String | 链上交易哈希 |

**错误处理：** 与 [PaymentResult](#paymentresult) 相同的模式——请使用 `try/catch`。

| Error message | Cause |
|---|---|
| `user_rejected` | 用户点击了取消或拒绝了生物识别 |
| `to, amount, and token are required` | 缺少必填参数 |
| `unsupported token: X` | 代币符号不在注册表中 |

---

### WebhookPayload

当订单被支付时交付给 dApp 的 `callbackUrl` 的载荷。

| Field | Type | Description |
|---|---|---|
| `orderId` | String | Nexlink 分配的订单 UUID |
| `externalOrderId` | String | _可选。_ dApp 分配的订单标识符（若在创建时提供则存在） |
| `status` | Integer | 始终为 `2`（已支付） |
| `amount` | Integer | 以最小单位表示的支付金额 |
| `symbol` | String | 代币符号（`"USDK"` 或 `"CNYT"`） |
| `txHash` | String | 链上交易哈希 |
| `paidAt` | Integer | 支付确认时的 Unix 时间戳 |
| `paidByUserId` | Integer | 实际付款人的 NexLink 用户 ID。未知时为 `0`。在[代付](PAYMENT.md#43-delegated-payment-pay-on-behalf)场景下与订单创建者不同。 |

**Headers：**

| Header | Description |
|---|---|
| `X-Nexlink-Timestamp` | Webhook 交付的 Unix 时间戳 |
| `X-Nexlink-Signature` | 用于校验的 HMAC-SHA256 十六进制签名 |

> 关于 Webhook 签名验证，请参阅 [PAYMENT.md 第 6 节](PAYMENT.md#6-webhook-callbacks)。

---

### ContractCallResult

`NexlinkApp.contract.call()`（应用内 JS SDK）的返回值。该 Promise 仅在成功时 **resolve**。在取消或失败时，该 Promise 会以一个错误 **reject**。

| Field | Type | Description |
|---|---|---|
| `status` | String | 始终为 `"sent"`（Promise 仅在成功时 resolve） |
| `txHash` | String | 链上交易哈希 |

**错误处理：** 与 [PaymentResult](#paymentresult) 相同的模式——请使用 `try/catch`。

| Error message | Cause |
|---|---|
| `user_rejected` | 用户点击了取消或拒绝了生物识别 |
| `contract, abi, method, and args are required` | 缺少必填参数 |
| ABI encoding errors | 参数与 ABI 类型不匹配 |

---

### ContractSession

表示一个合约调用会话。由 [POST /dapp/contract/create](#post-dappcontractcreate)、[POST /browser/contract/info](#post-browsercontractinfo)、[POST /dapp/contract/status](#post-dappcontractstatus) 和 [POST /browser/contract/confirm](#post-browsercontractconfirm) 返回。

| Field | Type | Description |
|---|---|---|
| `sessionToken` | String | 一次性合约会话令牌（UUID） |
| `contractAddress` | String | 目标合约地址 |
| `calldata` | String | ABI 编码的 calldata（十六进制） |
| `value` | String | 以 wei 计的原生代币数额 |
| `methodName` | String | _可选。_ 人类可读的方法名 |
| `abiJson` | String | _可选。_ 该合约的 ABI JSON |
| `status` | Integer | `1` = 待处理，`2` = 已确认，`3` = 失败，`4` = 已过期 |
| `txHash` | String | _可选。_ 交易哈希。当 `status` 为 `2` 时存在。 |
| `error` | String | _可选。_ 错误消息。当 `status` 为 `3` 时存在。 |
| `expireAt` | Integer | 该会话过期时的 Unix 时间戳 |

---

## Signature Verification

在信任载荷之前，dApp 后端必须验证 [InitData](#initdata) 中的 `hash` 字段。

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
| Signature | `computed_hash` 必须与 `hash` 匹配（在 Go 中使用 `hmac.Equal`，在 Node.js 中使用 `timingSafeEqual`） |
| Expiry | `now() - auth_date` 必须小于最大有效期（默认：86400 秒 / 24 小时） |
| Replay | 应跟踪 `query_id` 以防止复用（推荐：使用带 24 小时 TTL 的 Redis `SET NX`） |

> 完整的验证代码示例（Go 和 Node.js），请参阅 [AUTH.md 第 4 节](AUTH.md#4-initdata-signature-verification)。

---

## Nexlink Platform API

这些接口托管在 **Nexlink 后端**上。dApp 后端调用它们以支持二维码登录和远程验证。

**Base URL：** `https://nexlink-api`（请替换为你实际的 Nexlink API 域名）

---

### POST /browser/init\_data

为在 Nexlink 移动应用内打开某个 dApp 的用户生成一个已签名的 [InitData](#initdata) 载荷。这是一个由 Nexlink 应用调用的内部接口，dApp 后端不会直接调用它。

**Authorization：** Bearer 令牌（Nexlink 移动应用内部）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `dappId` | Integer | 是 | dApp 数字 ID |

**返回：** 位于响应封装内的 [InitData](#initdata) 字符串。

```json
{
  "errCode": 0,
  "data": {
    "initData": "user=%7B...%7D&auth_date=...&hash=..."
  }
}
```

**Errors：**

| errCode | Description |
|---|---|
| 1001 | 该 dApp 未配置 `secret_key`——无法生成 initData |

---

### POST /dapp/qr/create

创建一个新的二维码登录会话。当用户在通用浏览器中访问 dApp 时，由 dApp 后端调用此接口。成功时返回一个 [QrSession](#qrsession)。

**Authorization：** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `clientId` | Integer | 是 | 你的 dApp 的数字 ID |

**返回：** [QrSession](#qrsession)

```json
{
  "qrToken": "550e8400-e29b-41d4-a716-446655440000",
  "expiresAt": 1718700300
}
```

dApp 后端将该令牌编码为二维码的深链接：

```
nexlink://auth/qr?token=<qrToken>&dapp=<clientId>
```

> 二维码中**不含回调 URL**——只有令牌和 dApp ID。

**Errors：**

| HTTP Status | Description |
|---|---|
| 401 | API key 无效或缺失 |
| 404 | 给定 `clientId` 未找到对应 dApp |

---

### POST /dapp/qr/confirm

在用户扫描二维码并确认登录后，由 **Nexlink 移动应用**调用。后端生成已签名的 [InitData](#initdata) 并将其与 `qrToken` 一起存储，以便通过 [GET /dapp/qr/status](#get-dappqrstatus) 检索。

**Authorization：** `Bearer <user's nexlinkToken>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `qrToken` | String | 是 | 来自被扫描二维码的令牌 |
| `dappId` | Integer | 是 | 来自二维码深链接的 dApp 数字 ID |

**返回：**

```json
{ "errCode": 0 }
```

**Errors：**

| HTTP Status | Description |
|---|---|
| 400 | 令牌已过期或无效 |
| 401 | 用户未认证 |

**副作用：** 使用该 dApp 的 `secret_key` 生成已签名的 [InitData](#initdata) 并将其与 `qrToken` 一起存储。

---

### GET /dapp/qr/status

轮询二维码登录结果。支持**长轮询**——服务器最多保持连接 25 秒，在状态变化时立即返回。返回一个 [QrStatus](#qrstatus) 对象。

**Authorization：** `Bearer <dapp_api_key>`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `token` | String | 是 | 二维码会话令牌（查询参数） |
| `clientId` | Integer | 是 | dApp 数字 ID（查询参数） |

**返回：** [QrStatus](#qrstatus)

待处理（服务器保持约 25 秒后返回）：

```json
{ "status": "pending" }
```

已确认（包含已签名的 initData）：

```json
{
  "status": "confirmed",
  "initData": "user=%7B...%7D&auth_date=...&hash=..."
}
```

已过期：

```json
{ "status": "expired" }
```

| Status | Meaning | dApp Action |
|---|---|---|
| `pending` | 用户尚未扫描 | 立即重新连接 |
| `confirmed` | 用户已确认——包含 [InitData](#initdata) | 验证 initData，创建会话 |
| `expired` | 超过令牌 TTL | 显示"二维码已过期"，提供刷新选项 |

> **一次性读取：** 在返回 `confirmed` 之后，服务器会删除已存储的 initData。第二次请求将返回 `expired`。

---

### POST /dapp/verify

远程 [InitData](#initdata) 签名验证。适用于不希望在本地存储 `secret_key` 的 dApp。服务器按 `clientId` 查找密钥并在服务端验证。

**Authorization：** 无

| Parameter | Type | Required | Description |
|---|---|---|---|
| `initData` | String | 是 | 完整的 [InitData](#initdata) 查询字符串 |
| `clientId` | Integer | 是 | 你的 dApp 的数字 ID |

**返回：**

```json
{
  "errCode": 0,
  "data": {
    "valid": true,
    "user": {
      "uid": "79552634",
      "openim_id": "8566642169",
      "nickname": "John",
      "avatar": "https://cdn.nexlink.app/avatar/79552634.jpg"
    }
  }
}
```

**Errors：**

| errCode | Description |
|---|---|
| 40002 | 签名错误——initData 无效或被篡改 |

---

## dApp Authentication

所有 `/dapp/*` 接口（订单和合约管理）使用 **MD5 签名认证**，而非 Bearer 令牌。dApp 后端使用其 `secret_key` 对每个请求签名。

### Required Headers

| Header | Description |
|---|---|
| `dapp_id` | 你的 dApp 的数字 ID（字符串） |
| `request_time` | 当前 Unix 时间戳（字符串） |
| `sign` | MD5 签名（小写十六进制） |

### Signature Computation

```
sign = lowercase_hex( MD5( dapp_id + secret_key + request_time + request_body_bytes ) )
```

服务器验证签名，并拒绝 `request_time` 与服务器时钟相差过大的请求（时钟偏差容差可配置）。

### Example (Node.js)

```javascript
import { createHash } from 'crypto';

function signRequest(dappId, secretKey, body) {
  const requestTime = Math.floor(Date.now() / 1000).toString();
  const bodyStr = JSON.stringify(body);
  const sign = createHash('md5')
    .update(dappId + secretKey + requestTime + bodyStr)
    .digest('hex');

  return {
    headers: {
      'dapp_id': dappId,
      'request_time': requestTime,
      'sign': sign,
      'Content-Type': 'application/json',
    },
    body: bodyStr,
  };
}
```

> **注意：** 上文记录的二维码登录接口（`/dapp/qr/*`）使用的是 `Bearer <dapp_api_key>`。这些接口尚未实现——其认证机制可能会变更。

---

## Payment API

这些接口处理支付订单管理和二维码支付会话。关于架构细节，请参阅[支付集成](PAYMENT.md)。

**Base URL：** `https://nexlink-api`（与 Nexlink Platform API 相同）

---

### POST /dapp/order/create

创建一个新的支付订单。dApp 后端在发起支付前调用此接口。成功时返回一个 [PaymentOrder](#paymentorder)。

**Authorization：** MD5 签名（dApp 认证中间件——参见下文[认证](#dapp-authentication)）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `externalOrderId` | String | 否 | dApp 分配的订单标识符，用于关联对账（若提供则在每个 dApp 内唯一） |
| `amount` | Integer | 是 | 以最小单位表示的金额（5 位小数：`100000` = `1.00`） |
| `symbol` | String | 是 | 代币符号：`"USDK"` 或 `"CNYT"` |
| `recipientAddress` | String | 是 | 收款方钱包地址（十六进制，带校验和） |
| `callbackUrl` | String | 否 | 用于支付通知的 Webhook URL |
| `expireSeconds` | Integer | 否 | 订单 TTL（秒）（默认：3600） |

**返回：** [PaymentOrder](#paymentorder)

```json
{
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "externalOrderId": "shop-001",
  "amount": 10000000,
  "symbol": "USDK",
  "recipientAddress": "0x1234...abcd",
  "status": 1,
  "expireAt": 1718703600
}
```

> **幂等：** 如果提供了 `externalOrderId`，使用相同值创建订单会返回现有订单，而不会创建重复订单。

**Errors：**

| HTTP Status | Description |
|---|---|
| 401 | API key 无效或缺失 |
| 400 | 金额、符号或地址无效 |

---

### POST /dapp/order/query

查询支付订单的当前状态。提供 `orderId` 或 `externalOrderId` 其一即可。

**Authorization：** MD5 签名（dApp 认证中间件）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | 否 | Nexlink 订单 UUID（`orderId` 与 `externalOrderId` 至少需提供其一） |
| `externalOrderId` | String | 否 | dApp 分配的订单标识符 |

**返回：** [PaymentOrder](#paymentorder)

```json
{
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "externalOrderId": "shop-001",
  "amount": 10000000,
  "symbol": "USDK",
  "status": 2,
  "txHash": "0xabc123...",
  "paidAt": 1718700100
}
```

**Errors：**

| HTTP Status | Description |
|---|---|
| 401 | API key 无效或缺失 |
| 404 | 未找到订单 |

---

### POST /dapp/order/cancel

取消一个待支付订单。只有状态为 `1`（待支付）的订单才能被取消。

**Authorization：** MD5 签名（dApp 认证中间件）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | 是 | Nexlink 订单 UUID |

**返回：**

```json
{ "errCode": 0 }
```

**Errors：**

| HTTP Status | Description |
|---|---|
| 401 | API key 无效或缺失 |
| 400 | 订单不处于待支付状态 |
| 404 | 未找到订单 |

---

### POST /browser/order/mark\_paid

将订单标记为已支付。在用户确认应用内支付且链上交易已广播后，由 **NexLink 移动应用**调用。这是一个内部接口——dApp 后端通过 Webhook 而非调用此接口来接收确认。

**Authorization：** Bearer 令牌（NexLink 移动应用内部——`nexlinkToken`）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | 是 | Nexlink 订单 UUID |
| `txHash` | String | 是 | 链上交易哈希 |

**返回：** [PaymentOrder](#paymentorder)（已更新）

**Errors：**

| HTTP Status | Description |
|---|---|
| 400 | 订单已过期或已支付 |
| 401 | 用户未认证 |
| 404 | 未找到订单 |

**副作用：**
- 将订单转换为 `paid`（状态 `2`）
- 记录 `txHash`、`paidAt` 和 `paidByUserId`（调用此接口的已认证 NexLink 用户）
- 将 Webhook 交付入队发送到 `callbackUrl`

---

### Browser Payment Flow (QR Code)

对于基于浏览器的支付，dApp 直接将 `orderId` 包含在二维码深链接中。不需要单独的支付会话。

1. 通过 `POST /dapp/order/create` 创建订单 → 获得 `orderId`
2. 将深链接编码进二维码：`nexlink://pay?orderId=<orderId>`
3. 用户使用 NexLink 应用扫描二维码 → 应用获取订单详情、显示确认、执行支付
4. dApp 接收发送到 `callbackUrl` 的 Webhook 回调，或通过 `POST /dapp/order/query` 轮询

> **安全性：** `orderId` 是 UUID——不可猜测。支付仍需要用户通过生物识别确认。

---

## Contract API

这些接口处理面向外部浏览器的、基于二维码的合约调用会话。关于架构细节，请参阅[合约交互](CONTRACT.md)。

**Base URL：** `https://nexlink-api`（与 Nexlink Platform API 相同）

---

### POST /dapp/contract/create

为基于二维码的交互创建一个新的合约调用会话。当用户需要从外部浏览器签署一个合约调用时，由 dApp 后端调用此接口。成功时返回一个 [ContractSession](#contractsession)。

**Authorization：** MD5 签名（dApp 认证中间件）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `contractAddress` | String | 是 | 目标合约地址（十六进制，带校验和） |
| `calldata` | String | 是 | ABI 编码的 calldata（十六进制，以 `0x` 为前缀） |
| `value` | String | 否 | 以 wei 计的原生代币数额（默认 `"0"`） |
| `methodName` | String | 否 | 用于展示的人类可读方法名（例如 `"freeze"`） |
| `abiJson` | String | 否 | 该合约的 ABI JSON（启用解码后的确认 UI） |
| `callbackUrl` | String | 否 | 用于交易通知的 Webhook URL |
| `expireSeconds` | Integer | 否 | 会话 TTL（秒）（默认：300） |

**返回：** [ContractSession](#contractsession)

```json
{
  "sessionToken": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "contractAddress": "0x3d8b4425...",
  "calldata": "0x57e871e7...",
  "value": "0",
  "methodName": "freeze",
  "status": 1,
  "expireAt": 1718700300
}
```

dApp 后端将该令牌编码为二维码的深链接：

```
nexlink://contract?token=<sessionToken>&dapp=<clientId>
```

> 二维码中**不含合约地址、不含 calldata、不含 value**——只有令牌和 dApp ID。

**Errors：**

| HTTP Status | Description |
|---|---|
| 401 | API key 无效或缺失 |
| 400 | 合约地址或 calldata 无效 |

---

### POST /browser/contract/info

获取被扫描二维码的合约调用详情。在用户扫描合约调用二维码后，由 **NexLink 移动应用**调用。这是一个内部接口——dApp 后端应改用 [POST /dapp/contract/status](#post-dappcontractstatus)。

**Authorization：** Bearer 令牌（NexLink 移动应用内部——`nexlinkToken`）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `sessionToken` | String | 是 | 合约会话令牌 |

**返回：** [ContractSession](#contractsession)

**Errors：**

| HTTP Status | Description |
|---|---|
| 400 | 令牌已过期或无效 |
| 401 | 用户未认证 |
| 404 | 未找到合约会话 |

---

### POST /browser/contract/confirm

在用户确认一个二维码合约调用且链上交易已广播后，由 **NexLink 移动应用**调用。后端存储结果，以便通过 [POST /dapp/contract/status](#post-dappcontractstatus) 轮询。这是一个内部接口——dApp 后端应改用状态接口。

**Authorization：** Bearer 令牌（NexLink 移动应用内部——`nexlinkToken`）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `sessionToken` | String | 是 | 合约会话令牌 |
| `txHash` | String | 是 | 链上交易哈希 |

**返回：** [ContractSession](#contractsession)（已更新）

**Errors：**

| HTTP Status | Description |
|---|---|
| 400 | 令牌已过期或已使用 |
| 401 | 用户未认证 |

**副作用：**
- 将 `txHash` 与会话一起记录
- 将会话转换为 `confirmed` 状态（`2`）

---

### POST /dapp/contract/status

查询合约调用会话的当前状态。dApp 后端调用此接口以检查用户是否已确认该二维码合约调用。

**Authorization：** MD5 签名（dApp 认证中间件）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `sessionToken` | String | 是 | 合约会话令牌 |

**返回：** [ContractSession](#contractsession)

待处理：

```json
{
  "sessionToken": "7c9e6679-...",
  "contractAddress": "0x3d8b4425...",
  "status": 1,
  "expireAt": 1718700300
}
```

已确认（包含交易哈希）：

```json
{
  "sessionToken": "7c9e6679-...",
  "contractAddress": "0x3d8b4425...",
  "status": 2,
  "txHash": "0xabc123...",
  "expireAt": 1718700300
}
```

已过期：

```json
{
  "sessionToken": "7c9e6679-...",
  "contractAddress": "0x3d8b4425...",
  "status": 4,
  "expireAt": 1718700300
}
```

| Status | Code | Meaning | dApp Action |
|---|---|---|---|
| Pending | `1` | 用户尚未扫描或确认 | 再次轮询 |
| Confirmed | `2` | 用户已确认——交易已广播 | 更新 UI，处理结果 |
| Failed | `3` | 交易失败 | 显示错误 |
| Expired | `4` | 超过会话 TTL | 显示"二维码已过期"，提供刷新选项 |

---

## Escrow API

Escrow（担保）在 NEXLK 链上已部署的 **C2C** 和 **Guarantee** 合约上结算。它**不引入任何新的平台签名接口**——每个担保操作都是一次普通的合约调用：

| Escrow action | In-app | External browser |
|---|---|---|
| `approve`、`createTrade` / `createTransaction` / `openGuarantee`、`pay`、`reimburse`、`reclaimTrade` / `reclaimDeposit` | [`NexlinkApp.contract.call()`](#nexlinkappcontractcall) | 每个操作一个 [`POST /dapp/contract/create`](#post-dappcontractcreate) 二维码会话 |
| `getCountTransactions`、`getCase`、状态读取 | [`NexlinkApp.contract.read()`](#nexlinkappcontractread) | 直接 RPC `eth_call` |

关于架构、角色、ABI 和生命周期，请参阅[担保 / 担保支付](ESCROW.md)。

### Deployed contracts (chain `2026777`, current testnet)

| Contract | Address |
|---|---|
| C2C | `0x7781D90613061513aF33F99E9161473D76515AD0` |
| Guarantee | `0x0675Fe67E77F598868F6134eE1Cb9C47337F1e09` |
| JuryArbitrator | `0x86e43067a077Dea4806C57bfc29de9299122f622` |

> 请按环境覆盖地址。结算代币为 USDK / CNYT（5 位小数）——参见[代币注册表](PAYMENT.md#2-token-registry)。

### Backend correlation record

担保的资金模型是**客户端签名、后端关联**（参见 [ESCROW.md 第 8 节](ESCROW.md#8-backend-correlation)）。dApp 后端针对其自身订单存储以下字段——这是 dApp 侧的存储，而非平台接口：

| Field | Type | Source |
|---|---|---|
| `escrowTradeId` | Integer | 在 open 调用之前立即通过 `getCountTransactions()` 快照获取 |
| `buyerAddress` / `sellerAddress` | String | 各方在操作时在客户端捕获的 AA 钱包地址 |
| `freezeTx` | String | open（`createTrade` / `openGuarantee`）调用的 `txHash` |
| `releaseTx` | String | release（`pay` / `reclaim`）调用的 `txHash` |

### Legacy partner-custody correlation

在其自有的以 `bytes32` 为键的担保合约中托管资金的第三方平台，通过 dApp 的 `/external` API（danbao 专用）报告冻结/释放交易哈希以进行关联。关于旧版 ABI，请参阅 [ESCROW.md 第 9 节](ESCROW.md#9-legacy-single-escrow-abi)。第一方担保不使用此路径。

---

## JS SDK — Payment Methods

这些方法在 NexLink dApp 浏览器内的 `window.NexlinkApp.payment` 上可用。它们在外部浏览器中**不可用**。

用于检测：

```javascript
if (window.NexlinkApp && NexlinkApp.payment) {
  // In-app: payment methods available
} else {
  // External browser: use QR payment flow instead
}
```

---

### NexlinkApp.payment.pay

为预先创建的订单触发支付。显示一个原生确认 UI。返回一个 [PaymentResult](#paymentresult)。

```javascript
const result = await NexlinkApp.payment.pay({
  orderId: "550e8400-e29b-41d4-a716-446655440000"
});
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | 是 | 来自 [POST /dapp/order/create](#post-dappordercreate) 的订单 UUID |

**返回：** [PaymentResult](#paymentresult)

---

### NexlinkApp.payment.transfer

请求一次不带后端订单的直接代币转账。显示一个原生确认 UI。返回一个 [TransferResult](#transferresult)。**仅限应用内。**

```javascript
const result = await NexlinkApp.payment.transfer({
  to: "0x1234...abcd",
  amount: "100.00",
  token: "USDK"
});
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `to` | String | 是 | 收款方钱包地址（十六进制，带校验和） |
| `amount` | String | 是 | 人类可读的金额（例如 `"100.00"`） |
| `token` | String | 是 | 代币符号：`"USDK"` 或 `"CNYT"` |

**返回：** [TransferResult](#transferresult)

---

### NexlinkApp.payment.getOrderStatus

无需后端往返，从 NexLink 应用查询订单的当前状态。

```javascript
const status = await NexlinkApp.payment.getOrderStatus({
  orderId: "550e8400-e29b-41d4-a716-446655440000"
});
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orderId` | String | 是 | 订单 UUID |

**返回：**

| Field | Type | Description |
|---|---|---|
| `orderId` | String | 订单 UUID |
| `status` | String | `"pending"`、`"paid"`、`"cancelled"`、`"expired"` 或 `"unknown"` |

---

## JS SDK — Contract Methods

这些方法在 NexLink dApp 浏览器内的 `window.NexlinkApp.contract` 上可用。它们在外部浏览器中**不可用**——请改用[二维码合约流程](#contract-api)。

用于检测：

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

向智能合约发送一笔状态变更交易。SDK 会根据 ABI 编码 calldata，NexLink 应用会显示一个解码后的确认 UI。返回一个 [ContractCallResult](#contractcallresult)。

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
| `contract` | String | 是 | 合约地址（十六进制，带校验和） |
| `abi` | Array | 是 | ABI 数组（标准 Solidity ABI JSON 或人类可读格式） |
| `method` | String | 是 | 函数名（例如 `"freeze"`） |
| `args` | Array | 是 | 按顺序排列的函数参数 |
| `value` | String | 否 | 以 wei 计的原生代币数额（默认 `"0"`） |

**返回：** [ContractCallResult](#contractcallresult)

---

### NexlinkApp.contract.read

调用智能合约上的 `view` 或 `pure` 函数。无需签名、无确认 UI、无 gas 成本。返回解码后的返回值。

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
| `contract` | String | 是 | 合约地址（十六进制，带校验和） |
| `abi` | Array | 是 | ABI 数组 |
| `method` | String | 是 | 函数名（必须为 `view` 或 `pure`） |
| `args` | Array | 是 | 按顺序排列的函数参数 |

**返回：** 按照 ABI 输出规范解码后的值。单个返回值直接返回；多个值以数组形式返回。

---

### NexlinkApp.contract.encode

在不发送交易的情况下编码 ABI calldata。适用于手动构建 calldata 以配合 `NexlinkApp.wallet.sendTransaction()` 使用，或用于调试。

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
| `abi` | Array | 是 | ABI 数组 |
| `method` | String | 是 | 函数名 |
| `args` | Array | 是 | 按顺序排列的函数参数 |

**返回：** `String` —— 以 `0x` 为前缀的十六进制编码 calldata。

---

## Danbao Backend API

这些接口在 **danbao 后端**（`danbao-api`）中实现。它们处理 danbao 应用的基于 Nexlink 的登录、账户绑定和账户创建。

**Base URL：** `/api/v1`

> 其他 dApp 后端可将这些作为参考实现。其模式（验证 initData → 查找用户 → 签发令牌）对任何 dApp 都是相同的。

---

### POST /user/login-nexlink

验证 [InitData](#initdata) 并登录。如果 Nexlink 用户有一个已关联的 danbao 账户，则返回一个 [TokenResponse](#tokenresponse)。如果没有，则返回一个 [UnboundResponse](#unboundresponse)，以便前端能够显示授权弹窗。

**Authorization：** 无

| Parameter | Type | Required | Description |
|---|---|---|---|
| `initData` | String | 是 | 来自 Nexlink 的已签名 [InitData](#initdata) |

**成功时返回：** [TokenResponse](#tokenresponse)

```json
{
  "status": "ok",
  "access_token": "abc123...",
  "refresh_token": "def456...",
  "token_type": "Bearer",
  "expires_in": 7200,
  "user": { "id": 1, "username": "john_doe", "nexlink_user_id": 79552634 }
}
```

**未绑定时返回：** [UnboundResponse](#unboundresponse)

```json
{
  "status": "unbound",
  "nexlinkUser": { "uid": "79552634", "openim_id": "8566642169", "nickname": "John", "avatar": "https://..." }
}
```

**Errors：**

| HTTP Status | Body | Description |
|---|---|---|
| 401 | `"unauthorized"` | InitData 签名无效 |
| 403 | `"initData already used"` | 检测到重放（`query_id` 复用） |

---

### POST /user/bind-nexlink

将一个 Nexlink 身份关联到一个现有的 danbao 账户。用户必须提供其 danbao 凭据以证明账户所有权。成功时返回一个 [TokenResponse](#tokenresponse)。

**Authorization：** 无

| Parameter | Type | Required | Description |
|---|---|---|---|
| `username` | String | 是 | 现有的 danbao 用户名 |
| `password` | String | 是 | 现有的 danbao 密码 |
| `initData` | String | 是 | 来自 Nexlink 的已签名 [InitData](#initdata) |

**返回：** [TokenResponse](#tokenresponse)

```json
{
  "status": "ok",
  "access_token": "abc123...",
  "refresh_token": "def456...",
  "token_type": "Bearer",
  "expires_in": 7200,
  "user": { "id": 1, "username": "john_doe", "nexlink_user_id": 79552634 }
}
```

**Errors：**

| HTTP Status | Body | Description |
|---|---|---|
| 401 | `"unauthorized"` | InitData 签名无效 |
| 401 | `"invalid credentials"` | 用户名或密码错误 |
| 403 | `"initData already used"` | 检测到重放 |
| 409 | `"account already bound to a Nexlink user"` | 此 danbao 账户已存在一个 Nexlink 绑定 |
| 409 | `"nexlink user already bound to another account"` | 此 Nexlink 用户已关联到另一个 danbao 账户 |
| 429 | `"too many attempts, try again later"` | 超过速率限制（每 10 分钟 5 次尝试） |

---

### POST /user/create-nexlink

从一个 Nexlink 身份创建一个新的无密码 danbao 账户。用户名自动生成为 `nx_{uid}`。成功时返回一个 [TokenResponse](#tokenresponse)。

**Authorization：** 无

| Parameter | Type | Required | Description |
|---|---|---|---|
| `initData` | String | 是 | 来自 Nexlink 的已签名 [InitData](#initdata) |

**返回：** [TokenResponse](#tokenresponse)

```json
{
  "status": "ok",
  "access_token": "abc123...",
  "refresh_token": "def456...",
  "token_type": "Bearer",
  "expires_in": 7200,
  "user": { "id": 7, "username": "nx_79552634", "display_name": "John", "nexlink_user_id": 79552634 }
}
```

**Errors：**

| HTTP Status | Body | Description |
|---|---|---|
| 401 | `"unauthorized"` | InitData 签名无效 |
| 403 | `"initData already used"` | 检测到重放 |
| 429 | `"too many attempts"` | 超过速率限制（每 10 分钟 3 次尝试） |

> 如果 Nexlink 用户已经拥有账户（并发请求 / 双击），该接口会回退为登录，而不是返回错误。

---

### POST /user/unbind-nexlink

将一个 Nexlink 身份从一个 danbao 账户断开。要求用户已登录并确认其密码。解绑后撤销所有现有会话。

**Authorization：** `Bearer <session_token>`（用户必须已登录）

| Parameter | Type | Required | Description |
|---|---|---|---|
| `password` | String | 是 | 用于确认的当前账户密码 |

**返回：**

```json
{ "status": "ok" }
```

**Errors：**

| HTTP Status | Body | Description |
|---|---|---|
| 400 | `"set a password before unbinding"` | 无密码账户无法解绑（否则将失去所有访问权限） |
| 401 | `"invalid password"` | 密码错误 |
| 404 | `"not found"` | 未找到用户 |

**副作用：**
- 在用户记录上设置 `nexlink_user_id = NULL`
- 撤销所有现有会话（所有令牌立即失效）

> 解绑后，用户可在下次 Nexlink 登录时通过授权弹窗重新绑定。

---

## Proposed APIs (Design)

以下能力被记录为**设计规范**——其接口和 SDK 命名空间为提案（未实现），尚未实现。完整的请求/响应结构位于各自的文档中，构建完成后将被提升到本参考文档。

| Capability | Proposed surface | Spec |
|---|---|---|
| **订阅（Subscription）** | `NexlinkApp.subscription.*`；`/dapp/subscription/*` + `/browser/subscription/confirm`；订阅 Webhook | [SUBSCRIPTION.md](SUBSCRIPTION.md#6-proposed-backend-api) |
| **治理（Governance）** | 通过 [`NexlinkApp.contract`](#nexlinkappcontractcall) 使用标准 OpenZeppelin Governor / ERC20Votes ABI；可选的索引器接口 | [GOVERNANCE.md](GOVERNANCE.md#5-using-governance-from-a-dapp) |
| **NFT 发行（NFT issuance）** | 通过 [`NexlinkApp.contract`](#nexlinkappcontractcall) 使用 ERC-721（普通 + 灵魂绑定）；可选的门户发行 + 索引器接口 | [NFT.md](NFT.md#5-minting-via-the-contract-sdk) |

治理和 NFT 发行如今**不需要任何新的平台接口**——它们运行在现有的 [Contract API](#contract-api) 上。订阅是唯一引入新后端接口的能力，而所有这些接口都遵循现有的 [MD5 签名认证](#dapp-authentication)和 [Webhook](PAYMENT.md#6-webhook-callbacks) 约定。
