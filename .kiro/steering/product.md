# Company AI Memory - Product Steering

## Product Vision

Company AI Memory is an organizational-memory system that captures, structures, and retrieves institutional knowledge. The system provides employees with evidence-backed answers by connecting to authorized company systems and building a queryable memory graph.

## Core Capabilities

### Memory Capture
- **Automated Event Detection**: Specialized department agents monitor connected systems (GitHub, Jira, Slack, documents) and identify meaningful events
- **Structured Memory Types**: Decisions, risks, actions, people, projects, relationships, and other domain-specific entities
- **Rich Metadata**: Every memory includes provenance (source system, timestamps), permissions, and contextual relationships

### Memory Retrieval
- **Permission-Aware Queries**: Employees can only access memories they're authorized to see
- **Evidence-Backed Answers**: Main AI agent provides answers with citations and provenance trails
- **Relationship Navigation**: Traverse connections between people, projects, decisions, and events

### System Integration
- **MCP-Based Connectivity**: External systems connect through Model Context Protocol servers
- **Supported Systems**: GitHub (code, PRs, issues), Jira (tickets, sprints), Slack (conversations, decisions), document stores
- **Extensible Design**: New system integrations via additional MCP servers

## User Experience Principles

- **Trust Through Transparency**: Always show sources and reasoning paths
- **Respect Permissions**: Never surface unauthorized information
- **Contextual Relevance**: Prioritize recent, related, and high-signal memories
- **Low Friction**: Integrate naturally into existing workflows

## Non-Goals

- Real-time chat/messaging platform (use existing tools)
- Universal data warehouse (focus on high-value knowledge)
- Replacement for source systems (augment, don't replace)
- Public/external knowledge base (internal organizational use only)
