# Conjoint

**Port:** REST `:3130` · gRPC `:5130`

Real-time messaging infrastructure. Manages messaging identities (accounts), conversation spaces (channels), contact directories, channel memberships, and messages. Message delivery uses EMQX/MQTT via the `publisher` worker.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Accounts | `/conjoint/accounts` | Messaging identity |
| Channels | `/conjoint/channels` | Conversation container |
| Contacts | `/conjoint/contacts` | Contact directory entries |
| Members | `/conjoint/members` | Account-to-channel membership |
| Messages | `/conjoint/messages` | Messages inside a channel |

## `conjoint/accounts`

A messaging identity that can participate in channels and messages.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `AccountType` | `HUMAN`, `ROBOT` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `profile` | MongoId | `identity/profiles` reference |
| `bio` | string | Short bio text |
| `status` | string | Local status string |

### Population

| Path | Collection |
| --- | --- |
| `profile` | `identity/profiles` |

## `conjoint/channels`

A conversation space — group, broadcast stream, or direct conversation.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `ChannelType` | `GROUP`, `BROADCAST`, `CONVERSATION` |
| `scope` | ✅ | `ChannelScope` | `PUBLIC`, `PRIVATE`, `PROTECTED` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Short channel name |
| `title` | string | Human-readable title |
| `profile` | MongoId | `identity/profiles` reference |
| `account` | MongoId | Owning `conjoint/accounts` |
| `pinned_messages` | MongoId[] | Pinned message IDs (raw, not populatable) |

### Population

| Path | Collection |
| --- | --- |
| `profile` | `identity/profiles` |
| `account` | `conjoint/accounts` |

## `conjoint/contacts`

Contact directory entry, optionally linked to a messaging account.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `ContactType` | `MAIN`, `HOME`, `WORK`, `OTHER` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `phone` | string | Phone number |
| `email` | string | Email address |
| `account` | MongoId | `conjoint/accounts` reference |
| `nickname` | string | Display nickname |

### Population

| Path | Collection |
| --- | --- |
| `account` | `conjoint/accounts` |

## `conjoint/members`

Joins an account to a channel with optional role and permissions.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `account` | ✅ | MongoId | `conjoint/accounts` |
| `channel` | ✅ | MongoId | `conjoint/channels` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `role` | string | Membership role label |
| `permissions` | string[] | Fine-grained permission labels |

> The `{ account, channel }` pair has a unique compound index — duplicate memberships are not valid.

### Population

| Path | Collection |
| --- | --- |
| `account` | `conjoint/accounts` |
| `channel` | `conjoint/channels` |

## `conjoint/messages`

Polymorphic messages inside a channel.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `MessageType` | `TEXT`, `FILE`, `IMAGE`, `VIDEO`, `AUDIO`, `STICKER`, `GALLERY`, `CONTACT`, `COMMAND`, `DOCUMENT`, `LOCATION`, `PULL` |
| `content` | ✅ | any | Polymorphic message body |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `account` | MongoId | Sender account |
| `channel` | MongoId | Target channel |
| `caption` | string | Optional caption |
| `mentions` | MongoId[] | Mentioned **account** IDs (not user IDs) |
| `reply_to` | MongoId | Parent message ID (raw, not populatable) |
| `hashtags` | string[] | Hashtag strings |
| `reactions` | object[] | Reaction objects |
| `scheduled_at` | date | Scheduled send timestamp |
| `forwarded_from` | MongoId | Forwarding account ID |
| `originate_from` | MongoId | Original sender account ID |

> `mentions` are `conjoint/accounts` IDs — not `identity/users` IDs.

### Population

| Path | Collection |
| --- | --- |
| `account` | `conjoint/accounts` |
| `channel` | `conjoint/channels` |
| `mentions` | `conjoint/accounts` |
| `originate_from` | `conjoint/accounts` |
| `forwarded_from` | `conjoint/accounts` |

## Query Tips

- Scope message queries by `channel` for conversation history.
- Use `created_at` descending sort for message history pagination.
- Query memberships by `channel` to list members, or by `account` to list channel memberships.
