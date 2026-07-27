# 04 — Data Model (conceptual)

Entities and relationships, independent of database choice. Field lists are
illustrative, not final.

## Core entities

### Account
The connection to a mailbox.
- `id`, `provider` (imap | jmap | gmail), `address`, `display_name`
- `auth_ref` — pointer to credentials in secure storage (never the secret itself)
- `sync_state` — cursor/UID/history-id for incremental sync

### Message
A single email.
- `id` (internal), `provider_id`, `account_id`, `thread_id`
- `from`, `to`, `cc`, `bcc`, `subject`, `date`
- `body_text`, `body_html`, `snippet`
- `headers` (raw, for threading/debug), `attachments[]`
- `flags` (read, starred, …), `folder`/`labels`

### Thread
A conversation grouping messages.
- `id`, `account_id`, `subject`, `message_ids[]`
- `participants[]`, `last_activity_at`

### Attachment
- `id`, `message_id`, `filename`, `mime_type`, `size`, `local_path`

## AI-derived entities

These are *computed* artifacts, stored so they can be reused and don't need
recomputation. They are never sources of truth about the mail itself.

### Triage
Per-thread classification result.
- `thread_id`, `priority` (now | later | fyi | noise), `category`
- `summary_line`, `model`, `computed_at`, `confidence`
- `user_override` — set when the user corrects the triage (feeds the learning loop)

### Summary
- `thread_id`, `text`, `source_message_ids[]` (provenance), `model`, `computed_at`

### Draft
- `id`, `thread_id`, `intent` (optional user instruction), `text`
- `tone`, `model`, `created_at`, `status` (suggested | edited | sent | discarded)

### Embedding / Index entry
- `object_type` (message | thread), `object_id`
- `vector`, `model`, `chunk_ref` (if chunked), `computed_at`

## Relationships

```
Account 1───* Thread 1───* Message 1───* Attachment
                 │              │
                 │              └──* Embedding
                 ├──1 Triage
                 ├──* Summary
                 └──* Draft
```

## Storage implications

- **Canonical mail** (Account/Message/Thread/Attachment) is the source of truth;
  it must be durable and crash-safe.
- **AI-derived data** is a cache: it can be regenerated from canonical mail +
  a model, so it can be versioned by `model` and invalidated when models change.
- **Vectors** live in a vector index; keep a link back to the canonical object.
- **Secrets** (`auth_ref`) point into OS-level secure storage — the DB stores a
  reference, never the credential.

## Open modeling questions

- Threading algorithm: trust provider threading vs. compute our own from headers?
- Chunking strategy for long messages before embedding.
- How to represent "action items"/deadlines if we add them (own entity vs. field).
- Retention: do AI-derived artifacts expire? (See privacy doc.)
