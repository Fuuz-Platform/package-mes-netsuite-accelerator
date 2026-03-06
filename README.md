# package-mes-netsuite-accelerator

**Version:** 2025.4.0
**Platform:** Fuuz ≥ 2025.4.0
**Enterprise:** appdev
**Spec Version:** 2.0.0

---

## Overview

The MES-NetSuite Accelerator is the bi-directional integration layer connecting the Fuuz Manufacturing Execution System to Oracle NetSuite ERP. It provides 36 data flows covering inbound NetSuite-to-Fuuz record synchronization (work orders, assembly builds, items, BOMs, locations, customers) and outbound Fuuz-to-NetSuite transaction posting (production completions, assembly build confirmations, scrap, inventory adjustments).

The integration uses a webhook/inbound receiver model for NetSuite-initiated events and scheduled polling flows for periodic master data synchronization. A routing framework dispatches each incoming NetSuite record to the correct handler flow based on `baserecordtype` and `type` fields — enabling a single inbound endpoint to handle all NetSuite record types without custom routing configuration per record.

---

## Package Contents

```
mes-netsuite/
├── manifest.json
├── definition.json
├── package-data.json
├── data/                        14 seed data files
├── dataFlows/                   36 integration flows
└── screens/                     6 screens
```

---

## Architecture

### Inbound Router: MES-NS-NetSuite Inbound (`System`, module: `integration`)

The central inbound endpoint. All NetSuite webhook calls arrive here:

1. **Request** — Receives the NetSuite webhook payload
2. **Log: INCOMING RECORD/TYPE** — Logs the incoming `baserecordtype`, `type`, and `id` for tracing
3. **Merge Context: REQ** — Stores the raw payload in flow context under `REQUEST.DATA`
4. **Query: INTEGRATION** — Looks up the active NetSuite `Integration` record and its `IntegrationModel` for the incoming record type (`$uppercase(baserecordtype)`)
5. **Execute Model Flow** — Calls `$executeFlow(model.inboundFlowId, ...)` to dispatch to the type-specific handler flow with the integration context, model metadata, and raw payload

This architecture means adding support for a new NetSuite record type only requires creating a new `IntegrationModel` record pointing to a new handler flow — no changes to the router itself.

---

## Data Flows (36)

### Inbound (NetSuite → Fuuz)

| Flow | Description |
|------|-------------|
| MES-NS-NetSuite Inbound | Central inbound webhook router; dispatches by record type |
| MES-NS-Work Order Inbound | Processes incoming NetSuite Work Orders → creates/updates Fuuz production runs and work order records |
| MES-NS-Assembly Build Inbound | Handles Assembly Build records → updates production setup outputs and quantities |
| MES-NS-Item/Part Inbound | Syncs NetSuite Items/Parts → Fuuz Product records |
| MES-NS-BOM Inbound | Syncs NetSuite BOM structures → Fuuz product strategy processes |
| MES-NS-Customer Inbound | Syncs NetSuite Customers → Fuuz Customer records |
| MES-NS-Location Inbound | Syncs NetSuite Locations → Fuuz Facility/Location records |
| MES-NS-Vendor Inbound | Syncs NetSuite Vendors → Fuuz Supplier records |
| MES-NS-Employee Inbound | Syncs NetSuite Employees → Fuuz user/operator records |
| MES-NS-Purchase Order Inbound | Handles inbound PO records for receiving workflow integration |
| MES-NS-Sales Order Inbound | Handles inbound SO records for shipment/fulfillment workflow integration |
| MES-NS-Inventory Transfer Inbound | Syncs inventory transfer records from NetSuite |

### Outbound (Fuuz → NetSuite)

| Flow | Description |
|------|-------------|
| MES-NS-Production Complete Outbound | Posts Fuuz production history records to NetSuite as Work Order Completions |
| MES-NS-Assembly Build Complete Outbound | Posts assembly build completions to NetSuite |
| MES-NS-Scrap Outbound | Posts scrap transactions to NetSuite for inventory adjustment |
| MES-NS-Inventory Adjustment Outbound | Posts inventory adjustment transactions to NetSuite |
| MES-NS-Receipt Confirm Outbound | Posts purchase order receipt confirmations to NetSuite |
| MES-NS-Shipment Outbound | Posts shipment records to NetSuite Sales Orders |

### Master Data Sync (Scheduled)

| Flow | Description |
|------|-------------|
| MES-NS-Item Master Sync | Scheduled bulk sync of NetSuite Items to Fuuz Products |
| MES-NS-Customer Master Sync | Scheduled sync of NetSuite Customers |
| MES-NS-Location Master Sync | Scheduled sync of NetSuite Locations |
| MES-NS-Work Order Sync | Scheduled sync of open Work Orders from NetSuite |
| Additional sync flows (14+) | BOM sync, vendor sync, employee sync, and supporting reference data |

---

## Seed Data (14 records)

Pre-configured reference data:

- **Integration record** — The `NetSuite` integration configuration record with authentication and endpoint settings
- **IntegrationModels** — One record per supported NetSuite record type, linking the type name to its inbound handler flow ID
- **TransactionType mappings** — NetSuite-to-Fuuz transaction type cross-references for outbound posting
- **Connection configuration** — NetSuite REST API connection settings template

---

## Screens (6)

- **NetSuite Integration Status** — Dashboard showing active `IntegrationModel` configurations, last transaction timestamps, and error indicators
- **Integration Model Configuration** — CRUD screen for managing `IntegrationModel` records (add/remove supported record types, change handler flow assignments)
- **Transaction Log** — Searchable log of all inbound and outbound NetSuite transactions with status and payload viewer
- **Outbound Queue** — View of pending outbound transactions waiting to be posted to NetSuite
- **Error Review** — Screen for reviewing and reprocessing failed integration transactions
- **Connection Settings** — NetSuite API connection configuration screen (endpoint, credentials, token management)

---

## Installation

1. Ensure `application-mes-accelerator` or `application-mes-core-accelerator` is installed (provides target data models)
2. Import this package via Fuuz Package Manager
3. Configure the NetSuite `Connection` record with API endpoint URL, OAuth token credentials, and account ID
4. Activate the NetSuite `Integration` seed record
5. Configure `IntegrationModel` records for each NetSuite record type your implementation will use, pointing to the appropriate inbound handler flows
6. Register the inbound webhook URL (the `MES-NS-NetSuite Inbound` flow's HTTP trigger URL) in NetSuite as a Script Deployment or Webhook endpoint
7. Test with a sample NetSuite record by triggering a manual send and reviewing the Transaction Log
8. Activate outbound flows and schedule master data sync flows via `package-master-data-flow-config-accelerator`

---

## Dependencies

- **Fuuz Platform** ≥ 2025.4.0
- **`application-mes-accelerator`** or **`application-mes-core-accelerator`** — provides target data models
- **Oracle NetSuite ERP** with REST Web Services and SuiteScript enabled
- **NetSuite OAuth 2.0** credentials configured
- **`package-master-data-flow-config-accelerator`** (optional) — for scheduled master data sync management

---

*Built on the [Fuuz Industrial Operations Platform](https://fuuz.com)*
