# Nightscout on Azure — ARM Templates

Deploy [Nightscout](https://nightscout.github.io/) to Azure App Service using the **free F1 tier** via ARM templates.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FYOUR_ORG%2Fnightscout-azure%2Fmain%2Fazuredeploy.json)

> **Replace `YOUR_ORG`** in the button URL above with your GitHub username or org name after creating the repo.

---

## What this deploys

| Resource | SKU | Cost |
|---|---|---|
| App Service Plan | F1 (Free, Linux) | $0/month |
| App Service (Web App) | Linux container | $0/month |

> ⚠️ **F1 plan limits**: 60 CPU minutes/day, 1 GB RAM, shared infrastructure. Suitable for personal use. Not recommended for FreeAPS X or OpenAPS closed-loop systems — consider upgrading to D1 Shared ($) for those use cases.

> ⚠️ **No Azure database**: Azure Cosmos DB (MongoDB API) is **not fully compatible** with Nightscout. Use an external MongoDB provider such as [MongoDB Atlas](https://www.mongodb.com/atlas) (free M0 tier available).

---

## Prerequisites

1. **Azure account** — [Create a free account](https://azure.microsoft.com/en-us/free/)
2. **MongoDB connection string** — [Set up MongoDB Atlas (free)](https://nightscout.github.io/nightscout/database/)
3. Azure CLI installed locally **or** deploy via GitHub Actions / the "Deploy to Azure" button above

---

## Quick start: Deploy to Azure button

1. Click the **Deploy to Azure** button above (update the URL with your GitHub username first)
2. Fill in the required parameters:
   - **App Name** — unique lowercase name (becomes `https://<appName>.azurewebsites.net`)
   - **MongoDB URI** — your Atlas connection string
   - **API Secret** — at least 12 characters, no spaces
   - **Display Units** — `mg/dl` or `mmol/L`
3. Click **Review + create**, then **Create**
4. Wait ~2 minutes for deployment, then browse to your URL

---

## Deploy with Azure CLI

```bash
# 1. Login
az login

# 2. Create a resource group
az group create --name nightscout-rg --location eastus

# 3. Deploy (you will be prompted for secure parameters)
az deployment group create \
  --resource-group nightscout-rg \
  --template-file azuredeploy.json \
  --parameters appName=<your-unique-name> \
               displayUnits=mg/dl \
               mongodbUri='<your-mongodb-uri>' \
               apiSecret='<your-api-secret>'
```

---

## Deploy with GitHub Actions

GitHub Actions can deploy automatically on pushes to `main` or via manual trigger.

### Required secrets (Settings → Secrets and variables → Actions)

| Secret | Description |
|---|---|
| `AZURE_CREDENTIALS` | Service principal JSON from `az ad sp create-for-rbac` (see below) |
| `MONGODB_URI` | Your MongoDB Atlas connection string |
| `NIGHTSCOUT_API_SECRET` | Nightscout API secret (12+ chars) |

### Required variables

| Variable | Description |
|---|---|
| `AZURE_RESOURCE_GROUP` | Resource group name (default: `nightscout-rg`) |
| `AZURE_LOCATION` | Azure region (default: `eastus`) |
| `NIGHTSCOUT_APP_NAME` | Your app name (used on push deploys) |
| `DISPLAY_UNITS` | `mg/dl` or `mmol/L` |

### Create an Azure service principal

```bash
az ad sp create-for-rbac \
  --name "nightscout-github-actions" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/nightscout-rg \
  --sdk-auth
```

Copy the JSON output into the `AZURE_CREDENTIALS` secret.

---

## Parameters reference

| Parameter | Required | Default | Description |
|---|---|---|---|
| `appName` | ✅ | — | Globally unique web app name |
| `mongodbUri` | ✅ | — | MongoDB Atlas connection string |
| `apiSecret` | ✅ | — | Nightscout password (12+ chars) |
| `displayUnits` | — | `mg/dl` | `mg/dl` or `mmol/L` |
| `enablePlugins` | — | standard set | Space-separated plugin list |
| `nightscoutDockerImage` | — | `nightscout/cgm-remote-monitor:latest` | Docker image tag |
| `location` | — | resource group location | Azure region |

### Dexcom Share (optional)

To pull CGM data directly from Dexcom Share, add `bridge` to `enablePlugins` and set these app settings manually in the Azure portal after deployment (or extend the template):

- `BRIDGE_USER_NAME` — Dexcom account username
- `BRIDGE_PASSWORD` — Dexcom account password
- `BRIDGE_SERVER` — `US` or `EU`

---

## After deployment

1. Browse to `https://<appName>.azurewebsites.net`
2. Complete the initial profile setup
3. Authenticate with your API secret
4. Review [Nightscout security settings](https://nightscout.github.io/nightscout/security/)

---

## Updating Nightscout

To pull a newer Nightscout image, restart the App Service in the Azure portal:

**App Service → Overview → Restart**

Azure will pull the latest `nightscout/cgm-remote-monitor:latest` image on next start.

To pin a specific version, set the `nightscoutDockerImage` parameter to e.g. `nightscout/cgm-remote-monitor:15.0.2`.
