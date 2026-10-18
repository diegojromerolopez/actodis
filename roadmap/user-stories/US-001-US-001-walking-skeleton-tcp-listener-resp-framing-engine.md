# US-001: Walking Skeleton TCP Listener And RESP Framing Engine

## Overview
Build the minimal runnable actodis node: rebar3 project scaffolding, Makefile targets, supervision tree, and TCP listener that spawns a client actor per connection. Implement the full RESP2 binary-pattern-matching encoder and streaming decoder with pipelining, inline commands, and chunked reassembly. Dispatch PING, ECHO, and QUIT as the first working command set and smoke test.

## Domain Capability
Core Networking & RESP Protocol

## Dependencies
depends_on: []
change_type: "new"

## Definition of Done (DoD)
1. Observable Entry Point & Black-Box Protocol Contracts: Real working implementation with zero stubs or placeholders.
2. Standard I/O & Formatting: Real domain responses, exit code 0 for success, non-zero on error.
3. Verification Criteria: 100% passing unit, integration, and black-box E2E test suites with zero failures.
4. Modular File Paths:
   - Implementation: src/us_001/
   - Co-located Tests: tests/us_001/

```noctifab-contract
{
  "story_id": "US-001",
  "public_contracts": [{
    "id": "us-001.baseline",
    "interface": "CLI or Socket",
    "applicable_path_prefixes": ["src/"],
    "allowed_executables": ["make"],
    "exit_codes": [0],
    "stdout_contains": [],
    "stderr_prefixes": []
  }]
}
```
