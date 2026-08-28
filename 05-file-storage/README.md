
# File Storage Service

## 1. Problem

Design a file storage backend similar to Dropbox/Google Drive.

The system supports folders, large file uploads, versioning, sharing, permissions, and public links.

---

## 2. Requirements

### Functional

- Per-user storage with folders/subfolders
- Files up to 5GB
- Per-user storage quota
- Last 10 file versions
- User-to-user sharing with view/edit permissions
- Revocable public links
- Folder-level permissions with inheritance

### Non-Functional

- Resumable large-file uploads
- Durable storage
- Fast permission checks despite deep folder nesting
- Immediate permission/link revocation

---

## 3. Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Quota validation | Before chunk upload | Avoids wasting bandwidth |
| Large uploads | Client-side chunking + parallel upload | Resumability and performance |
| Chunk ordering | Sequence number | Reconstruct correct order |
| Chunk integrity | Per-chunk hash | Detects corruption |
| Resume | Upload session tracks received chunks | Only missing chunks are retried |
| Durability | Primary upload + asynchronous DR copy | Balances latency and durability |
| DR failure | Reconciliation job | Prevents silent durability failures |
| Versioning | Block-level hashing/deduplication | Efficient for binary files |
| Permissions | Folder inheritance + explicit override | Models hierarchical access |
| Permission performance | Cache permission decision | Avoids repeated tree traversal |
| Permission invalidation | Active invalidation + TTL fallback | Revocation must be immediate |
| Copied files | New independent object | Breaks original inheritance |
| Public links | Revoked flag + immediate cache deletion | Immediate revocation |

---

## 4. Architecture

```mermaid
flowchart TD
    User[User / Client]

    API[File API]
    Metadata[(Metadata DB)]
    Permission[(Permission Store)]
    Cache[(Redis<br/>Permission Cache)]

    Upload[Upload Session]
    Object[(Primary Object Storage)]
    DR[(DR Object Storage)]

    Queue[Durability Queue]
    Reconcile[Reconciliation Job]

    User --> API

    API --> Metadata
    API --> Permission
    API --> Cache

    API --> Upload
    Upload -->|Parallel Chunks| Object

    Object -->|Async Replication| Queue
    Queue --> DR

    Reconcile --> Metadata
    Reconcile -->|Retry Failed DR Copies| DR

    API -->|Permission Change| Cache
    API -->|Revocation| Cache
````

---

## 5. Critical Flow — Large File Upload

```text
Client
  ↓
Create upload session
  ↓
Validate quota + file size
  ↓
Split file into chunks
  ↓
Upload chunks in parallel
  ↓
Track received chunks
  ↓
Retry only missing chunks
  ↓
Complete upload
```

Sequence numbers determine **ordering**.

Hashes determine **integrity**.

These are separate concerns.

---

## 6. Critical Flow — Durability

```text
Primary Object Storage
        ↓
   User sees success
        ↓
Async DR copy
        ↓
durability_status = CONFIRMED
```

If the DR copy fails:

```text
Pending file
    ↓
Reconciliation Job
    ↓
Retry / Alert
```

---

## 7. Permission Model

Permissions are inherited:

```text
Folder
 ├── Subfolder
 │    └── File
 └── File
```

An explicit file-level permission overrides an inherited folder permission.

Permission decisions are cached using:

```text
(user_id, file_or_folder_id)
```

When permissions change, the relevant cache entries are actively invalidated.

---

## 8. Mistakes & Corrections

1. Quota checked after upload started → validate before transfer
2. Chunk tracking not defined → explicit upload sessions
3. Sequence and hash treated as one concept → separate ordering and integrity
4. DR upload treated as fire-and-forget → reconciliation
5. File content cached for permission performance → cache permission decisions
6. TTL assumed sufficient for revocation → active invalidation
7. File/folder permission conflict unresolved → explicit grant takes precedence
8. Binary versioning considered using line diffs → block-level hashing

---

## 9. Core Takeaways

* Validate constraints before expensive operations.
* Ordering and integrity are different concerns.
* Durability vs latency is a conscious architectural trade-off.
* Cache the decision when computation is expensive, not necessarily the underlying data.
* Security-sensitive caches require active invalidation.
* Specific rules should override inherited/general rules.
* Binary data requires binary-aware techniques.

---

## 10. Patterns Learned

* Chunked uploads
* Resume-by-missing-chunks
* Validate-before-expensive-operation
* Block-level deduplication
* Asynchronous durability
* Reconciliation
* Hierarchical permission inheritance
* Active cache invalidation

