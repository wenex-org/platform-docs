# Content

**Port:** REST `:3110` · gRPC `:5110`

Rich content management for user-facing documents: personal/shared notes, published posts and articles, and support tickets.

> Comments on posts and tickets live in `general/comments`, not in this service.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Notes | `/content/notes` | Personal or shared notes and note threads |
| Posts | `/content/posts` | Published articles and content records |
| Tickets | `/content/tickets` | Support, issue, and task-like records |

## `content/notes`

Personal or shared notes. Notes can be threaded and can reference a post or ticket through `relation`.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `NoteType` | `NOTE`, `REVIEW`, `COMMENT`, `FEEDBACK` |
| `content` | ✅ | string | Note body |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `parent` | MongoId | Parent note for threading |
| `level` | number | Thread nesting depth |
| `relation` | MongoId | Related `content/posts` or `content/tickets` ID |
| `visibility` | string | Visibility marker |
| `mentions` | MongoId[] | Raw user IDs — not populatable |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `reactions` | object[] | Reaction objects |

### Population

| Path | Collection |
| --- | --- |
| `relation` | `content/posts` or `content/tickets` |

## `content/posts`

Published content with a lifecycle centered on post status.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `title` | ✅ | string | Post title |
| `content` | ✅ | string | Post body |
| `status` | ✅ | `PostStatus` | `DRAFT`, `ARCHIVED`, `PUBLISHED` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `slug` | string | URL-friendly identifier |
| `subtitle` | string | Subtitle |
| `summary` | string | Excerpt |
| `parent` | MongoId | Parent post |
| `categories` | string[] | Category values |
| `keywords` | string[] | SEO keywords |
| `thumbnail` | MongoId | Raw `special/files` ID — not populatable |
| `featured_image` | MongoId | Raw `special/files` ID — not populatable |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `related_posts` | MongoId[] | Raw related post IDs — not populatable |
| `publication_date` | date | Scheduled or actual publication date |

### Special Operation

`content/posts` exposes a service-local `search` operation in addition to standard CRUD:

```
POST /content/posts/search
```

Use `search` when full-text search behavior is needed rather than standard filtering.

## `content/tickets`

Support, issue, and task-like records with priority, lifecycle status, and optional resolution feedback.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `title` | ✅ | string | Ticket title |
| `content` | ✅ | string | Ticket body |
| `status` | ✅ | `TicketStatus` | `OPEN`, `CLOSED`, `RESOLVED`, `IN_PROGRESS` |
| `priority` | ✅ | `Priority` | `LOW`, `MEDIUM`, `HIGH`, `URGENT` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `parent` | MongoId | Parent ticket |
| `department` | string | Routing department |
| `due_date` | date | Due date |
| `assigned_to` | MongoId | Assigned user ID (raw — not populatable) |
| `solution` | string | Resolution text |
| `feedback` | MongoId | `content/notes` ID used as formal resolution note |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `related_tickets` | MongoId[] | Raw related ticket IDs — not populatable |

### Population

| Path | Collection |
| --- | --- |
| `feedback` | `content/notes` |

## Query Tips

- Filter posts by `status` for publication lifecycle views.
- Filter tickets by `status` and `priority` for support queues.
- Use `relation` on notes to fetch all notes attached to a specific post or ticket.
- For comments on posts or tickets, query `general/comments` with the post or ticket ID.
