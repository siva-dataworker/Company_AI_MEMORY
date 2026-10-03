# Company AI Memory - Security Steering

## Security Principles

1. **Zero Trust**: Verify every request, never assume authorization
2. **Tenant Isolation**: Complete separation between organizations at data and compute layers
3. **Least Privilege**: Minimal permissions for every component and user
4. **Defense in Depth**: Multiple layers of security controls
5. **Audit Everything**: Comprehensive logging for security events and access

## Tenant Isolation

### Data Layer
- **Physical Separation**: Separate DynamoDB tables or Neptune graphs per tenant
- **Partition Keys**: Include tenant ID in all partition keys (belt-and-suspenders)
- **Encryption**: Separate KMS keys per tenant for data encryption at rest
- **Backups**: Per-tenant backup retention and recovery

### Compute Layer
- **Context Propagation**: Tenant ID in every Lambda invocation context
- **Pre-Flight Checks**: Validate tenant ID before any data access
- **Resource Tags**: Tag all AWS resources with tenant ID for cost tracking and audit
- **No Shared State**: Agents are stateless, no cross-tenant information leakage

### Network Layer
- **VPC Isolation**: Separate VPCs per tenant for sensitive deployments (or shared VPC with strict security groups)
- **API Gateway**: Tenant ID extracted from auth token, validated before routing
- **Private Endpoints**: Use VPC endpoints for AWS service access (avoid public internet)

## Authentication & Authorization

### User Authentication
- **Identity Provider**: AWS Cognito user pools, integrated with company SSO (SAML/OIDC)
- **Token-Based**: JWT tokens with tenant ID, user ID, and roles in claims
- **Session Management**: Short-lived access tokens (15 min), refresh tokens with rotation
- **MFA**: Enforce for admin users and sensitive operations

### Service Authentication
- **IAM Roles**: Lambda execution roles with minimal permissions per function
- **Service-to-Service**: API keys or IAM authentication for internal service calls
- **MCP Server Credentials**: Stored in Secrets Manager, rotated regularly, scoped per tenant

### Permission Model
- **User Permissions**: Sync from source systems (GitHub org, Jira project, Slack workspace)
- **Memory Permissions**: Inherit from source system + optional override
- **Permission Cache**: TTL-based cache (5 min) with invalidation on permission change events
- **Fail Closed**: If permission check fails or times out, deny access

## Data Security

### Data Classification
- **Sensitive**: Source code, private documents, DMs, strategic decisions
- **Confidential**: Project plans, feature roadmaps, team discussions
- **Internal**: General company information

All data treated as **Sensitive** by default.

### Encryption
- **At Rest**: AWS KMS encryption for all storage (DynamoDB, Neptune, S3)
- **In Transit**: TLS 1.3 for all network communication
- **Key Management**: Separate CMKs per tenant, automatic rotation enabled

### Data Retention
- **Memory Lifecycle**: Configurable retention per memory type (default: indefinite with audit trail)
- **Deletion Requests**: Support GDPR-style deletion (remove user data, anonymize memories)
- **Backup Retention**: 30-day point-in-time recovery, 1-year cold backup

## Secrets Management

- **No Hardcoded Secrets**: Never commit credentials, API keys, or tokens
- **AWS Secrets Manager**: Store all MCP server credentials, API keys
- **Secret Rotation**: Automatic rotation every 90 days where supported
- **Access Logging**: CloudTrail logs all secret retrievals

## API Security

### Input Validation
- **Schema Validation**: Validate all API inputs against JSON schemas
- **Sanitization**: Escape/sanitize user inputs before storage or LLM prompts
- **Size Limits**: Max request size, rate limiting per user/tenant
- **SQL/NoSQL Injection**: Use parameterized queries, ORM protections

### Rate Limiting
- **Per User**: 100 requests/minute for query API
- **Per Tenant**: 1000 requests/minute aggregate
- **Adaptive**: Increase limits for verified high-usage tenants

### CORS & CSP
- **CORS**: Restrict origins to approved domains only
- **CSP**: Content Security Policy headers to prevent XSS
- **HSTS**: Enforce HTTPS with Strict-Transport-Security headers

## Monitoring & Incident Response

### Security Monitoring
- **CloudWatch Logs**: All Lambda invocations, API requests, permission denials
- **AWS GuardDuty**: Threat detection for AWS account
- **CloudTrail**: All API calls to AWS services, especially IAM and data access
- **Alerting**: SNS/PagerDuty alerts for suspicious activity

### Audit Logging
Log all security-relevant events:
- Authentication attempts (success/failure)
- Permission grants/denials
- Memory access (who, what, when)
- Configuration changes
- MCP server credential usage

### Incident Response
1. **Detect**: Automated alerts for anomalies
2. **Contain**: Disable compromised tenant/user immediately
3. **Investigate**: Review audit logs, determine scope
4. **Remediate**: Rotate credentials, patch vulnerabilities
5. **Post-Mortem**: Document incident, update controls

## Compliance Considerations

- **GDPR**: Support data export, deletion requests, consent management
- **SOC 2**: Audit logging, access controls, encryption at rest/transit
- **HIPAA**: (If applicable) Additional PHI protections, BAAs with AWS
- **Regular Audits**: Quarterly security reviews, annual penetration testing

## Security Checklist for New Features

Before shipping any feature:
- [ ] Tenant ID validated in all data access paths
- [ ] Permission checks enforce least privilege
- [ ] Input validation prevents injection attacks
- [ ] Sensitive data encrypted at rest and in transit
- [ ] Security events logged to CloudWatch
- [ ] Error messages don't leak sensitive information
- [ ] Rate limiting applied to new endpoints
- [ ] Dependencies scanned for known vulnerabilities
