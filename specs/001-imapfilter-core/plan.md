# Implementation Plan: IMAPFilter Core Application

**Branch**: `feature/SDO` | **Date**: 2026-06-29 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-imapfilter-core/spec.md`

## Summary

IMAPFilter is a C-based CLI mail filtering utility that connects to remote IMAP servers,
sends searching queries, and processes mailboxes based on results. It uses Lua as the
configuration and extension language, embedding a Lua interpreter to expose its API for
user-defined filtering logic.

The implementation follows a layered architecture:
- **C core** handles protocol (IMAP), networking (socket/SSL), buffering, and process management
- **Lua bindings** bridge C functionality into the Lua runtime
- **Shared Lua modules** provide high-level abstractions (Account, Mailbox, Results Set)
- **Configuration files** are pure Lua scripts that users write to define filtering rules

## Technical Context

**Language/Version**: C99 (POSIX-compliant), compiled with `gcc -Wall -Wextra -O`

**Primary Dependencies**:
- Lua 5.1–5.5 (embedded interpreter, API via `lua.h`)
- PCRE2 v10.00+ (regular expression matching for search filters)
- OpenSSL v1.0.2+ (SSL/TLS connections, certificate validation)

**Storage**: N/A (stateless; operates on remote IMAP servers)

**Testing**: Manual testing against real IMAP servers; debug mode (`-d`) captures full protocol communication to file for verification.

**Target Platform**: Linux/Unix systems with POSIX sockets and standard C library. Cross-platform compatible with macOS, BSD.

**Project Type**: CLI application / mail filtering utility

**Performance Goals**: Efficient batch processing of mailbox operations; sequence set ranges sent instead of individual message IDs where possible to reduce network round-trips.

**Constraints**:
- Single-threaded execution model (simplifies state management)
- Must support IMAP4rev1 and legacy IMAP4 servers
- Memory-efficient: processes large mailboxes without loading entire mailbox into memory
- Build must remain Makefile-based with minimal external tooling

**Scale/Scope**: Designed for individual users managing 1–5 mail accounts. Each account may have hundreds of mailboxes with thousands of messages.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Based on IMAPFilter Constitution v1.0.0, all plans must verify:

- **Lua-First**: ✅ All public APIs exposed through Lua bindings (`lua.c`); shared Lua modules (`common.lua`, `set.lua`, `mailbox.lua`, etc.) are self-contained and installed to `$SHAREDIR/`.
- **CLI I/O**: ✅ POSIX-style short flags (`-c`, `-d`, `-e`, `-i`, `-l`, `-n`, `-p`, `-t`, `-v`, `-V`); dry-run mode (`-n`) suppresses writes; errors go to stderr via `log.c`.
- **IMAP Compliance**: ✅ Implements IMAP4rev1 (RFC 3501) and IMAP4 (RFC 1730) in `core.c`; RFC extensions: NAMESPACE, CHILDREN, AUTH=LOGIN, IDLE, UTF-8. Command generation follows RFC state machine.
- **Reliability**: ✅ Per-account connection tracking (`session.h`/`session.c`); recovery via `options.persist` and config-driven retry; PID file management in `signal.c`.
- **Simplicity**: ✅ Each C module has single responsibility: `socket.c` (networking), `cert.c` (SSL certs), `buffer.c` (dynamic buffers), `list.c`/`set.lua` (data structures). Build is Makefile-based. Complexity pushed to Lua.

**All gates pass — no violations.**

## Project Structure

### Documentation (this feature)

```text
specs/001-imapfilter-core/
├── plan.md              # This file
├── spec.md              # Feature specification
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output (CLI contract, Lua API)
└── tasks.md             # Phase 2 output (separate command)
```

### Source Code (repository root)

```text
imapfilter/
├── Makefile              # Top-level: delegates to src/Makefile
├── README                # Project description and installation
├── NEWS                  # Changelog per release
├── LICENSE               # MIT license
├── AUTHORS               # Contributor credits
├── .gitignore            # Build artifacts (*.o, binary)
│
├── src/
│   ├── Makefile          # Build rules for all source files
│   ├── imapfilter.c      # Main entry point: arg parsing, config loading
│   ├── imapfilter.h      # Core headers shared across modules
│   │
│   ├── core.c            # IMAP protocol engine: command generation, response parsing
│   ├── request.c         # Request builder (IMAP commands)
│   ├── response.c        # Response parser (server replies)
│   ├── session.c/h       # Per-account connection state management
│   ├── socket.c          # TCP/IPv6 socket abstraction
│   ├── cert.c            # SSL certificate handling and validation
│   │
│   ├── buffer.c/h        # Dynamic string/buffer utilities
│   ├── list.c/h          # Linked list data structure
│   ├── memory.c          # Custom memory allocation wrappers
│   ├── file.c            # File I/O helpers (config loading, PID file)
│   ├── log.c             # Logging to stderr and debug files
│   ├── signal.c          # Signal handling (SIGTERM, SIGUSR1/2 for IDLE interrupt)
│   ├── system.c          # System-level operations (PID file, environment)
│   ├── namespace.c       # IMAP NAMESPACE extension support
│   ├── pcre.c            # PCRE2 wrapper for regex compilation/matching
│   │
│   ├── lua.c             # Lua C API bindings: register all functions
│   ├── version.h         # VERSION, COPYRIGHT constants
│   ├── pathnames.h       # Default paths (config dir, share dir)
│   ├── regexp.h          # Regex type definitions
│   │
│   └── *.lua             # Shared Lua modules:
│       ├── common.lua    # Core types: Account, Mailbox, Results Set
│       ├── set.lua       # Set operations (union +, intersection *, difference -)
│       ├── mailbox.lua   # Mailbox methods: select_all, is_new, check_status, etc.
│       ├── message.lua   # Message methods: fetch_body, match_header, etc.
│       ├── account.lua   # Account methods: list_all, create_mailbox, etc.
│       ├── regex.lua     # Regex helper functions
│       └── options.lua   # Global options table
│
├── doc/
│   ├── imapfilter.1      # Man page (program options)
│   └── imapfilter_config.5  # Man page (config file format)
│
└── samples/
    ├── config.lua        # Example configuration with multiple accounts
    └── extend.lua        # Advanced examples: OAuth2, IDLE, custom extensions
```

**Structure Decision**: Single flat `src/` directory with C source files and Lua modules co-located. This reflects the project's Simplicity principle — no deep nesting, all build artifacts in one place. The top-level Makefile delegates to `src/Makefile`, maintaining standard Unix conventions.

## Complexity Tracking

> No constitution violations detected. All design decisions align with the five principles.
