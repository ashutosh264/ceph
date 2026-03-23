# GSoC 2026 Proposal: radosgw-admin UX and Documentation Improvements

**Project:** radosgw-admin UX and documentation improvements

**Organization:** Ceph

**Mentors:** Yuval Lifshitz (ylifshit@ibm.com), Jacques Heunis (jheunis@bloomberg.net)

**Project Hours:** 350

**Difficulty:** Advanced

---

## About Me
|||
|---|---|
| **Name** | Ashutosh Shaha |
| **Email** | ashutoshbshaha30@gmail.com |
| **GitHub** | [github.com/ashutosh264] |
| **College** | Pune Institute of Computer Technology  |
| **Timezone** | IST (GMT+5:30) |
| **Slack** | Ashutosh Shaha |

I have a strong background in data systems and software development, with experience working across databases, ETL pipelines, and system design. My work has involved building scalable solutions, integrating multiple data sources, and developing intuitive interfaces for complex data visualization. I’m particularly keen & very much interested in leveraging technology to solve real-world problems through efficient and well-structured systems.

---

## Synopsis

`radosgw-admin` is the primary administration tool for Ceph's RADOS Gateway (RGW). Its current command-line parsing is hand-rolled: a giant `getopt_long()` loop, a ~400-line manual `usage()` function, and a long `if/else` chain matching string tokens to commands. Documentation (the `--help` output, the man page, and the admin guide) is maintained independently and manually, leading to significant drift between what the code actually supports and what the docs describe.

This project replaces the manual approach with a structured CLI framework that:

1. Declares command and argument semantics explicitly in code
2. Auto-generates context-aware help and usage output
3. Enables programmatic generation of the man page and admin guide
4. Maintains full backward compatibility with existing command syntax

---

## The Problem (Evidence from Documentation Audit)

I performed a systematic comparison of three documentation sources:
- The source code (`src/rgw/radosgw-admin/radosgw-admin.cc`, `usage()` function, lines 140-557)
- The help output (`radosgw-admin -h`)
- The man page (`doc/man/8/radosgw-admin.rst`)

### Key Findings

| Category | Count |
|----------|-------|
| Critical command name mismatches (man page has wrong command) | 3 |
| Entire features completely undocumented in man page | 4 (account, dedup, mfa, lua scripting) |
| Individual commands missing from man page | 50+ |
| Commands in man page but removed from source | 1-3 |
| Option description mismatches | 4+ |

### Critical Mismatches

These commands exist in the source code under one name, but the man page documents them under a different name:

| Source Code (correct) | Man Page (wrong) | Man Page Line |
|-----------------------|------------------|---------------|
| `role delete` | `role rm` | 452 |
| `role-trust-policy modify` | `role modify` | 461 |
| `role-policy delete` | `role-policy rm` | 473 |

The admin guide (`doc/radosgw/role.rst`) confirms the source code names are correct. I have prepared a PR fixing these mismatches.

**Fix PR:** https://github.com/ceph/ceph/pull/67890

### Undocumented Feature Areas

Entire command groups present in the source `usage()` but absent from the man page:

- **Account management** (6 commands): `account create`, `account modify`, `account get`, `account stats`, `account rm`, `account list`
- **Dedup** (7 commands): `dedup stats`, `dedup estimate`, `dedup exec`, `dedup abort`, `dedup pause`, `dedup resume`, `dedup throttle`
- **MFA/TOTP** (6 commands): `mfa create`, `mfa list`, `mfa get`, `mfa remove`, `mfa check`, `mfa resync`
- **Lua scripting** (7 commands): `script put/get/rm`, `script-package add/rm/list/reload`

The full audit is available in [doc_audit.md](doc_audit.md).

---

## CLI Framework Survey

I evaluated five C++ CLI/argument-parsing frameworks against criteria derived from the project requirements:

| Criterion | CLI11 | Boost.PO | Taywee/args | argparse | TCLAP |
|-----------|-------|----------|-------------|----------|-------|
| Maturity & maintenance | Yes | Yes | Partial | Yes | Partial |
| Header-only | Yes | No | Yes | Yes | Yes |
| Nested subcommand hierarchy | Yes | No | Yes | Yes | No |
| Auto help/usage generation | Yes | Partial | Yes | Yes | Yes |
| Per-command `--help` | Yes | No | Partial | Yes | No |
| Per-command arg declarations | Yes | Partial | Yes | Yes | No |
| Unknown verb suggestion | Yes | No | Partial | Partial | No |
| Min C++ standard | C++11 | C++11 | C++11 | C++17 | C++98 |
| License (LGPL compat) | BSD-3 | Boost | MIT | MIT | MIT |
| **Score** | **8/9** | **3/9** | **6/9** | **7/9** | **3/9** |

### Recommendation: CLI11

**CLI11** is the clear winner:

1. **First-class nested subcommands** -- `radosgw-admin bucket check olh` maps directly to `app->add_subcommand("bucket")->add_subcommand("check")->add_subcommand("olh")`
2. **Header-only** -- drop a single `CLI11.hpp` into the source tree; no build system changes
3. **Auto-generated help** -- eliminates the 400-line manual `usage()` function entirely
4. **Unknown verb listing** -- already supports the existing "Expected one of the following" behavior
5. **Used by CMake itself** -- Ceph's build system uses CMake, establishing precedent
6. **BSD-3-Clause license** -- compatible with Ceph's LGPL-2.1

The full evaluation is available in [framework_evaluation.md](framework_evaluation.md).

### Sample Migration

Current code pattern:
```cpp
// ~400 lines of manual usage output
void usage() {
  cout << "  user create               create a new user\n";
  cout << "  user modify               modify user\n";
  // ... hundreds more lines ...
}

// Giant if/else chain
if (opt_cmd == OPT::USER_CREATE) { ... }
else if (opt_cmd == OPT::USER_MODIFY) { ... }
```

Proposed CLI11 pattern:
```cpp
CLI::App app{"radosgw-admin -- Ceph Object Gateway admin tool"};

struct UserCreateParams {
    std::string uid;
    std::string display_name;
};

UserCreateParams ucp;
auto user = app.add_subcommand("user", "User management commands");
auto user_create = user->add_subcommand("create", "Create a new user");
user_create->add_option("--uid", ucp.uid, "User ID")->required();
user_create->add_option("--display-name", ucp.display_name, "Display name")->required();
user_create->callback([&ucp]() { handle_user_create(ucp); });
```

Each command gets its own parameter struct, and the callback captures only that struct. This constrains the callback to only access parameters that were explicitly declared on the CLI11 command. If a developer accidentally references a parameter that belongs to a different command, it is a compile-time error rather than a silent documentation mismatch. This design makes it as difficult as reasonably possible for the documentation (auto-generated from CLI11 declarations) to diverge from observed behavior.

This gives:
- `radosgw-admin --help` lists all top-level commands
- `radosgw-admin user --help` lists user subcommands
- `radosgw-admin user create --help` shows required/optional args for user create
- Missing required args produce automatic error messages
- Unknown subcommands suggest closest match

---

## Proposed Timeline (350 hours)

### Phase 0: Orientation (Weeks 1-2, ~40 hours)
- Deep-dive into `radosgw-admin.cc` internals: option parsing flow, command dispatch, how commands map to RGW store operations
- Study how existing tests (`src/test/cli/radosgw-admin/help.t`) validate the CLI
- Set up a reliable dev/test loop: build, vstart, run radosgw-admin, run tests
- Create a complete inventory of all commands, their arguments, which are required vs optional, and default values

### Phase 1: CLI11 Integration - Core Infrastructure (Weeks 3-5, ~60 hours)
- Add `CLI11.hpp` to `src/rgw/radosgw-admin/`
- Create the top-level `CLI::App` and register all first-level subcommands (`user`, `bucket`, `zone`, `realm`, etc.)
- Wire up the subcommand callbacks to the existing dispatch logic (initially just bridging CLI11 parsing to the existing `OPT::` enum flow)
- Ensure `radosgw-admin --help` produces output equivalent to the current `usage()` function
- Validate backward compatibility: all existing command invocations must continue to work identically

### Phase 2: Per-Command Argument Declarations (Weeks 6-9, ~80 hours)
- Migrate command-specific options from the global `getopt_long` array to per-subcommand CLI11 declarations
- Start with a representative subset: `user create/modify/rm/info`, `bucket list/stats/check`, `zone/zonegroup` commands
- Add `->required()` annotations where the current code manually checks for missing args and prints errors
- Validate that per-command `--help` output is accurate and useful
- Ensure the existing `help.t` test passes, update as needed

### Phase 3: Complete Migration (Weeks 10-12, ~60 hours)
- Extend CLI11 declarations to remaining command groups: role, caps, quota, ratelimit, sync, lifecycle, notifications, scripting, mfa, account, dedup
- Remove the manual `usage()` function once all commands are registered
- Remove the manual option-parsing code (`getopt_long` loop) once fully superseded
- Comprehensive backward-compatibility testing

### Phase 4: Auto-Generated Documentation (Weeks 13-15, ~60 hours)
- Write a Python script that introspects the CLI11 command tree (by invoking `radosgw-admin` with a special `--dump-cli-tree` flag that outputs JSON) and generates:
  - The man page in RST format (`doc/man/8/radosgw-admin.rst`)
  - The admin guide sections
- Integrate doc generation into the build/CI pipeline so docs are always in sync
- Add a CI check that compares generated docs against committed docs, failing if they diverge

### Phase 5: Polish and Edge Cases (Weeks 16-17, ~30 hours)
- Handle edge cases: commands with hyphens vs spaces (`role-trust-policy` vs `role trust policy`), backward-compat aliases
- Ensure the "Expected one of the following" behavior on unknown verbs works identically to today
- Add tests verifying that unexpected/unrecognized arguments emit a warning to stderr but do not prevent command execution, ensuring backward compatibility for existing scripts that may pass flags not registered for the specific command being invoked
- Performance: verify no measurable overhead from CLI11 parsing

### Buffer (Week 18, ~20 hours)
- Address mentor feedback
- Fix any integration issues discovered during review
- Final documentation pass

---

## Migration Strategy 

(I prepared this strategy with AI, still thinking on this. It would be better to finalise this post discussion with the Maintainers)
The migration must be **incremental and backward-compatible**. At no point should existing scripts or user workflows break.

### Approach: Parallel Parsing with Gradual Cutover

```
Phase 1:  CLI11 parses → maps to OPT:: enum → existing dispatch (bridge mode)
Phase 2:  CLI11 parses + declares per-command args → existing dispatch
Phase 3:  CLI11 parses + declares all args → direct callbacks (getopt removed)
```

1. **Phase 1 bridge**: CLI11 is added alongside the existing parser. It handles command/subcommand routing and produces the same `OPT::` enum values. The existing `getopt_long` code still handles option values. This lets us validate command routing without touching option handling.

2. **Phase 2 per-command args**: For each command, options are moved from the global getopt array to CLI11's per-subcommand declarations. This is done command-group by command-group. At each step, the `help.t` test and manual testing confirm no regressions.

3. **Phase 3 full cutover**: Once all commands and options are in CLI11, the old getopt code and manual `usage()` function are removed.

### Backward Compatibility Guarantees

- All existing command strings continue to work (e.g., `radosgw-admin bucket stats --bucket=foo`)
- Both `--option=value` and `--option value` forms are supported (CLI11 handles both natively)
- Unexpected or unrecognized arguments emit a warning but do not prevent command execution, so existing scripts that pass flags not specific to the invoked command continue to work
- Exit codes and error message formats are preserved where possible
- The `help.t` test file serves as the regression baseline

---

## Deliverables

| Deliverable | Description |
|-------------|-------------|
| CLI11-based argument parsing | Replace hand-rolled getopt with structured CLI11 subcommand declarations |
| Auto-generated `--help` | Per-command, context-aware help output replacing the manual `usage()` function |
| Per-command `--help` | `radosgw-admin bucket --help` shows bucket-specific commands and options |
| Doc generation script | Python tool that produces the man page and admin guide from CLI11's command tree |
| CI integration | Build/CI step ensuring documentation never drifts from implementation |
| Backward compatibility | All existing command invocations work identically |

---

## About My Preparation

### Step 1: Build and Test
I was on macOS (Apple Silicon), so I had to use Podman to run an Ubuntu 24.04 container.

I set up a development environment on Ubuntu 24.04 and successfully built Ceph using the minimal cmake configuration. I ran `vstart.sh` to bring up a local cluster and performed the following radosgw-admin exercises:

- Created a bucket using `s3cmd` with the default vstart user
- Uploaded objects to the bucket
- Created a new user with `radosgw-admin user create`
- Modified bucket ownership with `radosgw-admin bucket link`
- Verified ownership change with `radosgw-admin bucket stats`
- Confirmed that the old credentials could no longer upload to the bucket
- Verified that the new user's credentials work

### Step 2: Documentation Audit and Framework Survey

**Audit findings:** I compared the source code `usage()` function, the `--help` output, and the man page. I found 3 critical command name mismatches, 50+ commands missing from the man page, and 4 entire feature areas completely undocumented. Full details in [doc_audit.md](doc_audit.md).

**PR submitted:** I fixed the 3 critical man page mismatches (role delete, role-trust-policy modify, role-policy delete). PR: [https://github.com/ceph/ceph/pull/67890]

**Framework survey:** I evaluated 5 C++ CLI frameworks against the project's requirements and recommend CLI11. Full analysis in [framework_evaluation.md](framework_evaluation.md).

---
