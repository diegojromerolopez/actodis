# US-005: Project Hardening, Maintenance Readiness & Code Quality

## 1. Story Metadata
- **ID**: US-005
- **Title**: Project Hardening, Maintenance Readiness & Code Quality
- **Slug**: US-005-project-hardening-maintenance-readiness-code-quality
- **Domain Capability**: Hardening & Release Readiness
- **Depends On**: ["US-001", "US-002", "US-003", "US-004"]
- **Change Type**: "new"
- **Complexity**: 20 CU

---

## 2. Executive Summary
This story validates system robustness, code quality, and release readiness. The story enforces static analysis with Dialyzer. It also enforces compiler settings with zero warnings. The story configures the containerized black-box test harness. The harness runs end-to-end command tests with official `redis-cli`. Documentation and operational guides receive final validation.

---

## 3. User Story Statement
As a systems operator and release engineer,
I want automated static type validation, clean compilation, and a containerized test harness,
So that actodis reliably handles Redis client traffic without runtime type crashes or regressions.

---

## 4. Pinned Modular Directory Layout & Required File Paths
All components must exist in the exact paths below:

```text
actodis/
├── rebar.config                     # Compiler flags, dialyzer configuration, eunit profile
├── Makefile                         # Required build, test, lint, e2e, clean targets
├── README.md                        # Deployment, architecture, command reference
├── docker-compose.yml               # Multi-service container test definition
├── src/
│   ├── actodis.app.src              # OTP application metadata
│   ├── actodis_app.erl              # Application entrypoint with typespecs
│   ├── actodis_sup.erl              # Root supervisor with typespecs
│   ├── actodis_listener.erl         # Non-blocking socket listener actor with typespecs
│   ├── actodis_client.erl           # TCP connection client actor with typespecs
│   ├── actodis_resp.erl             # Streaming RESP codec with typespecs
│   ├── actodis_commands.erl         # Command parser and envelope validator with typespecs
│   ├── actodis_store.erl            # Store routing facade with typespecs
│   ├── actodis_store_worker.erl     # In-memory worker gen_server with typespecs
│   ├── actodis_ttl.erl              # Active TTL eviction actor with typespecs
│   └── actodis_aof.erl              # Append-Only File durability actor with typespecs
├── test/
│   ├── actodis_resp_tests.erl       # Unit tests for RESP protocol engine
│   ├── actodis_store_tests.erl      # Unit tests for partitioned store and TTL
│   ├── actodis_commands_tests.erl   # Unit tests for Redis command semantics
│   ├── actodis_aof_tests.erl        # Unit tests for AOF persistence engine
│   └── actodis_integration_tests.erl# Integration tests with concurrent client actors
└── tests/
    └── e2e/
        ├── Dockerfile               # Alpine client container running official redis-cli
        └── run_tests.sh             # Executable black-box integration script
```

---

## 5. Observable Entry Point & Black-Box Protocol Contracts

### 5.1 Build & Static Analysis CLI Targets
The repository Makefile must expose these exact command targets:
1. `make build`:
   - Executes `rebar3 compile`.
   - Compiles all source files with `warnings_as_errors`.
   - Returns exit code `0`.
2. `make lint`:
   - Executes `rebar3 dialyzer`.
   - Analyzes all modules for type errors, unmatched returns, and contract violations.
   - Returns exit code `0` with zero warnings.
3. `make test`:
   - Executes `rebar3 eunit`.
   - Runs all unit and integration test suites.
   - Returns exit code `0` with zero failures.
4. `make e2e`:
   - Executes `docker compose down -v && docker compose up --build --abort-on-container-exit --exit-code-from e2e`.
   - Runs the black-box Redis client test suite against the live service.
   - Returns exit code `0`.
5. `make clean`:
   - Removes build artifacts, `_build/`, `.rebar3/`, and temporary test databases.
   - Returns exit code `0`.

### 5.2 Diagnostic Log Format
Every log emission written by actodis to standard output or standard error must start with the prefix `actodis: `.

### 5.3 Exit Status Codes
- Clean termination returns exit code `0`.
- Configuration error or runtime bind failure returns exit code `1` and prints `actodis: <reason>` to stderr.

---

## 6. Static Analysis & Type Contract Requirements
Every source file in `src/` must contain valid Erlang `-spec` declarations for all public functions.

### 6.1 `rebar.config` Dialyzer Configuration
The `rebar.config` file must contain these settings:
```erlang
{erl_opts, [debug_info, warnings_as_errors]}.
{eunit_opts, [verbose]}.
{dialyzer, [
    {warnings, [
        unmatched_returns,
        error_handling,
        underspecs
    ]},
    {plt_extra_apps, [kernel, stdlib]}
]}.
```

### 6.2 Negative Constraint on Type Silencing
Developers must not disable Dialyzer warnings using compiler directives or type wildcards.
All function arguments, state records, and reply envelopes must declare precise types.

---

## 7. Docker Compose & End-to-End Black-Box Harness

### 7.1 Compose Specification (`docker-compose.yml`)
`docker-compose.yml` must define two container services:
1. `api` service:
   - Built from repository root `Dockerfile`.
   - Starts actodis with `rebar3 shell --apps actodis`.
   - Sets `PORT=6379`, `ACTODIS_DATA_DIR=/data`, `ACTODIS_AOF_FSYNC=true`.
   - Mounts a shared persistent volume at `/data`.
2. `e2e` service:
   - Built from `./tests/e2e/Dockerfile`.
   - Depends on `api`.
   - Sets `REDIS_URL=redis://api:6379`.
   - Runs `/tests/run_tests.sh`.

### 7.2 Client Container (`tests/e2e/Dockerfile`)
The E2E test image must use `alpine:3.21`.
It must install `bash` and `redis` (`redis-cli`).
It must copy `tests/e2e/run_tests.sh` to `/tests/run_tests.sh`.

### 7.3 Test Script Protocol (`tests/e2e/run_tests.sh`)
The script must execute these exact steps without static sleep delays:
1. **Readiness Polling Loop**:
   Poll `redis-cli -u ${REDIS_URL} ping` every 100 milliseconds for up to 10 seconds.
   Continue when the server returns `PONG`.
2. **Command Verification Sequence**:
   - `PING`: assert output equals `PONG`.
   - `PING "hello world"`: assert output equals `hello world`.
   - `SET k1 v1`: assert output equals `OK`.
   - `GET k1`: assert output equals `v1`.
   - `SET k2 v2 NX`: assert output equals `OK`.
   - `SET k2 v2 NX`: assert output equals empty nil.
   - `SET k3 v3 EX 1`: assert output equals `OK`.
   - Wait 1100 milliseconds for expiration.
   - `GET k3`: assert output equals empty nil.
   - `TTL k3`: assert output equals `-2`.
   - `EXISTS k1 k2 missing`: assert output equals `2`.
   - `INCR counter`: assert output equals `1`.
   - `INCR counter`: assert output equals `2`.
   - `DECR counter`: assert output equals `1`.
   - `KEYS *`: assert output contains `k1`, `k2`, and `counter`.
   - `DEL k1 k2 counter`: assert output equals `3`.
   - `FLUSHALL`: assert output equals `OK`.
3. **Failure Handling**:
   Any mismatched assertion immediately prints an error message and terminates with exit code `1`.
   Successful completion of all steps terminates with exit code `0`.

---

## 8. Anti-Stub & Anti-Gaming Invariants
1. Placeholders like `pass`, `...`, and empty catch blocks are forbidden.
2. Shell commands must not mask errors with `|| true` or `|| exit 0`.
3. The E2E test script must verify real responses from the socket.
4. Mock tests must verify state mutations using In-Memory Test Doubles rather than mocked return stubs.

---

## 9. Comprehensive Scenario & Edge Case Matrix

| Scenario ID | Test Scope | Trigger / Input | Expected Result |
| :--- | :--- | :--- | :--- |
| SC-005-01 | Dialyzer Check | `make lint` | Zero errors and zero warnings across all modules. |
| SC-005-02 | Compiler Check | `make build` with `warnings_as_errors` | Clean compilation of all modules with exit code 0. |
| SC-005-03 | Full Unit Suite | `make test` | All unit tests in `test/*_tests.erl` pass with 100% success rate. |
| SC-005-04 | Socket Polling | `run_tests.sh` readiness loop | Successfully connects when the TCP port opens within timeout. |
| SC-005-05 | Command Conformance | `run_tests.sh` command suite | All Redis commands match standard Redis 7 CLI output. |
| SC-005-06 | Persistence Mount | Container restarts with mounted `/data` | Previously stored data remains accessible after restart. |
| SC-005-07 | Documentation Sanity | Readme examples | Documented commands match actual operational behavior. |

---

## 10. Verification Criteria
1. `make build` completes with exit code `0`.
2. `make lint` executes `rebar3 dialyzer` and exits with code `0`.
3. `make test` executes all EUnit tests and exits with code `0`.
4. `make e2e` builds containers, executes `run_tests.sh`, and exits with code `0`.
5. `README.md` documents build commands, configuration parameters, and supported operations.

---

## 11. Definition of Done (DoD)
- [ ] All Erlang modules in `src/` have `-spec` declarations for public functions.
- [ ] Dialyzer runs with zero warnings under `unmatched_returns`, `error_handling`, and `underspecs`.
- [ ] Docker Compose setup starts actodis and executes `run_tests.sh` successfully.
- [ ] `run_tests.sh` uses socket readiness polling and covers all 13 supported commands.
- [ ] All Makefile targets (`build`, `run`, `test`, `lint`, `e2e`, `clean`) operate as specified.
- [ ] The README file describes configuration, runtime options, and architectural flow.

---

```noctifab-contract
{
  "story_id": "US-005",
  "public_contracts": [
    {
      "id": "contract-make-lint",
      "interface": "cli",
      "allowed_executables": ["make", "rebar3"],
      "exit_codes": [0],
      "stdout_contains": [],
      "stderr_prefixes": []
    },
    {
      "id": "contract-make-test",
      "interface": "cli",
      "allowed_executables": ["make", "rebar3"],
      "exit_codes": [0],
      "stdout_contains": ["Passed:"],
      "stderr_prefixes": []
    },
    {
      "id": "contract-make-e2e",
      "interface": "cli",
      "allowed_executables": ["make", "docker"],
      "exit_codes": [0],
      "stdout_contains": ["All E2E tests passed"],
      "stderr_prefixes": []
    }
  ]
}
```
