# Microsoft Sentinel – Function App Data Connector Accelerator

This repository is an **ISV accelerator** for building and publishing a Microsoft Sentinel data connector using an **Azure Function** and the **Azure Monitor Ingestion API** (DCE/DCR). It provides a fully working end-to-end template — including the ARM deployment, Python function code, and Sentinel connector UI definition — that you can deploy as-is to validate the pattern, then adapt for your own data source.

---

## Overview

The accelerator follows the **standard two-ARM-template pattern** used by all Microsoft Sentinel solutions:

| Deployment | What it does | Template |
|---|---|---|
| **1 – Connector card** | Installs your connector UI into Sentinel Data Connectors | `FunctionApp/Package/mainTemplate.json` |
| **2 – Function App** | Deploys the Function App, DCE, DCR, Key Vault, and custom log table | `FunctionApp/Data Connectors/azuredeploy_FunctionApp_API_FunctionApp.json` |

```
Log Analytics Workspace + Microsoft Sentinel
    │
    └── Deploy mainTemplate.json  →  connector card appears in Data Connectors
            │
            └── Open connector → Deploy to Azure  →  Function App + DCE + DCR + Key Vault
                    │
                    └── Assign Monitoring Metrics Publisher on DCR
                            │
                            └── Function App runs every 10 min → data lands in FunctionAppSample_CL
```

---

## Scenario 1 – Test As-Is (No Code Changes)

Follow these steps to deploy the accelerator exactly as-is to verify the full ingestion pipeline end-to-end before customizing it for your data source.

---

### Prerequisites

- An Azure subscription with permission to create resource groups, register applications in Microsoft Entra ID, and assign RBAC roles
- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) (optional, for CLI-based deployments)
- [Git](https://git-scm.com/)

---

### Step 1 – Clone the Repositories

You need both the Azure Sentinel repository (for the packaging toolchain) and this accelerator.

```powershell
git clone https://github.com/Azure/Azure-Sentinel.git
git clone https://github.com/robertmoriarty12/Sentinel-Ingestion-Accelerator-Function-App.git
```

Copy the accelerator solution folder into the Azure Sentinel solutions directory:

```powershell
Copy-Item "Sentinel-Ingestion-Accelerator-Function-App\FunctionApp" `
          "Azure-Sentinel\Solutions\FunctionApp" -Recurse -Force
```

> **Note:** This copy step places your solution where the Sentinel packaging tools expect it. If you are submitting a PR to the official Azure Sentinel repository, this is where your solution folder will live permanently.

---

### Step 2 – Deploy a Log Analytics Workspace and Enable Microsoft Sentinel

If you do not already have a Sentinel workspace:

1. In the [Azure Portal](https://portal.azure.com), search for **Microsoft Sentinel** and click **Create**
2. Click **Create a new workspace**, fill in workspace name, resource group, and region, then click **Review + Create**
3. Once the workspace is created, click **Add** to enable Microsoft Sentinel on it

Note the following — you will need them later:

| Value | Where to find it |
|---|---|
| **Workspace name** | Log Analytics workspace → Overview |
| **Workspace Resource ID** | Log Analytics workspace → Properties → Resource ID |
| **Region** | Log Analytics workspace → Overview |

> **Region tip:** Deploy all resources to the same region. If you hit `SubscriptionIsOverQuotaForSku` errors in East US, try **Central US**.

---

### Step 3 – Deploy the Connector (mainTemplate.json)

This step installs the **"Function App Sample"** connector card into your Sentinel **Data Connectors** gallery. It does not deploy the Function App itself.

#### Option A – Azure Portal

1. In the Azure Portal, search for **"Deploy a custom template"** and select it
2. Click **Build your own template in the editor**
3. Paste the contents of [`FunctionApp/Package/mainTemplate.json`](FunctionApp/Package/mainTemplate.json) and click **Save**
4. Fill in the parameters:

| Parameter | Value |
|---|---|
| `workspace` | Your Log Analytics workspace name from Step 2 |
| `workspace-location` | Azure region of your workspace (e.g. `centralus`) |

5. Deploy to the **same resource group** as your Sentinel workspace

#### Option B – Azure CLI

```powershell
az deployment group create `
  --resource-group <your-sentinel-rg> `
  --template-file "Azure-Sentinel\Solutions\FunctionApp\Package\mainTemplate.json" `
  --parameters workspace="<your-workspace-name>" "workspace-location"="centralus"
```

#### Verify

Go to **Microsoft Sentinel → Data Connectors** and search for **"Function App Sample (Ingestion API)"**. The connector card should appear within 1–2 minutes.

---

### Step 4 – Create an Azure AD App Registration

The Function App uses a service principal to authenticate to Azure Monitor and push data into Sentinel.

1. In the Azure Portal, go to **Microsoft Entra ID → App registrations → New registration**
2. Enter a name (e.g. `FunctionApp-Sentinel-Connector`) and click **Register**
3. Under **Certificates & secrets → Client secrets → New client secret**, create a secret — copy the **Value** immediately (it is only shown once)
4. From the App Registration **Overview** page, note:

| Value | ARM parameter |
|---|---|
| Directory (tenant) ID | `TenantId` |
| Application (client) ID | `ClientId` |
| Client secret value | `ClientSecret` |

---

### Step 5 – Assign Key Vault Secrets Officer to the Deploying User

The Function App ARM template creates an Azure Key Vault and writes the `ClientSecret` into it. The deploying user or service principal must have the **Key Vault Secrets Officer** role (or **Key Vault Administrator**) on the target resource group or subscription **before** deployment — without it the deployment will fail with an authorization error when writing the secret.

1. In the Azure Portal, navigate to the **resource group** you plan to deploy into
2. Go to **Access control (IAM) → Add role assignment**
3. Select **Key Vault Secrets Officer**
4. Under **Members**, select the user or service principal that will run the deployment
5. Click **Review + assign**

---

### Step 6 – Deploy the Function App via the Connector Card

1. In **Microsoft Sentinel → Data Connectors**, find **"Function App Sample (Ingestion API)"** and click **Open connector page**
2. Follow **STEP 1** (already done in Step 4 above)
3. Under **STEP 2 – Deploy the Azure Function App**, click **Deploy to Azure**
4. Fill in the parameters:

| Parameter | Description |
|---|---|
| `WorkspaceName` | Log Analytics workspace name from Step 2 |
| `TenantId` | Tenant ID from Step 4 |
| `ClientId` | Client ID from Step 4 |
| `ClientSecret` | Client secret value from Step 4 |
| `AppInsightsWorkspaceResourceID` | Full Resource ID from Step 2 |
| `FunctionAppLocation` | Azure region (e.g. `centralus`). Can differ from RG location. |

5. Click **Review + create** and wait for deployment to complete (~3–5 minutes)

---

### Step 7 – Assign Monitoring Metrics Publisher on the Data Collection Rule

After the Function App deployment completes, grant the App Registration permission to ingest data into Sentinel.

1. In the Azure Portal, navigate to the **Data Collection Rule** created by the deployment (search by name in the resource group from Step 6, or find it in the deployment outputs)
2. Go to **Access control (IAM) → Add role assignment**
3. Select **Monitoring Metrics Publisher**
4. Under **Members**, choose **User, group, or service principal** and select the App Registration from Step 4
5. Click **Review + assign**

> **Critical:** This role must be assigned **directly on the Data Collection Rule resource** — assigning it at the resource group or subscription scope will **not work** and will result in a `403 Forbidden` error when the Function App attempts to ingest data. Wait **5–10 minutes** for RBAC propagation before testing.

---

### Step 8 – Verify Data Ingestion

The Function App runs automatically on a timer every 10 minutes. To verify data is flowing:

#### Check the function loaded

In the Azure Portal, go to your **Function App → Functions**. You should see `AzureFunctionFunctionApp` listed.

#### Manually trigger the function (optional)

1. Function App → **Functions** → `AzureFunctionFunctionApp`
2. Click **Test/Run → Run**
3. Watch the **Logs** tab — look for `Successfully ingested N event(s)`

#### Query Sentinel Logs

In **Microsoft Sentinel → Logs**, run:

```kql
FunctionAppSample_CL
| sort by TimeGenerated desc
| take 10
```

Allow **5–10 minutes** for ingestion lag after the function runs. Once data appears the connector status card in Sentinel will flip to **Connected**.

---

## Scenario 2 – Customize for Your Data Source

This scenario walks through the complete process of adapting the accelerator for your own connector — changing the table name, adding schema fields, updating the function code, and publishing everything from your own GitHub fork.

The example change is deliberately simple: rename the custom table from `FunctionAppSample_CL` to `ISVSecurity_CL` and add two new schema fields (`UserName` and `RiskScore`) to illustrate exactly which files need touching and why.

---

### Prerequisites for Scenario 2

- Scenario 1 completed — you have a working baseline connector deployed in Sentinel
- A GitHub account you can fork repositories into
- [PowerShell 7+](https://learn.microsoft.com/powershell/scripting/install/installing-powershell) for running the packaging tool
- [Node.js 18+](https://nodejs.org/) (required by the packaging tool)
- Python 3.11 (for rebuilding the function zip)

---

### Step 1 – Fork and Clone the Azure Sentinel Repository

The ISV works in their own fork of the official Azure Sentinel repository. This is where your solution permanently lives when you eventually submit a PR — and in the meantime, it gives you a publicly accessible GitHub raw URL for `WEBSITE_RUN_FROM_PACKAGE`.

1. Go to [https://github.com/Azure/Azure-Sentinel](https://github.com/Azure/Azure-Sentinel) and click **Fork**
2. Clone your fork locally:

```powershell
git clone https://github.com/<your-github-username>/Azure-Sentinel.git
cd Azure-Sentinel
```

3. Copy the accelerator's `FunctionApp/` folder into `Solutions/` in your fork:

```powershell
Copy-Item "..\Sentinel-Ingestion-Accelerator-Function-App\FunctionApp" `
          ".\Solutions\FunctionApp" -Recurse -Force
```

Your working directory for all Scenario 2 changes is:

```
Azure-Sentinel\Solutions\FunctionApp\
```

---

### Step 2 – Identify All Files That Need Changing

When you rename the table or add schema fields, these files must all be kept in sync:

| File | What to change |
|---|---|
| `Data Connectors/azuredeploy_FunctionApp_API_FunctionApp.json` | `tableName` variable, `streamName` variable, table column list, DCR stream declaration column list |
| `Data Connectors/FunctionApp_API_FunctionApp.json` | All KQL query references to the old table name; connector note text mentioning the table name |
| `Data Connectors/AzureFunctionFunctionApp/main.py` | Sample event payload — add new fields so test data matches the schema |
| `Data Connectors/FunctionAppSample.zip` | Rebuilt from `main.py` and all dependencies (see Step 4) |
| `Package/mainTemplate.json` | **Do not edit directly** — regenerated by the packaging tool in Step 5 |

---

### Step 3 – Update the ARM Template

Open `Data Connectors/azuredeploy_FunctionApp_API_FunctionApp.json`.

**3a – Rename the table and stream** (in the `variables` block):

```json
"tableName": "ISVSecurity_CL",
"streamName": "Custom-ISVSecurity_CL",
```

**3b – Add new columns** in two places — both the table schema and the DCR `streamDeclarations` must list identical columns:

```json
{ "name": "UserName", "type": "string" },
{ "name": "RiskScore", "type": "int"    }
```

The table schema (`Microsoft.OperationalInsights/workspaces/tables`) and the DCR stream declaration (`streamDeclarations`) must always be kept in sync — if they differ, ingestion will fail with a schema mismatch error.

**3c – Update `WEBSITE_RUN_FROM_PACKAGE`** to point to your fork (see Step 6):

```json
"WEBSITE_RUN_FROM_PACKAGE": "https://raw.githubusercontent.com/<your-github-username>/Azure-Sentinel/main/Solutions/FunctionApp/Data%20Connectors/FunctionAppSample.zip"
```

---

### Step 4 – Update the Connector UI Definition

Open `Data Connectors/FunctionApp_API_FunctionApp.json`.

- Replace every occurrence of the old table name (`FunctionAppSample_CL`) with your new name (`ISVSecurity_CL`) — this covers `graphQueries`, `sampleQueries`, `dataTypes`, `connectivityCriterias`, and the NOTE description text
- Update the **Deploy to Azure button URL** to point to the `azuredeploy` ARM template in your fork:

```
https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2F<your-github-username>%2FAzure-Sentinel%2Fmain%2FSolutions%2FFunctionApp%2FData%2520Connectors%2Fazuredeploy_FunctionApp_API_FunctionApp.json
```

Use PowerShell to do the table name rename in bulk rather than find-and-replace manually:

```powershell
(Get-Content "Data Connectors\FunctionApp_API_FunctionApp.json" -Raw) `
    -replace 'FunctionAppSample_CL', 'ISVSecurity_CL' |
  Set-Content "Data Connectors\FunctionApp_API_FunctionApp.json" -NoNewline
```

---

### Step 5 – Update main.py

Open `Data Connectors/AzureFunctionFunctionApp/main.py` and add the new fields to every event object in `build_sample_events()`:

```python
"UserName": "alice@contoso.com",
"RiskScore": 72,
```

Every field declared in the ARM template schema must be present in the payload (or explicitly set to `None`/omitted — Azure Monitor will accept missing optional fields). Sending a field that is **not** in the schema will be silently dropped.

---

### Step 6 – Rebuild the Function Zip

The zip must contain the updated `main.py` plus all Python dependencies pre-bundled for Linux. Run this from the `Data Connectors\` directory:

```powershell
$src     = ".\Data Connectors"
$staging = "$env:TEMP\fnapp_staging"
$pkgDir  = "$staging\.python_packages\lib\site-packages"

Remove-Item $staging -Recurse -Force -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Path $pkgDir -Force | Out-Null

# Install packages
python -m pip install azure-functions==1.21.3 azure-identity==1.19.0 `
  azure-monitor-ingestion==1.0.4 azure-core==1.32.0 `
  --target $pkgDir --no-user --quiet

# Replace Windows cryptography/cffi with Linux manylinux wheels (Function App runs Linux)
$wheelDir = "$env:TEMP\fnapp_wheels"
New-Item -ItemType Directory -Path $wheelDir -Force | Out-Null
Remove-Item "$pkgDir\cryptography*","$pkgDir\cffi*","$pkgDir\_cffi_backend*" -Recurse -Force -ErrorAction SilentlyContinue
python -m pip download cryptography cffi `
  --dest $wheelDir `
  --platform manylinux2014_x86_64 --python-version 311 `
  --implementation cp --only-binary=:all: --quiet
Get-ChildItem $wheelDir -Filter "*.whl" | ForEach-Object {
    Expand-Archive -Path $_.FullName -DestinationPath $pkgDir -Force
}

# Copy function files and zip
Copy-Item "$src\host.json","$src\requirements.txt","$src\proxies.json" $staging -Force
Copy-Item "$src\AzureFunctionFunctionApp" "$staging\AzureFunctionFunctionApp" -Recurse -Force
Push-Location $staging
Compress-Archive -Path ".\*" -DestinationPath "$src\FunctionAppSample.zip" -Force
Pop-Location

Write-Host "Zip size: $([int]((Get-Item "$src\FunctionAppSample.zip").Length / 1MB)) MB"
```

> **Critical packaging rules:**
> - All Python packages must be under `.python_packages/lib/site-packages/` inside the zip — they are NOT installed from `requirements.txt` when using `WEBSITE_RUN_FROM_PACKAGE` from a URL
> - `cryptography` and `cffi` must use Linux manylinux wheels — Windows `.pyd` builds will raise `ImportError` on the Linux runtime

---

### Step 7 – Re-run the Packaging Tool

Regenerate `Package/mainTemplate.json` from your updated source files. Run from the root of the `Azure-Sentinel` repo:

```powershell
pwsh -File ".\Tools\Create-Azure-Sentinel-Solution\V3\createSolutionV3.ps1" `
    -SolutionDataFolderPath ".\Solutions\FunctionApp\Data"
```

The tool will update `Package/mainTemplate.json` and `Package/3.0.0.zip`. The ARM-TTK check `IDs Should Be Derived From ResourceIDs` will report `Passed: False` — this is expected for all Sentinel solutions and is not a blocker.

> **Never edit `Package/mainTemplate.json` directly.** Always edit the source files and re-run the tool.

---

### Step 8 – Commit and Push to Your Fork

```powershell
git add "Solutions/FunctionApp/Data Connectors/azuredeploy_FunctionApp_API_FunctionApp.json"
git add "Solutions/FunctionApp/Data Connectors/FunctionApp_API_FunctionApp.json"
git add "Solutions/FunctionApp/Data Connectors/AzureFunctionFunctionApp/main.py"
git add "Solutions/FunctionApp/Data Connectors/FunctionAppSample.zip"
git add "Solutions/FunctionApp/Package/mainTemplate.json"
git add "Solutions/FunctionApp/Package/3.0.0.zip"
git commit -m "ISV: rename table to ISVSecurity_CL, add UserName+RiskScore schema fields"
git push origin main
```

Once pushed, the raw URL for your zip is live and `WEBSITE_RUN_FROM_PACKAGE` will resolve correctly when someone deploys the Function App.

---

### Step 9 – Re-deploy the Connector Card, Then Re-deploy the Function App

Your schema changes require re-deploying both ARM templates:

1. **Re-deploy `mainTemplate.json`** to your Sentinel workspace (same as Scenario 1 Step 3) — this updates the connector card with your new table name and KQL queries
2. **Open the updated connector in Data Connectors → Open connector page → STEP 2 → Deploy to Azure** — the Deploy to Azure button now points to your fork, which has the updated ARM template

> **Note on existing deployments:** If you previously deployed with the old table name, Azure will create the new table alongside the old one. The old `FunctionAppSample_CL` table will persist but stop receiving data. You can delete it via the Log Analytics workspace → Tables blade.

---

### Scenario 2 Summary – What Changes and Why

| What changed | Why |
|---|---|
| `tableName` / `streamName` variables in ARM template | Controls what table and DCR stream name Azure creates |
| Column list in table schema AND DCR stream declaration | Must be identical — DCR validates schema on ingestion |
| `main.py` payload | New fields must be in the payload so test data actually populates the new columns |
| `FunctionAppSample.zip` rebuilt | `WEBSITE_RUN_FROM_PACKAGE` serves a static file — the Function App won't see `main.py` changes until the zip is replaced |
| `FunctionApp_API_FunctionApp.json` KQL queries | Connector status card and sample queries would break with the old table name |
| `WEBSITE_RUN_FROM_PACKAGE` URL | Must point to your fork — the accelerator repo is the ISV's starting point, not their deployment target |
| `mainTemplate.json` regenerated | This file is the installed connector artifact — it must be current with your source files |

---

## Repository Structure

```
FunctionApp/
├── Data/
│   └── Solution_FunctionApp.json          # Solution metadata for Sentinel packaging tool
├── Data Connectors/
│   ├── FunctionApp_API_FunctionApp.json   # Connector UI definition (source — edit this)
│   ├── azuredeploy_FunctionApp_API_FunctionApp.json  # Function App ARM template
│   ├── AzureFunctionFunctionApp/
│   │   ├── main.py                        # Python function — replace with your API logic
│   │   └── function.json                  # Timer trigger config (every 10 min)
│   ├── FunctionAppSample.zip              # Pre-built deployment package
│   ├── host.json
│   ├── requirements.txt
│   └── proxies.json
├── Package/
│   ├── mainTemplate.json                  # Generated by packaging tool — do not edit directly
│   ├── 3.0.0.zip                          # Solution package
│   ├── createUiDefinition.json
│   └── testParameters.json
├── SolutionMetadata.json
├── README.md
└── ReleaseNotes.md
```

> **Important:** Never edit `Package/mainTemplate.json` directly. Always edit `Data Connectors/FunctionApp_API_FunctionApp.json` and re-run the packaging tool.

---

## Architecture

```
Azure Function (Timer, every 10 min)
    │
    ├── Reads CLIENT_SECRET via Key Vault reference (no plain-text secrets)
    ├── ClientSecretCredential → token for https://monitor.azure.com
    ├── LogsIngestionClient.upload() → POST to DCE endpoint
    │       HTTP 204 (success)
    │
    └── DCE → DCR (stream mapping) → FunctionAppSample_CL
                                            │
                                            └── Microsoft Sentinel Logs
```

### Key App Settings (set automatically by ARM template)

| Setting | Source |
|---|---|
| `TENANT_ID` | ARM parameter |
| `CLIENT_ID` | ARM parameter |
| `CLIENT_SECRET` | Key Vault reference — resolved at runtime by managed identity |
| `DCE_ENDPOINT` | Auto-resolved from DCE resource via ARM `reference()` |
| `DCR_ID` | Auto-resolved from DCR `immutableId` via ARM `reference()` |
| `STREAM_NAME` | `Custom-FunctionAppSample_CL` |

