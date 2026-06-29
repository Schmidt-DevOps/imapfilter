# Data Model: IMAPFilter Core

## Entity: Account

Represents an IMAP server connection configuration.

| Field | Type | Description |
|-------|------|-------------|
| `server` | string | Hostname or IP of the IMAP server |
| `port` | integer (optional) | Port number; defaults to 143 (IMAP) or 993 (IMAPS) based on SSL mode |
| `username` | string | Login username for authentication |
| `password` | string | Password for authentication (stored in config file) |
| `ssl` | enum | Connection security: `"none"`, `"starttls"`, `"sslv23"`, `"sslv3"`, `"tlsv1"`, `"tls"` |
| `timeout` | integer (optional) | Connection timeout in seconds; default 120 |
| `persist` | boolean (optional) | Whether to attempt indefinite reconnection on failure |
| `hostnames` | table (optional) | List of hostnames for multi-server setups |

**Validation Rules**:
- `server` MUST be non-empty
- `username` and `password` MUST be present for basic auth
- `ssl` values are validated against supported modes at connection time

**Relationships**:
- One Account → Many Mailboxes (discovered via `list_all()`)
- One Account → Zero or More Results Sets (from search operations)

---

## Entity: Mailbox

Represents a named collection of messages within an account.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Full mailbox path (e.g., "INBOX", "Archive/2024") |
| `delimiter` | char (optional) | Hierarchical delimiter from NAMESPACE extension |
| `flags` | set (read-only) | IMAP flags for the mailbox (\Seen, \Answered, etc.) |
| `uid_validity` | integer (read-only) | UID validity value from server |

**Validation Rules**:
- Mailbox names MAY contain spaces and Unicode characters (RFC 6855)
- Hierarchical names use the delimiter returned by NAMESPACE extension

**Relationships**:
- One Mailbox → Many Messages (identified by UID or sequence number)
- Mailbox is a child of exactly one Account

---

## Entity: Results Set

An ordered collection of message identifiers returned by search operations.

| Field | Type | Description |
|-------|------|-------------|
| `messages` | list of integers | Message UIDs or sequence numbers |
| `account` | Account reference | The account this result set belongs to |
| `mailbox` | Mailbox reference | The mailbox that was searched |

**Set Operations** (via Lua metatable):
- **Union (`+`)**: Combine two result sets, deduplicate UIDs
- **Intersection (`*`)**: Keep only UIDs present in both sets
- **Difference (`-`)**: Remove UIDs from first set that appear in second

**Action Methods**:
- `delete_messages()` — Mark messages for deletion on server
- `copy_messages(target)` — Copy to target mailbox (same or different account)
- `move_messages(target)` — Copy then delete (cross-account move)
- `mark_flagged()` / `mark_seen()` — Set standard flags
- `add_flags(flags)` / `remove_flags(flags)` — Add/remove custom flags
- `delete_mailbox()` — Delete the mailbox this set belongs to

**Validation Rules**:
- UIDs MUST be valid for the target mailbox's UID validity
- Cross-account operations require both accounts to be connected successfully

---

## Entity: Configuration Options

Global settings that affect program behavior.

| Field | Type | Description |
|-------|------|-------------|
| `timeout` | integer | Connection timeout in seconds (default 120) |
| `subscribe` | boolean | Whether to auto-subscribe to mailboxes on creation (default true) |
| `persist` | boolean | Enable persistent connection recovery (default false) |

**Validation Rules**:
- `timeout` MUST be a positive integer
- Boolean options default to documented values if not specified

---

## Entity: Message Metadata

Information about individual messages fetched from the server.

| Field | Type | Description |
|-------|------|-------------|
| `uid` | integer | Unique identifier for the message |
| `size` | integer | Message size in octets |
| `flags` | set | IMAP flags (\Seen, \Flagged, etc.) |
| `headers` | table | Parsed email headers (From, To, Subject, Date, etc.) |
| `body` | string/binary | Raw message body content |

**Validation Rules**:
- Body MAY contain binary data (null bytes, non-printable characters)
- Headers are parsed from the raw IMAP BODYSTRUCTURE response
