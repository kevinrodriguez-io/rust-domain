# Push delivery: APNs and FCM

Verified 2026-09-19 against Apple and Google primary documentation. Library pins in versions.md.

## Libraries

- **APNs: `@parse/node-apn` 8.1.0.** Maintained; supports Node 20/22/24.
- **FCM: `firebase-admin` 14.4.0.** Requires Node ≥ 22.
- **Never `node-apn`.** Abandoned at 3.0.0 from November 2020, still declaring Node 4.6 support. The maintained fork is the Parse Community one.

`apns2` 12.2.0 is a viable alternative but has not released since May 2025.

## APNs authentication

Token-based (`.p8`) auth. The provider JWT has exactly **four** fields:

| Location | Key | Value |
|---|---|---|
| Header | `alg` | `ES256` only — "APNs supports only the ES256 algorithm" |
| Header | `kid` | 10-character Key ID |
| Claims | `iss` | 10-character **Team ID** |
| Claims | `iat` | Epoch seconds UTC |

**There is no `exp` claim, and no `sub` claim.** If you have seen `sub` required, that is Safari/Web Push VAPID at `web.push.apple.com` — a different scheme where `sub` must be a `mailto:` or URL. An APNs provider JWT with `sub` is not what Apple documents.

**Refresh is a two-sided window:** "Refresh your token no more than once every 20 minutes and no less than once every 60 minutes." Below the floor → `429 TooManyProviderTokenUpdates`. Above the ceiling → `403 ExpiredProviderToken`. **Minting a fresh JWT per request is a bug**, not a safe default — cache and reuse for roughly 30–50 minutes.

**One connection pool per (developer account, key, environment).** APNs binds a connection to the team/key used on its first push; mixing keys, key types, or environments on one connection yields `403 UnrelatedKeyIdInToken` or `403 BadEnvironmentKeyIdInToken`. APNs also "doesn't support authentication tokens from multiple developer accounts over a single connection."

Certificate-based (`.p12`) auth is **not deprecated** — no sunset is announced. But it supports only a subset of push types (`location` requires token auth) and needs one connection per app. Use token auth anyway.

## APNs headers

| Header | Required | Semantics |
|---|---|---|
| `:method` | Yes | `POST` only |
| `:path` | Yes | `/3/device/<device_token>` (hex) |
| `authorization` | Yes (token auth) | `bearer <jwt>` |
| `apns-push-type` | Required on watchOS 6+, recommended always | 11 values, below |
| `apns-topic` | Yes | Bundle ID, plus a push-type suffix for some types |
| `apns-id` | No | Canonical UUID; APNs generates one if omitted |
| `apns-expiration` | No | Epoch seconds. **`0` = single attempt, no storage** |
| `apns-priority` | No | **`10`, `5`, and `1` are all valid.** Defaults to `10` |
| `apns-collapse-id` | No | Merge key, **max 64 bytes**; over → `400 BadCollapseId` |

**`apns-priority` values:** `10` sends immediately; `5` considers device power; `1` prioritises power "over all other factors… and prevent awakening the device."

**Priority is constrained per type.** `background` pushes **must** use priority 5 — "Using priority `10` is an error." Live Activities accept only 5 or 10. Bad values → `400 BadPriority`.

The 11 valid `apns-push-type` values and their `apns-topic` suffix:

| Value | Topic suffix | Notes |
|---|---|---|
| `alert` | none | |
| `background` | none | Priority 5 mandatory |
| `complication` | `h.complication` | watchOS/iOS only |
| `controls` | `.push-type.controls` | |
| `fileprovider` | `.pushkit.fileprovider` | |
| `liveactivity` | `.push-type.liveactivity` | iOS/iPadOS only |
| `location` | `.location-query` | Token auth only |
| `mdm` | UID from MDM cert subject | |
| `pushtotalk` | `.voip-ptt` | |
| `voip` | `.voip` | |
| `widgets` | `.push-type.widgets` | |

Nothing was added in 2025 or 2026. Apple's docs contradict themselves on the Live Activity suffix — one page omits the leading dot. **Use the dotted form**; it is the one in Apple's working sample code.

`apns-unique-id` is **response-only and Development-only**. Never send it, and do not build production observability on it.

## APNs payload limits

| Kind | Limit |
|---|---|
| Regular (alert, background, liveactivity via device token, widgets) | **4096 bytes** |
| VoIP (PushKit) | **5120 bytes** |
| Broadcast push (Live Activity via channel) | **5120 bytes** |

**"You must not use a compressed JSON payload."** Do not gzip.

## APNs endpoints

| Purpose | Host | Ports |
|---|---|---|
| Device push, production | `api.push.apple.com` | 443 or **2197** |
| Device push, sandbox | `api.sandbox.push.apple.com` | 443 or **2197** |
| Channel management, production | `api-manage-broadcast.push.apple.com` | **2196** |
| Channel management, sandbox | `api-manage-broadcast.sandbox.push.apple.com` | **2195** |

TLS 1.2+ and HTTP/2 throughout. Port 2197 exists to let APNs traffic through a firewall that blocks other HTTPS.

**Channel management is not on 443** — port 443 on that host serves an unrelated certificate, so hostname verification fails.

Apple mandates: "Make an uncached DNS query to resolve the APNs server name, before each connection." Apple load-balances via DNS, so caching pins you to a server subset.

## APNs errors

| Code | Reason | Action |
|---|---|---|
| `410` | `Unregistered` | **Delete the token.** Body carries a `timestamp` |
| `410` | `ExpiredToken` | Stop sending |
| `400` | `BadDeviceToken` | Invalid **or wrong environment**. Never retry |
| `403` | `ExpiredProviderToken` | Mint a new JWT |
| `429` | `TooManyRequests` | Too many pushes to one token; retry with delay |
| `429` | `TooManyProviderTokenUpdates` | JWT refreshed more than once per 20 min |
| `413` | `PayloadTooLarge` | Never retry |

**Never retry** `BadDeviceToken`, `DeviceTokenNotForTopic`, `Forbidden`, `ExpiredToken`, `Unregistered`, or `PayloadTooLarge`. 5xx may be retried after 15 minutes.

**Apple publishes no numeric rate limit.** Instead: "An error code beginning with 4XX slows down your ability to send notifications… APNs disconnects a provider connection with too many error conditions," sooner for `BadDeviceToken`. Note that **`410` is explicitly not counted as an error condition**, so token cleanup traffic is safe. Prune aggressively.

**Read concurrency from the wire.** Use the HTTP/2 `SETTINGS` frame's `MAX_CONCURRENT_STREAMS` rather than a constant — "don't assume a specific number of streams." With token auth, APNs allows only one stream until the first authenticated request lands.

**TLS pinning:** pin only the **USERTrust RSA Certification Authority** root, never the leaf or intermediate. The leaf rotates roughly quarterly. Prefer the OS trust store. The 2025 SHA-2 root migration is complete; nothing is pending.

## FCM HTTP v1

```
POST https://fcm.googleapis.com/v1/projects/<PROJECT_ID>/messages:send
```

Auth is OAuth 2.0 Bearer from a service account, scope `https://www.googleapis.com/auth/firebase.messaging`. Use Application Default Credentials where possible; `firebase-admin` handles token refresh.

Envelope:

```json
{
  "message": {
    "data": { "k": "v" },
    "notification": { },
    "android": { },
    "apns": { "headers": { }, "payload": { "aps": { } } },
    "webpush": { },
    "fid": "…"
  }
}
```

All `data` values must be strings. Exactly one target: `fid`, `topic`, `condition`, or the deprecated `token`.

**Target `fid`, not `token`.** The REST reference marks `token` as "Deprecated: Use `fid` instead." This is a client-and-server migration: clients opt in via `FirebaseMessagingInstallationIdEnabled` (Apple) or `firebase_messaging_installation_id_enabled` (Android), after which the SDK delivers FIDs instead of tokens. On Android, `getToken()`, `deleteToken()`, and `onNewToken()` are deprecated, and **once any component opts into FID registration, legacy `getToken()` is disabled app-wide** — which breaks third-party SDKs still calling it. No removal date is announced and both paths work today, so design the device-registration schema around FIDs while tolerating tokens.

### FCM's APNs defaults will bite you

**"The backend sets a default value for `apns-expiration` of 30 days and a default value for `apns-priority` of 10 if not explicitly set."** Priority 10 is *an error* for background pushes. Always set it explicitly in `apns.headers`.

Also: **"When sending payloads containing only data fields to iOS devices, only normal priority (`apns-priority: 5`) is allowed in `ApnsConfig`."**

### FCM limits

| Limit | Value |
|---|---|
| Payload (most messages) | **4096 bytes**, keys and values |
| Payload (**topic** messages) | **2048 bytes** — half |
| `ttl` | 0 to 2,419,200 seconds (4 weeks) |
| `sendEachForMulticast()` | up to **500** FIDs/tokens per call |
| Downstream quota | 600,000 messages/minute/project default |
| Topics per app instance | 2,000 |
| Concurrent fanouts per project | 1,000 |
| Topics per condition | 5 |

The topic cap being half the direct cap is easy to miss: a payload that delivers fine to an FID is rejected on a topic send.

### FCM errors

`UNREGISTERED` (404) is the analogue of APNs 410 — **delete the target**. `THIRD_PARTY_AUTH_ERROR` (401) means "APNs certificate or web push auth key was invalid or missing," i.e. your uploaded `.p8` is broken. Others: `INVALID_ARGUMENT` (400), `SENDER_ID_MISMATCH` (403), `QUOTA_EXCEEDED` (429), `UNAVAILABLE` (503), `INTERNAL` (500).

### Legacy APIs are gone, not deprecated

`fcm.googleapis.com/fcm/send` and `/batch` both return **404** today. `sendAll()`, `sendMulticast()`, `sendToDevice()`, `sendToTopic()`, and `sendToCondition()` were **removed** from `firebase-admin` in v13.0.0 (November 2024).

Replacements: `sendAll()` → **`sendEach()`**, `sendMulticast()` → **`sendEachForMulticast()`**.

The .NET SDK is an outlier — `SendAllAsync`/`SendMulticastAsync` still compile there and fail at runtime with a bare 404. Not relevant to a Node service, but worth recognising if you see that symptom elsewhere.

## FCM for iOS delivery

You must upload the **`.p8` auth key plus its Key ID** to Firebase (Settings → Cloud Messaging). "At least one is required" of development/production. Missing or invalid → `THIRD_PARTY_AUTH_ERROR`.

On Apple platforms, **method swizzling must stay enabled** — "Swizzling is required by the SDK, and without it key Firebase features such as FCM registration handling don't function properly."

## Token lifecycle

FCM garbage-collects Android registrations after **270 days** of inactivity. With the deprecated token APIs "the client SDK does not automatically manage refreshes on routine syncs," so refresh server-side — Google recommends monthly, and "There is no benefit to doing the refresh more frequently than weekly." **FID-based registrations are synced automatically by the SDK**, which is an operational argument for migrating beyond the deprecation itself.

Delete a target immediately on APNs `410`/`BadDeviceToken` or FCM `UNREGISTERED`. A dirty list gets you disconnected from APNs.
