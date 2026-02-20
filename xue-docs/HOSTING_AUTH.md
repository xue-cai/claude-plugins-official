# Hosting Claude Code as a Service — Authentication

> **Scenario**: You host Claude Code in a cloud sandbox (Docker container, cloud VM, or managed service) and want colleagues to use it — possibly connected to company data sources like a knowledge base, CRM, email, chat history, or production database. How should authentication be configured?

This document covers three layers of authentication: how users access the hosted instance (Layer 1), how Claude Code authenticates with the AI model provider (Layer 2), and how the hosted instance authenticates with your company's MCP servers and data sources (Layer 3). For MCP auth patterns in the plugin ecosystem more broadly, see [PLUGIN_ANALYSIS.md](./PLUGIN_ANALYSIS.md#mcp-server-authentication).

## Three Authentication Layers

When hosting Claude Code for a team with connected data sources, there are **three distinct auth layers** to configure:

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: USER → HOSTED CLAUDE CODE                 │
│  (How colleagues access the sandbox)                │
│  SSH keys, SSO, VPN, or web-based access            │
├─────────────────────────────────────────────────────┤
│  Layer 2: CLAUDE CODE → AI MODEL PROVIDER           │
│  (How Claude Code calls the LLM)                    │
│  API key, IAM role, OAuth, or service account       │
├─────────────────────────────────────────────────────┤
│  Layer 3: CLAUDE CODE → MCP SERVERS (DATA SOURCES)  │
│  (How Claude Code connects to your company services)│
│  OAuth, Bearer tokens, env vars, service accounts   │
└─────────────────────────────────────────────────────┘
```

## Layer 2: Claude Code → Model Provider Authentication

Claude Code supports **five deployment backends**, each with different auth mechanisms:

### Option A: Anthropic API (Direct)

The simplest setup — use an Anthropic API key:

```bash
export ANTHROPIC_API_KEY="sk-ant-api03-..."
```

| Aspect | Details |
|--------|---------|
| **Auth method** | API key (`ANTHROPIC_API_KEY` env var) |
| **Per-user attribution** | None — shared key means shared billing |
| **Team management** | Claude for Teams / Claude for Enterprise plans offer SSO, RBAC, and admin tools |
| **Best for** | Quick setup, small teams, development/testing |
| **Security risk** | High if key leaks — use env vars, never hardcode |

**For team use**: Prefer Claude for Enterprise (see [enterprise deployment overview](https://code.claude.com/docs/en/third-party-integrations)) for SSO (Okta, Azure AD, Google), RBAC, audit logging, and centralized billing.

### Option B: Amazon Bedrock

Route Claude Code through AWS, using IAM for auth:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-east-1
export AWS_PROFILE=your-sso-profile  # if using SSO
```

| Aspect | Details |
|--------|---------|
| **Auth method** | AWS IAM (credentials chain: env vars → SSO → instance profile) |
| **Per-user attribution** | Complete with IAM Identity Center / IdP federation |
| **Team management** | AWS IAM roles + SSO via Okta, Azure AD, Auth0, or Cognito |
| **Best for** | AWS-centric teams, enterprise compliance, cost tracking |
| **Required IAM permissions** | `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`, `bedrock:ListInferenceProfiles` |

**Recommended for production teams**: Use Direct IdP Integration (OIDC federation) for:
- Seamless SSO with your identity provider
- MFA enforcement
- Per-user cost allocation via CloudWatch/OpenTelemetry
- Temporary credentials (no long-lived API keys)

**Automation**: Use the [official CloudFormation template](https://github.com/aws-solutions-library-samples/guidance-for-claude-code-with-amazon-bedrock) to bootstrap IAM roles, OIDC provider, and monitoring.

### Option C: Google Vertex AI

Route Claude Code through GCP, using Google Cloud auth:

```bash
export CLAUDE_CODE_USE_VERTEX=1
export CLOUD_ML_REGION=us-east5
export ANTHROPIC_VERTEX_PROJECT_ID=your-project-id
```

| Aspect | Details |
|--------|---------|
| **Auth method** | Google Cloud SDK (`gcloud auth`) or service account JSON key |
| **Per-user attribution** | Via GCP IAM roles and billing labels |
| **Team management** | GCP IAM + Google Workspace SSO |
| **Best for** | GCP-centric teams, teams already using Google Cloud |
| **Required IAM role** | `Vertex AI User` (minimum) |

**No Anthropic API key needed** — auth and billing are unified through Google Cloud.

### Option D: Microsoft Azure AI Foundry

Route Claude Code through Azure, using Entra ID:

```bash
export CLAUDE_CODE_USE_FOUNDRY=1
export ANTHROPIC_FOUNDRY_RESOURCE=your-resource-name
# Auth Option 1: API key
export ANTHROPIC_FOUNDRY_API_KEY="your-key"
# Auth Option 2: Entra ID (recommended for teams)
az login
```

| Aspect | Details |
|--------|---------|
| **Auth method** | API key OR Azure Entra ID (Azure SDK credential chain) |
| **Per-user attribution** | Complete with Entra ID |
| **Team management** | Azure RBAC, Entra ID groups, Conditional Access |
| **Best for** | Azure-centric teams, Microsoft 365 organizations |
| **Required Azure roles** | `Azure AI User`, `Cognitive Services User` |

**Recommended**: Use Entra ID (`az login`) instead of API keys for enterprise security.

### Option E: Docker Sandbox (Self-Hosted)

Run Claude Code in a Docker container with any of the above backends:

```bash
docker run -it --rm \
  -e ANTHROPIC_API_KEY \
  -v "$(pwd)/.claude-state:/home/sandbox/state" \
  -v "$(pwd):/home/sandbox/workspace" \
  claude-sandbox
```

| Aspect | Details |
|--------|---------|
| **Auth method** | Depends on backend (API key, IAM, etc. — passed as env vars) |
| **Sandboxing** | Filesystem isolation + network isolation (whitelisted domains only) |
| **State persistence** | Mount `.claude-state` volume for credentials/history |
| **Best for** | Self-hosted, CI/CD, maximum control over environment |

## Layer 1: User → Hosted Instance Access

How your colleagues actually reach the hosted Claude Code instance:

| Access Method | How It Works | Best For |
|---------------|-------------|----------|
| **SSH** | Each user SSHs into the sandbox container/VM with their own key | Small teams, developers comfortable with terminal |
| **VPN + SSH** | VPN for network access, SSH for container access | Corporate networks, security-conscious orgs |
| **Web terminal** (e.g., Wetty, ttyd) | Browser-based terminal over HTTPS | Teams without SSH access, non-technical colleagues |
| **Claude Code in VS Code** | Remote SSH extension connects to hosted sandbox | Teams using VS Code |
| **Cloud provider console** | AWS Cloud9, GCP Cloud Shell, Azure Cloud Shell | Teams already using cloud provider tooling |
| **Managed platforms** | Modal, Cloudflare Sandboxes, or `clwd` CLI for automated provisioning | Rapid team onboarding, managed infrastructure |

## Comparison: Which Backend to Choose?

| Feature | Anthropic Direct | Amazon Bedrock | Google Vertex AI | Azure AI Foundry | Docker Self-Hosted |
|---------|-----------------|----------------|------------------|------------------|--------------------|
| **Setup time** | Minutes | Hours | Hours | Hours | Minutes–Hours |
| **Auth method** | API key | IAM / SSO / IdP | Google Cloud auth | Entra ID / API key | Any (env vars) |
| **SSO support** | Enterprise plan | ✅ IAM Identity Center | ✅ Google Workspace | ✅ Entra ID | Depends on backend |
| **Per-user billing** | Enterprise plan | ✅ CloudWatch | ✅ GCP Billing | ✅ Cost Management | Manual |
| **MFA** | Enterprise plan | ✅ | ✅ | ✅ Conditional Access | Depends on backend |
| **Audit logging** | Enterprise plan | ✅ CloudTrail | ✅ Cloud Audit Logs | ✅ Azure Monitor | Manual |
| **Data residency** | Anthropic-managed | AWS regions | GCP regions | Azure regions | Your infrastructure |
| **Network isolation** | N/A | VPC | VPC | VNet | Docker networking |
| **Best for** | Quick start | AWS teams | GCP teams | Azure teams | Maximum control |

## Best Practices for Hosting Claude Code for a Team

1. **Never share a single API key across users** — Use IAM/SSO-based auth (Bedrock, Vertex, Foundry) for per-user attribution, audit trails, and cost allocation

2. **Pin model versions** to prevent unexpected behavior:
   ```bash
   export ANTHROPIC_DEFAULT_SONNET_MODEL="claude-sonnet-4-20250514"
   ```

3. **Use sandbox isolation** — Docker containers with filesystem and network restrictions prevent accidental damage

4. **Separate state from workspace** — Mount `.claude-state` as a separate volume from the working directory

5. **Add sensitive directories to `.gitignore`** — Credential/state directories should never be committed

6. **Use your cloud provider's secret management** — AWS Secrets Manager, GCP Secret Manager, Azure Key Vault — never embed credentials in Dockerfiles or scripts

7. **For enterprise**: Use Claude for Enterprise (Anthropic direct) or cloud provider federation (Bedrock/Vertex/Foundry) for SSO, RBAC, and compliance

## Quick Start: Host for Your Team (Recommended Path)

**Smallest team (2–5 people, quick setup)**:
1. Get a Claude for Teams plan from Anthropic
2. Each colleague installs Claude Code locally and logs in with their own account
3. SSO and billing are handled by Anthropic

**Medium team (5–50 people, AWS-centric)**:
1. Set up Amazon Bedrock with IAM Identity Center
2. Create IAM roles for developers with Bedrock permissions
3. Deploy [official CloudFormation stack](https://github.com/aws-solutions-library-samples/guidance-for-claude-code-with-amazon-bedrock) for automated setup
4. Developers run `aws sso login` then use Claude Code with `CLAUDE_CODE_USE_BEDROCK=1`

**Shared cloud sandbox (any size, maximum control)**:
1. Provision Docker containers on your cloud provider
2. Configure backend auth (Bedrock/Vertex/Foundry) via env vars
3. Provide SSH or web terminal access to each colleague
4. Use separate state volumes per user for session isolation

---

## Layer 3: MCP Server Authentication for Company Data Sources

This is the layer most relevant when building a **domain-specific assistant** — connecting your hosted Claude Code instance to internal company services via MCP (Model Context Protocol) servers.

### The Architecture

```
                           ┌─────────────────────────────────┐
                           │     HOSTED CLAUDE CODE          │
                           │     (Docker / VM / Cloud)       │
                           │                                 │
                           │  ┌───────────────────────────┐  │
  Users ──SSH/Web──────────┤  │  .mcp.json                │  │
                           │  │  (defines all MCP servers) │  │
                           │  └──────┬──┬──┬──┬──┬────────┘  │
                           └─────────┼──┼──┼──┼──┼───────────┘
                                     │  │  │  │  │
                    ┌────────────────┘  │  │  │  └──────────────────┐
                    │         ┌────────┘  │  └────────┐            │
                    ▼         ▼           ▼           ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
              │Knowledge │ │  Chat    │ │  Email   │ │Production│ │  CRM /   │
              │Base MCP  │ │ History  │ │ History  │ │   DB     │ │ Tickets  │
              │ Server   │ │ MCP Srv  │ │ MCP Srv  │ │ MCP Srv  │ │ MCP Srv  │
              └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
                 (HTTP)       (HTTP)       (HTTP)      (stdio)      (HTTP)
```

### Example: Customer 360 Assistant

Suppose you want a hosted Claude Code instance that acts as a **Customer 360 assistant** — answering questions about any customer by pulling data from multiple internal systems:

- **Knowledge base** — Company wiki/docs (Confluence, Notion, or custom)
- **Chat history** — Customer conversations (Intercom, Zendesk, or internal)
- **Email history** — Customer email threads (Gmail API, Exchange, or custom)
- **Production DB** — Customer records, orders, subscriptions (PostgreSQL, MySQL)
- **CRM / Tickets** — Salesforce, HubSpot, or internal ticketing

#### The `.mcp.json` Configuration

Place this file in the hosted Claude Code instance's working directory or in a plugin's root:

```json
{
  "knowledge-base": {
    "type": "http",
    "url": "https://mcp.internal.yourcompany.com/knowledge",
    "headers": {
      "Authorization": "Bearer ${KB_API_TOKEN}",
      "X-Tenant-ID": "${COMPANY_TENANT_ID}"
    }
  },
  "chat-history": {
    "type": "http",
    "url": "https://mcp.internal.yourcompany.com/chat",
    "headers": {
      "Authorization": "Bearer ${CHAT_API_TOKEN}"
    }
  },
  "email-history": {
    "type": "http",
    "url": "https://mcp.internal.yourcompany.com/email",
    "headers": {
      "Authorization": "Bearer ${EMAIL_API_TOKEN}"
    }
  },
  "production-db": {
    "command": "python",
    "args": ["-m", "mcp_server_postgres"],
    "env": {
      "DATABASE_URL": "${PROD_DB_READ_REPLICA_URL}",
      "DB_USER": "${PROD_DB_USER}",
      "DB_PASSWORD": "${PROD_DB_PASSWORD}",
      "READ_ONLY": "true"
    }
  },
  "crm": {
    "type": "http",
    "url": "https://mcp.internal.yourcompany.com/crm",
    "headers": {
      "Authorization": "Bearer ${CRM_API_TOKEN}",
      "X-User-Email": "${USER_EMAIL}"
    }
  }
}
```

#### Environment Variables to Set

On the hosted instance, set these before starting Claude Code:

```bash
# Knowledge base
export KB_API_TOKEN="your-kb-service-token"
export COMPANY_TENANT_ID="acme-corp"

# Chat history
export CHAT_API_TOKEN="your-chat-service-token"

# Email history
export EMAIL_API_TOKEN="your-email-service-token"

# Production database (read replica!)
export PROD_DB_READ_REPLICA_URL="postgresql://readonly-replica.internal:5432/production"
export PROD_DB_USER="claude_readonly"
export PROD_DB_PASSWORD="from-secrets-manager"

# CRM
export CRM_API_TOKEN="your-crm-service-token"
export USER_EMAIL="analyst@yourcompany.com"
```

#### Docker Configuration for Multi-MCP

When running in Docker, pass all MCP credentials as environment variables:

```bash
docker run -it --rm \
  -e ANTHROPIC_API_KEY \
  -e KB_API_TOKEN \
  -e COMPANY_TENANT_ID \
  -e CHAT_API_TOKEN \
  -e EMAIL_API_TOKEN \
  -e PROD_DB_READ_REPLICA_URL \
  -e PROD_DB_USER \
  -e PROD_DB_PASSWORD \
  -e CRM_API_TOKEN \
  -e USER_EMAIL \
  -v "$(pwd)/.claude-state:/home/sandbox/state" \
  -v "$(pwd)/workspace:/home/sandbox/workspace" \
  claude-sandbox
```

Or use a `.env` file (keep out of git!):

```bash
docker run -it --rm \
  --env-file .env.customer360 \
  -v "$(pwd)/.claude-state:/home/sandbox/state" \
  -v "$(pwd)/workspace:/home/sandbox/workspace" \
  claude-sandbox
```

### MCP Auth Patterns for Internal Services

Depending on your internal infrastructure, choose the auth pattern that fits each data source:

| Data Source Type | Recommended MCP Auth | Transport | Configuration |
|-----------------|---------------------|-----------|--------------|
| **Internal API with OAuth** (e.g., Confluence Cloud, Google Workspace) | Auto OAuth — just provide URL | HTTP/SSE | `{ "type": "http", "url": "https://mcp.service.com/mcp" }` |
| **Internal API with tokens** (e.g., custom REST services, Elasticsearch) | Bearer token via `${ENV_VAR}` | HTTP | `{ "headers": { "Authorization": "Bearer ${TOKEN}" } }` |
| **Internal API with API keys** (e.g., custom microservices) | Custom headers via `${ENV_VAR}` | HTTP | `{ "headers": { "X-API-Key": "${API_KEY}" } }` |
| **Database** (PostgreSQL, MySQL, MongoDB) | Env vars passed to stdio server | stdio | `{ "command": "python", "args": ["-m", "mcp_server_db"], "env": { "DATABASE_URL": "${DB_URL}" } }` |
| **Service behind VPN** | stdio server running locally in the sandbox | stdio | Server process runs inside Docker with VPN access |
| **Service requiring mTLS** | stdio wrapper with client certs | stdio | `{ "command": "mtls-wrapper", "args": ["--cert", "${CLIENT_CERT}"] }` |
| **Service with short-lived tokens** | `headersHelper` script generates fresh tokens | HTTP/SSE | `{ "headersHelper": "./scripts/get-fresh-token.sh" }` |
| **SaaS with OAuth** (Salesforce, HubSpot, Zendesk) | Auto OAuth or explicit OAuth | HTTP/SSE | URL only, or `{ "oauth": { "clientId": "...", "callbackPort": 3118 } }` |

### Per-User vs Shared MCP Credentials

A critical decision for hosted instances: should each user have their own MCP credentials, or should the instance use shared service accounts?

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Shared service account** | One set of credentials for all users | Simple setup; one `.env` file | No per-user audit trail; overly broad access |
| **Per-user credentials** | Each user's env vars loaded at session start | Full audit trail; least-privilege per user | More complex setup; credential management overhead |
| **Hybrid** | Shared for read-only sources; per-user for write access | Balanced security and simplicity | Need clear policy on which sources are shared |

**Recommended for Customer 360**: Use the **hybrid** approach:
- **Shared read-only** credentials for knowledge base, chat history, email history (read-only API tokens)
- **Per-user** credentials for CRM and production DB (where writes or sensitive queries are possible)
- **Read-only database user** always — never give the MCP server write access to production

#### Per-User MCP Setup with Docker

```bash
# User-specific env file loaded at login
cat > /home/$USER/.env.mcp << 'EOF'
CRM_API_TOKEN=user-specific-crm-token
USER_EMAIL=jane@yourcompany.com
EOF

# Start Claude Code with user's MCP credentials
docker run -it --rm \
  --env-file /shared/.env.customer360-shared \
  --env-file /home/$USER/.env.mcp \
  -e ANTHROPIC_API_KEY \
  -v "/home/$USER/.claude-state:/home/sandbox/state" \
  -v "$(pwd)/workspace:/home/sandbox/workspace" \
  claude-sandbox
```

### Credential Lifecycle: Obtain → Store → Inject → Rotate

The table above helps you decide **which approach** to use. This section explains **how to manage credentials** through their full lifecycle for each approach.

```
┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌──────────┐
│ OBTAIN   │───▶│  STORE   │───▶│   INJECT     │───▶│  ROTATE  │
│          │    │          │    │              │    │          │
│ Create   │    │ Secret   │    │ Env vars →   │    │ Schedule │
│ tokens / │    │ manager  │    │ Claude Code  │    │ or on-   │
│ accounts │    │ or vault │    │ / Docker     │    │ demand   │
└──────────┘    └──────────┘    └──────────────┘    └──────────┘
```

#### Step 1: Obtain Credentials

How you create/get credentials differs by approach and data source:

**For shared service accounts:**

| Data Source | How to Create Credentials | What You Get |
|-------------|--------------------------|-------------|
| **Internal REST API** | Create a service account in your API management platform with read-only scope | API key or bearer token |
| **Production DB** | Create a read-only database user: `CREATE USER claude_readonly WITH PASSWORD '...' LOGIN; GRANT SELECT ON ALL TABLES IN SCHEMA public TO claude_readonly;` | Username + password or connection string |
| **Knowledge base** (Confluence, Notion) | Create a service integration / API token in admin settings with read-only permissions | API token |
| **Chat platform** (Slack, Intercom) | Create a bot user or app with `read` scopes only | Bot token or OAuth client credentials |
| **Email** (Gmail, Exchange) | Create a service account with domain-wide delegation (Google) or application permissions (Microsoft) | Service account JSON key or client secret |
| **CRM** (Salesforce, HubSpot) | Create a connected app / private app with limited scopes | Client ID + secret, or API key |

**For per-user credentials:**

| Data Source | How to Create Credentials | What You Get |
|-------------|--------------------------|-------------|
| **Internal REST API** | Each user generates their own personal API token from the service's developer settings | Personal API token per user |
| **CRM** (Salesforce, HubSpot) | Each user connects via OAuth; token stored per session | Per-user OAuth tokens (managed by Claude Code for auto-OAuth servers) |
| **Production DB** | Create per-user read-only DB users (or use IAM-based DB auth): `CREATE USER jane_readonly WITH PASSWORD '...' LOGIN; GRANT SELECT ON ALL TABLES TO jane_readonly;` | Per-user connection strings |
| **SaaS with OAuth** | Auto OAuth — Claude Code handles per-user auth automatically when user first invokes a tool | Per-user tokens stored by Claude Code |

> **Tip**: For OAuth-based SaaS services (Salesforce, HubSpot, Zendesk, Jira), you may not need to manage credentials at all. If the MCP server supports auto OAuth, Claude Code handles per-user authentication automatically — just provide the URL in `.mcp.json`.

#### Step 2: Store Credentials

**Never store credentials in:**
- Docker images / Dockerfiles
- Git repositories
- Plaintext files on shared filesystems
- Environment variables in shell profiles that get committed

**Use a secret manager:**

| Cloud Provider | Secret Manager | CLI to Retrieve |
|---------------|---------------|-----------------|
| **AWS** | AWS Secrets Manager | `aws secretsmanager get-secret-value --secret-id <name> --query SecretString --output text` |
| **GCP** | Google Secret Manager | `gcloud secrets versions access latest --secret=<name>` |
| **Azure** | Azure Key Vault | `az keyvault secret show --vault-name <vault> --name <name> --query value -o tsv` |
| **Self-hosted** | HashiCorp Vault | `vault kv get -field=value secret/<path>` |

**Organizing secrets in your secret manager:**

```
# Shared credentials (one set for the hosted instance)
customer360/shared/kb-api-token
customer360/shared/chat-api-token
customer360/shared/email-api-token
customer360/shared/db-readonly-url        # postgresql://claude_readonly:***@replica:5432/prod

# Per-user credentials (one set per user)
customer360/users/jane/crm-token
customer360/users/jane/db-user
customer360/users/bob/crm-token
customer360/users/bob/db-user
```

**Example: Storing credentials in AWS Secrets Manager:**

```bash
# Store a shared credential
aws secretsmanager create-secret \
  --name "customer360/shared/kb-api-token" \
  --secret-string "your-kb-api-token-here" \
  --description "Read-only knowledge base API token for Customer 360"

# Store a per-user credential
aws secretsmanager create-secret \
  --name "customer360/users/jane/crm-token" \
  --secret-string "jane-specific-crm-token" \
  --description "Jane's CRM API token for Customer 360"
```

#### Step 3: Inject Credentials into Claude Code

Credentials must arrive as environment variables when Claude Code starts. The `.mcp.json` references them with `${VAR_NAME}` syntax, and Claude Code expands them at connection time.

**Method A: Startup script fetches from secret manager (recommended)**

Create a launcher script that fetches secrets and passes them to Docker:

```bash
#!/bin/bash
# launch-customer360.sh — Called when a user starts their session

USER=$1  # e.g., "jane"

# Fetch shared credentials from secret manager
KB_TOKEN=$(aws secretsmanager get-secret-value \
  --secret-id "customer360/shared/kb-api-token" \
  --query SecretString --output text)

CHAT_TOKEN=$(aws secretsmanager get-secret-value \
  --secret-id "customer360/shared/chat-api-token" \
  --query SecretString --output text)

EMAIL_TOKEN=$(aws secretsmanager get-secret-value \
  --secret-id "customer360/shared/email-api-token" \
  --query SecretString --output text)

DB_URL=$(aws secretsmanager get-secret-value \
  --secret-id "customer360/shared/db-readonly-url" \
  --query SecretString --output text)

# Fetch per-user credentials
CRM_TOKEN=$(aws secretsmanager get-secret-value \
  --secret-id "customer360/users/$USER/crm-token" \
  --query SecretString --output text)

USER_EMAIL="$USER@yourcompany.com"

# Launch Claude Code with all credentials injected
docker run -it --rm \
  -e ANTHROPIC_API_KEY \
  -e KB_API_TOKEN="$KB_TOKEN" \
  -e CHAT_API_TOKEN="$CHAT_TOKEN" \
  -e EMAIL_API_TOKEN="$EMAIL_TOKEN" \
  -e PROD_DB_READ_REPLICA_URL="$DB_URL" \
  -e CRM_API_TOKEN="$CRM_TOKEN" \
  -e USER_EMAIL="$USER_EMAIL" \
  -e COMPANY_TENANT_ID="acme-corp" \
  -v "/home/$USER/.claude-state:/home/sandbox/state" \
  -v "$(pwd)/workspace:/home/sandbox/workspace" \
  claude-sandbox
```

**Method B: Docker with secret manager sidecar (for Kubernetes)**

```yaml
# kubernetes-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: claude-customer360-jane
spec:
  serviceAccountName: claude-customer360
  containers:
    - name: claude-sandbox
      image: claude-sandbox:latest
      env:
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: claude-api-key
              key: api-key
        # Shared credentials from Kubernetes secrets
        - name: KB_API_TOKEN
          valueFrom:
            secretKeyRef:
              name: customer360-shared
              key: kb-api-token
        - name: CHAT_API_TOKEN
          valueFrom:
            secretKeyRef:
              name: customer360-shared
              key: chat-api-token
        # Per-user credential from user-specific secret
        - name: CRM_API_TOKEN
          valueFrom:
            secretKeyRef:
              name: customer360-jane
              key: crm-token
```

**Method C: GCP — fetch from Secret Manager at startup**

```bash
#!/bin/bash
# launch-customer360-gcp.sh

USER=$1

# Shared credentials
KB_TOKEN=$(gcloud secrets versions access latest --secret=customer360-shared-kb-token)
CHAT_TOKEN=$(gcloud secrets versions access latest --secret=customer360-shared-chat-token)

# Per-user credentials
CRM_TOKEN=$(gcloud secrets versions access latest --secret=customer360-${USER}-crm-token)

docker run -it --rm \
  -e CLAUDE_CODE_USE_VERTEX=1 \
  -e CLOUD_ML_REGION=us-east5 \
  -e ANTHROPIC_VERTEX_PROJECT_ID=your-project-id \
  -e KB_API_TOKEN="$KB_TOKEN" \
  -e CHAT_API_TOKEN="$CHAT_TOKEN" \
  -e CRM_API_TOKEN="$CRM_TOKEN" \
  -v "/home/$USER/.claude-state:/home/sandbox/state" \
  -v "$(pwd)/workspace:/home/sandbox/workspace" \
  claude-sandbox
```

**Method D: Azure — fetch from Key Vault at startup**

```bash
#!/bin/bash
# launch-customer360-azure.sh

USER=$1

# Shared credentials
KB_TOKEN=$(az keyvault secret show --vault-name customer360 --name shared-kb-token --query value -o tsv)
CHAT_TOKEN=$(az keyvault secret show --vault-name customer360 --name shared-chat-token --query value -o tsv)

# Per-user credentials
CRM_TOKEN=$(az keyvault secret show --vault-name customer360 --name "${USER}-crm-token" --query value -o tsv)

docker run -it --rm \
  -e CLAUDE_CODE_USE_FOUNDRY=1 \
  -e ANTHROPIC_FOUNDRY_RESOURCE=your-resource-name \
  -e KB_API_TOKEN="$KB_TOKEN" \
  -e CHAT_API_TOKEN="$CHAT_TOKEN" \
  -e CRM_API_TOKEN="$CRM_TOKEN" \
  -v "/home/$USER/.claude-state:/home/sandbox/state" \
  -v "$(pwd)/workspace:/home/sandbox/workspace" \
  claude-sandbox
```

#### Step 4: Rotate Credentials

Credential rotation strategies depend on the credential type:

| Credential Type | Rotation Strategy | Frequency | Automation |
|----------------|-------------------|-----------|------------|
| **API tokens** (shared service accounts) | Generate new token → update secret manager → restart sessions | Every 90 days | Secret manager auto-rotation |
| **API tokens** (per-user) | Each user regenerates their own token → admin updates secret manager | Every 90 days, or on-demand | User-initiated + admin script |
| **Database passwords** | Rotate via secret manager auto-rotation → restart connections | Every 30–90 days | Secret manager auto-rotation |
| **OAuth tokens** (auto OAuth) | Claude Code handles refresh automatically — no manual rotation needed | Automatic | Built-in |
| **Service account keys** (GCP JSON keys) | Generate new key → update secret → delete old key | Every 90 days | GCP recommends Workload Identity Federation instead |
| **Client certificates** (mTLS) | Issue new cert → update secret manager → restart | Before expiry | PKI automation / cert-manager |

**Automated rotation with AWS Secrets Manager:**

```bash
# Enable auto-rotation for a shared credential (every 90 days)
aws secretsmanager rotate-secret \
  --secret-id "customer360/shared/kb-api-token" \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:111122223333:function:rotate-api-token \
  --rotation-rules '{"AutomaticallyAfterDays": 90}'
```

The rotation Lambda function should:
1. Call the service API to generate a new token
2. Update the secret in Secrets Manager
3. (Optional) Notify affected sessions to restart

**Manual rotation script (for any cloud provider):**

```bash
#!/bin/bash
# rotate-shared-credentials.sh
# Run on a schedule (cron) or on-demand

echo "Rotating Customer 360 shared credentials..."

# 1. Generate new KB token (call your service's API)
NEW_KB_TOKEN=$(curl -s -X POST https://kb.internal/api/tokens \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -d '{"name": "customer360-readonly", "scope": "read"}' | jq -r '.token')

# 2. Update secret manager
aws secretsmanager put-secret-value \
  --secret-id "customer360/shared/kb-api-token" \
  --secret-string "$NEW_KB_TOKEN"

# 3. Revoke old token (service-specific)
curl -s -X DELETE https://kb.internal/api/tokens/${OLD_TOKEN_ID} \
  -H "Authorization: Bearer ${ADMIN_TOKEN}"

echo "✅ KB token rotated. Active sessions will use new token on next restart."

# 4. (Optional) Signal running sessions to restart
# This depends on your orchestration — e.g., rolling restart of Docker containers
```

**Per-user credential rotation workflow:**

```
┌──────────────────────────────────────────────────────────┐
│  Admin sends rotation reminder to users                  │
│  (email / Slack / automated notification)                │
├──────────────────────────────────────────────────────────┤
│  Each user:                                              │
│  1. Generates new personal API token from the service    │
│  2. Runs: update-my-credentials.sh <new-token>          │
│     (script updates their secret in the secret manager)  │
├──────────────────────────────────────────────────────────┤
│  User's next Claude Code session picks up new credential │
│  (fetched from secret manager at startup)                │
├──────────────────────────────────────────────────────────┤
│  Admin verifies all users have rotated (check timestamps │
│  in secret manager) and revokes overdue tokens           │
└──────────────────────────────────────────────────────────┘
```

**User-facing rotation script:**

```bash
#!/bin/bash
# update-my-credentials.sh — Run by each user
# Usage: ./update-my-credentials.sh <new-crm-token>

USER=$(whoami)
NEW_CRM_TOKEN=$1

if [ -z "$NEW_CRM_TOKEN" ]; then
  echo "Usage: $0 <new-crm-token>"
  echo "Get your new token from: https://crm.yourcompany.com/settings/api-tokens"
  exit 1
fi

# Update in secret manager
aws secretsmanager put-secret-value \
  --secret-id "customer360/users/$USER/crm-token" \
  --secret-string "$NEW_CRM_TOKEN"

echo "✅ CRM token updated. It will take effect on your next Claude Code session."
echo "   To apply immediately, restart your session."
```

**For OAuth-based services (no manual rotation needed):**

When an MCP server uses auto OAuth (Pattern 1 — just a URL in `.mcp.json`), Claude Code manages the entire token lifecycle:
- Tokens are obtained automatically when the user first uses the MCP tool
- Refresh tokens are used to get new access tokens before they expire
- Users re-authorize if the refresh token expires or scopes change
- No admin intervention needed — this is the easiest approach

This is why **OAuth-based MCP servers are strongly preferred** for per-user access when available. If your internal service can implement an OAuth 2.0 / MCP-compatible auth flow, it eliminates all credential management overhead.

### Security Best Practices for Hosted MCP Connections

1. **Always use a read-only replica** for production database connections — never connect the MCP server to the primary/write DB
   ```bash
   # ✅ Read replica
   export PROD_DB_READ_REPLICA_URL="postgresql://readonly-replica.internal:5432/production"
   # ❌ Never the primary
   # export PROD_DB_URL="postgresql://primary.internal:5432/production"
   ```

2. **Scope tokens to minimum permissions** — Create dedicated API tokens for each MCP server with only the permissions it needs
   ```
   Knowledge Base: read-only access to articles
   Chat History: read-only access to conversations
   Email: read-only access to threads (no send permission!)
   CRM: read + limited write (e.g., add notes, not delete contacts)
   ```

3. **Use your cloud provider's secret management** — Don't store credentials in Docker images, env files on disk, or code
   ```bash
   # ✅ Fetch from AWS Secrets Manager at startup
   export PROD_DB_PASSWORD=$(aws secretsmanager get-secret-value \
     --secret-id customer360/db-password --query SecretString --output text)

   # ✅ Fetch from GCP Secret Manager
   export PROD_DB_PASSWORD=$(gcloud secrets versions access latest \
     --secret=customer360-db-password)

   # ✅ Fetch from Azure Key Vault
   export PROD_DB_PASSWORD=$(az keyvault secret show \
     --vault-name customer360 --name db-password --query value -o tsv)
   ```

4. **Network isolation** — Run MCP servers on an internal network; the hosted Claude Code instance should access them over VPN/VPC, not the public internet
   ```bash
   # Docker with host network (access internal services)
   docker run --network=host ...

   # Or Docker with a custom bridge connected to internal services
   docker run --network=internal-services ...
   ```

5. **Audit MCP tool usage** — Log which MCP tools are called, with what arguments, and by which user. Claude Code's `--debug` mode shows MCP calls; route this to your logging infrastructure

6. **Rotate credentials on a schedule** — MCP tokens should be rotated just like any API credential. Use short-lived tokens where possible (`headersHelper` for dynamic token generation)

7. **Restrict MCP tools in commands** — Use `allowed-tools` in command frontmatter to control which MCP tools each workflow can access:
   ```markdown
   ---
   allowed-tools: [
     "mcp__knowledge-base__search_articles",
     "mcp__chat-history__get_conversations",
     "mcp__crm__get_customer"
   ]
   ---
   # Don't allow: mcp__production-db__execute_query (too broad)
   ```

### Building Your Own MCP Server for Internal Data

If your company data sources don't have existing MCP servers, you'll need to build them. The MCP protocol is straightforward:

**Option 1: Use an existing MCP server package**

| Data Source | MCP Server Package | Transport |
|-------------|-------------------|-----------|
| PostgreSQL | `mcp-server-postgres` (Python), `@modelcontextprotocol/server-postgres` (Node) | stdio |
| MySQL | `mcp-server-mysql` | stdio |
| SQLite | `@modelcontextprotocol/server-sqlite` | stdio |
| Filesystem | `@modelcontextprotocol/server-filesystem` | stdio |
| Elasticsearch | `mcp-server-elasticsearch` | stdio |
| Redis | `mcp-server-redis` | stdio |

**Option 2: Build a custom MCP server** (for internal APIs)

Use the [MCP SDK](https://modelcontextprotocol.io/) to wrap your internal API:

```python
# Example: Custom knowledge base MCP server
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("knowledge-base")

@server.tool()
async def search_articles(query: str, limit: int = 10) -> list[TextContent]:
    """Search the company knowledge base for articles matching the query."""
    results = await your_kb_api.search(query, limit=limit)
    return [TextContent(type="text", text=format_result(r)) for r in results]

@server.tool()
async def get_article(article_id: str) -> TextContent:
    """Get the full content of a knowledge base article by ID."""
    article = await your_kb_api.get(article_id)
    return TextContent(type="text", text=article.content)
```

**Option 3: Use an HTTP proxy** — Expose your internal REST API as an MCP HTTP server using a generic MCP-to-REST adapter.

### Complete Customer 360 Setup Checklist

Here's a step-by-step checklist for setting up a hosted Claude Code instance as a Customer 360 assistant:

- [ ] **Layer 2**: Choose and configure an AI model backend (Bedrock, Vertex, Foundry, or Anthropic direct)
- [ ] **Layer 3 — MCP servers**:
  - [ ] Identify all data sources needed (KB, chat, email, DB, CRM)
  - [ ] For each source: choose MCP server (existing package or custom)
  - [ ] For each source: create a dedicated service account / API token with minimum permissions
  - [ ] Write `.mcp.json` with all MCP servers configured
  - [ ] Create environment variable documentation for operators
- [ ] **Security**:
  - [ ] Use read-only replicas for databases
  - [ ] Store all credentials in secret manager (not in env files or Docker images)
  - [ ] Set up network access (VPN/VPC) for internal services
  - [ ] Configure `allowed-tools` to restrict which MCP tools each workflow can access
  - [ ] Set up audit logging for MCP tool calls
  - [ ] Establish credential rotation schedule
- [ ] **Layer 1**: Configure user access (SSH, web terminal, or VS Code Remote)
- [ ] **Per-user isolation**:
  - [ ] Decide shared vs per-user MCP credentials for each source
  - [ ] Create per-user env files for user-specific credentials
  - [ ] Mount separate `.claude-state` volumes per user
- [ ] **Testing**:
  - [ ] Test each MCP server connection individually
  - [ ] Test a cross-source query (e.g., "Tell me everything about customer X")
  - [ ] Verify read-only access to production DB
  - [ ] Verify audit logs capture MCP tool usage
