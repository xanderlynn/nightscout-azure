# Nightscout on Azure — ARM Templates

Deploy [Nightscout](https://nightscout.github.io/) to Azure using **100% Azure resources** on the free tier via a single ARM template.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fxanderlynn%2Fnightscout-azure%2Fmain%2Fazuredeploy.json)

---

## What this deploys

| Resource | SKU | Cost |
|---|---|---|
| App Service Plan | F1 (Free, Linux) | $0/month |
| App Service (Web App) | Linux container | $0/month |
| Cosmos DB (MongoDB API) | Free tier (1000 RU/s, 25 GB) | $0/month |

> ⚠️ **Cosmos DB free tier** is limited to **one per Azure subscription**. If you already have a free tier Cosmos DB account, the deployment will fail — you'll need to remove `"enableFreeTier": true` from the template and accept the standard Cosmos DB pricing (or use a different subscription).

> ⚠️ **Cosmos DB compatibility**: Azure Cosmos DB for MongoDB API has known partial incompatibilities with Nightscout. The template applies the recommended workaround (`socketTimeoutMS=120000`) in the connection string. Most features work; some edge cases may not.

> ⚠️ **F1 App Service plan** includes 60 CPU minutes/day. Fine for personal CGM use. Not recommended for FreeAPS X or OpenAPS closed-loop users — consider upgrading to D1 Shared ($).

---

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `appName` | ✅ | — | Globally unique web app name (lowercase, hyphens ok). Becomes `https://<appName>.azurewebsites.net`. Also used to name the Cosmos DB account (`<appName>-db`). |
| `apiSecret` | ✅ | — | Nightscout password — 12+ characters, no spaces |
| `displayUnits` | — | `mg/dl` | `mg/dl` or `mmol/L` |
| `enablePlugins` | — | standard set | Space-separated Nightscout plugin list |
| `nightscoutDockerImage` | — | `nightscout/cgm-remote-monitor:latest` | Docker image tag |
| `location` | — | resource group location | Azure region |

The MongoDB connection string is **automatically constructed** from the Cosmos DB account — you don't need to provide it.

---

## Quick deploy — "Deploy to Azure" button

1. Click the **Deploy to Azure** button above
2. Select or create a **Resource Group**
3. Fill in `appName` and `apiSecret`
4. Click **Review + create** → **Create**
5. Wait ~5 minutes (Cosmos DB provisioning takes a few minutes)
6. Browse to `https://<appName>.azurewebsites.net`

---

## Deploy with Azure CLI

```bash
# Login
az login

# Create resource group
az group create --name nightscout-rg --location eastus

# Deploy — you will be prompted for apiSecret
az deployment group create \
  --resource-group nightscout-rg \
  --template-file azuredeploy.json \
  --parameters appName=<your-unique-name> \
               apiSecret='<your-secret-12-chars>'
```

---

## Deploy with GitHub Actions

The included workflow deploys on push to `main` or via manual trigger.

### Secrets required (Settings → Secrets and variables → Actions)

| Secret | Description |
|---|---|
| `AZURE_CREDENTIALS` | Service principal JSON (see below) |
| `NIGHTSCOUT_API_SECRET` | Nightscout API secret (12+ chars) |

### Variables required

| Variable | Description |
|---|---|
| `AZURE_RESOURCE_GROUP` | Resource group name (default: `nightscout-rg`) |
| `AZURE_LOCATION` | Azure region (default: `eastus`) |
| `NIGHTSCOUT_APP_NAME` | Your app name |
| `DISPLAY_UNITS` | `mg/dl` or `mmol/L` |

### Create a service principal

```bash
az ad sp create-for-rbac \
  --name "nightscout-github-actions" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/nightscout-rg \
  --sdk-auth
```

Paste the JSON output into the `AZURE_CREDENTIALS` secret.

---

## After deployment

1. Browse to `https://<appName>.azurewebsites.net`
2. Complete the initial profile setup (timezone, etc.)
3. Authenticate with your API secret
4. Review [Nightscout security settings](https://nightscout.github.io/nightscout/security/)

### Connect your CGM uploader

- **xDrip+** — set Nightscout URL in xDrip sync settings
- **Dexcom Share bridge** — add these app settings in Azure portal → App Service → Environment Variables:
  - `BRIDGE_USER_NAME`, `BRIDGE_PASSWORD`, `BRIDGE_SERVER` (US or EU)
  - Add `bridge` to your `ENABLE` app setting
- **Loop / AAPS** — configure Nightscout URL + API secret in the app

### Updating Nightscout

Restart the App Service to pull the latest image:

**Azure portal → App Service → Overview → Restart**

To pin a version: set `nightscoutDockerImage` to e.g. `nightscout/cgm-remote-monitor:15.0.2`.
