# Documentation Audit of radosgw-admin

**Sources compared:**
- Source code: `src/rgw/radosgw-admin/radosgw-admin.cc` (usage() function, lines 140-557)
- Man page: `doc/man/8/radosgw-admin.rst`

---

## Critical Mismatches (Command Name Differs Between Source and Man Page)

| # | Source Command | Man Page Command | Man Page Line | Severity | Notes |
|---|---------------|-----------------|---------------|----------|-------|
| 1 | `role delete` | `role rm` | 452 | HIGH | Source uses `delete`, man page uses `rm`. Admin guide (`doc/radosgw/role.rst` line 67) confirms `role delete` is correct. |
| 2 | `role-trust-policy modify` | `role modify` | 461 | HIGH | Completely different command string. Admin guide (line 165) confirms `role-trust-policy modify` is correct. |
| 3 | `role-policy delete` | `role-policy rm` | 473 | HIGH | Source uses `delete`, man page uses `rm`. Admin guide (line 313) confirms `role-policy delete` is correct. |

---

## Commands in Source (--help) but MISSING from Man Page

### Account Management (6 commands -- entire feature undocumented)

| Command | Description |
|---------|-------------|
| `account create` | create a new account |
| `account modify` | modify an existing account |
| `account get` | get account info |
| `account stats` | dump account storage stats |
| `account rm` | remove an account |
| `account list` | list all account ids |

### Dedup (7 commands -- entire feature undocumented)

| Command | Description |
|---------|-------------|
| `dedup stats` | Display dedup statistics from the last run |
| `dedup estimate` | Runs dedup in estimate mode |
| `dedup exec` | Execute dedup |
| `dedup abort` | Abort dedup |
| `dedup pause` | Pause dedup |
| `dedup resume` | Resume paused dedup |
| `dedup throttle` | Throttle dedup execution |

### User Managed Policies (3 commands missing)

| Command | Description |
|---------|-------------|
| `user policy attach` | attach a managed policy |
| `user policy detach` | detach a managed policy |
| `user policy list attached` | list attached managed policies |

### Role Managed Policies (3 commands missing)

| Command | Description |
|---------|-------------|
| `role policy attach` | attach a managed policy |
| `role policy detach` | detach a managed policy |
| `role policy list attached` | list attached managed policies |
| `role update` | update max_session_duration of a role |

### MFA (6 commands missing)

| Command | Description |
|---------|-------------|
| `mfa create` | create a new MFA TOTP token |
| `mfa list` | list MFA TOTP tokens |
| `mfa get` | show MFA TOTP token |
| `mfa remove` | delete MFA TOTP token |
| `mfa check` | check MFA TOTP token |
| `mfa resync` | re-sync MFA TOTP token |

### Notifications (3 commands missing)

| Command | Description |
|---------|-------------|
| `notification list` | list bucket notifications configuration |
| `notification get` | get a bucket notifications configuration |
| `notification rm` | remove a bucket notifications configuration |

### Lua Scripting (7 commands missing)

| Command | Description |
|---------|-------------|
| `script put` | upload a Lua script to a context |
| `script get` | get the Lua script of a context |
| `script rm` | remove the Lua scripts of a context |
| `script-package add` | add a Lua package to the scripts allowlist |
| `script-package rm` | remove a Lua package from the scripts allowlist |
| `script-package list` | get the Lua packages allowlist |
| `script-package reload` | install/remove Lua packages according to allowlist |

### Rate Limiting (8 commands missing)

| Command | Description |
|---------|-------------|
| `ratelimit get` | get ratelimit params |
| `ratelimit set` | set ratelimit params |
| `ratelimit enable` | enable ratelimit |
| `ratelimit disable` | disable ratelimit |
| `global ratelimit get` | view global ratelimit params |
| `global ratelimit set` | set global ratelimit params |
| `global ratelimit enable` | enable a ratelimit quota |
| `global ratelimit disable` | disable a ratelimit quota |

### Bucket Enhancements (4 commands missing)

| Command | Description |
|---------|-------------|
| `bucket check olh` | check for olh index entries pending removal |
| `bucket check unlinked` | check for object versions not visible in listing |
| `bucket set-min-shards` | set minimum number of shards for dynamic resharding |
| `bucket sync checkpoint` | poll bucket sync status until caught up |

### Object (1 command missing)

| Command | Description |
|---------|-------------|
| `object put` | put object |

### Lifecycle (1 command missing)

| Command | Description |
|---------|-------------|
| `lc reshard fix` | fix LC for a resharded bucket |

### Reshard (2 subcommands missing)

| Command | Description |
|---------|-------------|
| `reshard stale-instances list` | list stale-instances from bucket resharding |
| `reshard stale-instances delete` | cleanup stale-instances from bucket resharding |

### Reshard Log (2 commands missing)

| Command | Description |
|---------|-------------|
| `reshardlog list` | list bucket resharding log |
| `reshardlog purge` | trim bucket resharding log |

### Metadata / Data Log Enhancements (6 commands missing)

| Command | Description |
|---------|-------------|
| `mdlog autotrim` | auto trim metadata log |
| `bilog autotrim` | auto trim bucket index log |
| `bilog status` | read bucket index log status |
| `datalog type` | change datalog type to --log_type={fifo,omap} |
| `datalog semaphore list` | List recovery semaphores |
| `datalog semaphore reset` | Reset recovery semaphore |

### Usage (1 command missing)

| Command | Description |
|---------|-------------|
| `usage clear` | reset all the usage stats for the cluster |

### Realm (1 command missing)

| Command | Description |
|---------|-------------|
| `realm default rm` | clear the current default realm |

### Zonegroup / Zone Placement (2 commands missing)

| Command | Description |
|---------|-------------|
| `zonegroup placement get` | get a placement target of a specific zonegroup |
| `zone placement get` | get a zone placement target |

---

## Commands in Man Page but MISSING from Source usage()

| # | Man Page Command | Notes |
|---|-----------------|-------|
| 1 | `object manifest` | Present in man page (line 156-157), not in source usage() -- needs verification if actually implemented |

---

## Option / Description Mismatches

| # | Option | Source says | Man page says | Severity |
|---|--------|------------|---------------|----------|
| 1 | `--quota-scope` | "scope of quota (bucket, user, account)" | "scope of quota (bucket, user)" | MEDIUM -- `account` scope missing from man page |
| 2 | `bucket check` | "check bucket index by verifying size and object count stats" | "Check bucket index." (very brief) | LOW |
| 3 | `--subscription` | Not mentioned in source usage() | Present in man page Options | LOW -- leftover from old pubsub model |
| 4 | `--event-id` | Not mentioned in source usage() | Present in man page Options | LOW -- leftover from old pubsub model |

---

## Summary

| Category | Count |
|---------|-------|
| Critical name mismatches (wrong command in man page) | 3 |
| Entire features completely undocumented | 4 (account, dedup, mfa, scripting) |
| Individual missing commands | ~50+ |
| Commands in man page but removed from source | ~1-3 (needs verification) |
| Option description mismatches | 4+ |
