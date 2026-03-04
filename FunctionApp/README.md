# Microsoft Sentinel – Function App Sample Data Connector

This solution demonstrates how to build a production-ready Microsoft Sentinel data connector using an **Azure Function** and the **Azure Monitor Ingestion API** (DCE/DCR pattern). Use it as a starting template for ingesting custom JSON events from any source into a Sentinel custom log table.

---

## Architecture Overview

```
Azure Function (Timer Trigger, every 10 min)
    │
    ├── Authenticates via App Registration (ClientSecretCredential)
    ├── Builds sample JSON events
    └── POSTs to Data Collection Endpoint (DCE)
            │
            └── Data Collection Rule (DCR) → FunctionAppSample_CL table
                    │
                    └── Microsoft Sentinel (Log Analytics Workspace)
```

The ARM template **automatically creates** all required Azure Monitor infrastructure:
- Data Collection Endpoint (DCE)
- Custom log table (`FunctionAppSample_CL`)
- Data Collection Rule (DCR) with stream mapping
- **Monitoring Metrics Publisher** role assignment on the DCR (for the App Registration)
- Application Insights (linked to your workspace)
- Storage Account, App Service Plan (Y1/Consumption), Function App

---

## Prerequisites

Before deploying, ensure your Azure subscription has the following **resource providers registered**:

```powershell
az provider register --namespace Microsoft.Web
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.Authorization
```

Check registration status:
```powershell
az provider show --namespace Microsoft.Web --query registrationState
```

---

## Step 1 – Create an Azure AD App Registration

The Function App uses a service principal (App Registration) to authenticate to Azure Monitor.

1. In the [Azure Portal](https://portal.azure.com), go to **Microsoft Entra ID → App registrations → New registration**
2. Name it (e.g. `FunctionApp-Sentinel-Connector`) and click **Register**
3. Go to **Certificates & secrets → New client secret** — note the **Client Secret value** (shown once only)
4. On the **Overview** page, note:
   - **Application (client) ID** → used as `ClientId`
   - **Directory (tenant) ID** → used as `TenantId`
   - **Object ID** → used as `AzureClientObjectId` (required for the DCR role assignment)

> **Important:** The Object ID on the App Registration Overview page is the **service principal object ID** — this is what the ARM template uses for the role assignment. Do not confuse it with the Enterprise Application Object ID.

---

## Step 2 – Deploy via ARM Template

### Option A – Deploy to Azure Button

[![Deploy To Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Frobertmoriarty12%2FAzure-Sentinel%2Fmaster%2FSolutions%2FFunctionApp%2FData%2520Connectors%2Fazuredeploy_FunctionApp_API_FunctionApp.json)

### Option B – Azure CLI

```powershell
az deployment group create \
  --resource-group <your-resource-group> \
  --template-uri "https://raw.githubusercontent.com/robertmoriarty12/Azure-Sentinel/master/Solutions/FunctionApp/Data%20Connectors/azuredeploy_FunctionApp_API_FunctionApp.json" \
  --parameters \
      FunctionName="FunctionAppSample" \
      WorkspaceName="<your-sentinel-workspace-name>" \
      TenantId="<tenant-id>" \
      ClientId="<client-id>" \
      ClientSecret="<client-secret>" \
      AzureClientObjectId="<object-id>" \
      AppInsightsWorkspaceResourceID="<workspace-resource-id>"
```

### ARM Template Parameters

| Parameter | Description | Example |
|---|---|---|
| `FunctionName` | Prefix for all created resources (max 18 chars) | `FunctionAppSample` |
| `WorkspaceName` | Name of your Log Analytics / Sentinel workspace | `my-sentinel-ws` |
| `TenantId` | Azure AD Tenant ID from Step 1 | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `ClientId` | App Registration Client ID from Step 1 | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `ClientSecret` | App Registration Client Secret from Step 1 | `abc123~...` |
| `AzureClientObjectId` | App Registration **Object ID** from Step 1 | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `AppInsightsWorkspaceResourceID` | Full Resource ID of the Log Analytics workspace | `/subscriptions/{sub}/resourcegroups/{rg}/providers/microsoft.operationalinsights/workspaces/{name}` |

> **Finding `AppInsightsWorkspaceResourceID`:** In the Azure Portal, go to **Log Analytics workspace → Properties → Resource ID** and copy the full value.

### Region Requirement

Deploy the resource group to **Central US** or another region where Azure Functions Consumption (Y1/Dynamic) plan quota is available. East US may block Consumption plan deployments on trial/MSDN subscriptions.

```powershell
az group create --name <your-rg> --location centralus
```

---

## Step 3 – Verify Deployment

After deployment completes (~3-5 minutes), verify the resources were created:

```powershell
az resource list --resource-group <your-rg> --output table
```

You should see:
- `Microsoft.Insights/dataCollectionEndpoints`
- `Microsoft.Insights/dataCollectionRules`
- `Microsoft.OperationalInsights/workspaces/tables` (`FunctionAppSample_CL`)
- `Microsoft.Web/serverfarms`
- `Microsoft.Web/sites` (Function App)
- `Microsoft.Storage/storageAccounts`
- `Microsoft.Insights/components` (Application Insights)

Verify the role assignment on the DCR:
```powershell
$dcrId=$(az monitor data-collection rule list --resource-group <your-rg> --query "[0].id" -o tsv)
az role assignment list --scope $dcrId --query "[].{role:roleDefinitionName, principal:principalId}" -o table
```
Should show `Monitoring Metrics Publisher` assigned to your App Registration Object ID.

---

## Step 4 – Verify the Function App Loaded

In the Azure Portal, go to your Function App → **Functions**. You should see `AzureFunctionFunctionApp` listed.

If no functions appear, check **Log stream** for startup errors. Common causes:
- `WEBSITE_RUN_FROM_PACKAGE` URL unreachable
- Missing Python packages in the zip (see [Troubleshooting](#troubleshooting))

---

## Step 5 – Manually Trigger the Function

The function runs automatically every 10 minutes. To trigger it immediately:

### Portal
1. Function App → **Functions** → `AzureFunctionFunctionApp`
2. Click **Test/Run** → **Run**
3. Watch the **Logs** tab — you should see live output within seconds

### PowerShell (via Admin API)
```powershell
# Get master key
$key = az functionapp keys list --name <function-app-name> --resource-group <your-rg> --query "masterKey" -o tsv

# Trigger
Invoke-WebRequest `
  -Uri "https://<function-app-name>.azurewebsites.net/admin/functions/AzureFunctionFunctionApp" `
  -Method Post `
  -Headers @{"x-functions-key" = $key} `
  -ContentType "application/json" `
  -Body "{}" `
  -UseBasicParsing
```

### Azure CLI
```bash
az functionapp function list --name <function-app-name> --resource-group <your-rg>
```

---

## Step 6 – Verify Data in Sentinel

After a successful run, query the table in **Microsoft Sentinel → Logs** (allow ~5 minutes for ingestion):

```kql
FunctionAppSample_CL
| sort by TimeGenerated desc
| take 10
```

Filter by event type:
```kql
FunctionAppSample_CL
| where EventType == "Alert"
| sort by TimeGenerated desc
```

---

## File Structure

```
Solutions/FunctionApp/
├── Data Connectors/
│   ├── azuredeploy_FunctionApp_API_FunctionApp.json   # ARM deployment template
│   ├── FunctionApp_API_FunctionApp.json               # Sentinel connector UI definition
│   ├── FunctionAppSample.zip                          # Deployed to Function App via WEBSITE_RUN_FROM_PACKAGE
│   ├── host.json                                      # Azure Functions host config (bundled in zip)
│   ├── requirements.txt                               # Python dependencies (bundled in zip)
│   ├── proxies.json                                   # Azure Functions proxies (bundled in zip)
│   ├── .funcignore                                    # Files excluded from deployment
│   └── AzureFunctionFunctionApp/
│       ├── main.py                                    # Timer trigger function logic
│       └── function.json                              # Trigger binding (timer, every 10 min)
├── Package/
│   ├── mainTemplate.json                              # Sentinel Content Hub solution template
│   └── createUiDefinition.json
├── Data/
│   └── Solution_FunctionApp.json                     # Solution metadata
└── README.md
```

---

## How It Works – Code Walkthrough

### `function.json` – Timer Trigger
```json
{
  "bindings": [{
    "type": "timerTrigger",
    "schedule": "0 */10 * * * *",
    "runOnStartup": false
  }]
}
```
Runs every 10 minutes. `runOnStartup: false` prevents firing immediately on cold start.

### `main.py` – Ingestion Logic
```python
# 1. Read config from App Settings (set by ARM template automatically)
TENANT_ID     = os.environ.get("TENANT_ID")
CLIENT_ID     = os.environ.get("CLIENT_ID")
CLIENT_SECRET = os.environ.get("CLIENT_SECRET")
DCE_ENDPOINT  = os.environ.get("DCE_ENDPOINT")   # set from DCE resource at deploy time
DCR_ID        = os.environ.get("DCR_ID")          # set from DCR immutableId at deploy time
STREAM_NAME   = os.environ.get("STREAM_NAME")     # "Custom-FunctionAppSample_CL"

# 2. Authenticate
creds = ClientSecretCredential(tenant_id=TENANT_ID, client_id=CLIENT_ID, client_secret=CLIENT_SECRET)
client = LogsIngestionClient(endpoint=DCE_ENDPOINT, credential=creds)

# 3. Upload events
client.upload(rule_id=DCR_ID, stream_name=STREAM_NAME, logs=events)
```

### ARM Template – App Settings (auto-configured)
The ARM template uses `reference()` to populate DCE/DCR values at deploy time:
```json
"DCE_ENDPOINT": "[reference(resourceId('Microsoft.Insights/dataCollectionEndpoints', variables('endpointName'))).logsIngestion.endpoint]",
"DCR_ID":       "[reference(resourceId('Microsoft.Insights/dataCollectionRules', variables('mainRuleName'))).immutableId]"
```

---

## Adapting This Template for Your Own Connector

1. **Modify `main.py`** — replace `build_sample_events()` with your actual data source API calls
2. **Update the table schema** in the ARM template (`Microsoft.OperationalInsights/workspaces/tables`) to match your fields
3. **Update `tableName` and `streamName`** variables in the ARM template
4. **Update the DCR `dataFlow`** column mappings to match your schema
5. **Update `requirements.txt`** with any additional Python packages your API client needs
6. **Rebuild the zip** (see [Rebuilding the Zip](#rebuilding-the-zip)) and push to GitHub
7. **Update `WEBSITE_RUN_FROM_PACKAGE`** in the ARM template to point to your new zip URL

---

## Rebuilding the Zip

When you modify function code or Python dependencies, rebuild and re-push the zip:

```powershell
$src   = "path/to/Data Connectors"
$staging = "$env:TEMP\fnapp_staging"
$pkgDir  = "$staging\.python_packages\lib\site-packages"

# Clean staging
Remove-Item $staging -Recurse -Force -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Path $pkgDir -Force

# Install packages
python -m pip install `
  azure-functions==1.21.3 azure-identity==1.19.0 `
  azure-monitor-ingestion==1.0.4 azure-core==1.32.0 `
  --target $pkgDir --no-user

# Replace cryptography with Linux-compatible manylinux wheels
$wheelDir = "$env:TEMP\fnapp_wheels"
New-Item -ItemType Directory -Path $wheelDir -Force
Remove-Item "$pkgDir\cryptography*","$pkgDir\cffi*","$pkgDir\_cffi_backend*" -Recurse -Force -ErrorAction SilentlyContinue
python -m pip download cryptography cffi `
  --dest $wheelDir `
  --platform manylinux2014_x86_64 --python-version 311 `
  --implementation cp --only-binary=:all:
Get-ChildItem $wheelDir -Filter "*.whl" | ForEach-Object {
    Expand-Archive -Path $_.FullName -DestinationPath $pkgDir -Force
}

# Copy function files into staging and zip
Copy-Item "$src\host.json","$src\requirements.txt","$src\proxies.json" $staging -Force
Copy-Item "$src\AzureFunctionFunctionApp" "$staging\AzureFunctionFunctionApp" -Recurse -Force
Push-Location $staging
Compress-Archive -Path ".\*" -DestinationPath "$src\FunctionAppSample.zip" -Force
Pop-Location
```

> **Critical:** Python packages must be bundled inside the zip under `.python_packages/lib/site-packages/`. The `cryptography` package must use Linux manylinux wheels (not Windows `.pyd` files) because the Function App runs on Linux.

After rebuilding, push the zip to GitHub and restart the Function App:
```powershell
git add "Data Connectors/FunctionAppSample.zip"
git commit -m "Update function zip with latest code/packages"
git push

az functionapp restart --name <function-app-name> --resource-group <your-rg>
```

---

## Troubleshooting

### Functions Not Showing in Portal
- Check that `WEBSITE_RUN_FROM_PACKAGE` in App Settings points to a publicly accessible URL
- Go to **Function App → Log stream** — look for startup errors
- Verify the zip contains `host.json`, `requirements.txt`, and the function folder

### `ModuleNotFoundError: No module named 'azure.identity'`
The zip does not contain the Python packages. Packages must be **pre-bundled** inside the zip under `.python_packages/lib/site-packages/`. See [Rebuilding the Zip](#rebuilding-the-zip).

### `ImportError: cannot import name 'x509' from cryptography.hazmat.bindings._rust`
The `cryptography` package was built for Windows but the function runs on Linux. You must use Linux manylinux wheels. See [Rebuilding the Zip](#rebuilding-the-zip).

### `SubscriptionIsOverQuotaForSku` on Deployment
Your subscription has no quota for the VM SKU in the selected region. Try deploying to **Central US** — the Y1/Consumption plan has separate quota per region. Create the resource group in Central US before deploying.

### Authentication Errors (`ClientAuthenticationError`)
- Verify `TenantId`, `ClientId`, `ClientSecret` in App Settings are correct
- Verify the App Registration's **Monitoring Metrics Publisher** role is assigned on the DCR:
  ```powershell
  $dcrId=$(az monitor data-collection rule list -g <rg> --query "[0].id" -o tsv)
  az role assignment list --scope $dcrId -o table
  ```

### Data Not Appearing in `FunctionAppSample_CL`
- Allow up to 5 minutes for ingestion lag
- Check **Invocations** tab on the function for success/failure status
- Verify the table was created: in Sentinel Logs, run `search "FunctionAppSample_CL"`

### Check Live Logs
In the portal: Function App → function → **Logs** tab (shows real-time streaming output when connected). Or use Application Insights:
```kql
AppExceptions
| where TimeGenerated > ago(30m)
| project TimeGenerated, OuterMessage, InnermostMessage
| order by TimeGenerated desc
```

---

## Key App Settings (set automatically by ARM template)

| Setting | Description |
|---|---|
| `TENANT_ID` | Azure AD Tenant ID |
| `CLIENT_ID` | App Registration Client ID |
| `CLIENT_SECRET` | App Registration Client Secret |
| `DCE_ENDPOINT` | Data Collection Endpoint logs ingestion URL |
| `DCR_ID` | Data Collection Rule immutableId |
| `STREAM_NAME` | `Custom-FunctionAppSample_CL` |
| `WEBSITE_RUN_FROM_PACKAGE` | URL to the function zip file on GitHub |
| `FUNCTIONS_EXTENSION_VERSION` | `~4` |
| `FUNCTIONS_WORKER_RUNTIME` | `python` |
| `AzureWebJobsStorage` | Storage account connection string |

---

## Resources

- [Azure Monitor Ingestion API overview](https://learn.microsoft.com/azure/azure-monitor/logs/logs-ingestion-api-overview)
- [Azure Functions Python developer guide](https://learn.microsoft.com/azure/azure-functions/functions-reference-python)
- [Data Collection Rules overview](https://learn.microsoft.com/azure/azure-monitor/essentials/data-collection-rule-overview)
- [Microsoft Sentinel data connector development guide](https://learn.microsoft.com/azure/sentinel/create-custom-connector)
- [Azure Functions pricing](https://azure.microsoft.com/pricing/details/functions/)
- [Azure Monitor pricing](https://azure.microsoft.com/pricing/details/monitor/)
