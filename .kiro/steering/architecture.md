# Company AI Memory - Architecture Steering

## Architectural Principles

1. **Event-Driven**: System responds to events from connected systems, avoiding polling where possible
2. **Multi-Agent**: Specialized agents for departments/domains rather than monolithic processing
3. **Permission-Native**: Authorization checks built into every layer, not bolted on
4. **Provenance-First**: Every memory maintains its source chain and audit trail
5. **Simple Over Clever**: Choose maintainable patterns over complex infrastructure

## System Architecture

### Core Components

**MCP Integration Layer**
- MCP servers for each external system (GitHub, Jira, Slack, documents)
- Standardized event schemas across systems
- Authentication/authorization per integration

**Department Agents** (Specialized Processors)
- Engineering Agent: Monitors GitHub, identifies code decisions, architectural changes, technical debt
- Product Agent: Monitors Jira, identifies feature decisions, prioritization changes, roadmap shifts
- Communication Agent: Monitors Slack, identifies key decisions, action items, consensus
- Document Agent: Monitors document stores, identifies policy changes, strategic decisions
- Each agent: stateless, idempotent, processes specific event types

**Memory Storage**
- Graph database for memories and relationships (consider AWS Neptune, or DynamoDB with GSIs)
- Schema: nodes (memories, entities) + edges (relationships)
- Indexed by: timestamp, entity type, permissions, source system, keywords
- Partition strategy: by tenant/organization for isolation

**Main AI Agent** (Query/Retrieval)
- Receives employee questions
- Permission filtering based on user identity
- Retrieves relevant memories via graph traversal + semantic search
- Constructs evidence-backed answers with citations
- Stateless request/response model

**Permission Service**
- Syncs authorization models from source systems
- Evaluates "can user X access memory Y?"
- Cached with TTL, refreshed on permission events

### Technology Stack (AWS-Focused)

- **Compute**: AWS Lambda for agents and API handlers (event-driven, scales to zero)
- **Storage**: DynamoDB or Neptune for memory graph (evaluate based on query patterns)
- **Events**: EventBridge for event routing between components
- **APIs**: API Gateway + Lambda for employee queries
- **Auth**: Cognito for user identity, integrate with company SSO
- **Secrets**: AWS Secrets Manager for MCP server credentials
- **Monitoring**: CloudWatch + X-Ray for observability

### LangGraph/LangChain Usage

Use **only** where they provide clear value:
- ✅ LangGraph for multi-step retrieval workflows (query planning, memory traversal, answer synthesis)
- ✅ LangChain for prompt templates and LLM interaction patterns
- ✅ LangSmith for tracing complex agent reasoning (debugging, evaluation)
- ❌ Do NOT use for simple API calls, basic data transformations, or where standard code is clearer

## Data Flow

1. **Ingestion**: External system event → MCP server → EventBridge → Department Agent → Structured Memory → Storage
2. **Query**: Employee question → Main Agent → Permission Check → Memory Retrieval → Answer Synthesis → Response with Citations

## Deployment Model

- **Multi-Tenant**: Single deployment serves multiple organizations with strict tenant isolation
- **Per-Tenant Resources**: Separate DynamoDB tables or Neptune graphs per tenant
- **Shared Infrastructure**: Lambda functions, API Gateway, EventBridge rules (tenant ID in context)

## Scalability Considerations

- Department agents process events asynchronously (decouple from source systems)
- Main agent caches frequently accessed memories (Redis/ElastiCache if needed)
- Graph queries optimized with appropriate indexes
- Rate limiting per tenant to prevent abuse
