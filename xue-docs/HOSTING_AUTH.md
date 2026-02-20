# Hosting Claude Code as a Service — Authentication

> **Scenario**: You host Claude Code in a cloud sandbox (Docker container, cloud VM, or managed service) and want colleagues to use it. How should authentication be configured?

This is a different layer from MCP server authentication (covered in [PLUGIN_ANALYSIS.md](./PLUGIN_ANALYSIS.md#mcp-server-authentication)). MCP auth controls how plugins connect to external services (Stripe, GitHub, etc.). **Hosting auth** controls how Claude Code itself authenticates with the underlying AI model provider, and how users access the hosted instance.

## Two Authentication Layers

When hosting Claude Code for a team, there are **two distinct auth layers** to configure:

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: USER → HOSTED CLAUDE CODE                 │
│  (How colleagues access the sandbox)                │
│  SSH keys, SSO, VPN, or web-based access            │
├─────────────────────────────────────────────────────┤
│  Layer 2: CLAUDE CODE → AI MODEL PROVIDER           │
│  (How Claude Code calls the LLM)                    │
│  API key, IAM role, OAuth, or service account       │
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
