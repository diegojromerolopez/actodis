# US-002: Sharded Store Actors And String Numeric Commands

## Overview
Implement actodis_store facade with phash2 shard routing to supervised actodis_store_worker gen_server actors holding key/value state with injected clock_fun. Implement SET (with NX/XX/EX flags), GET, DEL, EXISTS, INCR, DECR wired through the command dispatcher to RESP replies.

## Domain Capability
Strings, Numeric Ops & Store Partitioning

## Dependencies
depends_on: ["US-001"]
change_type: "new"

## Definition of Done (DoD)
1. Observable Entry Point & Black-Box Protocol Contracts: Real working implementation with zero stubs or placeholders.
2. Standard I/O & Formatting: Real domain responses, exit code 0 for success, non-zero on error.
3. Verification Criteria: 100% passing unit, integration, and black-box E2E test suites with zero failures.
4. Modular File Paths:
   - Implementation: src/us_002/
   - Co-located Tests: tests/us_002/

```noctifab-contract
{
  "story_id": "US-002",
  "public_contracts": [{
    "id": "us-002.baseline",
    "interface": "CLI or Socket",
    "applicable_path_prefixes": ["src/"],
    "allowed_executables": ["make"],
    "exit_codes": [0],
    "stdout_contains": [],
    "stderr_prefixes": []
  }]
}
```
