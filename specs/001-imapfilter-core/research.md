# Research Findings: IMAPFilter Implementation

## Decision 1: Lua as Primary Configuration Language

**Decision**: Use embedded Lua interpreter (not a separate process) for configuration and extension.

**Rationale**: 
- Lua is lightweight, embeddable, and has a small C API surface
- Single-process architecture simplifies state management vs. fork/exec models
- Users can write complex filtering logic in Lua without learning a new DSL
- The `IMAP{}` constructor pattern provides natural object-oriented feel from Lua
- Historical precedent: IMAPFilter has used Lua since v1.x (2004+)

**Alternatives considered**:
- JSON config + separate filter engine: Simpler but less expressive; requires parsing layer
- XML config: Verbose, not scriptable
- Python embedding: Heavier dependency, larger memory footprint
- Pure C config structs: Less flexible for user customization

## Decision 2: C Core Architecture — Single File Per Concern

**Decision**: Each C source file has a single, well-defined responsibility.

**Rationale**:
- `core.c` handles IMAP protocol state machine and command/response cycle
- `socket.c` handles TCP/IPv6 connection abstraction
- `cert.c` handles SSL certificate loading and validation
- `lua.c` handles all Lua C API registration
- This matches the Simplicity principle: "each C module has a single clear responsibility"

**Alternatives considered**:
- Monolithic `imapfilter.c`: Simpler for small projects but harder to maintain as features grow
- Object-oriented C with vtables: Adds complexity without significant benefit for this use case

## Decision 3: IMAP Protocol Implementation — RFC-Conformant State Machine

**Decision**: Implement a full IMAP4rev1 state machine in `core.c`.

**Rationale**:
- Must support both IMAP4rev1 (RFC 3501) and legacy IMAP4 (RFC 1730)
- Server capability negotiation required before using extensions (NAMESPACE, IDLE, etc.)
- Response parsing must handle multi-line responses, untagged updates, literal strings
- `request.c` builds commands; `response.c` parses replies — clean separation

**Alternatives considered**:
- Use libcurl or libimap: Adds dependency; less control over protocol behavior
- Minimal IMAP subset: Would miss features like NAMESPACE, IDLE, UTF-8 mailbox names

## Decision 4: Results Set as Lua Table with C Backing

**Decision**: Search results are represented as a Lua table (via `set.lua`) backed by C data structures.

**Rationale**:
- Set operations (`+`, `*`, `-`) map naturally to Lua's operator overloading via metatables
- `common.lua` defines the Results Set type with methods like `delete_messages()`
- C layer handles actual IMAP command generation; Lua layer handles expression composition
- This is the "Lua-First" principle in action: user-facing API is pure Lua

**Alternatives considered**:
- Pure C results structure: Less flexible for user-defined operations
- Pure Lua results (no C): Would require fetching all data into Lua, memory inefficient

## Decision 5: SSL/TLS via OpenSSL with Compile-Time Defaults

**Decision**: Use OpenSSL for SSL/TLS; compile-time defaults for CA paths, overridable at runtime.

**Rationale**:
- OpenSSL is the de facto standard on Linux/Unix systems
- `SSLCAPATH` and `SSLCAFILE` are compile-time defaults (`#define`) but can be overridden via `-t` flag
- Certificate validation supports both system trust store and user-provided certificates file
- STARTTLS negotiation handled in `cert.c`; explicit SSL modes (ssl23, ssl3, tls1) supported

**Alternatives considered**:
- GnuTLS: Alternative but less common on Linux; different API surface
- OpenSSL 3.x vs 1.x: Code supports both via version detection at compile time

## Decision 6: PCRE2 for Regular Expression Matching

**Decision**: Use PCRE2 (not legacy PCRE) for regex pattern matching.

**Rationale**:
- PCRE2 is the modern API; IMAPFilter v2.7+ switched from PCRE to PCRE2
- Supports UTF-8 patterns for international mailbox names
- `pcre.c` wraps PCRE2 compilation and execution; `regexp.lua` exposes to Lua

**Alternatives considered**:
- POSIX regex (`regcomp`/`regexec`): Simpler but less powerful (no alternation, limited quantifiers)
- Lua pattern matching: Built-in but not full PCRE feature set

## Decision 7: Build System — GNU Make Delegation Pattern

**Decision**: Top-level `Makefile` delegates to `src/Makefile`; standard targets (`all`, `install`, `uninstall`, `clean`).

**Rationale**:
- Standard Unix convention; familiar to all target users
- No autoconf/automake overhead — simple, transparent build
- Lua modules installed alongside C binary in `$PREFIX/share/imapfilter/`
- Man pages installed to standard locations

**Alternatives considered**:
- CMake: More portable but adds dependency; overkill for a single-binary project
- Meson/Ninja: Modern but less familiar to traditional Unix users

## Decision 8: Per-Account Connection State Tracking

**Decision**: Each IMAP account maintains its own connection state (`session.c/h`).

**Rationale**:
- Accounts may connect to different servers, use different credentials
- Cross-account operations (copy/move) require separate connections per account
- Recovery mechanism tracks state per-account independently
- PID file is global but connection management is per-account

**Alternatives considered**:
- Global connection pool: Simpler for single-account but doesn't scale to multi-account
- Connection-per-operation: Less efficient for batch operations on same account
