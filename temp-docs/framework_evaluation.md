# CLI Framework Evaluation for radosgw-admin

**Purpose:** GSoC proposal research - selecting a C++ CLI framework to replace manual arg parsing

---

## Current Approach (Problem)

`radosgw-admin.cc` uses a hand-rolled parser:
- `generic_getopt_dumper()` / manual `getopt_long()` with a giant flat options array
- Commands are matched via a long `if/else` chain comparing string tokens
- `usage()` is a manually maintained wall of `cout` statements (~400 lines)
- No per-command help, no auto-generated docs, no structured argument declarations
- Result: docs drift from reality whenever commands are added/changed

---

## Frameworks Evaluated

### 1. CLI11

**Repo:** https://github.com/CLIUtils/CLI11
**Stars:** ~4.2k | **License:** BSD-3-Clause

| Criterion | Result |
|-----------|--------|
| Maturity & maintenance | Yes - actively maintained (2024 releases), used in CMake itself |
| Header-only | Yes - single `CLI11.hpp` header |
| Nested command hierarchies | Yes - `add_subcommand()` with unlimited nesting |
| Auto usage/help generation | Yes - full auto-generated help with descriptions |
| Per-command `--help` | Yes - `radosgw-admin bucket --help` works out of the box |
| Per-command arg declarations | Yes - each subcommand declares its own options; auto error messages |
| List verbs on unknown input | Yes - suggests similar commands on unknown subcommand |
| License compat with LGPL | Yes - BSD-3-Clause is compatible |

Sample code for radosgw-admin:
```cpp
CLI::App app{"radosgw-admin -- Ceph Object Gateway admin tool"};

auto bucket = app.add_subcommand("bucket", "Bucket management commands");
auto bucket_stats = bucket->add_subcommand("stats", "Return bucket statistics");
bucket_stats->add_option("--bucket", bucket_name, "Bucket name")->required();
bucket_stats->add_option("--uid", uid, "User ID");

auto user = app.add_subcommand("user", "User management commands");
auto user_create = user->add_subcommand("create", "Create a new user");
user_create->add_option("--uid", uid, "User ID")->required();
user_create->add_option("--display-name", display_name, "Display name")->required();
```

Best fit for this project.

---

### 2. Boost.Program_options

**Already in Ceph's dependencies**

| Criterion | Result |
|-----------|--------|
| Maturity & maintenance | Yes - very mature (Boost 1.32+), stable |
| Header-only | No - requires compiled Boost libraries |
| Nested command hierarchies | No - no native subcommand support, manual workaround needed |
| Auto usage/help generation | Partial - auto-generates option list but not command hierarchy |
| Per-command `--help` | No - requires manual implementation per command |
| Per-command arg declarations | Partial - options declared globally, not per-command |
| List verbs on unknown input | No |
| License compat with LGPL | Yes - Boost License compatible |

Not suitable. Already in Ceph but lacks subcommand support, which is the core need. Would require significant manual scaffolding.

---

### 3. Taywee/args

**Repo:** https://github.com/Taywee/args
**Stars:** ~1.3k | **License:** MIT

| Criterion | Result |
|-----------|--------|
| Maturity & maintenance | Partial - maintained but low activity |
| Header-only | Yes - single `args.hxx` header |
| Nested command hierarchies | Yes - subcommand support via `Subparser` |
| Auto usage/help generation | Yes |
| Per-command `--help` | Partial - requires manual setup |
| Per-command arg declarations | Yes |
| List verbs on unknown input | Partial |
| License compat with LGPL | Yes - MIT compatible |

Acceptable but weaker than CLI11. Less active, smaller community.

---

### 4. p-ranav/argparse

**Repo:** https://github.com/p-ranav/argparse
**Stars:** ~3.5k | **License:** MIT

| Criterion | Result |
|-----------|--------|
| Maturity & maintenance | Yes - actively maintained (2024 releases) |
| Header-only | Yes - single `argparse.hpp` |
| Nested command hierarchies | Yes - subcommand support (added in v2.x) |
| Auto usage/help generation | Yes |
| Per-command `--help` | Yes |
| Per-command arg declarations | Yes |
| List verbs on unknown input | Partial |
| License compat with LGPL | Yes - MIT compatible |

Good alternative. C++17 requirement is fine since Ceph now requires C++23. CLI11 still has an edge on unknown-verb listing and ecosystem maturity.

---

### 5. TCLAP

**Repo:** http://tclap.sourceforge.net

| Criterion | Result |
|-----------|--------|
| Maturity & maintenance | Partial - mature but barely maintained (last release ~2019) |
| Header-only | Yes |
| Nested command hierarchies | No |
| Auto usage/help generation | Yes |
| Per-command `--help` | No |
| Per-command arg declarations | No - global only |
| List verbs on unknown input | No |
| License compat with LGPL | Yes - MIT compatible |

Not suitable. No subcommand support, effectively abandoned.

---

## Comparison Table

| Criterion | CLI11 | Boost.PO | args | argparse | TCLAP |
|-----------|-------|----------|------|----------|-------|
| Maturity | Yes | Yes | Partial | Yes | Partial |
| Header-only | Yes | No | Yes | Yes | Yes |
| Subcommand hierarchy | Yes | No | Yes | Yes | No |
| Auto help generation | Yes | Partial | Yes | Yes | Yes |
| Per-command help | Yes | No | Partial | Yes | No |
| Per-command args | Yes | Partial | Yes | Yes | No |
| Unknown verb listing | Yes | No | Partial | Partial | No |
| C++ standard | C++11 | C++11 | C++11 | C++17 | C++98 |
| License | BSD-3 | Boost | MIT | MIT | MIT |
| **Score** | **8/9** | **3/9** | **6/9** | **7/9** | **3/9** |

---

## My Recommendation: CLI11

CLI11 scores highest on all criteria relevant to this project:

1. Multi-level subcommands (`bucket stats`, `bucket check olh`) map directly to `app.add_subcommand("bucket")->add_subcommand("stats")`
2. Header-only - drop `CLI11.hpp` into the source tree, no build system changes
3. Auto-generates help output, replacing the ~400-line manual `usage()` function
4. C++11 minimum - Ceph's main branch requires C++23 (`set(CMAKE_CXX_STANDARD 23)` in `src/CMakeLists.txt`), so all evaluated frameworks are compatible
5. Used by CMake - precedent for acceptance in the Ceph build ecosystem
6. BSD-3 license - compatible with Ceph's LGPL
