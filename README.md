# Website Uptime Monitor

A fully serverless uptime monitor built with Terraform and an Azure Function: checks a target website every 5 minutes, writes every result to Table Storage, and fires an email + SMS alert within seconds of a failure, all on a Consumption plan that costs under $1/month.

![Azure](https://img.shields.io/badge/Azure-Functions-0078D4?logo=microsoftazure&logoColor=white)
![Terraform](https://img.shields.io/badge/IaC-Terraform-844FBA?logo=terraform&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.10-3776AB?logo=python&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Beginner--Intermediate-orange)
![Cost](https://img.shields.io/badge/Cost-Under_%241%2Fmonth-brightgreen)

## 🎥 Watch Me Live
[Watch me build this lab live →](PASTE_YOUR_LINK_HERE)

## Overview

| Field | Value |
|---|---|
| Time to complete | 4 to 5 hours |
| Difficulty | Beginner to Intermediate |
| Tools used | Terraform, Azure Functions (Python), Table Storage, Application Insights, Log Analytics, Azure Monitor |
| Career relevance | Site Reliability Engineering, Cloud Operations, Serverless Development, Incident Response |

## The Business Problem This Lab Solves

Every business with a website lives with this fear: the site goes down, a customer tries to visit and gets an error, they leave and never come back, and the business owner doesn't find out until someone calls to complain, or until they check sales numbers the next day and wonder why everything dropped off.

This lab eliminates that blind spot entirely. The system checks the target website every five minutes, 24 hours a day, testing three things on every check: does it load, how fast does it load, and is it showing the right content. It fires an alert within seconds of any failure, and writes every check result to storage, so there's a complete history of every incident, not just the ones someone happened to notice.

**The interview framing:** "I built a system that detects website downtime within seconds and alerts the business owner immediately, turning an invisible problem into something that can be responded to in real time."

## Architecture Overview

| Component | What it is and why this project needs it |
|---|---|
| Azure Function (`check_website.py`) | A small piece of Python code that runs in the cloud on a schedule, no servers to manage. This is the actual monitoring logic: make an HTTP request to the target site, decide pass or fail, write the result somewhere |
| Timer Trigger (`function.json`) | What makes the Function run on a schedule, a cloud-based cron job. Fires every 5 minutes using a CRON expression. Without a trigger, a Function never runs |
| Table Storage (`uptimechecks`) | A cheap, fast, simple database table storing every check result: timestamp, PASS/SLOW/FAIL, response time in milliseconds, error detail. This is the historical record of every incident |
| Application Insights | Automatically collects everything the Function logs and makes it searchable. When the function calls `logging.error("SITE DOWN...")`, this is what receives that log at severity level 3, the signal the alert rule watches for |
| Log Analytics Workspace | The backend store Application Insights writes into. The alert rule runs KQL queries against this workspace |
| Monitor Alert Rule | A KQL query Azure runs every 5 minutes against Log Analytics. If it finds error-level traces containing "SITE DOWN" in the last 5 minutes, it fires. This is what connects "site is down" to "owner gets notified" |
| Action Group | The list of people and channels to notify, email and SMS in this project, extensible to Slack, Teams, or PagerDuty later |
| App Service Plan (Y1 Consumption) | The compute hosting the Function App. Y1 means pay only when the function actually runs, at 12 executions per hour this costs fractions of a cent per day |

## Architecture Diagram

![Diagram showing a timer trigger firing an Azure Function every 5 minutes, which checks the target website, writes results to Table Storage, and on failure logs an error that flows through Application Insights and Log Analytics to an alert rule, which notifies an Action Group by email and SMS](./screenshots/uptime-monitor-diagram.png)

## Skills Demonstrated

| Skill | Real-world application |
|---|---|
| Serverless compute with Azure Functions | Running scheduled logic without managing or patching a VM |
| Timer-triggered automation via CRON expressions | The scheduling pattern behind most background jobs in cloud environments |
| Table Storage for lightweight structured data | A cheaper, simpler alternative to a full database for append-heavy, low-complexity data like check logs |
| Application Insights + Log Analytics + KQL alerting | The standard Azure observability pipeline: logs flow into a queryable workspace, and alert rules run scheduled queries against it |
| Two-stage deployment (Terraform for infra, CLI for code) | A real production pattern, separating rare high-risk infrastructure changes from frequent low-risk code changes |
| Debugging real deployment failures | Four actual bugs hit during development are documented and fixed in this lab, a genuinely useful record of what breaks and why |

## What Gets Built

```
rg-uptime-monitor-[yourname]
├── Storage Account + Table (uptimechecks)   → check history
├── Log Analytics Workspace
├── Application Insights                     → linked to the workspace
├── App Service Plan (Y1 Consumption)
├── Linux Function App                        → runs check_website.py every 5 min
├── Action Group                              → email + SMS
└── Monitor Alert Rule                        → KQL query watching for "SITE DOWN"
```

## Project Files — What Each File Does

This project has two separate parts, deployed separately: Terraform files that create Azure resources, and Python files that contain the monitoring logic.

**Why two separate deployments?** Terraform creates the infrastructure, the Function App container, the storage account, the alert rules, but it does **not** upload the Python code. It creates an empty Function App and waits. The code gets deployed separately via the Azure CLI's `config-zip` command. This separates infrastructure changes (rare, high-risk) from code changes (frequent, lower-risk), a developer can update the monitoring logic without touching Terraform state at all.

| File | What it does |
|---|---|
| `variables.tf` | Declares the input parameters Terraform accepts. No real values here, just names and types |
| `terraform.tfvars` | Real values: name, target URL, email, phone. Stays out of Git via `.gitignore` |
| `main.tf` | Declares every Azure resource |
| `outputs.tf` | Prints resource names after deployment, needed for the code deploy step |
| `function_app/host.json` | Required by the Functions runtime to identify the runtime version. Without it: "Runtime version: Error" and the function never executes |
| `function_app/requirements.txt` | Python library list, installed when packaging the function |
| `function_app/check_website/function.json` | The trigger binding, tells Azure what fires this function and how often. Must live inside a subfolder matching the function name |
| `function_app/check_website/check_website.py` | The monitoring logic itself |

> **The zip structure matters.** Azure Functions on Linux requires function files inside a named subfolder matching the function name (`check_website/`). Files at the root level aren't recognized as a function. This is one of the most common deployment failures, the function registers but never appears in the portal.

## Prerequisites

| Requirement | How to check / install |
|---|---|
| Azure CLI | `az version`. If missing: `brew install azure-cli` (Mac) or aka.ms/installazurecliwindows (Windows). Then `az login` |
| Terraform | `terraform -version`. If missing: `brew tap hashicorp/tap && brew install hashicorp/tap/terraform` (Mac) or developer.hashicorp.com/terraform/downloads (Windows) |
| Python 3 | `python3 --version` (Mac) or `python --version` (Windows). Check "Add Python to PATH" during Windows install |
| Azure subscription | `az account show`. Log in with `az login` |
| Available vCPU quota | `az vm list-usage --location "East US" --query "[?name.value=='cores']" -o table`. If quota is 0 or very low, use West US 2 instead |

> **Quota issue:** free Azure subscriptions sometimes have a VM quota of 0 in certain regions. The error reads: `"Operation cannot be completed without additional quota. Current Limit (Total VMs): 0"`. If you see this on `terraform apply`, change `location` in `terraform.tfvars` from `"East US"` to `"West US 2"` and re-apply.

## Step 1 — Create the Project Folder and Empty Files

`check_website.py` and `function.json` go inside a subfolder called `check_website/`, not directly in `function_app/`. This matches the Azure Functions Linux requirement.

**Mac / Linux:**
```bash
mkdir -p ~/uptime-monitor-001/function_app/check_website
cd ~/uptime-monitor-001

# Root Terraform files
touch main.tf variables.tf outputs.tf terraform.tfvars .gitignore

# Function App files
touch function_app/host.json
touch function_app/requirements.txt

# Function files — MUST be inside check_website/ subfolder
touch function_app/check_website/function.json
touch function_app/check_website/check_website.py

# Verify the structure
find . -type f | sort
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$HOME\uptime-monitor-001\function_app\check_website" | Out-Null
cd "$HOME\uptime-monitor-001"

New-Item -ItemType File -Force -Path @(
    "main.tf"
    "variables.tf"
    "outputs.tf"
    "terraform.tfvars"
    ".gitignore"
    "function_app\host.json"
    "function_app\requirements.txt"
    "function_app\check_website\function.json"
    "function_app\check_website\check_website.py"
) | Out-Null

Get-ChildItem -Recurse -File | Select-Object FullName
```

**Expected structure:**
```
uptime-monitor-001/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── .gitignore
└── function_app/
    ├── host.json                      ← Azure Functions runtime identifier
    ├── requirements.txt               ← Python library list
    └── check_website/                 ← Subfolder named after the function
        ├── function.json              ← Timer trigger binding
        └── check_website.py           ← Monitoring logic
```

### `.gitignore`

```
terraform.tfvars
*.tfvars
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.backup
*.tfplan
__pycache__/
*.pyc
.python_packages/
function_deploy.zip
deploy_temp/
```

## Step 2 — `variables.tf`

```hcl
variable "yourname" {
  description = "Your name, lowercase, no spaces. Used to name all resources."
  type        = string
}

variable "location" {
  description = "Azure region. If you hit a quota error in East US, use West US 2."
  type        = string
  default     = "East US"
}

variable "target_url" {
  description = "The website URL to monitor. Must include https://."
  type        = string
}

variable "alert_email" {
  description = "Email address that receives downtime alerts."
  type        = string
}

variable "alert_phone" {
  description = "Phone number for SMS alerts. E.164 format: +14045550100"
  type        = string
}

variable "tags" {
  type = map(string)
  default = {
    project    = "uptime-monitor"
    managed_by = "terraform"
  }
}
```

## Step 3 — `terraform.tfvars`

```hcl
yourname    = "yourname"
location    = "East US"         # Change to "West US 2" if you hit a quota error
target_url  = "https://example.com"
alert_email = "your.email@example.com"
alert_phone = "+14045550100"    # E.164 format: +1 then 10 digits, no spaces or dashes
```

> **E.164 phone format:** `+` sign, country code (`1` for US), then 10-digit number. No spaces, dashes, or parentheses. `+14045550100` is correct, `404-555-0100` will fail.

## Step 4 — Write the Function Code

Writing the function code before the Terraform infrastructure means understanding what the function does before understanding why the infrastructure is configured the way it is.

### `function_app/host.json`

Required. Without this, the Function App shows "Runtime version: Error" in the portal and the function never executes. Must live at the root of `function_app/`, not inside `check_website/`.

```json
{
    "version": "2.0",
    "logging": {
      "applicationInsights": {
        "samplingSettings": {
          "isEnabled": true
        }
      }
    }
}
```

### `function_app/requirements.txt`

```
azure-data-tables==12.4.3
requests==2.31.0
```

### `function_app/check_website/function.json`

Must go inside the `check_website/` subfolder, not in `function_app/` directly. Azure Functions identifies functions by looking for `function.json` files inside named subfolders.

```json
{
    "scriptFile": "check_website.py",
    "bindings": [
      {
        "name": "mytimer",
        "type": "timerTrigger",
        "direction": "in",
        "schedule": "0 */5 * * * *"
      }
    ]
}
```

> `"0 */5 * * * *"` is a CRON expression with six fields: seconds, minutes, hours, day-of-month, month, day-of-week. `*/5` in minutes means every 5 minutes. The leading `0` means "at second zero." Net effect: fires at the start of every 5th minute, every hour, every day.

### `function_app/check_website/check_website.py`

Four bugs were hit and fixed during real deployment of this project, documented below rather than silently patched, because knowing what breaks and why is as valuable as the working code itself.

```python
import azure.functions as func
from azure.data.tables import TableServiceClient
import requests
import datetime
import os
import logging

def main(mytimer: func.TimerRequest) -> None:
    target_url   = os.environ["TARGET_URL"]
    storage_conn = os.environ["AzureWebJobsStorage"]

    check_time    = datetime.datetime.utcnow()
    result_status = "PASS"
    error_detail  = None
    response_ms   = None

    try:
        response    = requests.get(target_url, timeout=10)
        response_ms = response.elapsed.total_seconds() * 1000

        if response.status_code != 200:
            result_status = "FAIL"
            error_detail = f"HTTP {response.status_code}"

        elif response_ms > 5000:
            result_status = "SLOW"
            error_detail = f"Response time {response_ms:.0f}ms exceeded 5000ms"

        elif "error" in response.text.lower() and "404" in response.text:
            result_status = "FAIL"
            error_detail = "Page contains error indicators"

    except requests.exceptions.ConnectionError:
        result_status = "FAIL"
        error_detail = "Connection refused — server unreachable"

    except requests.exceptions.Timeout:
        result_status = "FAIL"
        error_detail = "Request timed out after 10 seconds"

    except Exception as e:
        result_status = "FAIL"
        error_detail = str(e)

    # ── Write result to Table Storage ────────────────────────────────────
    #
    # IMPORTANT — PartitionKey restrictions:
    # Azure Table Storage does not allow / : # ? in PartitionKey or RowKey.
    # The target URL (https://example.com) contains / and : which cause
    # silent write failures — rows appear to succeed but never show up.
    # Use the hardcoded string "uptime" as the PartitionKey instead.
    #
    # IMPORTANT — Use TableServiceClient, not TableClient:
    # create_table_if_not_exists() is a method on TableServiceClient.
    # TableClient does not have this method and raises AttributeError.
    # Always call create_table_if_not_exists() on the service client,
    # then get the table client separately for entity operations.
    #
    table_service = TableServiceClient.from_connection_string(storage_conn)
    table_service.create_table_if_not_exists("uptimechecks")
    table_client = table_service.get_table_client("uptimechecks")

    entity = {
        # "uptime" instead of target_url — URLs contain / and : which are
        # forbidden characters in Azure Table Storage partition keys.
        "PartitionKey": "uptime",
        "RowKey":       check_time.strftime("%Y%m%d%H%M%S"),
        "Timestamp":    check_time.isoformat(),
        "Status":       result_status,
        "ResponseMs":   int(response_ms) if response_ms else 0,
        "ErrorDetail":  error_detail or "",
        "TargetUrl":    target_url,
    }

    try:
        table_client.upsert_entity(entity)
        logging.info(f"Check: {result_status} | {response_ms:.0f}ms | {target_url}")
    except Exception as e:
        logging.error(f"Failed to write result to table: {e}")

    if result_status != "PASS":
        logging.error(f"SITE DOWN: {target_url} | {result_status} | {error_detail}")
```

**What was wrong and what's fixed:**

| What was wrong | What is fixed and why |
|---|---|
| `PartitionKey` set to `target_url` | Azure Table Storage forbids `/ : # ?` in PartitionKey. The URL `https://example.com` contains `:` and `/`. Rows appeared to write successfully but silently failed and never appeared in query results. Fixed by hardcoding PartitionKey to `"uptime"`, the target URL is still stored as a separate `TargetUrl` column |
| `table_client.create_table_if_not_exists()` | This method exists on `TableServiceClient`, not `TableClient`. Calling it on the wrong object raises `AttributeError`. Fixed by calling `table_service.create_table_if_not_exists("uptimechecks")` on the service client, then getting the table client separately |
| `logging.info` with `response_ms` when `response_ms` is `None` | If the request fails before timing is recorded, `response_ms` is `None` and `f"{response_ms:.0f}ms"` raises a `TypeError`. Fixed with the conditional `int(response_ms) if response_ms else 0` and handling `None` in the log line |
| No `host.json` in `function_app/` | Azure Functions v2 on Linux requires a `host.json` at the root of the zip package to identify the runtime version. Without it, "Runtime version: Error" and the function never runs. Fixed by adding `host.json` |

## Step 5 — `main.tf`

### Provider

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

data "azurerm_client_config" "current" {}
```

### Resource group

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-uptime-monitor-${var.yourname}"
  location = var.location
  tags     = var.tags
}
```

### Storage Account and Table

Serves two purposes: the Function runtime uses it for its own state, and the Python function writes check results into the `uptimechecks` table inside it. `account_replication_type = "LRS"` is sufficient, this is operational monitoring data, and if the storage account failed the Function would also be down, so geo-replication adds cost without meaningful protection here.

```hcl
resource "azurerm_storage_account" "main" {
  # Must be globally unique across all of Azure: 3-24 chars, lowercase + numbers only.
  name                     = "stuptime${var.yourname}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  min_tls_version          = "TLS1_2"
  tags                     = var.tags
}

resource "azurerm_storage_table" "uptime_checks" {
  name                 = "uptimechecks"
  storage_account_name = azurerm_storage_account.main.name
}
```

### Log Analytics and Application Insights

```hcl
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-uptime-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = var.tags
}

resource "azurerm_application_insights" "main" {
  name                = "appi-uptime-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  # workspace_id links App Insights to Log Analytics.
  # Without this, the alert rule cannot query App Insights logs.
  workspace_id        = azurerm_log_analytics_workspace.main.id
  application_type    = "web"
  tags                = var.tags
}
```

### App Service Plan and Function App

`WEBSITE_RUN_FROM_PACKAGE` is intentionally **not** set in `app_settings`. Setting it to `"1"` via Terraform and then deploying via `az functionapp deployment source config-zip` causes a 409 conflict, the two deployment methods are mutually exclusive. The config-zip approach doesn't require this setting.

```hcl
resource "azurerm_service_plan" "main" {
  name                = "asp-uptime-${var.yourname}"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  os_type             = "Linux"
  sku_name            = "Y1"
  tags                = var.tags
}

resource "azurerm_linux_function_app" "monitor" {
  name                       = "func-uptime-${var.yourname}"
  resource_group_name        = azurerm_resource_group.main.name
  location                   = var.location
  storage_account_name       = azurerm_storage_account.main.name
  storage_account_access_key = azurerm_storage_account.main.primary_access_key
  service_plan_id            = azurerm_service_plan.main.id

  site_config {
    application_stack {
      python_version = "3.10"
    }
  }

  app_settings = {
    "TARGET_URL"                            = var.target_url
    "APPINSIGHTS_INSTRUMENTATIONKEY"        = azurerm_application_insights.main.instrumentation_key
    "APPLICATIONINSIGHTS_CONNECTION_STRING" = azurerm_application_insights.main.connection_string
    "AzureWebJobsStorage"                   = azurerm_storage_account.main.primary_connection_string
    "FUNCTIONS_WORKER_RUNTIME"              = "python"
    # WEBSITE_RUN_FROM_PACKAGE is intentionally omitted.
    # Setting it here AND using config-zip for deployment causes a 409 conflict.
    # The config-zip method works without this setting.
  }

  tags = var.tags
}
```

### Action Group

```hcl
resource "azurerm_monitor_action_group" "downtime_alerts" {
  name                = "ag-uptime-${var.yourname}"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "uptime"

  email_receiver {
    name                    = "owner-email"
    email_address           = var.alert_email
    use_common_alert_schema = true
  }

  sms_receiver {
    name         = "owner-sms"
    country_code = "1"
    phone_number = replace(replace(var.alert_phone, "+1", ""), "-", "")
  }

  tags = var.tags
}
```

### Monitor Alert Rule

The KQL query watches for log entries at severity level 3 (the level Python's `logging.error()` maps to) that contain the string `"SITE DOWN"`, exactly what the function logs on a failure. Changing that string in the Python code means updating it here too.

```hcl
resource "azurerm_monitor_scheduled_query_rules_alert_v2" "site_down" {
  name                = "alert-site-down-${var.yourname}"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  description         = "Fires when the uptime monitor detects site failure."
  severity            = 1
  enabled             = true

  scopes                  = [azurerm_log_analytics_workspace.main.id]
  evaluation_frequency    = "PT5M"
  window_duration         = "PT5M"
  auto_mitigation_enabled = true

  criteria {
    query = <<-QUERY
      AppTraces
      | where SeverityLevel == 3
      | where Message contains "SITE DOWN"
      | summarize count() by bin(TimeGenerated, 5m)
      | where count_ > 0
    QUERY

    time_aggregation_method = "Count"
    threshold               = 0
    operator                = "GreaterThan"
  }

  action {
    action_groups = [azurerm_monitor_action_group.downtime_alerts.id]
  }

  tags = var.tags
}
```

## Step 6 — `outputs.tf`

```hcl
output "function_app_name" {
  value = azurerm_linux_function_app.monitor.name
}

output "application_insights_name" {
  value = azurerm_application_insights.main.name
}

output "storage_account_name" {
  value = azurerm_storage_account.main.name
}

output "storage_table_name" {
  value = azurerm_storage_table.uptime_checks.name
}

output "log_analytics_workspace_id" {
  value = azurerm_log_analytics_workspace.main.id
}
```

## Step 7 — Deploy Infrastructure with Terraform

Run from the project root (where `main.tf` is), not from `function_app/`.

```bash
terraform init

terraform plan
# Expected: 11 resources to add.
# If you see a quota error in the plan, change location in terraform.tfvars to "West US 2".

terraform apply
# Type yes when prompted. Takes 3-5 minutes.
# After completion, copy the function_app_name and storage_account_name from the output.
```

> If `terraform apply` fails with "quota" in the error message: open `terraform.tfvars`, change `location` to `"West US 2"`, save, and run `terraform apply` again. Terraform skips already-created resources and only creates the failed ones.

## Step 8 — Deploy the Function Code

Packages the Python code into a zip file with the correct folder structure, then uploads it. The zip structure matters, Azure Functions on Linux won't find the function if the files are in the wrong place.

**The required zip structure:**
```
function_deploy.zip
├── host.json                    ← at the root of the zip
├── requirements.txt             ← at the root of the zip
└── check_website/               ← subfolder named after the function
    ├── function.json
    ├── check_website.py
    └── (Python libraries go here after pip install)
```

### Part A — Install Python libraries

**Mac / Linux:**
```bash
cd ~/uptime-monitor-001/function_app

# Install libraries into the check_website subfolder.
# They must live alongside the function code, not at the root.
pip3 install -r requirements.txt --target check_website/.python_packages/lib/site-packages
```

**Windows (PowerShell):**
```powershell
cd "$HOME\uptime-monitor-001\function_app"

pip install -r requirements.txt --target check_website\.python_packages\lib\site-packages
```

### Part B — Create the zip package

**Mac / Linux:**
```bash
# Build the zip from inside function_app/ so paths are correct.
cd ~/uptime-monitor-001/function_app
zip -r ../function_deploy.zip .
cd ..

# Verify the zip has the right structure
unzip -l function_deploy.zip | head -20
# You should see host.json and requirements.txt at the root,
# and check_website/function.json and check_website/check_website.py inside a subfolder.
```

**Windows (PowerShell):**
```powershell
cd "$HOME\uptime-monitor-001\function_app"

Compress-Archive -Path * -DestinationPath ..\function_deploy.zip -Force
cd ..
```

### Part C — Upload to the Function App

Replace `yourname` with your actual `terraform.tfvars` value.

**Mac / Linux:**
```bash
az functionapp deployment source config-zip \
  --resource-group rg-uptime-monitor-yourname \
  --name func-uptime-yourname \
  --src function_deploy.zip

# Expected: JSON output with "status": "success" or "complete".
```

**Windows (PowerShell):**
```powershell
az functionapp deployment source config-zip `
  --resource-group rg-uptime-monitor-yourname `
  --name func-uptime-yourname `
  --src function_deploy.zip
```

## Step 9 — Verify the Monitor Is Running

Wait 5 minutes after deploying for the first execution to complete.

**Check 1 — Function is listed in the portal**
Azure portal → search `func-uptime-[yourname]` → open the Function App → **Functions** in the left menu → `check_website` should appear with **Status: Enabled**. If it doesn't appear, the zip structure is wrong.

**Check 2 — Table Storage has rows**
```bash
az storage entity query \
  --account-name stuptimeyourname \
  --table-name uptimechecks \
  --auth-mode login \
  --output table
```
Expected: rows with `Status = PASS` every 5 minutes.

**Check 3 — Application Insights is receiving data**
Portal → search `appi-uptime-[yourname]` → **Live Metrics**. Wait up to 5 minutes, expect a spike every time the function executes.

**Check 4 — Alert rule is enabled**
Portal → Monitor → Alerts → Alert rules → `alert-site-down-[yourname]` → status should show **Enabled**.

## Verification Checklist

| Check | What you should see |
|---|---|
| Function App running | Portal → `func-uptime-[yourname]` → Status: Running |
| `check_website` function listed | Function App → Functions → `check_website` appears, Status: Enabled |
| Table has rows | `az storage entity query` returns rows with PASS/SLOW/FAIL |
| PartitionKey is "uptime" | All rows have `PartitionKey = uptime`, not a URL |
| Application Insights active | `appi-uptime-[yourname]` → Live Metrics shows invocations every 5 minutes |
| Alert rule enabled | Monitor → Alert rules → `alert-site-down-[yourname]` is Enabled |
| Action Group configured | `ag-uptime-[yourname]` has both email and SMS receivers |

## Screenshots

*(Images go in the `/screenshots` folder in this repo. Send them over with a note on what each one shows, and they'll get embedded below with a proper takeaway for each.)*

Planned screenshots:

1. **Function App showing `check_website` listed and Enabled**
2. **Table Storage query output** — rows with PASS/SLOW/FAIL and correct PartitionKey
3. **Application Insights Live Metrics** — spikes on each 5-minute invocation
4. **The Monitor alert rule** — showing Enabled status and the KQL query
5. **A received downtime alert** — email and/or SMS, triggered by a deliberate test failure

## Troubleshooting

Every issue below was hit during real deployments of this project.

| Error / symptom | Root cause | Resolution |
|---|---|---|
| `terraform apply` fails: "Operation cannot be completed without additional quota. Current Limit (Total VMs): 0" | Free Azure subscription has vCPU quota 0 in East US | Change `location` in `terraform.tfvars` to `"West US 2"`, run `terraform apply` again, Terraform skips already-created resources |
| Function App shows "Runtime version: Error" after code deploy | `host.json` missing from the zip | Add `host.json` with `{"version":"2.0"}` to `function_app/`. Rebuild the zip and redeploy |
| `check_website` does not appear in Function App → Functions | Zip structure is wrong, files must be inside a named subfolder | Ensure `check_website.py` and `function.json` are inside a `check_website/` subfolder in the zip, not at the root |
| `config-zip` deploy returns 409 Conflict | `WEBSITE_RUN_FROM_PACKAGE = "1"` was set in Terraform `app_settings` | Remove it from `main.tf`, run `terraform apply`, then redeploy the zip |
| `AttributeError: object has no attribute create_table_if_not_exists` | Called on a `TableClient` instead of `TableServiceClient` | Use `table_service.create_table_if_not_exists("uptimechecks")` on the service client, then `get_table_client()` for the table client |
| Table Storage query returns no rows despite successful invocations | PartitionKey was set to the target URL, which contains forbidden `/` and `:` characters | Set PartitionKey to `"uptime"`, store the URL as a separate `TargetUrl` column |
| `ModuleNotFoundError: azure.data.tables` | pip installed libraries to the wrong path, or they weren't included in the zip | Run pip install with `--target check_website/.python_packages/lib/site-packages` |
| Alert never fires | Log Analytics takes 5 to 10 minutes to ingest data on first run; or `workspace_id` link missing | Wait 15 minutes after first execution. Verify `workspace_id` is set on `azurerm_application_insights` |
| SMS not received | Phone number format incorrect | Use E.164 format: `+1` followed by 10 digits, no spaces or dashes, e.g. `+14045550100` |
| Storage account name already taken | Storage account names are globally unique across all of Azure | Change `yourname` in `terraform.tfvars` to something more unique |

## Teardown

```bash
terraform destroy
```
Type `yes` when prompted. All 11 resources will be deleted. The Consumption plan is nearly free, but the storage account and Log Analytics workspace have small ongoing costs if left running.

## Related Labs

- **Project 1** — Azure Cost Visibility Dashboard
- **Project 2** — Automated Backup System
