<!--
SYNC IMPACT REPORT — v1.0.0 (initial constitution)

Version change: N/A → 1.0.0
Modified principles: N/A (first version)
Added sections: Core Principles (5), Build & Dependencies, Configuration & Deployment, Governance
Removed sections: N/A
Templates requiring updates: ⚠ pending — plan-template.md, spec-template.md, tasks-template.md reference generic "Constitution Check" but no specific principles yet
Follow-up TODOs: Populate Constitution Check gates in plan-template.md with concrete IMAPFilter-specific rules

-->

# IMAPFilter Constitution

## Core Principles

### I. Lua-First Configuration

Every user-facing configuration and extension point uses Lua as the primary language.
The program embeds a Lua interpreter and exposes its API through C-Lua bindings.
Lua modules in `src/` are shared at install time to `$SHAREDIR/`.
Configuration files follow the same Lua syntax, enabling users to script complex filtering logic.

- All public APIs (account operations, mailbox queries, message actions) MUST be accessible from Lua.
- Shared Lua modules (`common.lua`, `set.lua`, `mailbox.lua`, etc.) MUST be self-contained and documented.
- The Lua runtime is a first-class dependency — not an afterthought.

### II. CLI Text I/O Protocol

IMAPFilter is a command-line tool. Its interface follows strict text-based conventions:
command-line arguments drive behavior, stdout carries output, stderr carries errors.
The `--dry-run` flag provides safe preview mode without modifying server state.
Configuration can be loaded from file or piped via stdin (`-c -`).

- All program options MUST follow POSIX-style short flags with optional long-form equivalents.
- Error messages MUST go to stderr; informational output to stdout.
- Dry-run mode MUST suppress write operations while still reporting what would happen.

### III. IMAP Protocol Compliance

IMAPFilter implements IMAP4rev1 (RFC 3501) and IMAP4 (RFC 1730), plus key extensions:
NAMESPACE (RFC 2342), CHILDREN (RFC 3348), AUTH=LOGIN (RFC 2195), IDLE (RFC 2177),
UTF-8 (RFC 6855). Protocol correctness is non-negotiable.

- IMAP command generation MUST conform to RFC specifications for syntax and state machine.
- Server capability negotiation MUST respect advertised extensions before using them.
- Connection recovery MUST handle transient network errors gracefully without data loss.

### IV. Reliability & Recovery

IMAPFilter supports persistent connections with configurable recovery mechanisms.
The `persist` option enables indefinite reconnection on failure; the newer recovery
mechanism (v2.8.0+) provides structured error handling and config-driven retry policies.

- Connection state MUST be tracked per-account, not globally.
- Recovery configuration MUST be explicit in the user's config file — no silent defaults.
- PID file management MUST ensure cleanup on exit (normal or signal-induced).

### V. Simplicity & Minimal Dependencies

IMAPFilter keeps its C core lean and delegates complexity to Lua scripting.
Build dependencies are limited to three libraries: Lua, PCRE2, OpenSSL.
The program avoids unnecessary abstractions — each C module has a single clear responsibility.

- New C modules MUST have a single, well-defined purpose (e.g., `socket.c` for networking).
- Build system MUST remain Makefile-based with standard Unix conventions (`make install`).
- Configuration complexity is pushed to Lua; the C core stays focused on protocol and I/O.

## Build & Dependencies

### Technology Stack

| Component | Requirement |
|-----------|-------------|
| Language  | C (POSIX-compliant) |
| Scripting | Lua 5.1, 5.2, 5.3, 5.4, or 5.5 |
| Regex     | PCRE2 (v10.00+) |
| TLS       | OpenSSL (v1.0.2+) |
| Build     | GNU Make with standard `all/install/uninstall/clean` targets |

### Build Rules

- The top-level `Makefile` delegates to `src/Makefile`; no subdirectory may diverge from this pattern.
- Shared Lua modules are installed to `$PREFIX/share/imapfilter/`.
- Man pages install to `$PREFIX/man/man1/` and `$PREFIX/man/man5/`.
- SSL trust store paths (`SSLCAPATH`, `SSLCAFILE`) are compile-time defaults, overridable via `-t` flag.

## Configuration & Deployment

### File Layout

```
$HOME/.imapfilter/
├── config.lua          # Primary configuration (user passwords here)
└── certificates        # User-trusted SSL certificates
```

- `config.lua` MAY contain sensitive data; recommended permissions: `0600`.
- Environment variable `IMAPFILTER_HOME` overrides the default config directory.
- Sample configurations live in `samples/`; extend them rather than rewriting from scratch.

### Multi-Account Support

- Each IMAP account is a distinct `IMAP{}` Lua table with server, username, password.
- Accounts MAY share hostnames or usernames if ports differ (disambiguated at connection time).
- Cross-account operations (copy/move between accounts) MUST maintain separate connections per account.

## Governance

### Amendment Procedure

This constitution governs development decisions for IMAPFilter. Amendments require:

1. **Documentation**: Changes to principles must be recorded in this file with rationale.
2. **Version Bump**: Per the versioning policy below, reflecting the scope of change.
3. **Propagation**: Updated principles must be reflected in templates and documentation.

### Versioning Policy (Semantic Versioning)

| Bump Type | Trigger |
|-----------|---------|
| MAJOR     | Breaking protocol changes, Lua API removals/redefinitions, config format incompatibility |
| MINOR     | New features (e.g., new IMAP extension support), new Lua API methods, new options |
| PATCH     | Bug fixes, documentation improvements, compatibility adjustments for existing Lua versions |

### Compliance Expectations

- All code changes MUST be consistent with the principles in this constitution.
- Complexity additions MUST be justified against the Simplicity principle.
- The `NEWS` file MUST document all user-visible changes per release.
- This constitution supersedes ad-hoc development decisions when they conflict.

**Version**: 1.0.0 | **Ratified**: 2026-06-29 | **Last Amended**: 2026-06-29
