# Task: Implement worker lazy eviction and TTL state management

- **ID**: `US-003-TASK-001`
- **Story ID**: `US-003`
- **Status**: `PENDING`
- **Change Type**: `FEATURE`
- **Depends On**: `US-002`
- **Target Files**: `src/actodis_store_worker.erl`, `src/actodis_store.erl`, `test/actodis_store_tests.erl`

## Description

Add expire_at timestamps and lazy eviction logic to store workers with injected clock support and unit tests.
