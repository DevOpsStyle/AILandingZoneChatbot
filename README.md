# AI Foundry Private Landing Zone — ARM Nested Template

Template ARM **nested** (subscription-scope) che esegue il deploy end-to-end di una landing zone privata per workload AI.

## Deploy con un click (GitHub → Azure Portal)

Repo: [`DevOpsStyle/AILandingZoneChatbot`](https://github.com/DevOpsStyle/AILandingZoneChatbot) — branch `main`.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FDevOpsStyle%2FAILandingZoneChatbot%2Fmain%2Fmain.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2FDevOpsStyle%2FAILandingZoneChatbot%2Fmain%2FcreateUiDefinition.json)
[![Visualize](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/visualizebutton.svg?sanitize=true)](https://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2FDevOpsStyle%2FAILandingZoneChatbot%2Fmain%2Fmain.json)

> Se usi un branch diverso da `main`, sostituisci `main` negli URL sopra con il nome del branch (URL-encoded).

Il portale aprirà il **wizard** definito da [createUiDefinition.json](createUiDefinition.json) con questi step:

1. **Basics** — subscription, region di default, prefisso naming, nomi dei 3 RG (hub / workload / apim), toggle "usa RG esistente".
2. **Region per componente** — dropdown per ogni servizio (default = region scelta nei Basics; opzionale override).
3. **Componenti da deployare** — toggle on/off per ogni servizio opzionale (Key Vault, AI Search, APIM, Cosmos, Service Bus, Redis, **Application Gateway WAF, ACR, Container Apps, Application Insights**).
4. **SKU & sizing** — SKU/capacity per ogni componente abilitato, inclusi App Gateway SKU, WAF mode e autoscale min/max.
5. **Networking (hub-spoke)** — CIDR delle 3 VNet e relative subnet (hub: App Gateway; apim: APIM + PE; workload: Private Endpoints + Container Apps).
6. **AI Foundry & modelli** — selettore "**numero di modelli**" (0–3). Per ogni modello attivo: `deploymentName`, `modelName`, `format` (OpenAI/Microsoft/Meta/Mistral/Cohere/DeepSeek), `version`, `skuName`, `capacity`. Per ogni modello viene creato un **backend APIM** corrispondente (`aoai-<deploymentName>`) puntato all'endpoint privato di Foundry.
7. **API Management** — publisher email/name + SKU.
8. **Diagnostica** — master switch + toggle per singolo servizio (Foundry, Search, APIM, Cosmos, Service Bus, Redis, Key Vault).
9. **Defender for Cloud** — master switch + toggle per piano (AiServices, Api, CosmosDbs, KeyVaults, Storage, Containers, Arm, VM, AppServices, SQL, SQL VM, OSS DB).
10. **Tag** — workload, env, owner.

> **Nota**: per pubblicare su GitHub i template devono essere accessibili come **raw** (file pubblico nel repo) altrimenti il portale non li carica. Per repo privati usa "Template Specs" o un blob storage SAS.

## Risorse incluse

| Categoria        | Risorsa                                                                 |
|------------------|-------------------------------------------------------------------------|
| AI               | Azure AI Foundry (Cognitive Services `AIServices`) + N model deployments configurabili (0–3, scelti a runtime — esempi: `gpt-5-mini`, `text-embedding-3-large`, `gpt-4o-mini`) |
| Search           | Azure AI Search (Standard, semantic ranking)                            |
| API              | API Management (public access disabled, Private Endpoint nella VNet apim) **+ 1 backend per modello** (`aoai-<deploymentName>`) |
| Data             | Azure Cosmos DB (NoSQL / API configurabile)                             |
| Messaging        | Azure Service Bus (Premium)                                             |
| Cache            | Azure Managed Redis (Redis Enterprise — `Balanced_B5`)                  |
| Secrets          | Azure Key Vault (RBAC, public access disabled)                          |
| Container        | Azure Container Registry (Premium, public access disabled, Private Endpoint) |
| Compute          | Azure Container Apps Environment (VNet-injected, internal) + Container App "chatbot" |
| Edge             | Application Gateway **WAF_v2** (WAF Policy OWASP 3.2 + Bot Manager) + Public IP zone-redundant |
| Networking       | **Hub-spoke**: 3 VNet (hub / workload / apim) + 6 VNet peering + 14 Private DNS Zones + VNet Links |
| Observability    | Log Analytics Workspace + Application Insights + Diagnostic Settings su tutte le risorse |
| Security         | Microsoft Defender for Cloud (subscription-scope, plan-by-plan)         |

Tutti i servizi dati/PaaS (Foundry, Search, Cosmos, Service Bus, Key Vault, Redis, APIM, ACR) sono **`publicNetworkAccess: Disabled`** e raggiungibili solo via **Private Endpoint** verso le rispettive Private DNS Zones. L'**Application Gateway** è l'unico punto d'ingresso pubblico e instrada verso la Container App.

## Struttura nested

```
main.json (subscription scope)
├── 3x Resource Group               (hub / workload / apim)
├── deployment: defenderPlans-deploy   (subscription scope, condizionale)
├── deployment: hub-deploy             (RG hub: VNet hub + subnet snet-appgw)
├── deployment: workload-deploy        (RG workload)
│   ├── Log Analytics + Application Insights
│   ├── VNet workload + subnet snet-pe + subnet snet-aca (delegata a Microsoft.App/environments)
│   ├── 14 Private DNS Zones + VNet Links
│   ├── Key Vault
│   ├── Foundry + N model deployments
│   ├── AI Search
│   ├── Cosmos DB
│   ├── Service Bus Premium
│   ├── Redis Enterprise + database
│   ├── ACR Premium
│   ├── Container Apps Environment + Container App "chatbot"
│   ├── Private Endpoints + DNS Zone Groups (Foundry, Search, Cosmos, SB, Redis, KV, ACR)
│   ├── Diagnostic Settings (condizionali)
│   └── harden-private-access (chiude Foundry su public access dopo il PE)
├── deployment: apim-deploy            (RG apim: APIM + PE + diagnostica, condizionale)
├── 6x deployment: peering-*           (peering bidirezionali hub<->workload, hub<->apim, workload<->apim)
├── deployment: appgw-deploy           (RG hub: WAF Policy + Public IP + Application Gateway, condizionale)
└── deployment: dns-spoke-links-deploy (RG workload: link DNS zone alle VNet hub + apim)
```

Le dipendenze sono modellate esplicitamente con `dependsOn` per garantire ordine di creazione corretto (es. PE dopo VNet+servizio, DNS group dopo PE+zona, App Gateway dopo workload-deploy per leggere il FQDN della Container App).

## Toggle

### Region per-componente
- `location` (string): region di **default** usata da tutte le risorse.
- `locations` (object): override **per singolo componente**. Lascia `""` per usare il default.
  ```json
  { "network": "", "logAnalytics": "", "keyVault": "",
    "foundry": "westeurope", "search": "", "apim": "",
    "cosmos": "", "serviceBus": "", "redis": "" }
  ```
  - `network` = region delle 3 VNet **e di tutti i Private Endpoint** (i PE devono stare nella stessa region della VNet).
  - Le **Private DNS Zones** sono globali e non hanno region.
  - Per Cosmos DB, `locations.cosmos` è anche la primary write region dell'account.
  - Verifica che il modello scelto (es. `gpt-5-mini`) sia disponibile nella region di `foundry`.

### Diagnostica
- `enableDiagnostics` (bool master)
- `diagnosticsPerResource` (object granulare):
  ```json
  { "foundry": true, "search": true, "apim": true, "cosmos": true,
    "serviceBus": true, "redis": true, "keyVault": true }
  ```
La diagnostica viene applicata solo se `enableDiagnostics && diagnosticsPerResource.<svc>` sono entrambi `true`.

### Defender for Cloud
- `enableDefender` (bool master)
- `defenderPlans` (object granulare per piano):
  ```json
  { "AiServices": true, "Api": true, "CosmosDbs": true, "KeyVaults": true,
    "StorageAccounts": true, "Containers": true, "Arm": true, ... }
  ```
Imposta `false` per non attivare un piano. Se `enableDefender = false` l'intero blocco non viene deployato.

## Deploy

> **Subscription**: in ARM la sub è impostata dal comando, non dal template. Usa `--subscription` (CLI) o `Select-AzSubscription` (PowerShell). Il template contiene un **safety check** (`targetSubscriptionId`): se l'ID nei parametri non corrisponde alla sub corrente, il deploy fallisce subito.
>
> **Resource Group**: di default vengono **creati 3 RG** (hub / workload / apim). I nomi si impostano con `resourceGroupHubName`, `resourceGroupWorkloadName`, `resourceGroupApimName` (se vuoti, default `rg-<prefix>-<ruolo>`). Per usarne di esistenti imposta `useExistingResourceGroup: true`.
>
> **Provider da registrare**: la sub di destinazione deve avere registrati i resource provider usati — in particolare `Microsoft.Cache`, `Microsoft.ServiceBus`, `Microsoft.App`, `Microsoft.ContainerRegistry`, `Microsoft.CognitiveServices`. Esempio: `az provider register --namespace Microsoft.ServiceBus` (attendi `Registered` prima del deploy).

### Az CLI
```bash
# 1) seleziona la sub
az account set --subscription "<SUBSCRIPTION_ID>"

# 2) deploy (puoi anche passare --subscription qui)
az deployment sub create \
  --name aifoundry-private-deploy \
  --location swedencentral \
  --subscription "<SUBSCRIPTION_ID>" \
  --template-file main.json \
  --parameters @main.parameters.json \
  --parameters targetSubscriptionId="<SUBSCRIPTION_ID>"
```

### PowerShell
```powershell
# 1) seleziona la sub
Select-AzSubscription -SubscriptionId "<SUBSCRIPTION_ID>"

# 2) deploy
New-AzSubscriptionDeployment `
  -Name aifoundry-private-deploy `
  -Location swedencentral `
  -TemplateFile .\main.json `
  -TemplateParameterFile .\main.parameters.json `
  -targetSubscriptionId "<SUBSCRIPTION_ID>"
```

## Modelli (parametro `models`)

Array dinamico (0–N elementi). Per ogni modello:

| Campo | Tipo | Esempio | Note |
|---|---|---|---|
| `deploymentName` | string | `gpt-5-mini` | Nome del deployment in Foundry (usato anche come path nel backend APIM) |
| `modelName`      | string | `gpt-5-mini` | Nome modello del catalog |
| `format`         | string | `OpenAI` | OpenAI, Microsoft, Meta, Mistral AI, Cohere, DeepSeek |
| `version`        | string | `2025-08-07` | Versione del modello |
| `skuName`        | string | `GlobalStandard` | `GlobalStandard`, `Standard`, `DataZoneStandard`, `GlobalProvisionedManaged`, `ProvisionedManaged` |
| `capacity`       | int    | `50` | TPM x1000 (PayGo) o unità PTU (Provisioned) |

Esempio (3 modelli):
```json
"models": { "value": [
  { "deploymentName": "gpt-5-mini",            "modelName": "gpt-5-mini",            "format": "OpenAI", "version": "2025-08-07", "skuName": "GlobalStandard", "capacity": 50 },
  { "deploymentName": "text-embedding-3-large","modelName": "text-embedding-3-large","format": "OpenAI", "version": "1",          "skuName": "Standard",       "capacity": 50 },
  { "deploymentName": "gpt-4o-mini",           "modelName": "gpt-4o-mini",           "format": "OpenAI", "version": "2024-07-18", "skuName": "GlobalStandard", "capacity": 30 }
] }
```
Per ciascun modello, il template crea automaticamente un **backend APIM** chiamato `aoai-<deploymentName>` con URL `https://<foundry>.openai.azure.com/openai/deployments/<deploymentName>` (l'endpoint risolve via Private DNS).

## Note operative

- **Region**: default `swedencentral` (supporta `gpt-5-mini` + `text-embedding-3-large` + Managed Redis). Verifica disponibilità modelli/quota prima del deploy.
- **Provider registration**: registra i provider sulla sub di destinazione prima del deploy (`Microsoft.Cache`, `Microsoft.ServiceBus`, `Microsoft.App`, `Microsoft.ContainerRegistry`, `Microsoft.CognitiveServices`). Errori tipo *"is not registered against the RP"* o *"Zone information missing"* si risolvono così.
- **Model SKU per region**: lo SKU del modello (`Standard`, `GlobalStandard`, ...) deve essere supportato nella region di Foundry. Es. `text-embedding-3-large` con SKU `Standard` non è supportato in `italynorth` — usa `GlobalStandard` o un'altra region.
- **Subnet Container Apps**: `snet-aca` è delegata a `Microsoft.App/environments` (richiesto dall'ambiente ACA) e dimensionata `/23` per i workload profiles.
- **APIM**: in RG/VNet dedicati con Private Endpoint; `publicNetworkAccess` chiuso dopo la creazione del PE.
- **Redis Enterprise SKU**: `Balanced_B5` è il punto di ingresso "Azure Managed Redis". Adatta in base al carico.
- **Application Gateway**: è l'unico endpoint pubblico (Public IP zone-redundant) e instrada verso la Container App. WAF_v2 con policy OWASP 3.2 in modalità Prevention.
- **Defender pricings** sono risorse **subscription-scope**: il toggle è centralizzato in un nested deployment dedicato (`defenderPlans-deploy`).
- **Connettività privata**: dopo il deploy, per testare/usare i servizi serve connettività alla VNet (peering, VPN, Bastion + jumpbox, o Self-hosted runner).
- **Cosmos DB / Service Bus / KV**: hanno `disableLocalAuth: false` per compatibilità — irrigidisci a `true` dopo aver migrato a RBAC/Entra ID.
- **Quote**: i deployment di modello richiedono quota TPM disponibile per la sub nella region scelta.

## Validazione

```bash
az deployment sub validate \
  --location swedencentral \
  --template-file main.json \
  --parameters @main.parameters.json

az deployment sub what-if \
  --location swedencentral \
  --template-file main.json \
  --parameters @main.parameters.json
```
