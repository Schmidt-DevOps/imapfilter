# Feature Specification: IMAPFilter Core Application

**Feature Branch**: `feature/SDO`

**Created**: 2026-06-29

**Status**: Draft

**Input**: Reverse-engineer a sensible plan for implementing the existing IMAPFilter application. Make sure the plan would lead to the current implementation.

## Clarifications

### Session 2026-06-29

- Q: When processing multiple accounts, if some connections fail while others succeed, should IMAPFilter abort immediately or continue? → A: Continue processing remaining accounts and report all failures at end; exit code reflects worst outcome.
- Q: What are the quantitative performance targets for message processing throughput? → A: Moderate throughput (hundreds of messages/sec); memory-efficient streaming with sequence set optimization.
- Q: How should passwords be stored and protected in config.lua? → A: Plaintext acceptable; user responsible for file permissions (0600).
- Q: What is explicitly out of scope for this version of IMAPFilter? → A: SMTP, POP3, push notifications, web UI.
- Q: How should IMAPFilter handle server rate limiting and concurrent access from multiple instances? → A: No built-in rate limiting; rely on server-side limits and user scheduling via cron.

## User Scenarios & Testing

### User Story 1 - Configure and Run Mail Filtering (Priority: P1)

A user wants to filter emails across one or more mail accounts using a Lua configuration file. They write a config, run `imapfilter`, and have messages organized automatically.

**Why this priority**: This is the core value proposition — without it, IMAPFilter has no purpose.

**Independent Test**: Write a minimal config with one account, connect to a test server, select messages matching criteria, and verify actions (copy/move/delete) execute correctly.

**Acceptance Scenarios**:

1. **Given** a valid `config.lua` with one IMAP account, **When** the user runs `imapfilter`, **Then** it connects, processes mailboxes according to config rules, and exits cleanly.
2. **Given** multiple accounts in config, **When** cross-account operations are specified (e.g., copy from account1 to account2), **Then** separate connections are maintained per account and data transfers correctly.
3. **Given** 5 accounts where 2 fail to connect, **When** the user runs `imapfilter`, **Then** it processes the 3 successful accounts, reports errors for the 2 failed ones on stderr, and exits with a non-zero code indicating partial failure.

### User Story 2 - Safe Preview with Dry-Run Mode (Priority: P2)

A user wants to preview what filtering actions would be taken without actually modifying the server state.

**Why this priority**: Essential for safe configuration development; prevents accidental mass deletions.

**Independent Test**: Run `imapfilter --dry-run` and verify no write operations reach the server while informational messages are still printed.

**Acceptance Scenarios**:

1. **Given** a config with delete actions, **When** run with `-n` flag, **Then** matching messages are reported but not deleted on the server.
2. **Given** dry-run mode active, **When** multiple sequential actions would alter message counts, **Then** each action reports based on current (unchanged) state.

### User Story 3 - Robust Connection Recovery (Priority: P2)

A user runs IMAPFilter against unreliable connections and needs it to recover gracefully without manual intervention.

**Why this priority**: Network reliability is critical for a long-running mail filter; the recovery mechanism was a major v2.8.0 feature.

**Independent Test**: Simulate network interruption during execution, verify `persist` option causes reconnection and continued processing.

**Acceptance Scenarios**:

1. **Given** `options.persist = true`, **When** connection drops mid-execution, **Then** IMAPFilter reconnects and resumes from the last safe point.
2. **Given** a recovery config with timeout settings, **When** server returns transient errors, **Then** retries occur according to configuration before final failure.

### User Story 4 - SSL/TLS Secure Connections (Priority: P3)

A user connects to mail servers over encrypted channels and validates certificates.

**Why this priority**: Security is important for credentials stored in config files; SSL support was added incrementally across versions.

**Independent Test**: Connect to an IMAP server with STARTTLS, verify certificate validation using system trust store or custom certificates file.

**Acceptance Scenarios**:

1. **Given** `ssl = 'starttls'` in account config, **When** connecting, **Then** the connection upgrades via STARTTLS and validates against CA certificates.
2. **Given** a custom `-t` truststore path, **When** connecting with SSL, **Then** certificate validation uses the specified store.

### User Story 5 - Rich Search and Filter Expressions (Priority: P3)

A user writes complex filtering logic combining multiple criteria using Lua operators.

**Why this priority**: The Lua-first design enables powerful expression-based filtering that distinguishes IMAPFilter from simpler tools.

**Independent Test**: Write a config with compound search expressions (AND, OR, NOT via `*`, `+`, `-`), verify correct message selection.

**Acceptance Scenarios**:

1. **Given** `results = account.INBOX:is_unseen() * account.INBOX:contain_from('sender@example.com')`, **When** executed, **Then** only unseen messages from that sender are selected.
2. **Given** regex patterns via `match_header()` or `match_body()`, **When** PCRE2 is available, **Then** complex pattern matching works correctly.

## Edge Cases

- What happens when a mailbox name contains special characters (spaces, unicode)?
  - IMAPFilter must properly escape and handle non-ASCII mailbox names per RFC 6855.
- How does the program handle servers that don't support NAMESPACE extension?
  - Falls back to default namespace; `list_all()` still works but may miss some folders.
- What happens when config file contains syntax errors?
  - Lua interpreter reports error and exits with non-zero status; debug file captures details.
- How are binary message bodies handled during fetch operations?
  - Binary data is read into buffers without null-byte truncation; `fetch_message()` returns raw bytes.
- What happens when the server enforces rate limits (e.g., too many commands per minute)?
  - No built-in backoff or throttling; errors are logged and the program exits with non-zero status. Users manage scheduling via cron or shell scripts.

## Requirements

### Functional Requirements

- **FR-001**: System MUST connect to IMAP servers supporting IMAP4rev1 (RFC 3501) and IMAP4 (RFC 1730).
- **FR-002**: System MUST support multiple concurrent accounts with separate connections per account.
- **FR-003**: Users MUST be able to define filtering logic in a Lua configuration file (`config.lua`).
- **FR-004**: System MUST provide search methods: `select_all()`, `is_new()`, `is_unseen()`, `contain_from()`, `contain_subject()`, `contain_field()`, `match_header()`, `match_body()`.
- **FR-005**: System MUST support message actions: `delete_messages()`, `copy_messages()`, `move_messages()`, `mark_flagged()`, `mark_seen()`, `add_flags()`, `remove_flags()`.
- **FR-006**: System MUST support mailbox operations: `list_all()`, `list_subscribed()`, `create_mailbox()`, `subscribe_mailbox()`, `check_status()`, `delete_mailbox()`.
- **FR-007**: System MUST support SSL/TLS connections with STARTTLS and explicit SSL modes.
- **FR-008**: System MUST provide dry-run mode (`-n` flag) that suppresses write operations.
- **FR-009**: System MUST support PCRE2 regular expressions for pattern matching.
- **FR-010**: System MUST expose all functionality through Lua bindings accessible from config files.
- **FR-011**: System MUST support connection persistence/recovery via `options.persist`.
- **FR-012**: System MUST write PID file when `-p` flag is specified and clean up on exit.
- **FR-013**: System MUST support IPv6 connections.
- **FR-014**: System MUST provide OAuth2 authentication support (via extension).
- **FR-015**: When processing multiple accounts, if some connections fail, the system MUST continue processing remaining accounts and report all failures on stderr; exit code reflects worst outcome.
- **FR-016**: Passwords stored in `config.lua` are plaintext; users are responsible for file permissions (recommended 0600). No built-in encryption or credential manager integration.

### Key Entities

- **Account**: Represents an IMAP server connection with server, port, username, password, SSL mode, timeout settings. Created via `IMAP{}` Lua table constructor.
- **Mailbox**: A named collection of messages within an account. Accessed via dot notation (`account.INBOX`) or bracket notation (`account['folder/subfolder']`).
- **Results Set**: An ordered list of message identifiers returned by search methods. Supports set operations (union `+`, intersection `*`, difference `-`) and action methods (`delete_messages()`, etc.).
- **Configuration Options**: Global settings including timeout, subscribe default, persist mode, truststore path. Accessed via `options` Lua table.

## Success Criteria

### Measurable Outcomes

- **SC-001**: A user can configure filtering for 3+ accounts with cross-account operations in under 15 minutes using the sample config as starting point.
- **SC-002**: IMAPFilter handles connection drops and recovers within 30 seconds when `persist` is enabled.
- **SC-003**: SSL certificate validation correctly rejects expired/invalid certificates and accepts valid ones from system trust store.
- **SC-004**: Dry-run mode produces identical informational output to normal mode for read operations, with zero write commands sent to server.
- **SC-005**: Message processing achieves moderate throughput (hundreds of messages/sec) using sequence set range optimization; memory usage remains bounded regardless of mailbox size.

## Out of Scope

The following are explicitly NOT part of this version:

- **SMTP**: No outgoing mail support; IMAPFilter is read-only from the server perspective.
- **POP3**: Only IMAP protocol is supported.
- **Push notifications**: No push-based delivery; user must run `imapfilter` to process mail.
- **Web UI / API**: CLI-only application; no HTTP server or REST interface.
- **Daemon mode**: Runs as a batch tool; background execution is the user's responsibility (shell, cron).

## Assumptions

- Users have basic familiarity with Lua syntax (tables, functions, operators).
- Mail servers support at least IMAP4rev1 baseline capabilities.
- System has OpenSSL, PCRE2, and Lua development libraries installed for building.
- Configuration files are stored in `$HOME/.imapfilter/` by default; users may override via `IMAPFILTER_HOME`.
- The program runs as a batch tool (not daemon), though it supports background execution via shell.
