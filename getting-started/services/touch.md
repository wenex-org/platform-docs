# Touch

**Port:** REST `:3100` · gRPC `:5100`

Records and dispatches all outbound communications — email, in-app notices, web push, and SMS. Every create or send operation may trigger real-world communication. Always confirm recipient, content, and channel before acting.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Emails | `/touch/emails` | Outbound email records |
| Notices | `/touch/notices` | In-app notification records |
| Pushes | `/touch/pushes` | Web push subscription targets |
| Push Histories | `/touch/push-histories` | Push delivery attempt records |
| SMSs | `/touch/smss` | Outbound SMS records |

> `touch/push-histories` has **no MCP tools** — it is accessible via platform REST only.

## `touch/emails`

Stores outbound email envelope and body data.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `provider` | ✅ | `EmailProvider` | `NODEMAILER` |
| `to` | ✅ | string[] | Recipient email addresses |
| `from` | ✅ | string | Sender email (display name allowed) |
| `subject` | ✅ | string | Email subject |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `html` | string | HTML body |
| `text` | string | Plain text body |
| `cc` / `bcc` | string[] | Additional recipients |
| `reply_to` | string[] | Reply-to addresses |
| `date` | date | Scheduled send time |
| `attachments` | MongoId[] | Raw `special/files` IDs — not populatable |
| `smtp` | object | SMTP delivery metadata |

At least one of `html` or `text` must be present.

### Special Send Operations (platform REST only)

```
POST /touch/emails/send
```

## `touch/notices`

In-app notifications displayed inside client applications.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `NoticeType` | `INFO`, `EVENT`, `ALERT`, `WARNING`, `ERROR`, `SUCCESS`, `NOTICE`, `MESSAGE` |
| `title` | ✅ | string | Notice title |
| `content` | ✅ | string | Notice body |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `subtitle` | string | Secondary title |
| `category` | string | Custom grouping label |
| `visited` | boolean | Read status |
| `thumbnail` | MongoId | Raw file ID — not populatable |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `actions` | object[] | Action buttons |

## `touch/pushes`

Web push subscription targets for browser/device delivery.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `session` | ✅ | MongoId | Session ID |
| `keys` | ✅ | object | Push subscription keys — **write-only, never returned** |
| `endpoint` | ✅ | string | Push endpoint URL |
| `expiration` | ✅ | number | Expiration Unix timestamp |

### Special Send Operation (platform REST only)

```
POST /touch/pushes/send
```

## `touch/push-histories`

Delivery attempt records for push notifications.

> No MCP tools — use platform REST to inspect push delivery history.

### Population

| Path | Collection |
| --- | --- |
| `to` | `touch/pushes` |

## `touch/smss`

Outbound SMS records.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `provider` | ✅ | `SmsProvider` | `KAVENEGAR`, `MELIPAYAMAK` |
| `message` | ✅ | string | SMS body |
| `receptors` | ✅ | string[] | Recipient phone numbers (E.164 format) |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `sender` | string | Sender number or line ID |
| `res` | object | Provider response metadata |

### Special Send Operations (platform REST only)

| Endpoint | Purpose |
| --- | --- |
| `POST /touch/smss/send` | Direct SMS send |
| `POST /touch/smss/send/template` | Template SMS with parameters |
