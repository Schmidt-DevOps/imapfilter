# Quickstart Validation Guide

## Prerequisites

- IMAPFilter binary built and installed (`make all && make install`)
- Access to at least one IMAP mail server (test account recommended)
- Lua 5.1+ development libraries, PCRE2, OpenSSL installed on the build system

## Validation Scenario 1: Basic Connection and Mailbox Listing

**Goal**: Verify IMAPFilter can connect to a server and list mailboxes.

### Setup
Create `test-config.lua`:
```lua
-- Options
options.timeout = 30

-- Account
test_account = IMAP {
    server = 'imap.example.com',
    username = 'test@example.com',
    password = 'testpassword',
}

-- List all mailboxes
mailboxes, folders = test_account:list_all()
for _, mb in ipairs(mailboxes) do
    print('Mailbox: ' .. mb)
end
```

### Run
```bash
imapfilter -c test-config.lua -v
```

### Expected Outcome
- Connection established to `imap.example.com`
- Mailbox names printed to stdout (e.g., "INBOX", "Sent", etc.)
- Clean exit with code 0
- Debug output in stderr shows IMAP CAPABILITY, LIST commands and responses

---

## Validation Scenario 2: Message Search and Filtering

**Goal**: Verify search methods correctly identify messages.

### Setup
Create `test-filter.lua`:
```lua
options.timeout = 30

test_account = IMAP {
    server = 'imap.example.com',
    username = 'test@example.com',
    password = 'testpassword',
}

-- Select all unseen messages from a specific sender
results = test_account.INBOX:is_unseen() *
          test_account.INBOX:contain_from('newsletter@example.com')

print('Found ' .. #results .. ' unseen messages from newsletter@example.com')
for _, uid in ipairs(results) do
    print('  UID: ' .. uid)
end
```

### Run
```bash
imapfilter -c test-filter.lua -v
```

### Expected Outcome
- Correct number of matching messages reported
- UIDs printed for each match
- No write operations sent to server (only SEARCH commands)

---

## Validation Scenario 3: Dry-Run Mode

**Goal**: Verify dry-run suppresses writes while reporting actions.

### Setup
Create `test-dryrun.lua`:
```lua
options.timeout = 30

test_account = IMAP {
    server = 'imap.example.com',
    username = 'test@example.com',
    password = 'testpassword',
}

-- Select and delete old messages
results = test_account.INBOX:is_older(90)
print('Found ' .. #results .. ' messages older than 90 days')
results:delete_messages()
```

### Run (dry-run)
```bash
imapfilter -c test-dryrun.lua -n -v
```

### Expected Outcome
- Messages reported as found and marked for deletion
- No `STORE \Deleted` or `EXPUNGE` commands in verbose output
- Exit code 0

### Run (actual)
```bash
imapfilter -c test-dryrun.lua -v
```

### Expected Outcome
- Same messages reported as found
- `STORE \Deleted` and/or `EXPUNGE` commands visible in verbose output
- Messages actually deleted on server

---

## Validation Scenario 4: SSL/TLS Connection

**Goal**: Verify SSL connection and certificate validation.

### Setup
Create `test-ssl.lua`:
```lua
options.timeout = 30

test_account = IMAP {
    server = 'imap.example.com',
    username = 'test@example.com',
    password = 'testpassword',
    ssl = 'starttls',
}

status = test_account.INBOX:check_status()
print('Messages: ' .. status[1] .. ', Recent: ' .. status[2] .. ', Unseen: ' .. status[3])
```

### Run
```bash
imapfilter -c test-ssl.lua -v
```

### Expected Outcome
- STARTTLS negotiation succeeds
- Certificate validated against system trust store (`/etc/ssl/certs/`)
- Mailbox status printed to stdout
- If certificate is invalid, connection fails with error on stderr

---

## Validation Scenario 5: Cross-Account Copy

**Goal**: Verify cross-account operations maintain separate connections.

### Setup
Create `test-cross.lua`:
```lua
options.timeout = 30

account1 = IMAP {
    server = 'imap.example.com',
    username = 'user1@example.com',
    password = 'pass1',
}

account2 = IMAP {
    server = 'imap.other.com',
    username = 'user2@other.com',
    password = 'pass2',
}

-- Copy unseen messages from account1 to account2
results = account1.INBOX:is_unseen()
print('Copying ' .. #results .. ' messages')
results:copy_messages(account2.Archive)
```

### Run
```bash
imapfilter -c test-cross.lua -v
```

### Expected Outcome
- Two separate IMAP connections established (one per account)
- Messages copied from `account1.INBOX` to `account2.Archive`
- Both accounts' capabilities negotiated independently
- Clean exit with code 0
