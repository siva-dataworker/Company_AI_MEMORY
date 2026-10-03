# Company AI Memory - Development Steering

## Development Workflow

### Spec-Driven Development

**All features follow this process:**

1. **Requirements Phase**: Create spec file in `.kiro/specs/`, define user stories and acceptance criteria
2. **Design Phase**: Document architecture decisions, data models, API contracts in spec
3. **Task Breakdown**: Break implementation into discrete, testable tasks
4. **Implementation**: Kiro works through tasks incrementally
5. **Review**: Validate against spec requirements before closing

**Spec files** are the source of truth for:
- What we're building and why
- Design decisions and tradeoffs
- Implementation status
- Open questions and blockers

### Kiro Steering

**Persistent project knowledge lives in `.kiro/steering/`**:
- Product vision and principles → `product.md`
- Architecture patterns and tech stack → `architecture.md`
- Development practices (this file) → `development.md`
- Security requirements → `security.md`
- Testing standards → `testing.md`

Update steering files when:
- Architectural decisions change
- New patterns are established
- Security or compliance requirements evolve
- Team learns better practices

### Project Structure

```
/
├── .kiro/
│   ├── specs/           # Feature specifications
│   └── steering/        # Project knowledge and standards
├── src/
│   ├── agents/          # Department agents (engineering, product, communication, document)
│   ├── main-agent/      # Query/retrieval agent
│   ├── memory/          # Memory storage and graph operations
│   ├── permissions/     # Permission service
│   ├── mcp-integrations/ # MCP server adapters
│   └── shared/          # Common utilities, types, schemas
├── infrastructure/      # AWS CDK or CloudFormation
├── tests/
│   ├── unit/
│   ├── integration/
│   └── property/        # Property-based tests
└── docs/                # Additional documentation
```

## Code Standards

### Language and Tooling
- **Primary Language**: Python 3.11+ (AWS Lambda support, rich AI/data ecosystem)
- **Type Hints**: Required for all function signatures
- **Linting**: ruff for linting and formatting
- **Dependency Management**: Poetry or uv

### Code Organization
- **Single Responsibility**: Each agent, function, module has one clear purpose
- **Explicit Over Implicit**: Clear function names, avoid magic
- **Dependency Injection**: Pass dependencies (DB clients, LLM clients) rather than global state
- **Error Handling**: Explicit exception types, never swallow errors silently

### Naming Conventions
- `snake_case` for functions, variables, files
- `PascalCase` for classes
- `SCREAMING_SNAKE_CASE` for constants
- Prefix private methods/functions with `_`

### Documentation
- Docstrings for all public functions and classes (Google style)
- Inline comments for non-obvious logic
- README.md in each major module explaining its purpose

## Dependency Management

**Add dependencies thoughtfully:**
- Prefer AWS SDK (boto3) for AWS services
- Use LangGraph/LangChain only where they add clear value
- Avoid large frameworks that increase cold start times
- Pin versions explicitly (no loose version ranges)

## Git Workflow

- **Branch Naming**: `feature/description`, `fix/description`, `refactor/description`
- **Commits**: Clear, present-tense messages ("Add permission service", not "Added permission service")
- **PRs**: Reference spec file or issue, include testing notes
- **Main Branch**: Always deployable, protected

## Local Development

**Environment Setup:**
1. Install Python 3.11+, poetry/uv
2. Configure AWS credentials (local profile or SSO)
3. Set up local DynamoDB or Neptune emulator for testing
4. Install MCP servers for systems you're integrating

**Running Locally:**
- Department agents: Invoke with sample events
- Main agent: Run queries against local/dev memory store
- Integration tests: Use localstack for AWS services

## Configuration Management

- **Environment Variables**: Use `.env` files (never commit), load with `python-dotenv`
- **AWS Parameters**: Store configuration in AWS Systems Manager Parameter Store
- **Secrets**: Store in AWS Secrets Manager, never in code or environment variables
- **Per-Environment**: Separate configs for dev, staging, production
