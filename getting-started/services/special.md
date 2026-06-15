# Special

**Port:** REST `:3090` · gRPC `:5090`

Handles file metadata and storage management (via MinIO) and time-dimensioned statistical counters.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Files | `/special/files` | File metadata + MinIO object reference |
| Stats | `/special/stats` | Time-dimensioned counters and metrics |

## `special/files`

Stores file metadata — filename, MIME type, size, storage coordinates, and download URL. The binary content itself is stored in MinIO.

### File Upload (Recommended Path)

Use the upload endpoints for new binary files — **not** the raw CRUD create:

| Endpoint | Scope | Description |
| --- | --- | --- |
| `POST /special/files/upload/private` | `upload:special:files` | Upload a private file |
| `POST /special/files/upload/public` | `upload:special:files` | Upload a public file |

Use raw `create` only when object-storage metadata already exists from a pre-signed or external upload flow.

### Required Create Fields (raw create only)

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `original` | ✅ | string | Original filename |
| `mimetype` | ✅ | string | MIME type |
| `size` | ✅ | number | File size in bytes |
| `bucket` | ✅ | string | Object-storage bucket |
| `key` | ✅ | string | Object-storage key/path |
| `acl` | ✅ | string | Storage access level |
| `location` | ✅ | string | File download URL |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `title` | string | Display title |
| `field` | string | Multipart field name |
| `content_type` | string | Download content-type override |
| `storage_class` | string | Storage class |
| `etag` | string | Storage ETag |
| `state` | `State` | Lifecycle state |

### Special Operations (platform REST only)

| Operation | HTTP | Scope | Description |
| --- | --- | --- | --- |
| Share file | `POST /special/files/:id/share` | `share:special:files` | Generate a share link |
| Download file | `GET /special/files/:id/download` | `download:special:files` | Stream file contents |

## `special/stats`

Time-dimensioned counter and metric storage.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `StatType` | `DAILY`, `MONTHLY`, `YEARLY` |
| `key` | ✅ | `StatKey` | Stat identifier |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `obj` | MongoId | Related document |
| `flag` | string | Secondary key |
| `day` | number | Day dimension |
| `month` | number | Month dimension |
| `year` | number | Year dimension |
| `hours` | number[] | Exactly 24 elements |
| `days` | number[] | Exactly 31 elements |
| `months` | number[] | Exactly 12 elements |

> `hours`, `days`, and `months` must always be provided with the exact required element count. Partial arrays are rejected.

### Special Operations (platform REST only)

| Operation | HTTP | Scope | Description |
| --- | --- | --- | --- |
| Collect | `POST /special/stats/collect` | `collect:special:stats` | Upsert-style stat increment |
| Stackup | `POST /special/stats/stackup` | `collect:special:stats` | Aggregate / accumulate stats |
