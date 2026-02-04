# Azure MCP Server - ACA with Managed Identity

This document explains how to deploy the [Azure MCP Server 2.0-beta](https://mcr.microsoft.com/product/azure-sdk/azure-mcp) as a remote MCP server accessible over HTTPS. This enables AI agents from [Microsoft Foundry](https://azure.microsoft.com/products/ai-foundry) and [Microsoft Copilot Studio](https://www.microsoft.com/microsoft-copilot/microsoft-copilot-studio) to securely invoke MCP tool calls that perform Azure operations on your behalf.

This reference Azure Developer CLI (azd) template shows how to host the server on Azure Container Apps with your choice of Azure MCP Server namespaces enabled, using managed identity authentication for secure access to Azure resources.

## Prerequisites

- Azure subscription with **Owner** or **User Access Administrator** permissions
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)
- The list of Azure MCP Server tool areas (namespaces) you wish to enable (see [azmcp-commands.md](https://github.com/microsoft/mcp/blob/main/servers/Azure.Mcp.Server/docs/azmcp-commands.md))

## Available Namespaces

The Azure MCP Server supports multiple namespaces covering different Azure services:
- `storage` - Azure Storage operations (Blob, Queue, Table)
- `costmanagement` - Cost Management and billing operations
- `compute` - Azure Compute resources (VMs, Scale Sets)
- `network` - Azure Networking resources
- `monitor` - Azure Monitor and Application Insights
- And more - see the [full list](https://github.com/microsoft/mcp/blob/main/servers/Azure.Mcp.Server/docs/azmcp-commands.md)

You can enable 1-3 namespaces per deployment based on your needs.

## Quick Start

This reference template deploys the Azure MCP Server with **read-only** Azure tools enabled, accessible over HTTPS transport. For details on customizing server startup flags and configuration, see [Azure MCP Server documentation](https://github.com/microsoft/mcp/blob/main/servers/Azure.Mcp.Server/docs/azmcp-commands.md).

### Default Storage Deployment

By default, the template deploys with the `storage` namespace enabled:

```bash
azd up
```

You'll be prompted for:
- **Storage Account Resource ID** - The Azure resource ID of the storage account the MCP server will access
- **Microsoft Foundry Project Resource ID** - The Azure resource ID of the Microsoft Foundry project for agent integration

### Custom Namespace Deployment

To deploy with different namespaces (e.g., for cost management/FinOps), set the `AZURE_MCP_NAMESPACES` environment variable:

**For Cost Management:**
```bash
azd env set AZURE_MCP_NAMESPACES '["costmanagement"]'
azd up
```

**For Multiple Namespaces:**
```bash
azd env set AZURE_MCP_NAMESPACES '["costmanagement","monitor"]'
azd up
```

**For Storage with Compute:**
```bash
azd env set AZURE_MCP_NAMESPACES '["storage","compute"]'
azd env set STORAGE_RESOURCE_ID "/subscriptions/.../storageAccounts/..."
azd up
```

> **Note:** When using the `storage` namespace, you must provide a `STORAGE_RESOURCE_ID`. For other namespaces, this is optional and can be omitted.

## What Gets Deployed

- **Container App** - Runs Azure MCP Server with your specified namespace(s)
- **Role Assignments** - Container App managed identity granted appropriate roles based on enabled namespaces:
  - For `storage` namespace: Reader and Storage Blob Data Reader on specified storage account
  - For other namespaces: Subscription-level Reader role (read-only access)
- **Entra App Registration** - For incoming OAuth 2.0 authentication from clients (e.g., agents) with `Mcp.Tools.ReadWrite.All` role. This role is assigned to the managed identity of the Microsoft Foundry project specified by the input Microsoft Foundry resource ID
- **Application Insights** - Telemetry and monitoring

### Deployment Outputs

After deployment, retrieve `azd` outputs:

```bash
azd env get-values
```

Among the output there are useful values for the subsequent steps. Here is an example of these values.

```
CONTAINER_APP_URL="https://azure-mcp-server.wonderfulazmcp-a9561afd.eastus2.azurecontainerapps.io"
ENTRA_APP_CLIENT_ID="c3248eaf-3bdd-4ca7-9483-4fcf213e4d4d"
ENTRA_APP_IDENTIFIER_URI="api://c3248eaf-3bdd-4ca7-9483-4fcf213e4d4d"
ENTRA_APP_OBJECT_ID="a89055df-ccfc-4aef-a7c6-9561bc4c5386"
ENTRA_APP_ROLE_ID="3e60879b-a1bd-5faf-bb8c-cb55e3bfeeb8"
ENTRA_APP_SERVICE_PRINCIPAL_ID="31b42369-583b-40b7-a535-ad343f75e463"
```

## Using Azure MCP Server from Microsoft Foundry Agent

Once deployed, connect your Microsoft Foundry agent to the Azure MCP Server running on Azure Container Apps. The agent will authenticate using its managed identity and gain access to the configured Azure Storage tools.

1. Get your Container App URL from `azd` output: `CONTAINER_APP_URL`
2. Get Entra App Client ID from `azd` output: `ENTRA_APP_CLIENT_ID`
2. Navigate to your Foundry project: https://ai.azure.com/nextgen
3. Go to **Build** → **Create agent**  
4. Select the **+ Add** in the tools section
5. Select the **Custom** tab 
6. Choose **Model Context Protocol** as the tool and click **Create** ![Find MCP](images/azure__create-aif-agent-mcp-tool.png)
7. Configure the MCP connection ![Create MCP Connection](images/azure__add_aif_mcp_connection.png)
   - Enter the `CONTAINER_APP_URL` value as the Remote MCP Server endpoint. 
   - Select **Microsoft Entra** → **Project Managed Identity**  as the authentication method
   - Enter your `ENTRA_APP_CLIENT_ID` as the audience.
   - Click **Connect** to associate this connection to the agent

Your agent is now ready to assist you! It can answer your questions and leverage tools from the Azure MCP Server to perform Azure operations on your behalf.

## Clean Up

```bash
azd down
```

## Common Use Cases

### FinOps and Cost Management

For cost management and financial operations across all Azure resources:

```bash
# Set up cost management namespace
azd env set AZURE_MCP_NAMESPACES '["costmanagement"]'
azd up
```

This enables your AI agents to:
- Query Azure cost and usage data
- Analyze spending trends across subscriptions
- Generate cost reports and recommendations
- Monitor budget alerts and anomalies

The Container App's managed identity will need appropriate Cost Management Reader permissions at the subscription or management group level for full functionality.

### Multi-Service Monitoring

For comprehensive monitoring across compute, storage, and network:

```bash
# Set up multiple namespaces
azd env set AZURE_MCP_NAMESPACES '["compute","storage","network"]'
azd env set STORAGE_RESOURCE_ID "/subscriptions/.../storageAccounts/..."
azd up
```

## Template Structure

The `azd` template consists of the following Bicep modules:

- **`main.bicep`** - Orchestrates the deployment of all resources
- **`aca-infrastructure.bicep`** - Deploys Container App hosting the Azure MCP Server
- **`aca-role-assignment-resource.bicep`** - Assigns Azure storage RBAC roles to the Container App managed identity on the storage account (conditional, only if storage namespace is enabled)
- **`entra-app.bicep`** - Creates Entra App registration with custom app role for OAuth 2.0 authentication
- **`foundry-role-assignment-entraapp.bicep`** - Assigns Entra App role to the managed identity of the Microsoft Foundry project specified by the input Microsoft Foundry resource ID for the Azure MCP Server access
- **`application-insights.bicep`** - Deploys Application Insights for telemetry and monitoring (conditional deployment)

