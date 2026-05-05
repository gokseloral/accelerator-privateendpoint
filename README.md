# Azure Content Understanding behind Private Endpoint + Power Platform

End-to-end accelerator for hosting **Azure AI Content Understanding** behind a
**Private Endpoint** and consuming it from a **Power Platform** Managed
Environment via **Enterprise Policy / VNet injection**.

## One-click deploy

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fgokseloral%2Faccelerator-privateendpoint%2Fmain%2Finfra%2Fazuredeploy.json)
[![Visualize](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/visualizebutton.svg?sanitize=true)](http://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2Fgokseloral%2Faccelerator-privateendpoint%2Fmain%2Finfra%2Fazuredeploy.json)

The button above provisions everything in one resource group: VNet (primary +
paired secondary), AI Services account with Private Endpoint, Private DNS
zones, and the Power Platform Enterprise Policy resource.

The portal will prompt for:

| Parameter | Description | Example |
| --- | --- | --- |
| `baseName` | 3–11 chars, lowercase alphanumerics, used to derive resource names | `prvendcu` |
| `location` | Primary Azure region (Content Understanding-supported) | `westus`, `swedencentral`, `australiaeast` |
| `secondaryLocation` | Paired Azure region for the second PP-delegated subnet | `eastus` (paired with `westus`) |
| `powerPlatformGeo` | PP geo for the Enterprise Policy resource location (NOT an Azure region) | `unitedstates`, `europe`, `asia`, `australia` |
| `powerPlatformEnvironmentId` | GUID of the target PP environment (NOT the org URL) | `00000000-0000-0000-0000-000000000000` |
| `vnetAddressPrefix` / `peSubnetPrefix` / `ppSubnetPrefix` | Primary VNet + subnet CIDRs | `10.50.0.0/16` / `10.50.1.0/24` / `10.50.2.0/24` |
| `secondaryVnetAddressPrefix` / `secondaryPpSubnetPrefix` | Secondary VNet + delegated subnet CIDRs (must NOT overlap primary) | `10.51.0.0/16` / `10.51.2.0/24` |
| `enterprisePolicyName` | Name of the `Microsoft.PowerPlatform/enterprisePolicies` resource | `ep-vnet-prvendcu` |

Tenant ID, subscription ID, and the signed-in identity are taken from the
portal session — you don't have to enter them.

### Region pair reference

Pick `secondaryLocation` from the [Azure paired regions list](https://learn.microsoft.com/azure/reliability/cross-region-replication-azure#azure-paired-regions). The paired region for each supported primary:

| Primary (`location`) | Paired (`secondaryLocation`) | PP geo (`powerPlatformGeo`) |
| --- | --- | --- |
| `westus` | `eastus` | `unitedstates` |
| `swedencentral` | `swedensouth` | `europe` |
| `australiaeast` | `australiasoutheast` | `australia` |

### After the ARM deployment finishes

The ARM template stops at provisioning. To complete the integration, run the
final linking step locally (it requires Power Platform admin auth which can't
be done from ARM):

```powershell
# Copy .env.example -> .env, fill in PP_ENVIRONMENT_ID + PP_TENANT_ID
./scripts/link-enterprise-policy.ps1 -UseDeviceCode
./scripts/create-and-test-connector.ps1
```

---

## Manual / scripted path

> Why this matters: Azure AI Services lets you set `publicNetworkAccess=Disabled`,
> but Power Platform is **not** a "trusted Microsoft service" for Cognitive
> Services — `bypass=AzureServices` alone won't let Power Platform reach a
> private-endpoint-locked account. A Power Platform Enterprise Policy linked
> to a delegated subnet is the supported way to bridge that gap.

## What gets deployed

```
                                   ┌─ vnet-<base>          (region A, e.g. westus)
                                   │   ├─ snet-pe          ──► Private Endpoint
                                   │   │                       to Azure AI Services
                                   │   └─ snet-powerplatform   (delegated)
                                   │
Power Platform Managed Env ───► Enterprise Policy (vnet)
                                   │
                                   └─ vnet-<base>-sec      (region B, e.g. eastus)
                                       └─ snet-powerplatform   (delegated, paired)
```

* `Microsoft.CognitiveServices/accounts` (kind `AIServices`), public network access **disabled**, Private Endpoint into `snet-pe`, custom subdomain enabled.
* Three Private DNS zones linked to the primary VNet:
  `privatelink.cognitiveservices.azure.com`, `privatelink.openai.azure.com`,
  `privatelink.services.ai.azure.com`.
* Two delegated subnets (in paired Azure regions) for Power Platform VNet injection — required by the multi-region Power Platform geos (e.g. `unitedstates` requires `westus` + `eastus`).
* `Microsoft.PowerPlatform/enterprisePolicies` (kind `vnet`) referencing both delegated subnets.
* Power Platform custom connector for the Content Understanding REST API.

## Repo layout

```
infra/
  main.bicep                # VNet (primary + secondary), AI Services, PE, DNS
  main.bicepparam           # default address spaces (no env-specific values)
  enterprise-policy.bicep   # Microsoft.PowerPlatform/enterprisePolicies (vnet)
powerplatform/
  contentunderstanding-connector.swagger.json   # OpenAPI 2.0 source
  apiProperties.json                            # connection params / branding
scripts/
  load-env.ps1                     # parses .env into env vars
  deploy.ps1                       # deploys infra/* using values from .env
  link-enterprise-policy.ps1       # Enable/Disable subnet injection on PP env
  create-and-test-connector.ps1    # pac connector create + connectivity test
.env.example                       # copy to .env and fill in
```

## Prerequisites

| Requirement | Notes |
| --- | --- |
| Azure subscription Owner / Contributor | RG, networking, AI Services, PE, DNS, Enterprise Policy |
| Azure CLI ≥ `2.50` | older versions cannot accept some inline params |
| PowerShell 7+ | `pwsh` |
| `pac` CLI signed in to the target PP environment | `pac auth list` |
| `Microsoft.PowerPlatform.EnterprisePolicies` PowerShell module | auto-installed by `link-enterprise-policy.ps1` |
| Power Platform / Global Administrator | required to enable Managed Environment + link the policy |
| Target environment is a **Managed Environment** | Sandbox is not allowed; enable in PPAC |

## Configure

Copy `.env.example` → `.env` and fill in the values for your tenant:

```ini
AZURE_SUBSCRIPTION_ID=<sub guid>
AZURE_RESOURCE_GROUP=rg-prvendtest
AZURE_LOCATION=westus
AZURE_SECONDARY_LOCATION=eastus
BASE_NAME=prvendcu

PP_TENANT_ID=<tenant guid>
PP_ENVIRONMENT_ID=<env guid>          # NOT the org URL
PP_GEO=unitedstates                   # PP geo for the policy resource
ENTERPRISE_POLICY_NAME=ep-vnet-prvendcu
```

`.env` is gitignored; `.env.example` is the only file checked in.

## Run

```powershell
# 1. Provision Azure infra + Enterprise Policy (reads .env)
./scripts/deploy.ps1

# 2. Link the Enterprise Policy to the Managed PP environment
#    (interactive Az sign-in; add -UseDeviceCode if no browser popup appears)
./scripts/link-enterprise-policy.ps1

# 3. Push custom connector + run the connectivity test
./scripts/create-and-test-connector.ps1
```

`deploy.ps1` writes a local `scripts/deployment-outputs.json` that the other
scripts consume. That file is gitignored because it contains real resource
IDs and hostnames.

To unlink the policy (e.g. before changing the policy's subnets):

```powershell
./scripts/link-enterprise-policy.ps1 -Unlink
```

## Connectivity test semantics

`create-and-test-connector.ps1` calls
`GET /contentunderstanding/analyzers?api-version=<preview>` directly against
the AI Services endpoint:

| Run from | Expected | Meaning |
| --- | --- | --- |
| Your laptop (public internet) | `403 Public access is disabled` | ✅ lockdown is working |
| VM inside `snet-pe` (use `-InsideVnetTest`) | `200 OK` | ✅ private endpoint + DNS working |
| Power Automate flow in linked env | `200 OK` | ✅ end-to-end PP → PE working |

> The "Test operation" button in the **custom connector designer** routes
> through the connector authoring host (`*.azure-apihub.net`) and **does not
> use VNet injection**. Always validate from a real flow run.

## Verifying the Enterprise Policy link

```powershell
. ./scripts/load-env.ps1
$tok = az account get-access-token --resource 'https://service.powerapps.com/' --query accessToken -o tsv
$uri = "https://api.bap.microsoft.com/providers/Microsoft.BusinessAppPlatform/scopes/admin/environments/$env:PP_ENVIRONMENT_ID" + '?api-version=2019-10-01&$expand=properties.enterprisePolicies'
Invoke-RestMethod -Uri $uri -Headers @{ Authorization = "Bearer $tok" } |
  Select-Object -ExpandProperty properties |
  Select-Object -ExpandProperty enterprisePolicies
```

`linkStatus: Linked` indicates the policy is active.

## Troubleshooting

| Symptom | Likely cause / fix |
| --- | --- |
| `404` from `enterprisePolicies/vnet/link` | Environment is not a Managed Environment yet. Enable it in PPAC, then re-run. |
| `Environment location 'unitedstates' does not match the enterprise policy location 'westus'` | The Enterprise Policy resource's `location` must be the PP geo (`unitedstates`), not an Azure region. The provided Bicep handles this; redeploy if you set it manually. |
| `EnterprisePolicyUpdateNotAllowed` when redeploying the policy | Policy is currently linked to an environment. Run `link-enterprise-policy.ps1 -Unlink`, redeploy, then re-link. |
| Custom connector test from PP flow returns `403 ThrowExceptionDueToTrafficDenied` | Connection cached from before the policy link. Delete the connection (Power Apps → Connections), re-create it, retry. |
| Custom connector doesn't appear in flow designer's **Custom** tab | Add the connector to a Dataverse solution (`pac connector update --solution-unique-name <name>`), then refresh. |

## Cleanup

```powershell
./scripts/link-enterprise-policy.ps1 -Unlink
az group delete -n $env:AZURE_RESOURCE_GROUP --yes --no-wait
```

## License / disclaimer

Sample code provided as-is, no warranty. Review and adapt for production use
(naming conventions, resource tags, RBAC, diagnostic settings, address space
planning, etc.).
