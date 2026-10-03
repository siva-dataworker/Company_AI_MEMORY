# Company AI Memory - Testing Steering

## Testing Philosophy

- **Test Important Invariants**: Focus on correctness of critical paths, not 100% coverage
- **Property-Based Testing**: Use Hypothesis for testing invariants (permissions, tenant isolation, data integrity)
- **Fast Feedback**: Unit tests run in <1s, integration tests in <10s
- **Confidence Over Coverage**: Better to have fewer high-value tests than many low-value tests

## Test Categories

### Unit Tests

**What to Unit Test:**
- Pure functions (data transformations, parsing, formatting)
- Business logic in agents (memory extraction, relationship inference)
- Permission evaluation logic
- Event schema validation

**What NOT to Unit Test:**
- AWS SDK calls (mock/stub in integration tests instead)
- Simple getters/setters
- Framework code

**Unit Test Standards:**
- Fast (<1s for entire suite)
- No external dependencies (no network, database, AWS services)
- Use mocks/stubs sparingly (prefer pure functions that don't need mocks)
- One logical assertion per test (can be multiple assert statements if testing same invariant)

**Tools:** pytest, unittest.mock

### Integration Tests

**What to Integration Test:**
- Department agent end-to-end (event → memory storage)
- Main agent end-to-end (query → memory retrieval → answer)
- MCP server interactions
- Database operations (DynamoDB, Neptune)
- Permission service with cached and fresh data

**Test Environment:**
- LocalStack for AWS services (DynamoDB, EventBridge, Secrets Manager)
- In-memory or Docker containers for databases
- Mock MCP servers with known test data

**Tools:** pytest, localstack, docker-compose, moto (AWS mocking)

### Property-Based Tests

**Use Hypothesis to test invariants that must ALWAYS hold:**

**Tenant Isolation:**
- Property: "No query with tenant_id=A returns data from tenant_id=B"
- Generate: Random tenant IDs, random queries, random memory datasets
- Assert: Results contain only correct tenant's data

**Permission Enforcement:**
- Property: "User with permissions P can access exactly memories with permissions ⊆ P"
- Generate: Random permission sets, random user permissions, random memories
- Assert: Retrieved memories respect permission rules

**Memory Provenance:**
- Property: "Every memory has a valid source chain back to an MCP event"
- Generate: Random memories with broken/valid provenance
- Assert: System rejects invalid provenance, accepts valid

**Event Idempotency:**
- Property: "Processing same event N times produces same result as processing once"
- Generate: Random events, random N (1-100)
- Assert: Memory state identical after 1 vs N processings

**Data Integrity:**
- Property: "Memory graph has no orphaned relationships"
- Generate: Random memory additions/deletions
- Assert: All relationship edges connect valid nodes

**Tools:** Hypothesis (Python property-based testing)

### End-to-End Tests

**Smoke Tests for Critical Paths:**
1. User authenticates → queries system → receives answer with citations
2. External event arrives → department agent processes → memory stored → queryable
3. Permission change event → permission cache invalidated → access updated

**Run against:**
- Staging environment (real AWS services, test tenant)
- Scheduled nightly, or on-demand before production deploys

**Tools:** pytest, requests, boto3

### Performance Tests

**Load Testing:**
- Main agent: 100 concurrent queries, <500ms p95 latency
- Department agents: 1000 events/minute ingestion rate
- Memory storage: 10k memories, <100ms retrieval p95

**Tools:** Locust, AWS Lambda load testing, CloudWatch metrics

## Testing Standards

### Test Organization
```
tests/
├── unit/
│   ├── test_agents.py
│   ├── test_memory.py
│   └── test_permissions.py
├── integration/
│   ├── test_agent_workflows.py
│   ├── test_mcp_integrations.py
│   └── test_storage.py
├── property/
│   ├── test_tenant_isolation.py
│   ├── test_permissions.py
│   └── test_data_integrity.py
└── e2e/
    └── test_critical_paths.py
```

### Test Naming
- `test_<functionality>_<scenario>_<expected_outcome>`
- Examples:
  - `test_engineering_agent_processes_pr_event_creates_memory`
  - `test_main_agent_denies_access_when_user_lacks_permission`
  - `test_tenant_isolation_prevents_cross_tenant_queries`

### Test Data
- **Fixtures**: Reusable test data in `tests/fixtures/`
- **Factories**: Use factory pattern for generating test objects (consider `factory_boy`)
- **Realistic Data**: Test with realistic event shapes, memory structures
- **Edge Cases**: Empty inputs, max sizes, special characters, Unicode

### Assertions
- Use descriptive assertion messages: `assert result == expected, f"Expected {expected}, got {result}"`
- Test both positive and negative cases
- For property tests, include shrinking (Hypothesis automatic)

## CI/CD Testing

### Pre-Commit
- Linting (ruff)
- Type checking (mypy)
- Fast unit tests (<5s)

### Pull Request
- All unit tests
- Integration tests
- Property-based tests (with lower iteration count for speed)
- Coverage report (informational, not blocking)

### Pre-Deploy
- Full test suite (unit + integration + property with high iterations)
- E2E smoke tests against staging
- Security scans (dependency vulnerabilities, code analysis)

### Post-Deploy
- E2E smoke tests against production (test tenant)
- Synthetic monitoring (canary queries every 5 min)

## Test Maintenance

- **Review tests** when feature changes (update or delete outdated tests)
- **Flaky tests**: Fix immediately or delete (don't tolerate flakiness)
- **Slow tests**: Optimize or move to nightly suite
- **Test coverage**: Track trends, but don't obsess over percentage

## Testing Checklist for New Features

Before marking a feature complete:
- [ ] Unit tests for core logic
- [ ] Integration test for happy path
- [ ] Property-based test for key invariant (if applicable)
- [ ] Negative test cases (invalid input, permission denial)
- [ ] Performance test for high-scale scenarios
- [ ] E2E test added to smoke suite
- [ ] All tests passing in CI/CD
