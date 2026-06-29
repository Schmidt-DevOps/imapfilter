# Contract: Lua API

## Account Methods

Exposed via `account_name` (the result of `IMAP{...}` constructor).

### Connection Management
| Method | Signature | Description |
|--------|-----------|-------------|
| *(constructor)* | `IMAP{server, port, username, password, ssl, timeout}` | Create account connection object |

### Mailbox Discovery
| Method | Signature | Description |
|--------|-----------|-------------|
| `list_all()` | `account:list_all()` | Return (mailboxes, folders) — all available mailboxes |
| `list_subscribed()` | `account:list_subscribed()` | Return (mailboxes, folders) — only subscribed mailboxes |

### Mailbox Operations
| Method | Signature | Description |
|--------|-----------|-------------|
| `create_mailbox(name)` | `account:create_mailbox("Name")` | Create a new mailbox |
| `subscribe_mailbox(name)` | `account:subscribe_mailbox("Name")` | Subscribe to a mailbox |
| `delete_mailbox(name)` | `account:delete_mailbox("Name")` | Delete a mailbox |

### Mailbox Access (via dot/bracket notation)
```lua
-- Dot notation for simple names
account.INBOX:select_all()

-- Bracket notation for hierarchical names  
account['Archive/2024']:select_all()
```

## Mailbox Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `check_status()` | `mailbox:check_status()` | Return (messages, recent, unseen) counts |
| `select_all()` | `mailbox:select_all()` | Select all messages → Results Set |
| `is_new()` | `mailbox:is_new()` | Select new/unseen-arrival messages |
| `is_unseen()` | `mailbox:is_unseen()` | Select unseen messages |
| `is_recent()` | `mailbox:is_recent()` | Select recent messages |
| `contain_from(addr)` | `mailbox:contain_from("addr")` | Match From header |
| `contain_to(addr)` | `mailbox:contain_to("addr")` | Match To header |
| `contain_subject(subj)` | `mailbox:contain_subject("subj")` | Match Subject header |
| `contain_field(field, value)` | `mailbox:contain_field("field", "value")` | Match arbitrary header field |
| `is_older(days)` | `mailbox:is_older(30)` | Select messages older than N days |
| `is_smaller(bytes)` | `mailbox:is_smaller(50000)` | Select messages smaller than N bytes |
| `is_larger(bytes)` | `mailbox:is_larger(1000)` | Select messages larger than N bytes |
| `match_header(field, pattern)` | `mailbox:match_header("From", "pattern")` | Regex match on header field |
| `match_body(pattern)` | `mailbox:match_body("pattern")` | Regex match on message body |

## Results Set Methods (Set Operations)

| Operator | Method Equivalent | Description |
|----------|-------------------|-------------|
| `+` | `results1 + results2` | Union of two result sets |
| `*` | `results1 * results2` | Intersection of two result sets |
| `-` | `results1 - results2` | Difference (remove from first) |

## Results Set Methods (Actions)

| Method | Signature | Description |
|--------|-----------|-------------|
| `delete_messages()` | `results:delete_messages()` | Mark messages for deletion |
| `copy_messages(target)` | `results:copy_messages(account.mailbox)` | Copy to target mailbox |
| `move_messages(target)` | `results:move_messages(account.mailbox)` | Move to target mailbox |
| `mark_flagged()` | `results:mark_flagged()` | Set \Flagged flag |
| `mark_seen()` | `results:mark_seen()` | Set \Seen flag |
| `add_flags(flags)` | `results:add_flags({"\\Custom"})` | Add custom flags |
| `remove_flags(flags)` | `results:remove_flags({"\\Custom"})` | Remove custom flags |

## Message Fetch Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `fetch_message()` | `mailbox:fetch_message(uid)` | Return raw message content |
| `fetch_structure()` | `mailbox:fetch_structure(uid)` | Return BODYSTRUCTURE details |
| `fetch_header()` | `mailbox:fetch_header(uid)` | Return parsed headers |

## IDLE Support

| Method | Signature | Description |
|--------|-----------|-------------|
| `enter_idle()` | `account.mailbox:enter_idle()` | Enter IMAP IDLE mode; blocks until server sends update or signal received |
