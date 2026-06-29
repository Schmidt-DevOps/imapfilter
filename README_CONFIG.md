# IMAPFilter Configuration Guide

A comprehensive guide to using [IMAPFilter](https://github.com/lefcha/imapfilter), a powerful mail filtering utility that connects to remote mail servers via IMAP and processes messages based on customizable criteria.

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Installation & Configuration File](#installation--configuration-file)
3. [Command-Line Options](#command-line-options)
4. [Configuration Structure](#configuration-structure)
5. [Options](#options)
6. [Accounts](#accounts)
7. [Mailboxes](#mailboxes)
8. [Searching Messages](#searching-messages)
9. [Logical Operators & Meta-Searching](#logical-operators--meta-searching)
10. [Processing Results](#processing-results)
11. [Fetching Message Parts](#fetching-message-parts)
12. [Appending Messages](#appending-messages)
13. [Auxiliary Functions](#auxiliary-functions)
14. [Advanced Patterns](#advanced-patterns)
15. [Daemon Mode & IDLE](#daemon-mode--idle)
16. [Error Recovery](#error-recovery)
17. [OAuth2 Authentication](#oauth2-authentication)
18. [External Tool Integration](#external-tool-integration)

---

## Quick Start

```lua
-- 1. Set global options
options.timeout = 120

-- 2. Define an account
account = IMAP {
    server = 'imap.example.com',
    username = 'user@example.com',
    password = 'secret',
    ssl = 'auto'
}

-- 3. Search for unseen messages in INBOX
results = account.INBOX:is_unseen()

-- 4. Process the results (e.g., mark as read)
results:mark_seen()
```

---

## Installation & Configuration File

The default configuration file is located at `$HOME/.imapfilter/config.lua`. You can specify a custom path with `-c`:

```bash
imapfilter -c /path/to/myconfig.lua
```

Read from stdin:

```bash
cat config.lua | imapfilter -c -
```

---

## Command-Line Options

| Option | Description |
|--------|-------------|
| `-c configfile` | Path to configuration file (default: `$HOME/.imapfilter/config.lua`). Use `-` for stdin. |
| `-d debugfile` | File for detailed debug/communication logs. |
| `-e 'command'` | Execute a single Lua command inline (no config file loaded). |
| `-i` | Enter interactive mode after executing the configuration. |
| `-l logfile` | File for error log messages. |
| `-n` | Dry-run mode — actions are reported but not sent to the server. |
| `-t truststore` | Path to SSL CA trust store directory or file (default: `/etc/ssl/certs/`). |
| `-p pidfile` | Write process ID to this file on startup. |
| `-V` | Display version and copyright. |
| `-v` | Enable verbose output showing brief communication details. |

---

## Configuration Structure

A typical configuration has three sections:

```lua
-- 1. Global options
options.timeout = 120
options.namespace = true

-- 2. Account definitions
account1 = IMAP { server = '...', username = '...', password = '...' }
account2 = IMAP { server = '...', username = '...', oauth2 = '...' }

-- 3. Filtering logic
results = account1.INBOX:is_unseen() * account1.INBOX:contain_from('newsletter@example.com')
results:copy_messages(account2.archive)
```

---

## Options

Global settings are stored in the `options` table:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `cache` | boolean | `true` | Cache message parts locally during a session. |
| `certificates` | boolean | `true` | Accept and store server certificates for future validation. |
| `charset` | string | ASCII | Character set for search strings sent to the server. |
| `close` | boolean | `false` | Implicitly close selected mailbox after each operation (removes deleted messages). |
| `create` | boolean | `false` | Auto-create mailboxes when writing to non-existent ones. |
| `expunge` | boolean | `true` | Immediately expunge messages marked for deletion. |
| `hostnames` | boolean | `true` | Validate server hostname against the certificate. |
| `info` | boolean | `true` | Print a summary of actions while processing mailboxes. |
| `keepalive` | number | `29` | Minutes before re-issuing IDLE to keep connection alive. |
| `limit` | number | `0` | Break long requests into smaller batches (e.g., 50 messages at a time). |
| `namespace` | boolean | `true` | Auto-apply namespace prefix and delimiter to mailbox names. Use `/` as delimiter, empty string as prefix. Disable if manually specifying server-side mailbox names. |
| `range` | number | `0` | Limit message UID range in operations (e.g., 50). |
| `starttls` | boolean | `true` | Negotiate TLS connection via STARTTLS extension. |
| `subscribe` | boolean | `false` | Auto-subscribe newly created mailboxes. |
| `timeout` | number | `60` | Seconds to wait for server response (0 = indefinite). |
| `wakeonany` | boolean | `false` | IDLE returns on any event (not just RECENT/EXISTS), including FETCH and EXPUNGE. |

---

## Accounts

Accounts are initialized with the `IMAP()` function:

```lua
myaccount = IMAP {
    server   = 'imap.mail.server',   -- required
    username = 'me',                  -- required
    password = 'secret',              -- optional (prompted interactively if omitted)
    port     = 993,                   -- default: 143 (imap) or 993 (imaps)
    ssl      = 'auto',                -- 'auto' | 'tls1.2' | 'tls1.1' | 'tls1' | 'ssl3'
    oauth2   = 'base64encodedstring' -- optional: OAuth2 string for XOAUTH2 auth
}
```

### Listing Mailboxes

```lua
-- List all mailboxes/folders under a folder
mailboxes, folders = myaccount:list_all('')
mailboxes, folders = myaccount:list_all('myfolder', '*')  -- wildcard in mailbox name

-- List subscribed mailboxes/folders
mailboxes, folders = myaccount:list_subscribed('')
```

### Manipulating Mailboxes

```lua
myaccount:create_mailbox('archive')
myaccount:subscribe_mailbox('archive')
myaccount:unsubscribe_mailbox('old_folder/old_mailbox')
myaccount:rename_mailbox('old_name', 'new_name')
myaccount:delete_mailbox('temporary')
```

### Accessing Mailboxes

```lua
-- Simple names (letters, digits, underscores)
myaccount.INBOX
myaccount.sent

-- Complex names or folder paths
myaccount['INBOX.Trash']
myaccount['folder/subfolder/mailbox']
```

---

## Searching Messages

All search methods return a special table (a metatable implementing sets) containing matching message references.

### State-Based Searches

| Method | Description |
|--------|-------------|
| `:select_all()` | All messages in the mailbox. |
| `:is_answered()` | Messages that have been answered. |
| `:is_deleted()` | Messages marked for removal. |
| `:is_draft()` | Incomplete draft messages. |
| `:is_flagged()` | Flagged for urgent/special attention. |
| `:is_new()` | Recently arrived, unread (first session notified). |
| `:is_old()` | Not recently arrived, unread. |
| `:is_recent()` | Recently arrived (first session notified). |
| `:is_seen()` | Messages that have been read. |
| `:is_unanswered()` | Messages not yet answered. |
| `:is_undeleted()` | Not marked for removal. |
| `:is_undraft()` | Completed drafts. |
| `:is_unflagged()` | Not flagged. |
| `:is_unseen()` | Unread messages. |

### Size & Age Searches

```lua
account.INBOX:is_larger(100000)   -- larger than 100KB
account.INBOX:is_smaller(5000)    -- smaller than 5KB
account.INBOX:is_older(30)        -- older than 30 days
account.INBOX:is_newer(7)         -- newer than 7 days
```

### Date Searches (format: `day-month-year`, e.g., `'01-Jan-2007'`)

```lua
account.INBOX:arrived_before('01-Jan-2007')
account.INBOX:arrived_on('15-Mar-2024')
account.INBOX:arrived_since('01-Jan-2024')
account.INBOX:sent_before('01-Jun-2023')
account.INBOX:sent_on('25-Dec-2023')
account.INBOX:sent_since('01-Jan-2024')
```

### Keyword Searches (case-insensitive)

```lua
account.INBOX:contain_from('boss@company.com')
account.INBOX:contain_to('team@group.org')
account.INBOX:contain_cc('cc@example.com')
account.INBOX:contain_bcc('bcc@example.com')
account.INBOX:contain_subject('invoice')
account.INBOX:contain_field('Sender', 'noreply@service.com')
account.INBOX:contain_body('hello world')
account.INBOX:contain_message('entire message text')
```

### Regex Searches (case-sensitive, PCRE-based)

These download relevant parts locally — use with meta-searching to minimize downloads.

```lua
account.INBOX:match_from('.*(user1|user2)@host')
account.INBOX:match_subject('^Re:.*FW:.*')
account.INBOX:match_body('.*total.*amount.*\\d+\\.\\d+')
account.INBOX:match_header('^.+X-Spam-Status: Yes$')
account.INBOX:match_field('X-Custom', 'value123')
account.INBOX:match_bcc('hidden@recipient.com')
account.INBOX:match_cc('cc@list.example.com')
account.INBOX:match_to('to@example.com')
```

> **Note:** Lua uses `\` as an escape character. Use `\\` for a literal backslash in patterns (e.g., `'\\\\d+'`).

### Custom IMAP Queries

```lua
-- Full IMAP SEARCH criteria (RFC 3501 Section 6.4.4)
account.INBOX:send_query('UNSEEN SINCE 01-Jan-2024')
```

---

## Logical Operators & Meta-Searching

### Set Operations

| Operator | Logic | Description |
|----------|-------|-------------|
| `+` | OR | Union of result sets. |
| `*` | AND | Intersection of result sets. |
| `-` | NOT | Difference (remove matches). |

```lua
-- Unseen messages that are larger than 100KB OR contain "invoice" in subject
results = account.INBOX:is_unseen() +
          account.INBOX:contain_subject('invoice')

-- Unseen AND flagged messages with "urgent" in the body
results = account.INBOX:is_unseen() *
          account.INBOX:is_flagged() *
          account.INBOX:contain_body('urgent')

-- Unseen messages that do NOT contain "promo" in the subject
results = account.INBOX:is_unseen() -
          account.INBOX:contain_subject('promo')

-- Precedence: AND binds tighter than OR/NOT; use parentheses to override
results = (account.INBOX:is_unseen() +
           account.INBOX:is_larger(100000)) *
           account.INBOX:contain_subject('test')
```

### Meta-Searching

Apply further searches on existing result sets:

```lua
unseen = account.INBOX:is_unseen()
spammy = unseen:match_header('^.+X-Spam: Yes$')
spammy:delete_messages()

-- Chained search
account.INBOX:is_new():match_body('^Hello world!?$'):delete_messages()
```

### Composite Filters as Functions

```lua
myfilter = function ()
    return account.INBOX:is_unseen() +
           account.INBOX:is_larger(100000) *
           account.INBOX:contain_subject('test')
end

results = myfilter()

-- With arguments for dynamic filtering
myfilter = function (mailbox, min_size, subject_pattern)
    return mailbox:is_unseen() +
           mailbox:is_larger(min_size) *
           mailbox:contain_subject(subject_pattern)
end

results = myfilter(account.INBOX, 50000, 'report')
```

### Combining Multiple Mailboxes/Accounts

```lua
-- Same account, different mailboxes
results = account.newsletter:is_unseen() +
          account.security:is_old()

-- Different accounts
results = account1.INBOX:is_unseen() +
          account2.INBOX:contain_subject('alert')
```

---

## Processing Results

| Method | Description |
|--------|-------------|
| `:delete_messages()` | Delete matching messages. |
| `:copy_messages(destination)` | Copy to a mailbox (same or different account). |
| `:move_messages(destination)` | Move to a mailbox (same or different account). |

```lua
results = account.INBOX:is_unseen() * account.INBOX:contain_from('boss@work.com')
results:copy_messages(account['Work/Important'])

results = account.newsletter:select_all()
results:move_messages(account.archive)

-- Cross-account move
results:move_messages(otheraccount['folder/mailbox'])
```

### Flag Operations

| Method | Description |
|--------|-------------|
| `:mark_answered()` / `:unmark_answered()` | Mark/unmark as answered. |
| `:mark_deleted()` / `:unmark_deleted()` | Mark/unmark for deletion. |
| `:mark_draft()` / `:unmark_draft()` | Mark/unmark as draft. |
| `:mark_flagged()` / `:unmark_flagged()` | Mark/unmark urgent attention. |
| `:mark_seen()` / `:unmark_seen()` | Mark/unmark as read. |

```lua
results:add_flags({ '\\Seen', '\\Flagged' })
results:remove_flags({ '\\Seen' })
results:replace_flags({ '\\Answered' })
results:has_keyword('MyCustomFlag')
results:has_unkeyword('OldFlag')
```

---

## Fetching Message Parts

Access individual messages by UID:

```lua
-- Fetch full message (header + body)
message = account.INBOX[22]:fetch_message()

-- Fetch just the header
header = account.INBOX[22]:fetch_header()

-- Fetch just the body
body = account.INBOX[22]:fetch_body()

-- Fetch a specific header field
subject = account.INBOX[22]:fetch_field('Subject')

-- Fetch a specific MIME part (e.g., "1.1" for first sub-part of part 1)
part = account.INBOX[5]:fetch_part('1.1')
```

### Message Metadata

```lua
flags   = account.INBOX[22]:fetch_flags()       -- table of flag strings
date    = account.INBOX[22]:fetch_date()         -- internal date string
size    = account.INBOX[22]:fetch_size()         -- size in bytes
structure = account.INBOX[22]:fetch_structure()  -- body structure table

-- Structure example:
-- { "1"     = { type = "text/plain",  size = 5000, name = nil,  encoding = "7bit" },
--   "1.1"   = { type = "text/html",   size = 12000, name = nil,  encoding = "base64" },
--   "2"     = { type = "application/pdf", size = 50000, name = "doc.pdf", encoding = "base64" } }
```

### Iterating Over Results

```lua
results = account.INBOX:is_unseen()
for _, message_ref in ipairs(results) do
    mailbox, uid = table.unpack(message_ref)
    header = mailbox[uid]:fetch_header()
    print(uid, header)
end
```

---

## Appending Messages

```lua
-- Append raw message text to a mailbox
account.INBOX:append_message(raw_email_text)

-- Append with specific flags and date
account.INBOX:append_message(
    raw_email_text,
    { '\\Seen', '\\Flagged' },
    '15-Jan-2024 10:30:00 +0000'
)
```

### Adding Headers to Messages

```lua
all = account.INBOX:select_all()
for _, mesg in ipairs(all) do
    mbox, uid = table.unpack(mesg)
    header = mbox[uid]:fetch_header()
    body   = mbox[uid]:fetch_body()
    -- Add a custom header
    message = header:gsub('[\r\n]+$', '\r\n') ..
              'X-Custom-Header: my-value\r\n' .. '\r\n' .. body
    account.Archive:append_message(message)
end
```

---

## Auxiliary Functions

### Date Utilities

```lua
-- Generate a date N days ago in 'day-month-year' format
date = form_date(14)  -- date from 14 days ago
```

### Interactive Password Prompt

```lua
password = get_password('Enter password: ')
```

### Process Piping

```lua
-- Send data to a command's stdin; returns exit status
status = pipe_to('spamassassin', email_text)

-- Read from a command's stdout; returns (exit_status, output_string)
status, data = pipe_from('gpg --decrypt secret.txt')
data = data:gsub('[\r\n]', '')  -- strip newlines
```

### Regular Expression Matching

```lua
-- PCRE-based search; returns success boolean + any captures
success, capture1, capture2 = regex_search('^(?i)pcre: (\\w+)$', 'mystring')
if success then
    print('Matched:', capture1)
end
```

### Delay & Daemon Mode

```lua
-- Pause execution
sleep(300)  -- 5 minutes

-- Run as a background daemon, polling every N seconds
become_daemon(600, my_function)                    -- standard daemon
become_daemon(600, my_function, true)               -- don't change directory
become_daemon(600, my_function, true, true)         -- don't redirect stdio either
```

### Error Recovery with Exponential Backoff

```lua
function fragile_commands()
    results = account.INBOX:is_old()
    results:move_messages(account.archive)
end

-- Retry indefinitely on failure (with exponential backoff between retries)
recover(fragile_commands)

-- Retry up to N times
recover(fragile_commands, 5)

-- Handle errors explicitly
success, errormsg = recover(fragile_commands, 5)
if not success then
    print('Failed:', errormsg)
end

-- Capture return values on success
success2, results2 = recover(function()
    return account.INBOX:is_seen()
end)
if success2 then
    results2:delete_messages()
end
```

---

## Advanced Patterns

### IDLE-Based Real-Time Filtering

```lua
while true do
    -- Wait for server notification of new messages
    update, event = account.INBOX:enter_idle()
    if update then
        -- Process newly arrived unseen messages
        results = account.INBOX:is_unseen()
        results:move_messages(account.archive)
    end
end
```

### Smart IDLE (skip when no new mail)

```lua
function smart_idle(mailbox)
    if #mailbox:is_unseen() == 0 then
        if not mailbox:enter_idle() then
            sleep(300)  -- fallback polling for servers without IDLE support
        end
    end
end

while true do
    smart_idle(account.INBOX)
    results = account.INBOX:is_unseen()
    results:move_messages(account.archive)
end
```

### Continuous Multi-Account Filtering with Per-Account Recovery

```lua
function process_account1()
    results = account1.mailbox:is_old()
    results:move_messages(account1.archive)
end

function process_account2()
    results = account2.mailbox:is_seen()
    results:delete_messages()
end

while true do
    recover(process_account1, 4)  -- retry up to 4 times
    recover(process_account2, 2)  -- retry up to 2 times
    sleep(60)
end
```

### External Spam Filter Integration (Bayesian)

```lua
all = account.INBOX:select_all()
results = Set {}

for _, mesg in ipairs(all) do
    mbox, uid = table.unpack(mesg)
    text = mbox[uid]:fetch_message()
    if pipe_to('bayesian-spam-filter', text) == 1 then
        table.insert(results, mesg)
    end
end

results:delete_messages()
```

### Filter Only Text Parts of Attachments

```lua
all = account.INBOX:select_all()
results = Set {}

for _, mesg in ipairs(all) do
    mbox, uid = table.unpack(mesg)
    structure = mbox[uid]:fetch_structure()
    for partid, partinfo in pairs(structure) do
        if partinfo.type:lower() == 'text/plain' and (partinfo.size or 0) < 1024 then
            part = mbox[uid]:fetch_part(partid)
            if pipe_to('spam-checker', part) == 1 then
                table.insert(results, mesg)
                break
            end
        end
    end
end

results:delete_messages()
```

### Password Vault Integration (e.g., `pass`)

```lua
status, password = pipe_from('pass Email/imap-server')
password = password:gsub('[\r\n]', '')

account = IMAP {
    server   = 'imap.example.com',
    username = 'user@example.com',
    password = password
}
```

---

## Daemon Mode & IDLE

### Running as a Background Daemon

```lua
function filter_loop()
    results = account.INBOX:is_unseen()
    if #results > 0 then
        results:move_messages(account.archive)
    end
end

-- Run every 600 seconds (10 minutes) in the background
become_daemon(600, filter_loop)
```

### Running with IDLE (Event-Driven)

```lua
while true do
    account.INBOX:enter_idle()          -- blocks until server sends update
    results = account.INBOX:is_unseen()
    if #results > 0 then
        results:mark_seen()
    end
end
```

### Signal-Based Interruption

The IDLE mode can be interrupted by `SIGUSR1` or `SIGUSR2` signals at any time. After receiving a signal, execution continues from the line after `enter_idle()`. Only `true` is returned (no event string).

---

## Error Recovery

By default, if an error occurs (network failure, server timeout), IMAPFilter stops immediately. Use `recover()` for resilience:

```lua
-- recover() wraps a function and retries on error with exponential backoff
function batch_process()
    local results = account.INBOX:is_unseen()
    results:copy_messages(account.archive)
end

recover(batch_process, 5)  -- retry up to 5 times
```

For multi-account setups, wrap each account's logic independently so one failure doesn't block the others:

```lua
while true do
    recover(function() process_account1() end, 4)
    recover(function() process_account2() end, 3)
    recover(function() process_account3() end, 2)
    sleep(60)
end
```

---

## OAuth2 Authentication

For servers supporting XOAUTH2 (e.g., Gmail):

```lua
user       = 'me@gmail.com'
clientid   = 'xxxx.apps.googleusercontent.com'
clientsecret = 'your_secret'
refreshtoken = '1/xxx-refresh-token-yyy'

-- Generate access token
status, output = pipe_from('oauth2.py --client_id=' .. clientid ..
                   ' --client_secret=' .. clientsecret ..
                   ' --refresh_token=' .. refreshtoken)
_, _, accesstoken = string.find(output, 'Access Token: ([%w%p]+)\n')

-- Generate OAuth2 string
status, output = pipe_from('oauth2.py --generate_oauth2_string' ..
                           ' --access_token=' .. accesstoken ..
                           ' --user=' .. user)
_, _, oauth2string = string.find(output, 'OAuth2 argument:\n([%w%p]+)\n')

account = IMAP {
    server = 'imap.gmail.com',
    ssl    = 'tls1.2',
    username = user,
    oauth2   = oauth2string
}
```

Alternative OAuth2 token managers: [gmail-oauth2-tools](https://github.com/google/gmail-oauth2-tools), [oama](https://github.com/pdobsan/oama), [pizauth](https://github.com/ltratt/pizauth), [email-oauth2-proxy](https://github.com/simonrob/email-oauth2-proxy).

---

## Reference: Sample Files

| File | Purpose |
|------|---------|
| `samples/config.lua` | Basic filtering examples with multiple accounts and mailboxes. |
| `samples/extend.lua` | Advanced patterns: IDLE, daemon mode, external tools, OAuth2, recovery. |

---

## Quick Reference Cheat Sheet

```lua
-- Account setup
acc = IMAP { server = 'imap.example.com', username = 'u', password = 'p' }

-- Common searches
acc.INBOX:select_all()          -- all messages
acc.INBOX:is_unseen()           -- unread
acc.INBOX:is_flagged()          -- flagged
acc.INBOX:contain_from('x@y')   -- by sender
acc.INBOX:contain_subject('z')  -- by subject
acc.INBOX:match_body('.*regex') -- regex in body

-- Set operations
r1 + r2    -- OR (union)
r1 * r2    -- AND (intersection)
r1 - r2    -- NOT (difference)

-- Actions
results:delete_messages()
results:copy_messages(acc.Archive)
results:move_messages(acc.Trash)
results:mark_seen()
results:add_flags({ '\\Seen', '\\Flagged' })

-- Fetching
acc.INBOX[1]:fetch_header()     -- header only
acc.INBOX[1]:fetch_body()       -- body only
acc.INBOX[1]:fetch_field('From')-- specific field
acc.INBOX[1]:fetch_part('1.2')  -- MIME part

-- IDLE / daemon
while true do
    acc.INBOX:enter_idle()
    acc.INBOX:is_unseen():move_messages(acc.Archive)
end

become_daemon(60, my_function)

-- Recovery
recover(function() acc.INBOX:select_all():delete_messages() end, 5)
```
