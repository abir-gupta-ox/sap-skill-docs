# SAP ↔ OXmaint Sync Agent — Full Skill Documentation
> For use with NVIDIA NeMo Agent Toolkit (OpenShell / NeMo Agent Toolkit v1.6+)

---

## 1. Overview & Purpose

This skill enables a bi-directional synchronization agent between **OXmaint CMMS** and **SAP Plant Maintenance (PM)** using the **NVIDIA NeMo Agent Toolkit**. The agent:

- Polls OXmaint for new/updated Work Orders, PM Schedules, and Assets
- Pushes that data into SAP via OData REST APIs
- Polls SAP for new/updated Maintenance Orders, Notifications, and Preventive Maintenance (PM) Plans
- Pushes that data back into OXmaint via OXmaint REST API
- Runs as a scheduled or event-driven workflow inside NeMo Agent Toolkit

---

## 2. Architecture Diagram (Text)

```
┌─────────────────────────────────────────────────────────────────┐
│                     NeMo Agent Toolkit                          │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │ OXmaint Poll │    │  Sync Logic  │    │   SAP Push/Pull  │  │
│  │   Agent      │───▶│  Orchestrat. │───▶│     Agent        │  │
│  └──────────────┘    │  (NeMo Flow) │    └──────────────────┘  │
│                      └──────┬───────┘                          │
│                             │                                   │
│                      ┌──────▼───────┐                          │
│                      │  State Store │                          │
│                      │ (last_sync   │                          │
│                      │  timestamps) │                          │
│                      └──────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
         │                                          │
         ▼                                          ▼
┌─────────────────┐                    ┌─────────────────────────┐
│  OXmaint REST   │                    │  SAP S/4HANA OData v2   │
│  API            │                    │  /sap/opu/odata/sap/    │
│  (CMMS Portal)  │                    │  API_MAINTENANCEORDER   │
└─────────────────┘                    └─────────────────────────┘
```

---

## 3. Prerequisites

### 3.1 SAP Side
| Requirement | Detail |
|---|---|
| SAP Version | S/4HANA 2020+ (Cloud or On-Premise) |
| OData Services | `API_MAINTENANCEORDER`, `API_MAINTNOTIFICATION`, `API_PMORDER_OPERATION`, `API_PREVENTIVE_MAINT_PLAN` |
| Auth | OAuth 2.0 (Cloud) or Basic Auth (On-Premise) |
| Communication Scenario | `SAP_COM_0231` (Plant Maintenance Integration) |
| SAP BTP | Required if using Cloud Connector for On-Premise |

### 3.2 OXmaint Side
| Requirement | Detail |
|---|---|
| OXmaint API Key | Available under Settings → API Integrations |
| Base URL | `https://api.oxmaint.com/v1` (confirm with your instance) |
| Webhook (optional) | Can register for push events vs. polling |

### 3.3 NeMo Agent Toolkit
```bash
pip install nemo-agent-toolkit langchain
# Requires Python 3.11, 3.12, or 3.13
export NVIDIA_API_KEY="your-key-from-build.nvidia.com"
```

---

## 4. SAP API Reference

### 4.1 Authentication

**SAP S/4HANA Cloud (OAuth 2.0)**
```
POST https://<tenant>.authentication.sap.hana.ondemand.com/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=<your_client_id>
&client_secret=<your_client_secret>
```
Response includes `access_token` — pass as `Authorization: Bearer <token>` on all requests.

**SAP On-Premise (Basic Auth)**
```
Authorization: Basic base64(<username>:<password>)
x-csrf-token: Fetch    ← Required for POST/PATCH/DELETE
```
First do a `GET` with `x-csrf-token: Fetch` header → SAP returns `x-csrf-token: <token>` in response headers. Use that token for all write operations.

---

### 4.2 Core SAP OData Endpoints

Base path (On-Premise): `https://<host>:<port>/sap/opu/odata/sap/`
Base path (Cloud): `https://api.<region>.hana.ondemand.com/`

#### A. Maintenance Orders (Work Orders)

**Service:** `API_MAINTENANCEORDER`

| Operation | Method | URL |
|---|---|---|
| List all orders | GET | `.../API_MAINTENANCEORDER/MaintenanceOrder` |
| Get single order | GET | `.../API_MAINTENANCEORDER/MaintenanceOrder('<AUFNR>')` |
| Create order | POST | `.../API_MAINTENANCEORDER/MaintenanceOrder` |
| Update order | PATCH | `.../API_MAINTENANCEORDER/MaintenanceOrder('<AUFNR>')` |
| Get operations | GET | `.../API_MAINTENANCEORDER/MaintenanceOrderOperation` |
| Get components | GET | `.../API_MAINTENANCEORDER/MaintenanceOrderComponent` |

**Key fields for sync:**

```json
{
  "MaintenanceOrder": "000004000001",
  "MaintenanceOrderDesc": "Pump Inspection",
  "OrderType": "PM01",
  "MaintenanceActivityType": "001",
  "Equipment": "10000001",
  "FunctionalLocation": "FL-PUMP-01",
  "MainWorkCenter": "MECH",
  "MaintPlanningPlant": "1000",
  "BasicStartDate": "2025-04-15",
  "BasicFinishDate": "2025-04-16",
  "Priority": "1",
  "SystemStatus": "CRTD",
  "OrderProcessingStage": "Open",
  "MaintOrdPersonResponsible": "JSMITH",
  "MaintenanceOrderLongText": "Full inspection of pump P-101"
}
```

**SAP Order Types (PM):**
| Code | Type | Maps to OXmaint |
|---|---|---|
| PM01 | Corrective Maintenance | Corrective WO |
| PM02 | Breakdown Maintenance | Emergency WO |
| PM03 | Preventive Maintenance | PM Schedule WO |
| PM04 | Refurbishment | Refurbishment WO |
| PM05 | Calibration | Calibration WO |
| PM06 | Capital Investment | CapEx WO |

**System Status Values:**
| SAP Status | Meaning | OXmaint Equivalent |
|---|---|---|
| CRTD | Created | Draft |
| REL | Released | Open / In Progress |
| PCNF | Partially Confirmed | In Progress |
| CNF | Confirmed | Completed |
| TECO | Technically Complete | Closed |
| CLSD | Closed | Archived |

**OData Filter Examples:**
```
# Changed since last sync (delta query)
$filter=ChangedDateTime gt datetimeoffset'2025-04-14T00:00:00Z'

# Only released + open orders
$filter=SystemStatus eq 'REL'

# By functional location
$filter=FunctionalLocation eq 'FL-PUMP-01'

# Expand operations and components in one call
$expand=to_MaintOrderOperation,to_MaintOrderComponent
```

---

#### B. Maintenance Notifications

**Service:** `API_MAINTNOTIFICATION`

| Operation | Method | URL |
|---|---|---|
| List notifications | GET | `.../API_MAINTNOTIFICATION/MaintenanceNotification` |
| Get single | GET | `.../API_MAINTNOTIFICATION/MaintenanceNotification('<QMNUM>')` |
| Create | POST | `.../API_MAINTNOTIFICATION/MaintenanceNotification` |
| Update | PATCH | `.../API_MAINTNOTIFICATION/MaintenanceNotification('<QMNUM>')` |
| Set to In-Process | POST | function import `SetMaintNotifToInProcess` |
| Complete notification | POST | function import `CompleteMaintNotification` |

**Key Notification Fields:**
```json
{
  "MaintenanceNotification": "000010001234",
  "NotificationType": "M1",
  "MaintNotifLongText": "Leaking valve detected",
  "Equipment": "10000001",
  "FunctionalLocation": "FL-PUMP-01",
  "ReportedByUser": "JSMITH",
  "MaintNotificationDate": "2025-04-15",
  "MaintNotifProcessingPhase": "OSNO",
  "Priority": "2"
}
```

**Notification Types:**
| Code | Type |
|---|---|
| M1 | Activity Report |
| M2 | Malfunction Report |
| S4 | Service Notification |

---

#### C. Preventive Maintenance Plans

**Service:** `API_PREVENTIVE_MAINT_PLAN` (or `API_MAINTPLAN`)

| Operation | Method | URL |
|---|---|---|
| List PM plans | GET | `.../API_PREVENTIVE_MAINT_PLAN/MaintenancePlan` |
| Get single plan | GET | `.../API_PREVENTIVE_MAINT_PLAN/MaintenancePlan('<WARPL>')` |
| List items | GET | `.../API_PREVENTIVE_MAINT_PLAN/MaintenancePlanItem` |
| Schedule call | POST | `.../API_PREVENTIVE_MAINT_PLAN/CallMaintenancePlan` |

**Key PM Plan Fields:**
```json
{
  "MaintenancePlan": "00000100",
  "MaintenancePlanDesc": "Monthly Pump Inspection",
  "MaintPlanCategory": "PM",
  "CycleLength": 30,
  "CycleTimeUnit": "DAY",
  "StartDate": "2025-01-01",
  "NextPlannedDate": "2025-05-01",
  "MaintPlanCallObject": "PM01",
  "Equipment": "10000001",
  "FunctionalLocation": "FL-PUMP-01"
}
```

---

#### D. Equipment & Functional Locations (Asset Master)

**Service:** `API_EQUIPMENT` / `API_FUNCTIONALLOCATION`

| Operation | Method | URL |
|---|---|---|
| List equipment | GET | `.../API_EQUIPMENT/Equipment` |
| Get single | GET | `.../API_EQUIPMENT/Equipment('<EQUNR>')` |
| List func. locations | GET | `.../API_FUNCTIONALLOCATION/FunctionalLocation` |

---

### 4.3 OData Query Patterns

```
# Pagination
$top=100&$skip=0

# Select specific fields
$select=MaintenanceOrder,MaintenanceOrderDesc,SystemStatus,BasicStartDate

# Filter with AND
$filter=OrderType eq 'PM01' and SystemStatus eq 'REL'

# Order by creation date
$orderby=CreatedDateTime desc

# Format (always use JSON)
$format=json
```

---

## 5. OXmaint API Reference

Base URL: `https://api.oxmaint.com/v1`
Auth Header: `Authorization: Bearer <OXMAINT_API_KEY>`

### 5.1 Work Orders

| Operation | Method | URL |
|---|---|---|
| List work orders | GET | `/work-orders` |
| Get single | GET | `/work-orders/{id}` |
| Create | POST | `/work-orders` |
| Update | PATCH | `/work-orders/{id}` |
| List PM schedules | GET | `/pm-schedules` |
| Get assets | GET | `/assets` |

**OXmaint Work Order Payload:**
```json
{
  "title": "Pump P-101 Inspection",
  "description": "Full inspection required",
  "status": "open",
  "priority": "high",
  "asset_id": "asset_101",
  "assigned_to": "tech_user_id",
  "due_date": "2025-04-20",
  "work_order_type": "preventive",
  "custom_fields": {
    "sap_order_number": "000004000001",
    "sap_order_type": "PM03",
    "sap_functional_location": "FL-PUMP-01"
  }
}
```

### 5.2 Field Mapping: OXmaint ↔ SAP

| OXmaint Field | SAP Field | Notes |
|---|---|---|
| `id` | `ExternalOrderID` | Store OXmaint ID in SAP long text or custom field |
| `title` | `MaintenanceOrderDesc` | Truncate to 40 chars for SAP |
| `description` | `MaintenanceOrderLongText` | Full text |
| `status: open` | `SystemStatus: CRTD/REL` | Map on create vs release |
| `status: in_progress` | `SystemStatus: REL` | Release order in SAP |
| `status: completed` | `SystemStatus: TECO` | Tech complete in SAP |
| `priority: critical` | `Priority: 1` | |
| `priority: high` | `Priority: 2` | |
| `priority: medium` | `Priority: 3` | |
| `priority: low` | `Priority: 4` | |
| `asset_id` | `Equipment` | Pre-mapped table needed |
| `location` | `FunctionalLocation` | Pre-mapped table needed |
| `due_date` | `BasicFinishDate` | ISO 8601 → SAP date format |
| `scheduled_date` | `BasicStartDate` | |
| `work_order_type: corrective` | `OrderType: PM01` | |
| `work_order_type: emergency` | `OrderType: PM02` | |
| `work_order_type: preventive` | `OrderType: PM03` | |

---

## 6. NeMo Agent Toolkit — Workflow Configuration

### 6.1 Install & Setup

```bash
pip install nemo-agent-toolkit langchain requests python-dotenv

# .env file
SAP_BASE_URL=https://<your-sap-host>/sap/opu/odata/sap
SAP_CLIENT_ID=<oauth_client_id>
SAP_CLIENT_SECRET=<oauth_client_secret>
SAP_AUTH_URL=https://<tenant>.authentication.sap.hana.ondemand.com/oauth/token
OXMAINT_API_KEY=<your_oxmaint_key>
OXMAINT_BASE_URL=https://api.oxmaint.com/v1
NVIDIA_API_KEY=<from_build.nvidia.com>
```

### 6.2 Tool Definitions (Python)

```python
# tools/sap_tools.py
import requests, os
from nemo_agent_toolkit import tool

SAP_BASE = os.getenv("SAP_BASE_URL")

def get_sap_token():
    r = requests.post(os.getenv("SAP_AUTH_URL"), data={
        "grant_type": "client_credentials",
        "client_id": os.getenv("SAP_CLIENT_ID"),
        "client_secret": os.getenv("SAP_CLIENT_SECRET")
    })
    return r.json()["access_token"]

@tool(name="sap_get_maintenance_orders",
      description="Fetch maintenance orders from SAP PM updated since a given datetime")
def sap_get_maintenance_orders(since_datetime: str = None) -> dict:
    """
    Args:
        since_datetime: ISO 8601 e.g. '2025-04-14T00:00:00Z'. If None, fetches last 24h.
    Returns:
        dict with list of maintenance orders
    """
    token = get_sap_token()
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/json"
    }
    filter_str = ""
    if since_datetime:
        filter_str = f"?$filter=ChangedDateTime gt datetimeoffset'{since_datetime}'&$format=json"
    else:
        filter_str = "?$format=json&$top=50&$orderby=ChangedDateTime desc"
    
    url = f"{SAP_BASE}/API_MAINTENANCEORDER/MaintenanceOrder{filter_str}"
    resp = requests.get(url, headers=headers)
    resp.raise_for_status()
    return resp.json().get("d", {}).get("results", [])


@tool(name="sap_create_maintenance_order",
      description="Create a new maintenance order in SAP PM from OXmaint work order data")
def sap_create_maintenance_order(
    description: str,
    order_type: str,
    equipment: str,
    functional_location: str,
    start_date: str,
    finish_date: str,
    priority: str,
    long_text: str,
    external_id: str
) -> dict:
    """
    Creates a maintenance order in SAP.
    order_type: PM01 | PM02 | PM03 | PM04 | PM05
    priority: 1 (critical) | 2 (high) | 3 (medium) | 4 (low)
    """
    token = get_sap_token()
    # First get CSRF token for write operation
    csrf_resp = requests.get(
        f"{SAP_BASE}/API_MAINTENANCEORDER/MaintenanceOrder?$top=1",
        headers={"Authorization": f"Bearer {token}", "x-csrf-token": "Fetch"}
    )
    csrf_token = csrf_resp.headers.get("x-csrf-token")
    
    payload = {
        "MaintenanceOrderDesc": description[:40],
        "OrderType": order_type,
        "Equipment": equipment,
        "FunctionalLocation": functional_location,
        "BasicStartDate": f"/Date({start_date})/",
        "BasicFinishDate": f"/Date({finish_date})/",
        "Priority": priority,
        "MaintenanceOrderLongText": f"[OXmaint ID: {external_id}] {long_text}"
    }
    
    resp = requests.post(
        f"{SAP_BASE}/API_MAINTENANCEORDER/MaintenanceOrder",
        json=payload,
        headers={
            "Authorization": f"Bearer {token}",
            "x-csrf-token": csrf_token,
            "Content-Type": "application/json",
            "Accept": "application/json"
        }
    )
    resp.raise_for_status()
    return resp.json().get("d", {})


@tool(name="sap_update_maintenance_order",
      description="Update an existing SAP maintenance order status or fields")
def sap_update_maintenance_order(order_number: str, fields: dict) -> dict:
    """
    Args:
        order_number: SAP order number e.g. '000004000001'
        fields: dict of fields to update e.g. {'SystemStatus': 'TECO'}
    """
    token = get_sap_token()
    csrf_resp = requests.get(
        f"{SAP_BASE}/API_MAINTENANCEORDER/MaintenanceOrder('{order_number}')",
        headers={"Authorization": f"Bearer {token}", "x-csrf-token": "Fetch"}
    )
    csrf_token = csrf_resp.headers.get("x-csrf-token")
    
    resp = requests.patch(
        f"{SAP_BASE}/API_MAINTENANCEORDER/MaintenanceOrder('{order_number}')",
        json=fields,
        headers={
            "Authorization": f"Bearer {token}",
            "x-csrf-token": csrf_token,
            "Content-Type": "application/json",
            "Accept": "application/json"
        }
    )
    resp.raise_for_status()
    return {"updated": True, "order_number": order_number}


@tool(name="sap_get_pm_plans",
      description="Fetch preventive maintenance plans from SAP")
def sap_get_pm_plans(equipment: str = None) -> list:
    token = get_sap_token()
    filter_str = f"?$filter=Equipment eq '{equipment}'&$format=json" if equipment \
                 else "?$format=json&$top=100"
    url = f"{SAP_BASE}/API_PREVENTIVE_MAINT_PLAN/MaintenancePlan{filter_str}"
    resp = requests.get(url, headers={
        "Authorization": f"Bearer {token}",
        "Accept": "application/json"
    })
    resp.raise_for_status()
    return resp.json().get("d", {}).get("results", [])


@tool(name="sap_get_notifications",
      description="Fetch SAP maintenance notifications (malfunction reports, activity reports)")
def sap_get_notifications(since_datetime: str = None) -> list:
    token = get_sap_token()
    filter_str = ""
    if since_datetime:
        filter_str = f"?$filter=CreatedDateTime gt datetimeoffset'{since_datetime}'&$format=json"
    else:
        filter_str = "?$format=json&$top=50"
    url = f"{SAP_BASE}/API_MAINTNOTIFICATION/MaintenanceNotification{filter_str}"
    resp = requests.get(url, headers={
        "Authorization": f"Bearer {token}",
        "Accept": "application/json"
    })
    resp.raise_for_status()
    return resp.json().get("d", {}).get("results", [])
```

```python
# tools/oxmaint_tools.py
import requests, os
from nemo_agent_toolkit import tool

OXMAINT_BASE = os.getenv("OXMAINT_BASE_URL")
OXMAINT_KEY = os.getenv("OXMAINT_API_KEY")

def oxmaint_headers():
    return {
        "Authorization": f"Bearer {OXMAINT_KEY}",
        "Content-Type": "application/json",
        "Accept": "application/json"
    }

@tool(name="oxmaint_get_work_orders",
      description="Fetch work orders from OXmaint updated since a given datetime")
def oxmaint_get_work_orders(since_datetime: str = None, status: str = None) -> list:
    params = {}
    if since_datetime:
        params["updated_after"] = since_datetime
    if status:
        params["status"] = status
    resp = requests.get(f"{OXMAINT_BASE}/work-orders",
                        headers=oxmaint_headers(), params=params)
    resp.raise_for_status()
    return resp.json().get("data", [])


@tool(name="oxmaint_create_work_order",
      description="Create a new work order in OXmaint from SAP maintenance order data")
def oxmaint_create_work_order(
    title: str,
    description: str,
    status: str,
    priority: str,
    asset_id: str,
    due_date: str,
    work_order_type: str,
    sap_order_number: str,
    sap_order_type: str
) -> dict:
    payload = {
        "title": title,
        "description": description,
        "status": status,
        "priority": priority,
        "asset_id": asset_id,
        "due_date": due_date,
        "work_order_type": work_order_type,
        "custom_fields": {
            "sap_order_number": sap_order_number,
            "sap_order_type": sap_order_type,
            "source_system": "SAP"
        }
    }
    resp = requests.post(f"{OXMAINT_BASE}/work-orders",
                         json=payload, headers=oxmaint_headers())
    resp.raise_for_status()
    return resp.json().get("data", {})


@tool(name="oxmaint_update_work_order",
      description="Update an existing OXmaint work order")
def oxmaint_update_work_order(work_order_id: str, fields: dict) -> dict:
    resp = requests.patch(f"{OXMAINT_BASE}/work-orders/{work_order_id}",
                          json=fields, headers=oxmaint_headers())
    resp.raise_for_status()
    return resp.json().get("data", {})


@tool(name="oxmaint_get_pm_schedules",
      description="Fetch PM schedules from OXmaint")
def oxmaint_get_pm_schedules() -> list:
    resp = requests.get(f"{OXMAINT_BASE}/pm-schedules", headers=oxmaint_headers())
    resp.raise_for_status()
    return resp.json().get("data", [])
```

### 6.3 NeMo Agent Workflow Definition (YAML)

```yaml
# config/sap_oxmaint_sync.yaml
name: sap_oxmaint_bidirectional_sync
description: >
  Bi-directional sync agent between OXmaint CMMS and SAP Plant Maintenance.
  Runs every 15 minutes. Syncs work orders, PM plans, and notifications.

agents:
  - name: oxmaint_to_sap_agent
    description: >
      Checks OXmaint for new or updated work orders and pushes them to SAP PM.
    model: meta/llama-3.1-70b-instruct
    tools:
      - oxmaint_get_work_orders
      - oxmaint_get_pm_schedules
      - sap_create_maintenance_order
      - sap_update_maintenance_order
    system_prompt: |
      You are a sync agent responsible for pushing OXmaint CMMS data to SAP Plant Maintenance.

      Your workflow:
      1. Call oxmaint_get_work_orders with since_datetime = last successful sync timestamp
      2. For each work order returned:
         a. Check if custom_fields.sap_order_number exists
            - If YES: call sap_update_maintenance_order to sync status changes
            - If NO: call sap_create_maintenance_order to create it in SAP
      3. Log every created/updated order with its IDs for reconciliation
      4. Update the state store with the new sync timestamp

      Field mapping rules:
      - OXmaint status 'open' → SAP SystemStatus 'CRTD'
      - OXmaint status 'in_progress' → SAP SystemStatus 'REL'
      - OXmaint status 'completed' → SAP SystemStatus 'TECO'
      - OXmaint priority 'critical' → SAP Priority '1'
      - OXmaint priority 'high' → SAP Priority '2'
      - OXmaint priority 'medium' → SAP Priority '3'
      - OXmaint work_order_type 'corrective' → SAP OrderType 'PM01'
      - OXmaint work_order_type 'preventive' → SAP OrderType 'PM03'
      - OXmaint work_order_type 'emergency' → SAP OrderType 'PM02'

      Always store the OXmaint ID in the SAP order's long text field as [OXmaint ID: <id>].
      If a SAP API call fails, log the error and continue with the next record. Do not stop the sync.

  - name: sap_to_oxmaint_agent
    description: >
      Checks SAP PM for new or updated maintenance orders and syncs to OXmaint.
    model: meta/llama-3.1-70b-instruct
    tools:
      - sap_get_maintenance_orders
      - sap_get_pm_plans
      - sap_get_notifications
      - oxmaint_create_work_order
      - oxmaint_update_work_order
    system_prompt: |
      You are a sync agent responsible for pulling SAP Plant Maintenance data into OXmaint CMMS.

      Your workflow:
      1. Call sap_get_maintenance_orders with since_datetime = last successful sync timestamp
      2. Call sap_get_notifications with since_datetime = same timestamp
      3. Call sap_get_pm_plans to check for new or modified PM plans
      4. For each SAP maintenance order:
         a. Check MaintenanceOrderLongText for existing [OXmaint ID: <id>] tag
            - If found: call oxmaint_update_work_order with status/priority changes
            - If not found: call oxmaint_create_work_order to create it in OXmaint
      5. For each SAP notification (malfunction report M2):
         - Create a corresponding corrective work order in OXmaint if not already synced
      6. Log all operations for audit trail

      Field mapping rules (reverse of oxmaint_to_sap):
      - SAP 'CRTD' → OXmaint status 'open'
      - SAP 'REL'  → OXmaint status 'in_progress'
      - SAP 'TECO' → OXmaint status 'completed'
      - SAP 'CLSD' → OXmaint status 'closed'
      - SAP Priority '1' → OXmaint priority 'critical'
      - SAP Priority '2' → OXmaint priority 'high'
      - SAP 'PM01' → OXmaint work_order_type 'corrective'
      - SAP 'PM02' → OXmaint work_order_type 'emergency'
      - SAP 'PM03' → OXmaint work_order_type 'preventive'

workflows:
  - name: full_sync
    description: Run both agents sequentially
    steps:
      - agent: oxmaint_to_sap_agent
        input: "Sync all OXmaint work orders updated since last sync to SAP."
      - agent: sap_to_oxmaint_agent
        input: "Sync all SAP maintenance orders updated since last sync to OXmaint."
    schedule: "*/15 * * * *"   # Every 15 minutes
```

### 6.4 Running the Agent

```python
# main.py
from nemo_agent_toolkit import WorkflowRunner
from dotenv import load_dotenv

load_dotenv()

runner = WorkflowRunner.from_config("config/sap_oxmaint_sync.yaml")

# One-time full run
result = runner.run("full_sync")
print(result)

# Or register as scheduled service
runner.start_scheduler()
```

---

## 7. State Management (Last Sync Timestamps)

```python
# state/sync_state.py
import json, os
from datetime import datetime, timezone

STATE_FILE = "sync_state.json"

def get_last_sync(direction: str) -> str:
    """Returns ISO 8601 string of last successful sync"""
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE) as f:
            state = json.load(f)
            return state.get(direction, {}).get("last_sync_at",
                   "2025-01-01T00:00:00Z")
    return "2025-01-01T00:00:00Z"

def set_last_sync(direction: str):
    """Records current time as last successful sync"""
    state = {}
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE) as f:
            state = json.load(f)
    state.setdefault(direction, {})
    state[direction]["last_sync_at"] = datetime.now(timezone.utc).isoformat()
    with open(STATE_FILE, "w") as f:
        json.dump(state, f, indent=2)
```

---

## 8. ID Reconciliation Table

To avoid duplicate records, maintain a mapping table:

```sql
-- reconciliation_table (SQLite/PostgreSQL)
CREATE TABLE sync_map (
  id              SERIAL PRIMARY KEY,
  oxmaint_id      VARCHAR(100) NOT NULL,
  sap_order_no    VARCHAR(20),
  oxmaint_type    VARCHAR(50),   -- 'work_order', 'pm_schedule'
  sap_type        VARCHAR(20),   -- 'PM01', 'PM02', etc.
  last_synced_at  TIMESTAMP NOT NULL,
  sync_direction  VARCHAR(20),   -- 'oxmaint_to_sap', 'sap_to_oxmaint'
  sync_status     VARCHAR(20)    -- 'success', 'failed', 'pending'
);
```

```python
# In your tools, before creating a record:
def check_already_synced(oxmaint_id=None, sap_order_no=None) -> bool:
    # Query sync_map table
    # Return True if a mapping already exists
    pass
```

---

## 9. Error Handling & Retry Strategy

```python
import time
from functools import wraps

def with_retry(max_attempts=3, delay_seconds=5):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except requests.HTTPError as e:
                    if e.response.status_code == 401:
                        # Token expired — refresh and retry
                        refresh_token()
                        continue
                    elif e.response.status_code == 429:
                        # Rate limited — back off
                        time.sleep(delay_seconds * (attempt + 1))
                        continue
                    elif e.response.status_code >= 500:
                        # SAP server error — retry
                        time.sleep(delay_seconds)
                        continue
                    else:
                        # 4xx client error — log and skip
                        log_error(func.__name__, e, args, kwargs)
                        return None
            log_error(func.__name__, "Max retries exceeded", args, kwargs)
            return None
        return wrapper
    return decorator
```

---

## 10. Deployment Options

### Option A: Standalone Python Service (Recommended for start)
```bash
# systemd service or Docker container
docker run -d \
  --env-file .env \
  --name sap-oxmaint-sync \
  your-image:latest python main.py
```

### Option B: SAP BTP Integration Suite
- Deploy as a Cloud Integration iFlow
- Use SAP's standard OData adapters
- Connect to OXmaint via HTTP adapter with OAuth

### Option C: NVIDIA NeMo Agent Toolkit with OpenShell
```bash
# Install OpenShell runtime for policy-based security guardrails
# (NVIDIA's enterprise-grade deployment option)
pip install nemo-openShell
nemo-shell deploy config/sap_oxmaint_sync.yaml
```

---

## 11. Testing Checklist

- [ ] SAP OAuth token acquired successfully
- [ ] `GET /API_MAINTENANCEORDER/MaintenanceOrder?$top=1` returns data
- [ ] CSRF token fetched for write operations
- [ ] OXmaint API key valid — test with `GET /work-orders?$top=1`
- [ ] Field mapping table verified with actual SAP equipment numbers
- [ ] NeMo workflow runs without errors in dry-run mode
- [ ] Reconciliation table prevents duplicate creation
- [ ] Status change in OXmaint reflects in SAP within 15 min
- [ ] Status change in SAP reflects in OXmaint within 15 min
- [ ] Error logs captured and accessible

---

## 12. SAP Transaction Codes (Manual Reference)

| T-Code | Purpose |
|---|---|
| IW31 | Create Maintenance Order |
| IW32 | Change Maintenance Order |
| IW33 | Display Maintenance Order |
| IW21 | Create Notification |
| IP01 | Create Maintenance Plan |
| IP10 | Schedule Maintenance Plan |
| IE01 | Create Equipment |
| IL01 | Create Functional Location |
| /IWFND/MAINT_SERVICE | Activate/manage OData services |

---

## 13. Common Issues & Fixes

| Issue | Cause | Fix |
|---|---|---|
| 403 on POST | Missing CSRF token | Fetch `x-csrf-token` via GET first |
| 401 Unauthorized | Token expired | Re-call OAuth token endpoint |
| OData 404 | Service not activated | Go to `/IWFND/MAINT_SERVICE` in SAP and activate |
| Date format error | SAP expects `/Date(ms)/` format | Convert ISO date to ms epoch |
| Duplicate records | No reconciliation check | Query sync_map before creating |
| Long text truncated | SAP order desc is 40-char max | Use `MaintenanceOrderLongText` for full description |

---

*Last updated: April 2025 — based on SAP S/4HANA 2023 APIs and NVIDIA NeMo Agent Toolkit v1.6*
