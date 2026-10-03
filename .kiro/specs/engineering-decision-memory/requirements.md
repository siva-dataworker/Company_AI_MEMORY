# Requirements Document: Engineering Decision Memory

## Introduction

Engineering Decision Memory is the first vertical slice of Company AI Memory. It demonstrates the complete workflow from ingesting authorized GitHub activity, through identifying meaningful technical decisions, to storing structured organizational memory and retrieving evidence-backed answers. This focused implementation establishes the core patterns for memory capture, storage, permission enforcement, and retrieval that will extend to other departments.

The system monitors GitHub repositories for engineering activity (pull requests, code reviews, architectural discussions), extracts technical decisions with full context and provenance, stores them as queryable memories with permission inheritance, and enables the main AI agent to answer questions about past engineering decisions with citations to source evidence.

## Glossary

- **Engineering_Agent**: Specialized department agent that processes GitHub events to identify and extract technical decisions
- **Main_Agent**: Query and retrieval agent that answers employee questions using stored memories
- **Memory**: Structured record of a decision, event, or entity with metadata, provenance, permissions, and relationships
- **Decision_Memory**: Memory type representing a technical decision with context, rationale, alternatives, and outcome
- **Provenance**: Complete audit trail linking a memory back to its source event in GitHub
- **Permission_Service**: Component that evaluates whether a user can access a specific memory
- **MCP_Server**: Model Context Protocol server that connects external systems (GitHub) to the system
- **Tenant**: Organization using the system (single tenant for this vertical slice)
- **Memory_Store**: Graph database storing memories and their relationships
- **Evidence**: Source material (PR descriptions, code comments, review discussions) supporting a memory

## Requirements

### Requirement 1: GitHub Event Ingestion

**User Story:** As the system, I want to receive GitHub events through MCP, so that I can process engineering activity.

#### Acceptance Criteria

1. WHEN a pull request is opened, THE MCP_Server SHALL emit a standardized event to the Engineering_Agent
2. WHEN a pull request is merged, THE MCP_Server SHALL emit a standardized event to the Engineering_Agent
3. WHEN a code review comment is added, THE MCP_Server SHALL emit a standardized event to the Engineering_Agent
4. THE MCP_Server SHALL include the complete event payload with PR title, description, code diff, comments, and metadata
5. THE MCP_Server SHALL include tenant_id, repository name, timestamp, and event source URL in every event
6. WHEN the GitHub API is unreachable, THE MCP_Server SHALL log the error and retry with exponential backoff
7. THE MCP_Server SHALL authenticate using credentials stored in AWS Secrets Manager

### Requirement 2: Decision Identification

**User Story:** As the Engineering_Agent, I want to identify which GitHub events contain meaningful technical decisions, so that I can extract decision memories.

#### Acceptance Criteria

1. WHEN a pull request contains architectural keywords (architecture, design, refactor, migration, deprecation), THE Engineering_Agent SHALL classify it as a potential decision
2. WHEN a pull request modifies infrastructure files (terraform, cloudformation, dockerfile, kubernetes manifests), THE Engineering_Agent SHALL classify it as a potential decision
3. WHEN a pull request has more than 3 review comments discussing tradeoffs or alternatives, THE Engineering_Agent SHALL classify it as a potential decision
4. WHEN a pull request is classified as a potential decision, THE Engineering_Agent SHALL use an LLM to determine if it represents a meaningful technical decision
5. THE Engineering_Agent SHALL extract decision context including the problem statement, chosen solution, rejected alternatives, and rationale
6. WHEN an event does not contain a meaningful decision, THE Engineering_Agent SHALL discard it without creating a memory

### Requirement 3: Decision Memory Structure

**User Story:** As the Engineering_Agent, I want to create structured decision memories, so that they can be stored and queried consistently.

#### Acceptance Criteria

1. THE Engineering_Agent SHALL create a Decision_Memory with fields: decision_id, title, summary, problem_statement, chosen_solution, alternatives_considered, rationale, outcome, timestamp, and provenance
2. THE Engineering_Agent SHALL populate provenance with source_system (github), source_event_type (pull_request), source_event_id (PR number), source_url (GitHub PR URL), and extraction_timestamp
3. THE Engineering_Agent SHALL extract participant information (decision_maker, reviewers, contributors) as entities with relationships to the Decision_Memory
4. THE Engineering_Agent SHALL infer relationships between the Decision_Memory and mentioned projects, technologies, and components
5. THE Engineering_Agent SHALL assign permissions to the Decision_Memory based on GitHub repository visibility (public, private, team-restricted)
6. THE Engineering_Agent SHALL validate the Decision_Memory schema before storage

### Requirement 4: Memory Storage

**User Story:** As the system, I want to store decision memories in a graph database, so that they can be queried with relationships.

#### Acceptance Criteria

1. WHEN the Engineering_Agent creates a Decision_Memory, THE Memory_Store SHALL store it as a node with all metadata fields
2. THE Memory_Store SHALL create relationship edges between the Decision_Memory and participant entities (DECIDED_BY, REVIEWED_BY, CONTRIBUTED_BY)
3. THE Memory_Store SHALL create relationship edges between the Decision_Memory and mentioned entities (AFFECTS_PROJECT, USES_TECHNOLOGY, IMPACTS_COMPONENT)
4. THE Memory_Store SHALL index memories by tenant_id, timestamp, decision_type, permissions, and keywords for fast retrieval
5. THE Memory_Store SHALL enforce tenant_id in all partition keys to ensure tenant isolation
6. WHEN storing a memory with invalid provenance, THE Memory_Store SHALL reject it and return an error
7. THE Memory_Store SHALL encrypt all data at rest using AWS KMS with tenant-specific keys

### Requirement 5: Permission Inheritance

**User Story:** As the system, I want memories to inherit permissions from their source systems, so that unauthorized users cannot access restricted information.

#### Acceptance Criteria

1. WHEN a GitHub repository is private, THE Engineering_Agent SHALL mark the Decision_Memory with permissions matching the repository access list
2. WHEN a GitHub repository is public, THE Engineering_Agent SHALL mark the Decision_Memory as accessible to all tenant users
3. WHEN a GitHub repository has team-based access, THE Engineering_Agent SHALL mark the Decision_Memory with the specific team permissions
4. THE Permission_Service SHALL sync GitHub repository permissions to the system on a configurable schedule (default: every 5 minutes)
5. WHEN GitHub permissions change for a repository, THE Permission_Service SHALL update all related memory permissions within the cache TTL window

### Requirement 6: Query Processing

**User Story:** As an employee, I want to ask questions about past engineering decisions, so that I can understand why technical choices were made.

#### Acceptance Criteria

1. WHEN an employee submits a query, THE Main_Agent SHALL extract the tenant_id and user_id from the authentication token
2. THE Main_Agent SHALL validate the authentication token and reject expired or invalid tokens
3. THE Main_Agent SHALL parse the natural language query to identify relevant keywords, entities, and time constraints
4. THE Main_Agent SHALL retrieve the user's current permission set from the Permission_Service
5. THE Main_Agent SHALL formulate a query plan identifying which memory types, relationships, and filters to apply

### Requirement 7: Permission-Aware Retrieval

**User Story:** As the Main_Agent, I want to retrieve only memories the user is authorized to access, so that unauthorized information never enters the model context.

#### Acceptance Criteria

1. WHEN retrieving memories, THE Main_Agent SHALL filter results to only memories where user permissions match or exceed memory permissions
2. THE Main_Agent SHALL apply permission filters at the database query level, not in post-processing
3. WHEN the Permission_Service is unreachable or times out, THE Main_Agent SHALL deny access and return an error
4. THE Main_Agent SHALL never cache or log unauthorized memory content
5. THE Main_Agent SHALL retrieve related entities and relationships only if the user has permission for the source memory

### Requirement 8: Memory Retrieval Strategy

**User Story:** As the Main_Agent, I want to retrieve relevant memories efficiently, so that I can answer queries with low latency.

#### Acceptance Criteria

1. THE Main_Agent SHALL use semantic similarity search on the query and memory summaries to identify candidate memories
2. THE Main_Agent SHALL apply graph traversal to find related memories within 2 relationship hops of high-scoring candidates
3. THE Main_Agent SHALL rank retrieved memories by relevance score, recency, and relationship strength
4. THE Main_Agent SHALL limit retrieval to the top 10 most relevant memories to control context size and cost
5. WHEN multiple memories have similar relevance, THE Main_Agent SHALL prefer more recent memories

### Requirement 9: Evidence-Backed Answer Synthesis

**User Story:** As the Main_Agent, I want to generate answers with citations to source evidence, so that employees can verify the information.

#### Acceptance Criteria

1. WHEN synthesizing an answer, THE Main_Agent SHALL cite specific Decision_Memory records by title and source_url
2. THE Main_Agent SHALL include relevant excerpts from the problem_statement, chosen_solution, and rationale fields
3. THE Main_Agent SHALL provide the timestamp and decision_maker for each cited memory
4. WHEN the answer references multiple related decisions, THE Main_Agent SHALL explain the relationships between them
5. WHEN no relevant memories exist, THE Main_Agent SHALL state that no information is available and SHALL NOT generate speculative answers
6. THE Main_Agent SHALL format citations as inline links with descriptive text

### Requirement 10: Answer Provenance

**User Story:** As an employee, I want to trace every fact in the answer back to source evidence, so that I can trust the information.

#### Acceptance Criteria

1. WHEN presenting an answer, THE Main_Agent SHALL include a provenance section listing all retrieved memories with their source URLs
2. THE Main_Agent SHALL indicate which parts of the answer came from which memories
3. THE Main_Agent SHALL include the retrieval timestamp and permission scope applied to the query
4. THE Main_Agent SHALL log the complete query execution (query, retrieved memories, permissions checked, answer generated) for audit purposes

### Requirement 11: Event Idempotency

**User Story:** As the system, I want to handle duplicate events gracefully, so that processing the same event multiple times produces consistent results.

#### Acceptance Criteria

1. WHEN the Engineering_Agent receives a duplicate GitHub event, THE Engineering_Agent SHALL detect it using the event_id and tenant_id combination
2. WHEN a duplicate event is detected for an existing memory, THE Engineering_Agent SHALL skip processing and return success
3. WHEN a duplicate event is detected with changed content (e.g., PR description updated), THE Engineering_Agent SHALL update the existing memory with the new content
4. THE Engineering_Agent SHALL maintain a deduplication cache with a 24-hour TTL
5. FOR ALL valid events, processing the event N times SHALL produce the same memory state as processing it once

### Requirement 12: Error Handling and Logging

**User Story:** As a system administrator, I want comprehensive error logging, so that I can diagnose and resolve issues.

#### Acceptance Criteria

1. WHEN any component encounters an error, THE component SHALL log the error to CloudWatch with severity level, tenant_id, user_id (if applicable), and full error context
2. WHEN the Engineering_Agent fails to process an event, THE Engineering_Agent SHALL log the event payload and error, then send the event to a dead letter queue
3. WHEN the Main_Agent fails to retrieve memories, THE Main_Agent SHALL return a user-friendly error message without exposing internal details
4. WHEN a permission check fails, THE Permission_Service SHALL log the user_id, requested memory_id, and denial reason
5. THE system SHALL emit CloudWatch metrics for event processing rate, memory creation rate, query latency, and error rates

### Requirement 13: Tenant Isolation

**User Story:** As the system, I want to enforce complete tenant isolation, so that one organization's data never leaks to another.

#### Acceptance Criteria

1. THE Memory_Store SHALL partition all data by tenant_id using separate DynamoDB tables or Neptune graph partitions
2. THE Engineering_Agent SHALL validate tenant_id from the event source before processing any event
3. THE Main_Agent SHALL validate tenant_id from the authentication token before executing any query
4. FOR ALL queries with tenant_id=A, THE Memory_Store SHALL return only data where tenant_id=A (property-based test with random tenant IDs and memory datasets)
5. THE Permission_Service SHALL include tenant_id in all cache keys to prevent cross-tenant cache poisoning

### Requirement 14: Performance Requirements

**User Story:** As an employee, I want fast query responses, so that I can get answers without disrupting my workflow.

#### Acceptance Criteria

1. WHEN a user submits a query, THE Main_Agent SHALL return an answer within 2000ms at p95 latency for queries retrieving up to 10 memories
2. WHEN the Engineering_Agent processes a GitHub event, THE Engineering_Agent SHALL complete processing within 5000ms at p95 latency
3. THE Memory_Store SHALL retrieve memories with permission filtering in under 100ms at p95 latency for queries matching fewer than 1000 memories
4. THE Permission_Service SHALL resolve user permissions from cache in under 10ms at p95 latency

### Requirement 15: Testing and Validation

**User Story:** As a developer, I want property-based tests for critical invariants, so that I can be confident the system behaves correctly.

#### Acceptance Criteria

1. THE test suite SHALL include a property-based test verifying tenant isolation (generate random tenant IDs, memories, and queries; assert no cross-tenant data leakage)
2. THE test suite SHALL include a property-based test verifying permission enforcement (generate random permission sets and memories; assert retrieval respects permissions)
3. THE test suite SHALL include a property-based test verifying event idempotency (generate random events and repetition counts; assert memory state is identical after 1 vs N processings)
4. THE test suite SHALL include a property-based test verifying provenance integrity (generate random memories; assert every memory has a valid source chain)
5. THE test suite SHALL include integration tests for the complete workflow (GitHub event → memory storage → query retrieval → answer synthesis)

