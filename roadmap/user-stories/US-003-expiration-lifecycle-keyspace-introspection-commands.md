# US-003: Expiration Lifecycle And Keyspace Introspection Commands

## Metadata
- **Story ID:** US-003
- **Title:** Expiration Lifecycle And Keyspace Introspection Commands
- **Slug:** US-003-expiration-lifecycle-keyspace-introspection-commands
- **Domain Capability:** TTL Expiration & Keyspace Commands
- **Depends On:** ["US-002"]
- **Change Type:** new
- **Complexity:** 26 CU

## 1. Description
This story adds key expiration and keyspace introspection to the `actodis` key-value store.
Store workers track absolute expiration timestamps for each key.
The store worker removes expired keys when a client reads the key.
A dedicated sweep actor periodically checks shards and evicts expired keys.
The system implements the commands `EXPIRE`, `TTL`, `KEYS`, and `FLUSHALL`.
All time-dependent logic accepts an injected clock function for deterministic testing.

## 2. Observable Entry Point & Black-Box Protocol Contracts
Clients issue commands via RESP2 TCP frames.
The server processes requests and returns strict RESP2 wire-format replies.

### 2.1 Supported Commands & RESP Framing

| Command | Arguments | Success Envelope | Error Envelope |
| :--- | :--- | :--- | :--- |
| `EXPIRE` | `EXPIRE <key> <seconds>` | `:1\r\n` if set, `:0\r\n` if key absent | `-ERR wrong number of arguments for 'expire' command\r\n`<br>`-ERR value is not an integer or out of range\r\n` |
| `TTL` | `TTL <key>` | `:<seconds>\r\n` (remaining TTL)<br>`:-1\r\n` (key has no timeout)<br>`:-2\r\n` (key does not exist) | `-ERR wrong number of arguments for 'ttl' command\r\n` |
| `KEYS` | `KEYS <pattern>` | `*<count>\r\n$<len>\r\n<key1>\r\n...`<br>`*0\r\n` when no keys match | `-ERR wrong number of arguments for 'keys' command\r\n` |
| `FLUSHALL` | `FLUSHALL` | `+OK\r\n` | None |

### 2.2 Expiration Semantics
1. `EXPIRE <key> <seconds>` computes an absolute unix epoch timestamp: `CurrentEpochSeconds + Seconds`.
2. If `<seconds> <= 0`, the key expires immediately and the worker deletes it. The command returns `:1\r\n` if the key existed, else `:0\r\n`.
3. `TTL <key>` returns the remaining lifetime in seconds as a positive integer.
4. When a key has no associated expiration, `TTL` returns `:-1\r\n`.
5. When a key does not exist or has expired, `TTL` returns `:-2\r\n`.
6. `KEYS <pattern>` accepts standard glob wildcards (`*` matches any characters, `?` matches one character).
7. `FLUSHALL` clears all keys and expiration timers across all store shards immediately.

## 3. Pinned Modular Directory Layout
The following files define the required components and tests:

```
actodis/
├── src/
│   ├── actodis_ttl.erl           # Sweep actor for background key eviction
│   ├── actodis_store.erl         # Facade routing commands and cross-shard queries
│   ├── actodis_store_worker.erl  # Shard actor storing key values and expire_at epochs
│   └── actodis_commands.erl      # Command parser, validator, and response formatter
└── test/
    ├── actodis_store_tests.erl        # Unit tests with injected mock clock
    ├── actodis_commands_tests.erl     # Unit tests for EXPIRE, TTL, KEYS, FLUSHALL
    └── actodis_integration_tests.erl  # Black-box socket integration tests
```

## 4. Requirements & Implementation Details

### 4.1 Lazy Eviction on Store Workers
- Store entries in `actodis_store_worker` must retain an optional `expire_at` field (unix epoch seconds).
- Any read operation (`GET`, `EXISTS`, `TTL`, `KEYS`) checks `expire_at` against the current clock time.
- If `CurrentTime >= ExpireAt`, the worker deletes the key and treats it as absent.

### 4.2 Active Sweep Actor (`actodis_ttl`)
- Implement `actodis_ttl` as a supervised OTP `gen_server`.
- The actor runs an active sweep timer at a configurable interval (default: 100 milliseconds).
- During each tick, `actodis_ttl` sends a sweep request to each store worker.
- Workers evaluate a sample of keys and delete expired entries.

### 4.3 Keyspace Inspection (`KEYS`)
- `actodis_store:keys(Pattern)` queries all worker shards in parallel or sequence.
- Each shard performs lazy eviction on its keys before applying the pattern match.
- The pattern match checks key binaries using standard glob rules (`*` and `?`).
- The command returns a RESP array containing all unique matching keys.

### 4.4 Keyspace Purge (`FLUSHALL`)
- `actodis_store:flushall()` sends a purge message to all worker shards.
- Workers clear their internal storage maps and cancel pending key expiration timers.
- The command returns `+OK\r\n`.

### 4.5 Deterministic Time Injection
- `actodis_store_worker` and `actodis_ttl` accept a `clock_fun` option: `fun() -> integer() end`.
- The default clock is `fun() -> erlang:system_time(second) end`.
- Unit tests inject mock clock functions to test countdowns and expiration boundaries deterministically.

## 5. Comprehensive Scenario & Edge Case Matrix

| Scenario ID | Test Input | Mock Clock Action | Expected Output |
| :--- | :--- | :--- | :--- |
| `SC-EXP-01` | `SET k v`, `EXPIRE k 10` | Time = $T_0$ | Returns `:1\r\n`. `TTL k` returns `:10\r\n`. |
| `SC-EXP-02` | `EXPIRE missing 10` | Time = $T_0$ | Returns `:0\r\n`. |
| `SC-EXP-03` | `SET k v`, `EXPIRE k 10` | Advance clock to $T_0 + 6$ | `TTL k` returns `:4\r\n`. |
| `SC-EXP-04` | `SET k v`, `EXPIRE k 10` | Advance clock to $T_0 + 10$ | `GET k` returns `$-1\r\n`. `TTL k` returns `:-2\r\n`. |
| `SC-EXP-05` | `SET k v`, `EXPIRE k 0` | Time = $T_0$ | Returns `:1\r\n`. Key is immediately expired. `GET k` returns `$-1\r\n`. |
| `SC-EXP-06` | `SET k v`, `EXPIRE k -5` | Time = $T_0$ | Returns `:1\r\n`. Key is deleted immediately. |
| `SC-EXP-07` | `SET k v`, `TTL k` | No expiry set | Returns `:-1\r\n`. |
| `SC-EXP-08` | `TTL missing` | Key absent | Returns `:-2\r\n`. |
| `SC-EXP-09` | `SET k v EX 5`, `SET k new_v` | Time = $T_0 + 2$ | Setting key without EX removes TTL. `TTL k` returns `:-1\r\n`. |
| `SC-EXP-10` | `EXPIRE k invalid` | Non-integer argument | Returns `-ERR value is not an integer or out of range\r\n`. |
| `SC-EXP-11` | `EXPIRE` / `EXPIRE k` | Missing arguments | Returns `-ERR wrong number of arguments for 'expire' command\r\n`. |
| `SC-EXP-12` | `TTL` / `TTL a b` | Invalid arity | Returns `-ERR wrong number of arguments for 'ttl' command\r\n`. |
| `SC-KEY-01` | `SET user:1 a`, `SET user:2 b`, `SET order:1 c`, `KEYS user:*` | Time = $T_0$ | Returns array with `user:1` and `user:2`. |
| `SC-KEY-02` | `SET k1 a`, `SET k2 b`, `KEYS *` | Time = $T_0$ | Returns array containing both keys. |
| `SC-KEY-03` | `SET k1 a EX 2`, `KEYS *` | Advance clock to $T_0 + 3$ | Lazy eviction triggers. Returns `*0\r\n`. |
| `SC-KEY-04` | `KEYS missing:*` | No matches | Returns `*0\r\n`. |
| `SC-KEY-05` | `KEYS` / `KEYS a b` | Invalid arity | Returns `-ERR wrong number of arguments for 'keys' command\r\n`. |
| `SC-FLU-01` | Populate 10 keys across shards, `FLUSHALL` | N/A | Returns `+OK\r\n`. Next `KEYS *` returns `*0\r\n`. |
| `SC-FLU-02` | `FLUSHALL extra_arg` | Argument provided | Returns `-ERR wrong number of arguments for 'flushall' command\r\n`. |

## 6. Verification Criteria & Definition of Done
1. **Unit Test Pass Rate:** `rebar3 eunit` executes with 100% pass rate and zero failures.
2. **Mock Clock Verification:** Tests verify boundary transitions ($T_0$, $T_0 + TTL - 1$, $T_0 + TTL$, $T_0 + TTL + 1$) without sleep pauses.
3. **Active Sweep Verification:** Sweep actor triggers key eviction without explicit client read operations.
4. **Compilation Cleanliness:** `rebar3 compile` succeeds with zero compiler warnings under `warnings_as_errors`.
5. **RESP2 Compliance:** Command replies match standard Redis wire formats exactly.
6. **No Stubs Invariant:** Code contains no stub placeholders or shell error masks.

```noctifab-contract
{
  "story_id": "US-003",
  "public_contracts": [
    {
      "id": "RESP_EXPIRE_TTL_KEYS_FLUSHALL",
      "interface": "tcp_resp2",
      "allowed_executables": ["rebar3", "erl", "redis-cli"],
      "exit_codes": [0],
      "stdout_contains": ["actodis:"],
      "stderr_prefixes": ["actodis:"]
    }
  ]
}
```
