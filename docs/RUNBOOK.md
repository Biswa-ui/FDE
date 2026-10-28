# Step-by-Step Operational Runbook: Agent Identity & Auth Manager

This runbook documents every step, command, code change, and IAM policy binding performed to migrate `github-agent` and `salesforce-agent` from hardcoded secrets to **Google Cloud Agent Identity Auth Manager**.

> **Security Note:** All sensitive credentials (GitHub PAT, Salesforce Consumer Key/Secret) are stored exclusively in Google Cloud Agent Identity Auth Manager and redacted as placeholders (`<YOUR_...>`) in this repository.

---

## Environment Metadata

| Parameter | Value |
| :--- | :--- |
| **Project ID** | `qwiklabs-gcp-01-e1dba63e942f` |
| **Project Number** | `216706464857` |
| **Organization ID** | `616463121992` |
| **Location / Region** | `us-central1` |
| **Model** | `gemini-3.5-flash` (`GOOGLE_CLOUD_LOCATION=global`) |
| **GitHub Auth Provider URI** | `projects/qwiklabs-gcp-01-e1dba63e942f/locations/us-central1/authProviders/github-mcp-auth-provider` |
| **Salesforce Auth Provider URI** | `projects/qwiklabs-gcp-01-e1dba63e942f/locations/us-central1/authProviders/salesforce-2lo-auth-provider` |
| **Salesforce My Domain** | `flow-site-8276.my.salesforce.com` |
| **GitHub Agent Reasoning Engine ID** | `385113018599473152` |
| **GitHub Agent SPIFFE Principal** | `principal://agents.global.org-616463121992.system.id.goog/resources/aiplatform/projects/216706464857/locations/us-central1/reasoningEngines/385113018599473152` |
| **Salesforce Cloud Run Service Account** | `216706464857-compute@developer.gserviceaccount.com` |

---

## Task 1. Prepare Your Environment for Agent Identity

### 1.1 Set Environment Variables

```bash
export PROJECT_ID=qwiklabs-gcp-01-e1dba63e942f
export LOCATION=us-central1
export REGION=us-central1
gcloud config set project ${PROJECT_ID}

export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')
export ORGANIZATION_ID=$(gcloud projects get-ancestors ${PROJECT_ID} --format=json | jq -r '.[] | select(.type=="organization") | .id')
echo "Project: ${PROJECT_ID} (${PROJECT_NUMBER}), Org: ${ORGANIZATION_ID}, Location: ${LOCATION}, Agents: ${REGION}"
```

### 1.2 Copy Lab Setup Scripts & Stage Agent Code

```bash
gcloud storage cp -r gs://${PROJECT_ID}-bucket/apex_labs .
bash apex_labs/lab2_prereqs.sh
export PATH="$HOME/.local/bin:$PATH"
```

### 1.3 Configure `github-agent/.env`

```bash
cd ~/agents/github-agent
cat << EOF > .env
GOOGLE_CLOUD_PROJECT=$PROJECT_ID
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
GITHUB_MCP_URL=https://api.githubcopilot.com/mcp/
GITHUB_AUTH_PROVIDER_URI=projects/$PROJECT_ID/locations/$LOCATION/authProviders/github-mcp-auth-provider
EOF
```

### 1.4 Verify `agents-cli` Installation

```bash
agents-cli info
# CLI version: 1.0.0
```

### 1.5 Enable Required Agent Identity & Auth Manager APIs

```bash
gcloud services enable \
  iam.googleapis.com \
  agentidentity.googleapis.com \
  aiplatform.googleapis.com \
  agentidentitycredentials.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com \
  --project=${PROJECT_ID}
```

---

## Task 2. Create the GitHub API-Key Auth Provider

### 2.1 Capture the GitHub Personal Access Token

```bash
read -rs GITHUB_PAT && echo
```

### 2.2 Create `github-mcp-auth-provider`

```bash
gcloud alpha agent-identity auth-providers create github-mcp-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --api-key="${GITHUB_PAT}"
```

### 2.3 Verify the Auth Provider State is `ENABLED`

```bash
gcloud alpha agent-identity auth-providers list \
  --project=${PROJECT_ID} \
  --location=${LOCATION}

export GITHUB_AUTH_PROVIDER_URI="projects/${PROJECT_ID}/locations/${LOCATION}/authProviders/github-mcp-auth-provider"
echo ${GITHUB_AUTH_PROVIDER_URI}
```

---

## Task 3. Move the GitHub Agent onto the Auth Provider

### 3.1 Refactor `~/agents/github-agent/agent/agent.py`

Completed all three `TODO` items in [`agents/github-agent/agent/agent.py`](../agents/github-agent/agent/agent.py):

1. **TODO 1 — Register `GcpAuthProvider` with `CredentialManager`:**
   ```python
   CredentialManager.register_auth_provider(GcpAuthProvider())
   ```
2. **TODO 2 — Define `GcpAuthProviderScheme` pointing to `GITHUB_AUTH_PROVIDER_URI`:**
   ```python
   github_auth_scheme = GcpAuthProviderScheme(name=GITHUB_AUTH_PROVIDER_URI)
   ```
3. **TODO 3 — Build and return `McpToolset` with `header_provider`:**
   ```python
   toolset = McpToolset(
       connection_params=StreamableHTTPConnectionParams(url=GITHUB_MCP_URL),
       auth_scheme=github_auth_scheme,
       tool_filter=GITHUB_TOOL_FILTER,
       header_provider=github_header_provider,
   )
   return toolset
   ```

### 3.2 Deploy to Vertex AI Agent Runtime with Agent Identity

```bash
cd ~/agents/github-agent
chmod 777 ./deploy_with_identity.sh
REGION=us-central1 A2A_APP_NAME=agent AUTH_PROVIDER_NAME=github-mcp-auth-provider DISPLAY_NAME=github-agent-v2 ./deploy_with_identity.sh
```

### 3.3 Export the Provisioned SPIFFE Identity and Reasoning Engine URL

```bash
export GITHUB_AGENT_IDENTITY="principal://agents.global.org-616463121992.system.id.goog/resources/aiplatform/projects/216706464857/locations/us-central1/reasoningEngines/385113018599473152"
export GITHUB_ENGINE_ID="385113018599473152"
export GITHUB_AGENT_URL="https://${REGION}-aiplatform.googleapis.com/v1/projects/${PROJECT_NUMBER}/locations/${REGION}/reasoningEngines/${GITHUB_ENGINE_ID}"
echo ${GITHUB_AGENT_URL}
```

### 3.4 Grant IAM Access (`roles/agentidentity.user`) on `github-mcp-auth-provider`

```bash
# Grant to the GitHub agent's SPIFFE principal
gcloud alpha agent-identity auth-providers add-iam-policy-binding github-mcp-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="${GITHUB_AGENT_IDENTITY}"

# Grant to the lab user account for local testing
gcloud alpha agent-identity auth-providers add-iam-policy-binding github-mcp-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="user:$(gcloud config get-value account)"
```

### 3.5 Test the Deployed GitHub Agent over A2A

```bash
agents-cli run --url ${GITHUB_AGENT_URL} --mode a2a --app-name agent "List all my open GitHub issues"
```

---

## Task 4. Create the Salesforce 2LO Auth Provider

### 4.1 Set Salesforce Connected App Parameters

```bash
export SF_DOMAIN="flow-site-8276.my.salesforce.com"
export SF_CLIENT_ID="<YOUR_SALESFORCE_CONSUMER_KEY>"
read -rs SF_CLIENT_SECRET && echo
```

### 4.2 Create `salesforce-2lo-auth-provider`

```bash
gcloud alpha agent-identity auth-providers create salesforce-2lo-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --two-legged-oauth-client-id="${SF_CLIENT_ID}" \
  --two-legged-oauth-client-secret="${SF_CLIENT_SECRET}" \
  --two-legged-oauth-token-url="https://${SF_DOMAIN}/services/oauth2/token"
```

### 4.3 Verify Both Auth Providers Are `ENABLED`

```bash
gcloud alpha agent-identity auth-providers list \
  --project=${PROJECT_ID} \
  --location=${LOCATION}

export SALESFORCE_AUTH_PROVIDER_URI="projects/${PROJECT_ID}/locations/${LOCATION}/authProviders/salesforce-2lo-auth-provider"
echo ${SALESFORCE_AUTH_PROVIDER_URI}
```

---

## Task 5. Update the Salesforce Agent to Use the Auth Provider

### 5.1 Refactor `~/agents/salesforce-agent/app/tools.py`

Completed all three `TODO` items in [`agents/salesforce-agent/app/tools.py`](../agents/salesforce-agent/app/tools.py):

1. **TODO 1 — Register `GcpAuthProvider` and build `AuthConfig`:**
   ```python
   CredentialManager.register_auth_provider(GcpAuthProvider())
   salesforce_auth_config = AuthConfig(
       auth_scheme=GcpAuthProviderScheme(name=SALESFORCE_AUTH_PROVIDER_URI)
   )
   ```
2. **TODO 2 — Accept `credential: AuthCredential` in `search_salesforce` and extract the token:**
   ```python
   def search_salesforce(query: str, credential: AuthCredential) -> dict:
       access_token = _extract_token(credential)
   ```
3. **TODO 3 — Wrap `search_salesforce` in `AuthenticatedFunctionTool`:**
   ```python
   return AuthenticatedFunctionTool(
       func=search_salesforce,
       auth_config=salesforce_auth_config,
   )
   ```

### 5.2 Refactor `~/agents/salesforce-agent/app/agent.py` & `app/salesforce_client.py`

- Updated [`agents/salesforce-agent/app/agent.py`](../agents/salesforce-agent/app/agent.py) to remove `SALESFORCE_CLIENT_ID` and `SALESFORCE_CLIENT_SECRET` and initialize `SalesforceClient(domain=SALESFORCE_DOMAIN, ...)`.
- Updated [`agents/salesforce-agent/app/salesforce_client.py`](../agents/salesforce-agent/app/salesforce_client.py) to derive `self.instance_url = f"https://{self.domain}"` and remove `authenticate()`.

### 5.3 Update `.env` and `pyproject.toml`

```bash
cd ~/agents/salesforce-agent
cat << EOF > .env
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
SALESFORCE_DOMAIN=${SF_DOMAIN}
SALESFORCE_AUTH_PROVIDER_URI=${SALESFORCE_AUTH_PROVIDER_URI}
EOF

sed -i 's/google-adk\[gcp\]/google-adk[gcp,agent-identity]/' pyproject.toml
uv lock
```

### 5.4 Deploy `salesforce-agent` to Cloud Run

```bash
cd ~/agents/salesforce-agent
agents-cli deploy --project ${PROJECT_ID} --region ${REGION} \
  --no-confirm-project \
  --max-instances 4 \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

### 5.5 Grant the Cloud Run Service Account Access to `salesforce-2lo-auth-provider`

```bash
export SF_SA=$(gcloud run services describe salesforce-agent --region ${REGION} \
  --format='value(spec.template.spec.serviceAccountName)')
echo ${SF_SA}

gcloud alpha agent-identity auth-providers add-iam-policy-binding salesforce-2lo-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="serviceAccount:${SF_SA}"

gcloud alpha agent-identity auth-providers add-iam-policy-binding salesforce-2lo-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="user:$(gcloud config get-value account)"
```

### 5.6 Disable IAP and Test `salesforce-agent` over A2A

```bash
gcloud run services update salesforce-agent --region=${REGION} --no-iap

export SF_URL=$(gcloud run services describe salesforce-agent --region ${REGION} --format='value(status.url)')
agents-cli run --url ${SF_URL} --mode a2a "Search for any architecture decision records about timeouts."
```
