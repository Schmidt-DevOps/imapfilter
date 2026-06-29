# Contract: CLI Interface

## Program Entry Point

**Binary**: `imapfilter` (installed to `$PREFIX/bin/`)

## Command Line Options

```
imapfilter [options]
```

| Flag | Long Form | Argument | Description |
|------|-----------|----------|-------------|
| `-c` | `--config` | `<path>` | Path to config file; use `-` for stdin. Default: `$HOME/.imapfilter/config.lua` |
| `-d` | `--debug` | `<file>` | File path for debug output (full IMAP protocol communication) |
| `-e` | `--execute` | `<command>` | Execute a Lua command string from CLI |
| `-i` | `--interactive` | — | Enter interactive REPL after config execution |
| `-l` | `--log` | `<file>` | File path for error log output |
| `-n` | `--dry-run` | — | Suppress write operations; report what would happen |
| `-p` | `--pidfile` | `<file>` | Write PID to file on startup, delete on exit |
| `-t` | `--truststore` | `<path>` | Path to SSL CA trust store (directory or file) |
| `-v` | `--verbose` | — | Print brief communication details to stdout |
| `-V` | `--version` | — | Display version and copyright, then exit |

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Configuration error (Lua syntax error, missing file) |
| 2 | Connection error (cannot reach server) |
| 3 | Protocol error (IMAP command failed) |
| 4 | General runtime error |

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `HOME` | User home directory; default config location is `$HOME/.imapfilter/` |
| `IMAPFILTER_HOME` | Override the default configuration directory |

## Configuration File Format

The config file is a Lua script. It MUST:
1. Define one or more `Account` objects via `account = IMAP{...}`
2. Perform mailbox operations and message filtering using Lua syntax
3. Use set operators (`+`, `*`, `-`) for combining search results
4. Call action methods on Results Sets (e.g., `results:delete_messages()`)

### Account Definition Contract

```lua
account_name = IMAP {
    server = "<hostname>",
    port = <number>,          -- optional, defaults based on ssl mode
    username = "<user>",
    password = "<password>",
    ssl = "<mode>",           -- "none", "starttls", "sslv23", etc.
    timeout = <seconds>,      -- optional, default 120
}
```

### Global Options Contract

```lua
options.timeout = <seconds>     -- connection timeout
options.subscribe = true/false  -- auto-subscribe on mailbox creation
options.persist = true/false    -- persistent reconnection
```
