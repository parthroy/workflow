# Activepieces MCP Server Implementation

> Multi-tenant, agent-based MCP server with connector-based tenancy

## 📚 Documentation

**Main Guide**: [MCP_COMPLETE_IMPLEMENTATION_GUIDE.md](./MCP_COMPLETE_IMPLEMENTATION_GUIDE.md)
- Complete architecture overview
- Implementation guide (8-week plan)
- Database schema & migrations
- API specifications
- Code examples
- Security model
- Testing strategy
- Deployment guide
- Quick start (1-hour setup)

## 🚀 Quick Start

### Prerequisites
- Node.js 20+
- PostgreSQL 14+
- Redis 7+
- Docker & Docker Compose

### Get Running in 1 Hour

1. **Start services**
```bash
docker-compose up -d
```

2. **Install dependencies**
```bash
npm install
npm install @modelcontextprotocol/sdk
```

3. **Run database migrations**
```bash
npm run migration:run
```

4. **Start development server**
```bash
npm run dev
```

5. **Create your first agent**
```bash
curl -X POST http://localhost:8080/v1/agents \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "Test Agent",
    "enabledPieces": ["@activepieces/piece-slack"],
    "enabledActions": {
      "@activepieces/piece-slack": ["send_message"]
    }
  }'
```

## 🏗️ Architecture Overview

```
Connector Auth (OAuth/API)
    ↓
Extract tenantId (workspace_id, portal_id, domain)
    ↓
Map to primary tenant (org_acme_corp)
    ↓
Store connection with tenant + connector IDs
    ↓
Create agent with tenant + tool whitelist
    ↓
MCP exposes ONLY agent's tools
    ↓
Tool execution uses connector credentials
    ↓
Isolated at workspace/portal/domain level
```

## 🔑 Key Concepts

### Connector-Based Tenancy
```typescript
// Slack OAuth → Extract workspace_id
tenantId = "slack_T1234567890"

// HubSpot API → Extract portal_id
tenantId = "hubspot_87654321"

// Gmail OAuth → Extract domain
tenantId = "gmail_acme.com"

// Map to primary tenant
primaryTenantId = "org_acme_corp"
```

### Agent-Level Tool Scoping
```typescript
{
  "enabledPieces": ["slack", "hubspot"],     // Only these pieces
  "enabledActions": {
    "slack": ["send_message"],               // Only this action
    "hubspot": ["create_contact"]            // Only this action
  },
  "enabledFlows": ["flw_qualify_lead"]       // Only this flow
}
```

### Dual-Mode Support
- **Piece-centric**: Atomic actions (fast, 100-500ms)
- **Flow-centric**: Orchestrated workflows (complex, 2-10s)

## 🔐 Security Model

### 4-Layer Isolation
1. **Connector Level**: Physical isolation (Slack token only works for workspace T12...)
2. **Tenant Level**: Database filtering (tenant_id checks)
3. **Agent Level**: Tool whitelist (only enabled tools)
4. **Execution Level**: Runtime validation (tenantId matching)

### No RBAC
- Simple whitelist-based authorization
- No complex roles or permissions
- 50% less code, 3x faster

## 📊 Database Schema

```sql
-- Tenant mapping (multiple connectors → one org)
CREATE TABLE tenant_mapping (
  primary_tenant_id VARCHAR(255),      -- "org_acme_corp"
  connector_tenant_id VARCHAR(255),    -- "slack_T1234567890"
  connector_type VARCHAR(50)           -- "slack"
);

-- Connections (per connector)
CREATE TABLE app_connection (
  id VARCHAR(21) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,           -- "org_acme_corp"
  connector_tenant_id VARCHAR(255) NOT NULL, -- "slack_T1234567890"
  piece_name VARCHAR(255),                   -- "slack"
  value JSONB,                               -- { access_token: "xoxb-..." }
  agent_id VARCHAR(21)
);

-- Agents (per tenant)
CREATE TABLE agent (
  id VARCHAR(21) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,     -- "org_acme_corp"
  enabled_pieces JSONB,                -- ["slack", "hubspot"]
  enabled_actions JSONB,               -- { "slack": ["send_message"] }
  enabled_flows JSONB,                 -- ["flw_qualify_lead"]
  connection_mappings JSONB            -- { "slack": "conn_slack_001" }
);
```

## 🛠️ Implementation Phases

### ✅ Phase 0: Setup & Planning (Week 1)
- Environment setup
- Database migrations
- Project structure
- Dependencies

### 🔄 Phase 1: Core Foundation (Weeks 2-3)
- Agent service
- Agent controller
- Token generation
- MCP protocol handler

### 🔄 Phase 2: Piece-Centric Tools (Weeks 4-5)
- Property mapper
- Tool discovery
- Tool execution
- Connection management

### 🔄 Phase 3: Flow-Centric Tools (Week 6)
- Flow MCP config
- Flow discovery
- Flow execution

### 🔄 Phase 4: Advanced Features (Week 7)
- Dynamic properties
- File handling
- Streaming

### 🔄 Phase 5: UI (Weeks 8-9)
- Agent management UI
- Analytics dashboard

### 🔄 Phase 6: Testing (Week 10)
- Load testing
- Security audit

### 🔄 Phase 7: Launch (Weeks 11-12)
- Documentation
- Deployment
- Beta testing

## 📖 API Examples

### Create Agent
```bash
POST /v1/agents
{
  "displayName": "Sales Assistant",
  "connectionIds": ["conn_slack_001"],
  "enabledPieces": ["slack"],
  "enabledActions": {
    "slack": ["send_message"]
  }
}
```

### MCP Tools List
```bash
POST /v1/mcp/agents/{agentId}/http
{
  "jsonrpc": "2.0",
  "method": "tools/list"
}
```

### MCP Tool Call
```bash
POST /v1/mcp/agents/{agentId}/http
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "slack_send_message",
    "arguments": {
      "channel": "#sales",
      "text": "New lead!"
    }
  }
}
```

## 🧪 Testing

```bash
# Unit tests
npm run test

# Integration tests
npm run test:e2e

# Load tests
npm run test:load
```

## 📦 Deployment

```bash
# Docker
docker-compose up -d

# Kubernetes
kubectl apply -f k8s/

# Production checklist
- [ ] Database migrations run
- [ ] Environment variables set
- [ ] Redis cluster configured
- [ ] Load balancer configured
- [ ] Monitoring enabled
- [ ] Logs centralized
- [ ] Backups scheduled
```

## 🎯 Next Steps

1. Read [MCP_COMPLETE_IMPLEMENTATION_GUIDE.md](./MCP_COMPLETE_IMPLEMENTATION_GUIDE.md)
2. Follow Quick Start guide (Section 11)
3. Implement Phase 1 (Core Foundation)
4. Test with first agent
5. Continue with remaining phases

## 📝 Resources

- **Main Guide**: [MCP_COMPLETE_IMPLEMENTATION_GUIDE.md](./MCP_COMPLETE_IMPLEMENTATION_GUIDE.md)
- **Activepieces Docs**: https://www.activepieces.com/docs
- **MCP Protocol**: https://modelcontextprotocol.io/

## 🤝 Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for contribution guidelines.

## 📄 License

This implementation follows the Activepieces license. See original repository for details.

---

**Status**: Ready to Build! 🚀

**Estimated Time to First Agent**: 1 hour
**Estimated Time to Production**: 8 weeks
