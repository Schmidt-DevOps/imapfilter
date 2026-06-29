---
description: "Task list for IMAPFilter Core Application implementation"
---

# Tasks: IMAPFilter Core Application

**Input**: Design documents from `/specs/001-imapfilter-core/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: Manual testing against real IMAP servers; debug mode (`-d`) captures full protocol communication.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- Source code at repository root under `src/`
- Documentation in `doc/`
- Samples in `samples/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [X] T001 Create top-level project files: Makefile, README, LICENSE, AUTHORS, .gitignore
- [X] T002 Create `src/` directory with src/Makefile build rules
- [X] T003 Create `doc/` directory for man pages
- [X] T004 Create `samples/` directory for example configurations
- [X] T005 [P] Create version.h with VERSION and COPYRIGHT constants in src/version.h
- [X] T006 [P] Create pathnames.h with default config/share/man paths in src/pathnames.h

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Build System & Infrastructure
- [X] T010 Configure src/Makefile with CC, CFLAGS, LDFLAGS, LIBS (Lua, PCRE2, OpenSSL)
- [X] T011 [P] Implement buffer.c/h — dynamic string/buffer utilities for IMAP command building
- [X] T012 [P] Implement list.c/h — linked list data structure for message ID management
- [X] T013 [P] Implement memory.c — custom malloc/free wrappers with tracking
- [X] T014 [P] Implement log.c — logging to stderr and debug files (verbose/debug modes)
- [X] T015 [P] Implement signal.c — signal handling for SIGTERM, SIGUSR1/2 (IDLE interrupt)
- [X] T016 [P] Implement system.c — PID file management, environment variable access

### Core Headers & Types
- [X] T017 Create imapfilter.h — shared C headers: types, constants, function prototypes
- [X] T018 Create regexp.h — PCRE2 regex type definitions and wrapper declarations

### IMAP Protocol Foundation
- [X] T019 Implement request.c — IMAP command builder (SEARCH, STORE, COPY, MOVE, EXPUNGE)
- [X] T020 Implement response.c — IMAP response parser (untagged updates, multi-line responses, literals)
- [X] T021 Implement core.c — IMAP4rev1 state machine: LOGIN, CAPABILITY, SELECT, SEARCH execution

### Network Layer
- [X] T022 Implement socket.c — TCP/IPv6 socket abstraction with connect and read/write wrappers
- [X] T023 Implement cert.c — SSL certificate handling: STARTTLS negotiation, CA validation, hostname verification

### Session Management
- [X] T024 Create session.h — per-account connection state structure definition
- [X] T025 Implement session.c — account connection lifecycle: connect, login, select mailbox, disconnect

**Checkpoint**: Foundation ready — Lua bindings and shared modules can now be implemented in parallel.

---

## Phase 3: User Story 1 - Configure and Run Mail Filtering (Priority: P1) 🎯 MVP

**Goal**: Users can define mail filtering logic in a Lua config file, run imapfilter, and have messages organized across accounts.

**Independent Test**: Write a minimal config with one account, connect to a test server, select messages matching criteria, and verify actions (copy/move/delete) execute correctly.

### Shared Lua Modules (Foundation for US1)
- [X] T030 [P] [US1] Implement common.lua — core type definitions: Account constructor, Mailbox accessor, Results Set base type
- [X] T031 [P] [US1] Implement set.lua — Results Set metatable with union (+), intersection (*), difference (-) operators
- [X] T032 [P] [US1] Implement options.lua — global options table (timeout, subscribe, persist)

### Mailbox Search Methods
- [X] T033 [US1] Implement mailbox.lua — mailbox methods: check_status(), select_all(), is_new(), is_unseen(), is_recent()
- [X] T034 [US1] Implement message.lua — message methods: contain_from(), contain_to(), contain_subject(), contain_field(), is_older(), is_smaller(), is_larger()

### Search Extension Methods
- [X] T035 [P] [US1] Implement regex.lua — PCRE2 wrapper for match_header() and match_body() Lua bindings

### Account Operations
- [X] T036 [US1] Implement account.lua — account methods: list_all(), list_subscribed(), create_mailbox(), subscribe_mailbox()

### Results Set Actions (via set.lua metatable)
- [X] T037 [US1] Add delete_messages() method to Results Set in set.lua
- [X] T038 [US1] Add copy_messages(target) and move_messages(target) methods to Results Set in set.lua
- [X] T039 [US1] Add mark_flagged(), mark_seen(), add_flags(), remove_flags() methods to Results Set in set.lua

### Lua C API Bindings (bridge between C core and Lua modules)
- [X] T040 Implement lua.c — register all C functions with Lua: IMAP constructor, mailbox accessors, search methods, action methods
- [X] T041 Implement imapfilter.c — main entry point: arg parsing (-c, -d, -e, -i, -l, -n, -p, -t, -v, -V), config loading, execution flow

### File I/O & Configuration
- [X] T042 Implement file.c — config file loading (file path or stdin via `-c -`), debug file writing

**Checkpoint**: User Story 1 complete — a user can write a config.lua with accounts and filtering rules, run imapfilter, and have messages processed.

---

## Phase 4: User Story 2 - Safe Preview with Dry-Run Mode (Priority: P2)

**Goal**: Users can preview what filtering actions would be taken without actually modifying the server state.

**Independent Test**: Run `imapfilter --dry-run` and verify no write operations reach the server while informational messages are still printed.

### Dry-Run Implementation
- [X] T050 [US2] Add dry_run flag to imapfilter.c arg parsing (handle `-n` option)
- [X] T051 [US2] Propagate dry_run state through core.c — suppress STORE/EXPUNGE/COPY/MOVE commands when set
- [X] T052 [US2] Update request.c — conditionally build write commands based on dry_run flag
- [X] T053 [US2] Update response.c — still parse server responses in dry-run mode (read operations unaffected)
- [X] T054 [US2] Add verbose output in log.c — print "[DRY-RUN]" prefix for suppressed write actions

**Checkpoint**: User Story 2 complete — `-n` flag suppresses all write operations while maintaining identical informational output.

---

## Phase 5: User Story 3 - Robust Connection Recovery (Priority: P2)

**Goal**: IMAPFilter recovers gracefully from network interruptions without manual intervention.

**Independent Test**: Simulate network interruption during execution, verify `persist` option causes reconnection and continued processing.

### Persistence & Recovery
- [X] T060 [US3] Add persist field to options.lua — global `options.persist = true/false`
- [X] T061 [US3] Implement recovery logic in session.c — detect connection drops, attempt reconnection with configurable timeout
- [X] T062 [US3] Update core.c — wrap IMAP command execution with try/retry around persistent connections
- [X] T063 [US3] Add error logging in log.c — log network errors and recovery attempts to stderr/log file
- [X] T064 [US3] Implement namespace.c — NAMESPACE extension (RFC 2342) for mailbox discovery with fallback

### IDLE Support
- [X] T065 [US3] Add enter_idle() Lua binding in lua.c — block until server sends UPDATE or signal received
- [X] T066 [US3] Implement idle handling in core.c — send IDLE command, parse untagged EXISTS/EXPUNGE updates

**Checkpoint**: User Story 3 complete — persistent connections recover from drops; IDLE mode supported.

---

## Phase 6: User Story 4 - SSL/TLS Secure Connections (Priority: P3)

**Goal**: Users connect to mail servers over encrypted channels with certificate validation.

**Independent Test**: Connect to an IMAP server with STARTTLS, verify certificate validation using system trust store or custom certificates file.

### SSL/TLS Enhancement
- [X] T070 [US4] Extend cert.c — support explicit SSL modes: ssl23, sslv3, tlsv1, tls (in addition to starttls)
- [X] T071 [US4] Add `-t` truststore flag parsing in imapfilter.c — override compile-time CA path defaults
- [X] T072 [US4] Implement SSL hostname validation in cert.c — verify server certificate CN/SAN against connection host
- [X] T073 [US4] Update session.c — pass ssl mode and truststore path to socket.c during connect
- [X] T074 [US4] Add SNI (Server Name Indication) support in cert.c for TLS connections

### OAuth2 Extension Support
- [X] T075 [P] [US4] Add XOAUTH2 authentication helper in auxiliary.lua — token refresh and auth flow example

**Checkpoint**: User Story 4 complete — SSL/TLS with STARTTLS, explicit modes, hostname validation, SNI, and OAuth2 extension.

---

## Phase 7: User Story 5 - Rich Search and Filter Expressions (Priority: P3)

**Goal**: Users write complex filtering logic combining multiple criteria using Lua operators.

**Independent Test**: Write a config with compound search expressions (AND, OR, NOT via `*`, `+`, `-`), verify correct message selection.

### Advanced Search Features
- [X] T080 [US5] Add contain_field() implementation in mailbox.lua — match arbitrary IMAP header fields
- [X] T081 [US5] Add match_header() and match_body() implementations using PCRE2 regex in message.lua
- [X] T082 [US5] Implement fetch_message(), fetch_structure(), fetch_header() in message.lua — raw message retrieval
- [X] T083 [US5] Add UTF-8 mailbox name support (RFC 6855) in response.c — handle charset="UTF-8" SEARCH responses

### Cross-Account Operations
- [X] T084 [US5] Enhance copy_messages() and move_messages() in set.lua — validate target account connection before transfer
- [X] T085 [US5] Add cross-account error handling — report failures per-account, continue processing remaining accounts

**Checkpoint**: User Story 5 complete — complex compound search expressions with regex, fetch methods, UTF-8 support, and cross-account operations.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, samples, man pages, and quality improvements

### Man Pages
- [X] T090 Create doc/imapfilter.1 — manual page covering all CLI options (-c, -d, -e, -i, -l, -n, -p, -t, -v, -V)
- [X] T091 Create doc/imapfilter_config.5 — manual page documenting config file format, account definition, options table

### Sample Configurations
- [X] T092 Create samples/config.lua — example with multiple accounts, search expressions, cross-account operations
- [X] T093 Create samples/extend.lua — advanced examples: OAuth2 flow, IDLE usage, custom extensions

### NEWS & Documentation
- [X] T094 Update NEWS file — document all changes for current version (v2.8.5)
- [X] T095 Update README — ensure installation instructions match current build process

### Quality Improvements
- [X] T096 Add misc.c — utility functions: string escaping, date parsing, IMAP flag helpers
- [X] T097 Ensure all C modules have single responsibility per Simplicity principle
- [X] T098 Verify Makefile supports standard targets: all, install, uninstall, clean, dist, distclean
- [X] T099 Add Lua 5.1–5.5 compatibility checks in build system (version detection)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all user stories
- **User Stories (Phases 3–7)**: All depend on Foundational phase completion
  - US1 (Phase 3) is MVP — can be tested independently after Phase 2
  - US2-US5 build on US1 foundation but are independently testable
- **Polish (Phase 8)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1, MVP)**: Can start after Foundational (Phase 2) — no dependencies on other stories
- **User Story 2 (P2)**: Depends on US1 foundation (dry-run extends existing action methods)
- **User Story 3 (P2)**: Depends on US1 foundation (recovery wraps existing connection logic)
- **User Story 4 (P3)**: Depends on US1 + US3 foundation (SSL builds on socket layer, OAuth2 is extension)
- **User Story 5 (P3)**: Depends on US1 foundation (advanced search extends mailbox methods)

### Within Each User Story

- Shared Lua modules → Mailbox/Message methods → Account operations → Results Set actions
- C bindings (lua.c) must be updated alongside each new Lua method
- Test after each sub-phase to verify incremental functionality

### Parallel Opportunities

- Phase 1: T005–T006 can run in parallel (different header files)
- Phase 2: T011–T016 can run in parallel (different C modules, no cross-dependencies)
- Phase 3: T030–T032 shared Lua modules can run in parallel; T033–T041 depend on them
- Phase 8: T090–T095 documentation tasks can run in parallel (different files)

---

## Parallel Example: User Story 1 Shared Modules

```bash
# Launch all shared Lua modules together:
Task: "Implement common.lua — core type definitions"
Task: "Implement set.lua — Results Set metatable with operators"
Task: "Implement options.lua — global options table"

# Once shared modules complete, launch method implementations:
Task: "Implement mailbox.lua — search methods"
Task: "Implement message.lua — filter methods"
Task: "Implement account.lua — account operations"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (project structure, Makefile, headers)
2. Complete Phase 2: Foundational (IMAP protocol engine, network layer, session management)
3. Complete Phase 3: User Story 1 — shared Lua modules + search methods + actions
4. **STOP and VALIDATE**: Test with a minimal config against a real IMAP server
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 (dry-run) → Test independently → Deploy/Demo
4. Add User Story 3 (recovery/IDLE) → Test independently → Deploy/Demo
5. Add User Story 4 (SSL/TLS/OAuth2) → Test independently → Deploy/Demo
6. Add User Story 5 (advanced search/cross-account) → Test independently → Deploy/Demo
7. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1 (shared modules + methods)
   - Developer B: User Story 3 (recovery/IDLE) — can start after Phase 2
3. Stories complete and integrate independently
4. Polish phase completed by all team members together

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Verify functionality against real IMAP servers using `-v` (verbose) and `-d` (debug file)
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
