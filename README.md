# 🚀 Azure Functions — Full Production-Style Project (Python)

A complete, **ready‑to‑deploy** Azure Functions project with:

* Multiple functions (HTTP, Queue Trigger, Timer)
* Local development (Core Tools) + debugging
* Storage Queue integration
* Environment separation (local/dev/prod)
* **Terraform IaC** to provision all Azure resources
* **GitHub Actions CI/CD** for one‑click deploys
* Observability (App Insights), retries, and best practices

> Language: **Python 3.10+** (can adapt to Node/C# if needed)

---

## 📁 Repository Structure

```
azure-functions-full-project/
├── README.md
├── .gitignore
├── .funcignore
├── host.json
├── local.settings.json.example
├── requirements.txt
├── src/
│   ├── __init__.py
│   ├── shared/
│   │   ├── __init__.py
│   │   └── utils.py
│   ├── http_echo/__init__.py
│   ├── http_echo/function.json
│   ├── queue_enqueuer/__init__.py
│   ├── queue_enqueuer/function.json
│   ├── queue_worker/__init__.py
│   ├── queue_worker/function.json
│   ├── timer_cleanup/__init__.py
│   └── timer_cleanup/function.json
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   └── README.md
└── .github/workflows/deploy.yml
```

---

## 🧩 Functions Overview

1. **http\_echo** (HTTP Trigger): simple health/echo endpoint; returns request info.
2. **queue\_enqueuer** (HTTP Trigger): accepts JSON and enqueues to **Azure Storage Queue**.
3. **queue\_worker** (Queue Trigger): consumes queue messages and logs/processes them.
4. **timer\_cleanup** (Timer Trigger): runs nightly to simulate housekeeping.

---

## 🛠️ Prerequisites

* Azure Subscription + permissions to create resources
* **Azure Functions Core Tools** v4
* **Azure CLI** (`az`) logged in
* Python 3.10+

```bash
# macOS
brew update && brew install azure-functions-core-tools@4 az

# Windows (PowerShell)
winget install Microsoft.AzureFunctionsCoreTools --source winget
winget install Microsoft.AzureCLI

# Linux (Ubuntu example)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
# Functions Core Tools: https://learn.microsoft.com/azure/azure-functions/functions-run-local
```

Login:

```bash
az login
az account set --subscription "<SUBSCRIPTION_NAME_OR_ID>"
```

---

## ⚙️ App Settings (local)

`local.settings.json.example` — copy to `local.settings.json` for local runs.

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "APPINSIGHTS_INSTRUMENTATIONKEY": "",
    "QUEUE_NAME": "jobs",
    "WEBSITE_RUN_FROM_PACKAGE": "0"
  }
}
```

> For local dev, install **Azurite** or use the Storage Emulator. Quick start (Docker): `docker run -p 10000:10000 -p 10001:10001 -p 10002:10002 mcr.microsoft.com/azure-storage/azurite`

---

## 🐍 Python Dependencies

`requirements.txt`

```
azure-functions==1.20.0
azure-storage-queue==12.9.0
```

---

## 🔧 host.json

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": { "isEnabled": true }
    }
  },
  "extensions": {
    "queues": {
      "maxDequeueCount": 5,
      "visibilityTimeout": "00:00:30"
    }
  }
}
```

---

## 🧠 Shared Utilities

`src/shared/utils.py`

```python
import json
from datetime import datetime

def ok(body: dict, status_code: int = 200):
    return {
        "status_code": status_code,
        "mimetype": "application/json",
        "body": json.dumps(body, default=str)
    }

def now_iso():
    return datetime.utcnow().isoformat() + "Z"
```

---

## 🌐 Function: http\_echo (HTTP Trigger)

`src/http_echo/function.json`

```json
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    }
  ]
}
```

`src/http_echo/__init__.py`

```python
import logging
import azure.functions as func
from ..shared.utils import ok, now_iso

def main(req: func.HttpRequest) -> func.HttpResponse:
    logging.info("http_echo invoked")
    name = req.params.get("name") or (req.get_json(silent=True) or {}).get("name")
    return func.HttpResponse(
        **ok({
            "message": f"Hello {name or 'world'} 👋",
            "time": now_iso(),
            "method": req.method,
            "path": req.url
        })
    )
```

---

## 📥 Function: queue\_enqueuer (HTTP → Storage Queue)

`src/queue_enqueuer/function.json`

```json
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["post"]
    },
    { "type": "http", "direction": "out", "name": "res" }
  ]
}
```

`src/queue_enqueuer/__init__.py`

```python
import os, json, logging
import azure.functions as func
from azure.storage.queue import QueueClient
from ..shared.utils import ok, now_iso

QUEUE_NAME = os.getenv("QUEUE_NAME", "jobs")


def main(req: func.HttpRequest) -> func.HttpResponse:
    logging.info("queue_enqueuer invoked")

    try:
        payload = req.get_json()
    except Exception:
        payload = {"note": "no json body"}

    # Attach metadata
    payload.update({
        "_enqueued_at": now_iso(),
        "_source": "http"
    })

    # Use the same connection string as AzureWebJobsStorage
    conn = os.getenv("AzureWebJobsStorage")
    qc = QueueClient.from_connection_string(conn, QUEUE_NAME)
    qc.create_queue()  # idempotent
    qc.send_message(json.dumps(payload))

    return func.HttpResponse(**ok({"status": "enqueued", "queue": QUEUE_NAME, "payload": payload}))
```

---

## ⚙️ Function: queue\_worker (Queue Trigger)

`src/queue_worker/function.json`

```json
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "name": "msg",
      "type": "queueTrigger",
      "direction": "in",
      "queueName": "%QUEUE_NAME%",
      "connection": "AzureWebJobsStorage"
    }
  ]
}
```

`src/queue_worker/__init__.py`

```python
import json, logging
import azure.functions as func


def main(msg: func.QueueMessage):
    body = msg.get_body().decode("utf-8")
    try:
        data = json.loads(body)
    except Exception:
        data = {"raw": body}

    logging.info("[queue_worker] processing: %s", data)
    # TODO: add real work here (call APIs, write to DB, send email, etc.)
```

---

## ⏱️ Function: timer\_cleanup (Timer Trigger)

`src/timer_cleanup/function.json`

```json
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "name": "mytimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 0 20 * * *"  
    }
  ]
}
```

> Cron format: `sec min hour day month dayOfWeek` (this runs every day at 20:00 UTC). Adjust for IST as needed.

`src/timer_cleanup/__init__.py`

```python
import logging
from datetime import datetime, timezone


def main(mytimer) -> None:
    utc_now = datetime.now(timezone.utc).isoformat()
    logging.info("[timer_cleanup] tick at %s", utc_now)
    # TODO: add housekeeping (cleanup old blobs, rotate reports, etc.)
```

---

## ▶️ Run Locally

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\\Scripts\\activate
pip install -r requirements.txt
func start --python
```

Test:

```bash
curl -s http://127.0.0.1:7071/api/http_echo?name=Atul | jq
curl -s -X POST http://127.0.0.1:7071/api/queue_enqueuer \
  -H 'Content-Type: application/json' \
  -d '{"task":"send_report","user":"atul"}' | jq
```

Logs show `queue_worker` consuming messages.

---

## ☁️ Terraform — Provision Azure Resources

`terraform/providers.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.114"
    }
  }
}

provider "azurerm" {
  features {}
}
```

`terraform/variables.tf`

```hcl
variable "prefix" { type = string default = "afn" }
variable "location" { type = string default = "Central India" }
variable "environment" { type = string default = "dev" }
variable "queue_name" { type = string default = "jobs" }
```

`terraform/main.tf`

```hcl
resource "azurerm_resource_group" "rg" {
  name     = "${var.prefix}-${var.environment}-rg"
  location = var.location
}

resource "azurerm_storage_account" "sa" {
  name                     = replace("${var.prefix}${varenvironment}sa", "-", "")
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_queue" "q" {
  name                 = var.queue_name
  storage_account_name = azurerm_storage_account.sa.name
}

resource "azurerm_application_insights" "appi" {
  name                = "${var.prefix}-${var.environment}-appi"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  application_type    = "web"
}

resource "azurerm_service_plan" "plan" {
  name                = "${var.prefix}-${var.environment}-plan"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  os_type             = "Linux"
  sku_name            = "Y1" # Consumption
}

resource "azurerm_linux_function_app" "fa" {
  name                       = "${var.prefix}-${var.environment}-func"
  resource_group_name        = azurerm_resource_group.rg.name
  location                   = azurerm_resource_group.rg.location
  service_plan_id            = azurerm_service_plan.plan.id
  storage_account_name       = azurerm_storage_account.sa.name
  storage_account_access_key = azurerm_storage_account.sa.primary_access_key

  site_config {
    application_stack {
      python_version = "3.10"
    }
  }

  app_settings = {
    FUNCTIONS_WORKER_RUNTIME       = "python"
    AzureWebJobsStorage            = azurerm_storage_account.sa.primary_connection_string
    APPINSIGHTS_INSTRUMENTATIONKEY = azurerm_application_insights.appi.instrumentation_key
    QUEUE_NAME                     = var.queue_name
    WEBSITE_RUN_FROM_PACKAGE       = "1"
  }
}
```

`terraform/outputs.tf`

```hcl
output "function_default_hostname" {
  value = azurerm_linux_function_app.fa.default_hostname
}

output "queue_name" {
  value = var.queue_name
}
```

`terraform/README.md`

```md
# Terraform Use

export ARM_USE_OIDC=true   # if using GitHub OIDC; else az login works locally
az login

cd terraform
terraform init
terraform plan -out=tfplan
terraform apply -auto-approve tfplan
```

> After apply, note the `function_default_hostname` (e.g., `afn-dev-func.azurewebsites.net`).

---

## 🔄 CI/CD with GitHub Actions

`.github/workflows/deploy.yml`

```yaml
name: Deploy Azure Functions (Python)

on:
  push:
    branches: [ main ]
  workflow_dispatch: {}

permissions:
  id-token: write
  contents: read

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install deps
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Zip package
        run: |
          zip -r functionapp.zip host.json requirements.txt src

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Function App
        uses: azure/functions-action@v1
        with:
          app-name: ${{ secrets.FUNCTION_APP_NAME }}
          package: functionapp.zip
```

### Secrets required

* `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` (from Azure Entra ID app reg / Federated creds)
* `FUNCTION_APP_NAME` (output from Terraform)

> Alternative: use `publish-profile` secret from the Function App (less secure than OIDC).

---

## 🌍 Post‑Deploy Test (Cloud)

```bash
BASE=https://<FUNCTION_DEFAULT_HOSTNAME>

curl -s "$BASE/api/http_echo?name=Cloudnautic" | jq

curl -s -X POST "$BASE/api/queue_enqueuer" \
  -H 'x-functions-key: <FUNCTION_KEY>' \
  -H 'Content-Type: application/json' \
  -d '{"task":"daily_sync","user":"atul"}' | jq
```

> Get the **function key** from Azure Portal → Function App → Functions → `queue_enqueuer` → `Function Keys`.

---

## ✅ Production Considerations

* **Auth**: Use Function/Host keys for internal endpoints; for public APIs, add **Easy Auth** (AAD) or API Management.
* **Retries & Poison**: `maxDequeueCount` with a dedicated **poison queue** for failures.
* **Observability**: App Insights logs, Live Metrics, custom `logging` with correlation IDs.
* **Config**: Move secrets to **App Settings**/**Key Vault**; never commit secrets.
* **Concurrency**: Use `WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT` and queue batch settings if needed.
* **Cold Start**: Keep warm via timer trigger or ping, if appropriate.
* **IaC**: Keep Terraform the source of truth; pin provider versions.

---

## 🧪 Quick Health Checklist

* `func start` runs with no import errors
* `/api/http_echo` returns 200 JSON
* Posting to `/api/queue_enqueuer` shows message consumed by `queue_worker`
* Terraform `apply` completes; Function hostname reachable
* GitHub Actions deploys successfully on push to `main`

---

## 🧰 Useful CLI Snippets

```bash
# Get function default hostname
az functionapp show -g <rg> -n <funcapp> --query defaultHostName -o tsv

# Get function key
az functionapp function keys list -g <rg> -n <funcapp> --function-name queue_enqueuer -o tsv

# Tail logs
az webapp log tail -g <rg> -n <funcapp>
```

---

## 📦 .gitignore

```
.venv/
__pycache__/
local.settings.json
*.zip
.DS_Store
```

## 📦 .funcignore

```
.git/
.vscode/
.venv/
local.settings.json
*.md
terraform/
.github/
```

---

## 🧭 Next Steps / Variants

* Add **Cosmos DB** binding for worker output
* Add **Durable Functions** orchestrations (chaining, fan‑out/in)
* API Management front‑door with auth + rate limiting
* Blue/Green via staging slots (Premium plan)

---

**Happy shipping!** If you want the **Node.js** or **C#** version, or a **Durable Functions** workflow, say the word and I’ll add it. 💙
