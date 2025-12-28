# Deploying MCP-Atlassian on Azure Container Apps with Azure API Management

This guide provides detailed instructions for deploying the MCP-Atlassian server on Azure Container Apps (ACA) with Azure API Management (APIM) handling OAuth 2.1 authentication for secure access to Atlassian Cloud APIs.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Part 1: Atlassian OAuth 2.0 App Setup](#part-1-atlassian-oauth-20-app-setup)
- [Part 2: Azure Infrastructure Setup](#part-2-azure-infrastructure-setup)
- [Part 3: Deploy MCP-Atlassian to Azure Container Apps](#part-3-deploy-mcp-atlassian-to-azure-container-apps)
- [Part 4: Configure Azure API Management](#part-4-configure-azure-api-management)
- [Part 5: APIM OAuth 2.1 Configuration](#part-5-apim-oauth-21-configuration)
- [Part 6: Testing and Validation](#part-6-testing-and-validation)
- [Security Best Practices](#security-best-practices)
- [Troubleshooting](#troubleshooting)
- [Reference Links](#reference-links)

---

## Architecture Overview

```
┌─────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────┐
│                 │      │                     │      │                     │      │                 │
│   MCP Client    │──────│   Azure API         │──────│   Azure Container   │──────│   Atlassian     │
│   (AI Agent)    │ HTTPS│   Management        │ HTTPS│   Apps              │ HTTPS│   Cloud APIs    │
│                 │      │   (OAuth 2.1)       │      │   (mcp-atlassian)   │      │   (Jira/Conf)   │
│                 │      │                     │      │                     │      │                 │
└─────────────────┘      └─────────────────────┘      └─────────────────────┘      └─────────────────┘
                                   │
                                   │ Token Validation
                                   ▼
                         ┌─────────────────────┐
                         │   Atlassian Auth    │
                         │   (auth.atlassian   │
                         │    .com)            │
                         └─────────────────────┘
```

### Key Components

| Component | Purpose |
|-----------|---------|
| **Azure API Management** | API gateway handling OAuth 2.1 token validation, rate limiting, and request transformation |
| **Azure Container Apps** | Serverless container hosting for the MCP-Atlassian server |
| **Atlassian OAuth 2.0** | Authentication provider for Jira and Confluence Cloud access |
| **Azure Key Vault** | Secure storage for secrets and credentials (optional but recommended) |

### Authentication Flow

1. **Client Authorization**: MCP client initiates OAuth 2.1 flow via APIM
2. **Token Acquisition**: User authenticates with Atlassian, receives access token
3. **API Request**: Client sends request to APIM with Bearer token
4. **Token Validation**: APIM validates the JWT token against Atlassian's OIDC endpoint
5. **Backend Call**: APIM forwards request with token to Azure Container Apps
6. **Atlassian API**: MCP server uses token to call Jira/Confluence APIs

---

## Prerequisites

### Required Accounts and Access

- [ ] **Azure Subscription** with permissions to create:
  - Container Apps
  - API Management
  - Container Registry (optional)
  - Key Vault (recommended)
- [ ] **Atlassian Cloud Account** with admin access to create OAuth apps
- [ ] **Azure CLI** installed and configured (`az login`)
- [ ] **Docker** installed (for local testing)

### Required Information

Gather the following before starting:

| Item | Description | Example |
|------|-------------|---------|
| Atlassian Site URL | Your Atlassian Cloud URL | `https://yourcompany.atlassian.net` |
| Azure Subscription ID | Your Azure subscription | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| Resource Group Name | Azure resource group | `rg-mcp-atlassian-prod` |
| Azure Region | Deployment region | `eastus` |

---

## Part 1: Atlassian OAuth 2.0 App Setup

### Step 1.1: Create an Atlassian OAuth 2.0 App

1. Navigate to [Atlassian Developer Console](https://developer.atlassian.com/console/myapps/)
2. Click **Create** → **OAuth 2.0 integration**
3. Enter app details:
   - **Name**: `MCP Atlassian Azure` (or your preferred name)
   - **Description**: MCP server for AI agent integration

### Step 1.2: Configure Permissions (Scopes)

Navigate to **Permissions** and add the required scopes:

#### Jira API Scopes

| Scope | Permission | Required For |
|-------|------------|--------------|
| `read:jira-work` | Read issues, projects, boards | All read operations |
| `write:jira-work` | Create/update issues, comments | Write operations |
| `read:jira-user` | Read user information | User lookups |

#### Confluence API Scopes

| Scope | Permission | Required For |
|-------|------------|--------------|
| `read:confluence-content.all` | Read pages, spaces, content | All read operations |
| `write:confluence-content` | Create/update pages, comments | Write operations |
| `read:confluence-user` | Read user information | User lookups |

#### Required Scope

| Scope | Permission | Required For |
|-------|------------|--------------|
| `offline_access` | Refresh tokens | **Critical** - enables token refresh |

**Complete scope string:**
```
read:jira-work write:jira-work read:jira-user read:confluence-content.all write:confluence-content read:confluence-user offline_access
```

### Step 1.3: Configure Authorization Settings

1. Navigate to **Authorization** in your app settings
2. Click **Add** next to "OAuth 2.0 (3LO)"
3. Set the **Callback URL**:
   ```
   https://<your-apim-name>.azure-api.net/oauth2/callback
   ```
   > Note: You'll get the actual APIM URL after creating it in Part 4

### Step 1.4: Record OAuth Credentials

Save these values securely (you'll need them later):

| Credential | Location | Example |
|------------|----------|---------|
| **Client ID** | Settings → Authentication details | `AbCdEf123456...` |
| **Client Secret** | Settings → Authentication details | `ATOAxxxx...` |
| **Authorization URL** | `https://auth.atlassian.com/authorize` | Fixed URL |
| **Token URL** | `https://auth.atlassian.com/oauth/token` | Fixed URL |

### Step 1.5: Get Your Cloud ID

Your Cloud ID is required for API calls. Get it by:

1. **Option A**: Use the API after authentication:
   ```bash
   curl -H "Authorization: Bearer <access_token>" \
     https://api.atlassian.com/oauth/token/accessible-resources
   ```

2. **Option B**: Run the MCP-Atlassian OAuth wizard locally:
   ```bash
   docker run --rm -it -p 8080:8080 \
     ghcr.io/sooperset/mcp-atlassian:latest --oauth-setup -v
   ```

---

## Part 2: Azure Infrastructure Setup

### Step 2.1: Set Environment Variables

```bash
# Azure Configuration
export RESOURCE_GROUP="rg-mcp-atlassian-prod"
export LOCATION="eastus"
export ACA_ENV_NAME="mcp-atlassian-env"
export ACA_APP_NAME="mcp-atlassian"
export APIM_NAME="apim-mcp-atlassian"
export KEY_VAULT_NAME="kv-mcp-atlassian"

# Atlassian OAuth (from Part 1)
export ATLASSIAN_OAUTH_CLIENT_ID="your-client-id"
export ATLASSIAN_OAUTH_CLIENT_SECRET="your-client-secret"
export ATLASSIAN_OAUTH_CLOUD_ID="your-cloud-id"
```

### Step 2.2: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

### Step 2.3: Create Azure Key Vault (Recommended)

```bash
# Create Key Vault
az keyvault create \
  --name $KEY_VAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --enable-rbac-authorization true

# Store secrets
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "atlassian-oauth-client-id" \
  --value "$ATLASSIAN_OAUTH_CLIENT_ID"

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "atlassian-oauth-client-secret" \
  --value "$ATLASSIAN_OAUTH_CLIENT_SECRET"

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "atlassian-oauth-cloud-id" \
  --value "$ATLASSIAN_OAUTH_CLOUD_ID"
```

### Step 2.4: Create Container Apps Environment

```bash
# Create Container Apps environment
az containerapp env create \
  --name $ACA_ENV_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Get the environment ID
ACA_ENV_ID=$(az containerapp env show \
  --name $ACA_ENV_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)
```

---

## Part 3: Deploy MCP-Atlassian to Azure Container Apps

### Step 3.1: Create the Container App

```bash
az containerapp create \
  --name $ACA_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $ACA_ENV_NAME \
  --image ghcr.io/sooperset/mcp-atlassian:latest \
  --target-port 8000 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 3 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --env-vars \
    "ATLASSIAN_OAUTH_ENABLE=true" \
    "STREAMABLE_HTTP_PATH=/mcp" \
    "MCP_VERBOSE=true" \
  --args "--transport" "streamable-http" "--port" "8000" "--host" "0.0.0.0" "--path" "/mcp"
```

### Step 3.2: Configure Managed Identity (for Key Vault Access)

```bash
# Enable system-assigned managed identity
az containerapp identity assign \
  --name $ACA_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --system-assigned

# Get the identity principal ID
IDENTITY_PRINCIPAL_ID=$(az containerapp identity show \
  --name $ACA_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Grant Key Vault access
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/$KEY_VAULT_NAME"
```

### Step 3.3: Get the Container App URL

```bash
ACA_FQDN=$(az containerapp show \
  --name $ACA_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn -o tsv)

echo "Container App URL: https://$ACA_FQDN"
```

### Step 3.4: Verify Health Endpoint

```bash
curl -s "https://$ACA_FQDN/healthz" | jq
# Expected: {"status": "ok"}
```

---

## Part 4: Configure Azure API Management

### Step 4.1: Create API Management Instance

```bash
# Create APIM (Developer tier for testing, Standard for production)
az apim create \
  --name $APIM_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --publisher-email "admin@yourcompany.com" \
  --publisher-name "Your Company" \
  --sku-name Developer

# Wait for deployment (can take 30-45 minutes for new instances)
az apim show --name $APIM_NAME --resource-group $RESOURCE_GROUP --query provisioningState
```

### Step 4.2: Get APIM Gateway URL

```bash
APIM_GATEWAY_URL=$(az apim show \
  --name $APIM_NAME \
  --resource-group $RESOURCE_GROUP \
  --query gatewayUrl -o tsv)

echo "APIM Gateway URL: $APIM_GATEWAY_URL"
```

### Step 4.3: Import MCP API

Create an OpenAPI specification file `mcp-api-spec.yaml`:

```yaml
openapi: 3.0.1
info:
  title: MCP Atlassian API
  description: Model Context Protocol server for Atlassian integration
  version: "1.0"
servers:
  - url: https://{aca-fqdn}
    variables:
      aca-fqdn:
        default: your-container-app.azurecontainerapps.io
paths:
  /mcp:
    post:
      summary: MCP Endpoint
      description: Main MCP protocol endpoint for tool invocations
      operationId: mcpPost
      security:
        - oauth2: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
        '401':
          description: Unauthorized
  /healthz:
    get:
      summary: Health Check
      description: Kubernetes liveness/readiness probe endpoint
      operationId: healthCheck
      responses:
        '200':
          description: Service is healthy
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: ok
components:
  securitySchemes:
    oauth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.atlassian.com/authorize
          tokenUrl: https://auth.atlassian.com/oauth/token
          scopes:
            read:jira-work: Read Jira issues
            write:jira-work: Write Jira issues
            read:confluence-content.all: Read Confluence content
            write:confluence-content: Write Confluence content
            offline_access: Refresh token support
```

Import the API:

```bash
az apim api import \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --path mcp \
  --api-id mcp-atlassian \
  --specification-format OpenApiJson \
  --specification-path mcp-api-spec.yaml
```

### Step 4.4: Set Backend URL

```bash
az apim backend create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --backend-id mcp-atlassian-backend \
  --protocol http \
  --url "https://$ACA_FQDN"
```

---

## Part 5: APIM OAuth 2.1 Configuration

### Step 5.1: Register Atlassian as OAuth 2.0 Server

Navigate to **Azure Portal** → **API Management** → **OAuth 2.0 + OpenID Connect**:

1. Click **+ Add**
2. Configure:
   - **Display name**: Atlassian OAuth 2.0
   - **Client registration page URL**: `https://developer.atlassian.com/console/myapps/`
   - **Authorization endpoint URL**: `https://auth.atlassian.com/authorize`
   - **Token endpoint URL**: `https://auth.atlassian.com/oauth/token`
   - **Default scope**: `read:jira-work read:confluence-content.all offline_access`
   - **Client ID**: Your Atlassian OAuth Client ID
   - **Client secret**: Your Atlassian OAuth Client Secret
   - **Grant types**: ✓ Authorization code

### Step 5.2: Create Inbound Policy for Token Validation

Create an APIM policy to validate Atlassian OAuth tokens and forward them to the backend.

Navigate to **APIs** → **MCP Atlassian API** → **All operations** → **Inbound processing** → **Code editor**:

```xml
<policies>
    <inbound>
        <base />

        <!-- Rate limiting -->
        <rate-limit calls="100" renewal-period="60" />

        <!-- CORS for browser-based clients -->
        <cors allow-credentials="true">
            <allowed-origins>
                <origin>*</origin>
            </allowed-origins>
            <allowed-methods preflight-result-max-age="300">
                <method>*</method>
            </allowed-methods>
            <allowed-headers>
                <header>*</header>
            </allowed-headers>
        </cors>

        <!-- Validate OAuth 2.0 Bearer token -->
        <validate-jwt header-name="Authorization"
                      failed-validation-httpcode="401"
                      failed-validation-error-message="Unauthorized: Invalid or expired token"
                      require-expiration-time="true"
                      require-signed-tokens="false">
            <openid-config url="https://auth.atlassian.com/.well-known/openid-configuration" />
            <audiences>
                <audience>api.atlassian.com</audience>
            </audiences>
            <issuers>
                <issuer>https://auth.atlassian.com</issuer>
            </issuers>
        </validate-jwt>

        <!-- Extract Cloud ID from token claims (if present) -->
        <set-variable name="cloudId" value="@{
            var authHeader = context.Request.Headers.GetValueOrDefault("Authorization", "");
            if (string.IsNullOrEmpty(authHeader) || !authHeader.StartsWith("Bearer ")) {
                return "";
            }
            try {
                var token = authHeader.Substring(7);
                var handler = new System.IdentityModel.Tokens.Jwt.JwtSecurityTokenHandler();
                var jwtToken = handler.ReadJwtToken(token);
                var cloudIdClaim = jwtToken.Claims.FirstOrDefault(c => c.Type == "aud" || c.Type == "cloud_id");
                return cloudIdClaim?.Value ?? "";
            } catch {
                return "";
            }
        }" />

        <!-- Forward Cloud ID to backend if extracted -->
        <choose>
            <when condition="@(!string.IsNullOrEmpty((string)context.Variables["cloudId"]))">
                <set-header name="X-Atlassian-Cloud-Id" exists-action="override">
                    <value>@((string)context.Variables["cloudId"])</value>
                </set-header>
            </when>
        </choose>

        <!-- Ensure Authorization header is forwarded -->
        <set-header name="Authorization" exists-action="override">
            <value>@(context.Request.Headers.GetValueOrDefault("Authorization",""))</value>
        </set-header>

        <!-- Set backend URL -->
        <set-backend-service backend-id="mcp-atlassian-backend" />
    </inbound>

    <backend>
        <base />
    </backend>

    <outbound>
        <base />
        <!-- Add security headers -->
        <set-header name="X-Content-Type-Options" exists-action="override">
            <value>nosniff</value>
        </set-header>
        <set-header name="X-Frame-Options" exists-action="override">
            <value>DENY</value>
        </set-header>
    </outbound>

    <on-error>
        <base />
        <set-header name="Content-Type" exists-action="override">
            <value>application/json</value>
        </set-header>
        <set-body>@{
            return new JObject(
                new JProperty("error", context.LastError.Message),
                new JProperty("reason", context.LastError.Reason),
                new JProperty("requestId", context.RequestId)
            ).ToString();
        }</set-body>
    </on-error>
</policies>
```

### Step 5.3: Configure Health Check Bypass

For the `/healthz` endpoint, create a separate operation policy that bypasses authentication:

```xml
<policies>
    <inbound>
        <base />
        <!-- No authentication required for health checks -->
        <set-backend-service backend-id="mcp-atlassian-backend" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

### Step 5.4: Configure APIM Credential Manager (Alternative Approach)

For more advanced token management, use APIM's Credential Manager:

1. Navigate to **Credential manager** in APIM
2. Click **+ Create**
3. Configure provider:
   - **Provider**: Generic OAuth 2.0
   - **Identity provider URL**: `https://auth.atlassian.com`
   - **Authorization URL**: `https://auth.atlassian.com/authorize?audience=api.atlassian.com`
   - **Token URL**: `https://auth.atlassian.com/oauth/token`
   - **Refresh URL**: `https://auth.atlassian.com/oauth/token`
   - **Client ID**: Your Atlassian OAuth Client ID
   - **Client secret**: Your Atlassian OAuth Client Secret
   - **Scope**: `read:jira-work write:jira-work read:confluence-content.all offline_access`

4. Create a connection and authorize access

---

## Part 6: Testing and Validation

### Step 6.1: Test Health Endpoint

```bash
curl -s "$APIM_GATEWAY_URL/mcp/healthz" | jq
# Expected: {"status": "ok"}
```

### Step 6.2: Obtain OAuth Token

Use the OAuth 2.0 authorization code flow:

```bash
# Step 1: Open authorization URL in browser
AUTH_URL="https://auth.atlassian.com/authorize?\
audience=api.atlassian.com&\
client_id=$ATLASSIAN_OAUTH_CLIENT_ID&\
scope=read:jira-work%20read:confluence-content.all%20offline_access&\
redirect_uri=https://$APIM_NAME.azure-api.net/oauth2/callback&\
response_type=code&\
prompt=consent&\
state=random-state-string"

echo "Open this URL in your browser:"
echo $AUTH_URL

# Step 2: After authorization, exchange code for token
# (You'll receive a code via the callback URL)
curl -X POST "https://auth.atlassian.com/oauth/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "client_id=$ATLASSIAN_OAUTH_CLIENT_ID" \
  -d "client_secret=$ATLASSIAN_OAUTH_CLIENT_SECRET" \
  -d "code=YOUR_AUTHORIZATION_CODE" \
  -d "redirect_uri=https://$APIM_NAME.azure-api.net/oauth2/callback"
```

### Step 6.3: Test MCP Endpoint

```bash
# Replace with your actual access token
ACCESS_TOKEN="your-access-token"
CLOUD_ID="your-cloud-id"

# Test MCP tools listing (initialize request)
curl -X POST "$APIM_GATEWAY_URL/mcp" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-Atlassian-Cloud-Id: $CLOUD_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }' | jq
```

### Step 6.4: Test Jira Tool Invocation

```bash
curl -X POST "$APIM_GATEWAY_URL/mcp" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-Atlassian-Cloud-Id: $CLOUD_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "jira_get_projects",
      "arguments": {}
    }
  }' | jq
```

---

## Security Best Practices

### 1. Use Azure Key Vault for Secrets

Never store secrets in environment variables or code. Use Key Vault with managed identity:

```bash
# Reference secrets in Container Apps
az containerapp secret set \
  --name $ACA_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --secrets "oauth-client-id=keyvaultref:$KEY_VAULT_NAME/atlassian-oauth-client-id,identityref:system"
```

### 2. Enable Private Networking

Configure Container Apps to use internal ingress with APIM VNet integration:

```bash
# Create VNet
az network vnet create \
  --name mcp-vnet \
  --resource-group $RESOURCE_GROUP \
  --address-prefix 10.0.0.0/16

# Create subnet for Container Apps
az network vnet subnet create \
  --name aca-subnet \
  --vnet-name mcp-vnet \
  --resource-group $RESOURCE_GROUP \
  --address-prefix 10.0.0.0/23

# Create Container Apps environment with VNet
az containerapp env create \
  --name $ACA_ENV_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --infrastructure-subnet-resource-id "/subscriptions/.../subnets/aca-subnet" \
  --internal-only
```

### 3. Configure IP Restrictions

Restrict Container Apps ingress to APIM only:

```bash
az containerapp ingress access-restriction set \
  --name $ACA_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --rule-name apim-only \
  --ip-address <APIM-Public-IP> \
  --action Allow
```

### 4. Enable Logging and Monitoring

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name mcp-logs

# Enable Container Apps logging
az containerapp logs show \
  --name $ACA_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --follow
```

### 5. Token Security

- Always use HTTPS for all communications
- Implement token rotation policies
- Use short-lived access tokens with refresh tokens
- Monitor for suspicious token usage patterns

---

## Troubleshooting

### Common Issues

#### 1. Token Validation Fails (401 Unauthorized)

**Symptoms**: APIM returns `401 Unauthorized`

**Solutions**:
- Verify the token is not expired
- Check that scopes match what APIM expects
- Ensure the audience (`api.atlassian.com`) is correct
- Validate OIDC configuration URL is accessible

```bash
# Test OIDC configuration
curl -s https://auth.atlassian.com/.well-known/openid-configuration | jq
```

#### 2. Backend Connection Fails (502/503)

**Symptoms**: APIM returns gateway errors

**Solutions**:
- Verify Container App is running: `az containerapp show --name $ACA_APP_NAME ...`
- Check Container App logs: `az containerapp logs show --name $ACA_APP_NAME ...`
- Test health endpoint directly against Container App

#### 3. Cloud ID Not Found

**Symptoms**: Atlassian APIs return `Site not found` errors

**Solutions**:
- Ensure `X-Atlassian-Cloud-Id` header is being forwarded
- Verify Cloud ID is correct for your Atlassian site
- Check that the OAuth token has access to the specified cloud

```bash
# Verify accessible resources
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
  https://api.atlassian.com/oauth/token/accessible-resources | jq
```

#### 4. Scope Errors

**Symptoms**: `insufficient_scope` errors from Atlassian APIs

**Solutions**:
- Verify requested scopes in your OAuth app configuration
- Re-authorize with the correct scopes
- Check that `offline_access` is included for refresh tokens

### Debug Logging

Enable verbose logging in MCP-Atlassian:

```bash
az containerapp update \
  --name $ACA_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --set-env-vars "MCP_VERY_VERBOSE=true"
```

---

## Reference Links

### Atlassian Documentation

- [OAuth 2.0 (3LO) Apps](https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)
- [Authorization Code Grants](https://developer.atlassian.com/cloud/jira/platform/oauth-2-authorization-code-grants-3lo/)
- [Scopes for OAuth 2.0](https://developer.atlassian.com/cloud/jira/platform/scopes-for-oauth-2-3lo-and-forge-apps/)
- [Jira Cloud REST API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/)
- [Confluence Cloud REST API](https://developer.atlassian.com/cloud/confluence/rest/v2/intro/)

### Azure Documentation

- [Azure Container Apps Overview](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [Azure API Management Overview](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts)
- [APIM OAuth 2.0 Authorization](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-oauth2)
- [APIM Credential Manager](https://learn.microsoft.com/en-us/azure/api-management/credentials-overview)
- [Validate JWT Policy](https://learn.microsoft.com/en-us/azure/api-management/validate-jwt-policy)
- [Azure Key Vault Integration](https://learn.microsoft.com/en-us/azure/container-apps/manage-secrets)

### MCP-Atlassian Documentation

- [MCP-Atlassian GitHub Repository](https://github.com/sooperset/mcp-atlassian)
- [OAuth Setup Guide](https://github.com/sooperset/mcp-atlassian#c-oauth-20-authentication-cloud---advanced)
- [Docker Deployment](https://github.com/sooperset/mcp-atlassian#-2-installation)

---

## Appendix: Environment Variables Reference

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `ATLASSIAN_OAUTH_ENABLE` | Enable OAuth mode for user-provided tokens | Yes (for APIM) | `false` |
| `ATLASSIAN_OAUTH_CLIENT_ID` | OAuth 2.0 client ID | For full OAuth | - |
| `ATLASSIAN_OAUTH_CLIENT_SECRET` | OAuth 2.0 client secret | For full OAuth | - |
| `ATLASSIAN_OAUTH_CLOUD_ID` | Atlassian Cloud ID | For full OAuth | - |
| `ATLASSIAN_OAUTH_ACCESS_TOKEN` | Pre-existing access token (BYOT) | For BYOT mode | - |
| `STREAMABLE_HTTP_PATH` | HTTP endpoint path | Yes | `/mcp` |
| `MCP_VERBOSE` | Enable verbose logging | No | `false` |
| `MCP_VERY_VERBOSE` | Enable debug logging | No | `false` |
| `READ_ONLY_MODE` | Disable write operations | No | `false` |
| `JIRA_URL` | Jira instance URL | If using Jira | - |
| `CONFLUENCE_URL` | Confluence instance URL | If using Confluence | - |

---

## Quick Start Checklist

- [ ] Created Atlassian OAuth 2.0 app with correct scopes
- [ ] Recorded Client ID, Client Secret, and Cloud ID
- [ ] Created Azure Resource Group
- [ ] Created Azure Container Apps environment
- [ ] Deployed MCP-Atlassian container with `ATLASSIAN_OAUTH_ENABLE=true`
- [ ] Created Azure API Management instance
- [ ] Configured APIM OAuth 2.0 server for Atlassian
- [ ] Applied APIM inbound policy for token validation
- [ ] Tested health endpoint through APIM
- [ ] Tested MCP endpoint with valid OAuth token
- [ ] Configured Key Vault for production secrets
- [ ] Enabled monitoring and logging
