# Azure MCP Server - Deployment Examples

This document provides practical examples for deploying the Azure MCP Server with different namespace configurations to support various use cases.

## Example 1: FinOps and Cost Management

Deploy a server focused on cost management and financial operations:

```bash
# Create environment
azd init

# Configure for cost management
azd env set AZURE_MCP_NAMESPACES '["costmanagement"]'
azd env set FOUNDRY_PROJECT_RESOURCE_ID "/subscriptions/YOUR_SUB/resourceGroups/YOUR_RG/providers/Microsoft.MachineLearningServices/workspaces/YOUR_WORKSPACE"

# Deploy
azd up
```

**What you get:**
- Cost Management tools for querying spending data
- Budget and anomaly monitoring capabilities
- Usage and billing report generation
- No storage dependencies

**Agent Capabilities:**
- "Show me the Azure spending for last month"
- "What are the top 5 most expensive resources?"
- "Generate a cost breakdown by resource group"
- "Are we exceeding any budgets?"

## Example 2: Storage Operations (Default)

Deploy with storage namespace for blob, queue, and table operations:

```bash
# Create environment
azd init

# Configure for storage (or omit to use defaults)
azd env set AZURE_MCP_NAMESPACES '["storage"]'
azd env set STORAGE_RESOURCE_ID "/subscriptions/YOUR_SUB/resourceGroups/YOUR_RG/providers/Microsoft.Storage/storageAccounts/YOUR_STORAGE"
azd env set FOUNDRY_PROJECT_RESOURCE_ID "/subscriptions/YOUR_SUB/resourceGroups/YOUR_RG/providers/Microsoft.MachineLearningServices/workspaces/YOUR_WORKSPACE"

# Deploy
azd up
```

**What you get:**
- Read-only access to specified storage account
- Blob, queue, and table operations
- Storage analytics capabilities

## Example 3: Multi-Service Monitoring

Deploy with monitoring and compute for comprehensive infrastructure insights:

```bash
# Create environment
azd init

# Configure for monitoring and compute
azd env set AZURE_MCP_NAMESPACES '["monitor","compute"]'
azd env set FOUNDRY_PROJECT_RESOURCE_ID "/subscriptions/YOUR_SUB/resourceGroups/YOUR_RG/providers/Microsoft.MachineLearningServices/workspaces/YOUR_WORKSPACE"

# Deploy
azd up
```

**What you get:**
- Azure Monitor and Application Insights access
- VM and compute resource monitoring
- Metrics and log queries
- No storage dependencies

**Agent Capabilities:**
- "Show me the CPU usage for my VMs"
- "What are the recent Application Insights errors?"
- "List all VMs in the subscription"
- "Query logs for the last hour"

## Example 4: Hybrid Storage and Compute

Deploy with both storage and compute namespaces:

```bash
# Create environment
azd init

# Configure for storage and compute
azd env set AZURE_MCP_NAMESPACES '["storage","compute"]'
azd env set STORAGE_RESOURCE_ID "/subscriptions/YOUR_SUB/resourceGroups/YOUR_RG/providers/Microsoft.Storage/storageAccounts/YOUR_STORAGE"
azd env set FOUNDRY_PROJECT_RESOURCE_ID "/subscriptions/YOUR_SUB/resourceGroups/YOUR_RG/providers/Microsoft.MachineLearningServices/workspaces/YOUR_WORKSPACE"

# Deploy
azd up
```

**What you get:**
- Storage operations on specified account
- Compute resource management
- Combined infrastructure operations

## Example 5: Network and Security

Deploy with network namespace for networking insights:

```bash
# Create environment
azd init

# Configure for network operations
azd env set AZURE_MCP_NAMESPACES '["network"]'
azd env set FOUNDRY_PROJECT_RESOURCE_ID "/subscriptions/YOUR_SUB/resourceGroups/YOUR_RG/providers/Microsoft.MachineLearningServices/workspaces/YOUR_WORKSPACE"

# Deploy
azd up
```

**What you get:**
- Virtual network information
- Network security group queries
- Load balancer and gateway access
- No storage dependencies

## Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `AZURE_MCP_NAMESPACES` | No | `["storage"]` | Array of 1-3 namespaces to enable |
| `STORAGE_RESOURCE_ID` | Conditional | `""` | Required only if `storage` namespace is used |
| `FOUNDRY_PROJECT_RESOURCE_ID` | Yes | N/A | Microsoft Foundry project resource ID |
| `AZURE_LOCATION` | Yes | N/A | Azure region for deployment |

## Available Namespaces

Full list of available namespaces (see [Azure MCP Server docs](https://github.com/microsoft/mcp/blob/main/servers/Azure.Mcp.Server/docs/azmcp-commands.md) for complete details):

- `storage` - Azure Storage (Blob, Queue, Table)
- `costmanagement` - Cost Management and billing
- `compute` - Virtual Machines and Scale Sets
- `network` - Virtual Networks and security
- `monitor` - Azure Monitor and Application Insights
- `containerservice` - Azure Kubernetes Service
- `sql` - Azure SQL Database
- `cosmosdb` - Cosmos DB
- `keyvault` - Key Vault secrets and certificates
- And more...

## Tips and Best Practices

### 1. Choose Namespaces Based on Use Case

Don't enable more namespaces than you need. Each namespace adds tools that may not be relevant to your agent's purpose.

**Good:**
```bash
# For FinOps agent
azd env set AZURE_MCP_NAMESPACES '["costmanagement"]'
```

**Not ideal:**
```bash
# Unnecessary namespaces for FinOps
azd env set AZURE_MCP_NAMESPACES '["costmanagement","storage","compute"]'
```

### 2. Namespace Limits

You can enable up to 3 namespaces per deployment. If you need more, consider deploying multiple MCP servers with different namespace configurations.

### 3. Storage Namespace Requirement

The `storage` namespace requires a storage account resource ID. Other namespaces don't have this requirement:

```bash
# This works - no storage namespace
azd env set AZURE_MCP_NAMESPACES '["costmanagement"]'

# This requires STORAGE_RESOURCE_ID
azd env set AZURE_MCP_NAMESPACES '["storage"]'
azd env set STORAGE_RESOURCE_ID "/subscriptions/.../storageAccounts/..."
```

### 4. RBAC Permissions

The Container App's managed identity needs appropriate permissions:

- **Storage namespace**: Automatically assigned Reader and Storage Blob Data Reader on specified storage account
- **Other namespaces**: May require subscription-level Reader or specific resource-level permissions

For cost management, you may need to manually assign "Cost Management Reader" role at the subscription or management group level.

## Troubleshooting

### Issue: "Parameter 'storageResourceId' cannot be empty"

**Solution:** Either include the storage namespace and provide a storage resource ID, or remove storage from namespaces:

```bash
# Option 1: Remove storage from namespaces
azd env set AZURE_MCP_NAMESPACES '["costmanagement"]'

# Option 2: Provide storage resource ID
azd env set STORAGE_RESOURCE_ID "/subscriptions/.../storageAccounts/..."
```

### Issue: Cost management queries not working

**Solution:** Ensure the Container App's managed identity has "Cost Management Reader" role:

```bash
# Get the Container App's principal ID
PRINCIPAL_ID=$(azd env get-value CONTAINER_APP_PRINCIPAL_ID)

# Assign Cost Management Reader role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Cost Management Reader" \
  --scope "/subscriptions/YOUR_SUBSCRIPTION_ID"
```

### Issue: Cannot enable more than 3 namespaces

**Solution:** This is a design limit. Deploy multiple MCP servers if you need more namespaces, each with different configurations.
