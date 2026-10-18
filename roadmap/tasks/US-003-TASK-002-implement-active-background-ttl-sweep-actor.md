# Task: Implement active background TTL sweep actor

- **ID**: `US-003-TASK-002`
- **Story ID**: `US-003`
- **Status**: `PENDING`
- **Change Type**: `FEATURE`
- **Depends On**: `US-003-TASK-001`
- **Target Files**: `src/actodis_ttl.erl`, `src/actodis_sup.erl`, `test/actodis_store_tests.erl`

## Description

Create the actodis_ttl gen_server actor to periodically sweep store workers and evict expired keys.
