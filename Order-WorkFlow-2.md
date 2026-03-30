# Order Workflow

## Order Status Overview

| Status | Value | Description |
|--------|-------|-------------|
| New | 1 | Order created, editable (add/edit lines, documents, info) |
| Requested | 2 | Submitted for operator review |
| Rejected | 3 | Operator rejected the order |
| Confirmed | 4 | Approved, triggers WMS sync & shipping label |
| WarehouseProcessing | 5 | Warehouse is picking/packing the order |
| Shipped | 7 | Goods shipped out from warehouse |
| Delivered | 10 | Carrier confirmed delivery |
| Completed | 8 | Order finalized and closed |
| Abnormal | 9 | Error occurred during processing (WMS sync fail, etc.) |
| Holded | 100 | Temporarily on hold by WMS or operator |
| Canceled | 255 | Order cancelled |

---

## Complete Order Lifecycle Flow

```mermaid
flowchart TD
    Start([🧑‍💼 Customer / API])

  %% -- Creation --
    Start -->|"POST /orders <br> POST /orders/import"| New["📝 <b>New</b> <br> <i>Status = 1</i>"]

    %% -- Editing (while New) --
    New -.->|"PUT /orders/{id}<br>POST /lines · PUT /lines<br>DELETE /lines <br> POST /documents"| New

    %% -- Submit --
    New -->|"POST /orders/{id}/request<br>POST /orders/request-batch"| Requested[" <b>Requested</b><br><i>Status = 2</i>"]

    %% -- Operator Review --
    Requested -->|"Auto-confirm<br>(if enabled)"| Confirmed
    Requested -->|"Operator confirms<br>(AdminApi)"| Confirmed[" ✅ <b>Confirmed</b><br><i>Status = 4</i>"]
    Requested -->|"Operator rejects<br>(AdminApi)"| Rejected["❌ <b>Rejected</b><br><i>Status = 3</i>"]

    %% -- Rejected ? back to New --
    Rejected -->|"Operator moves<br>back to New<br>(AdminApi)"| New

    %% -- WMS & Warehouse --
    Confirmed -->|"Auto: Sync to WMS<br>+ Make/Upload Label"| WarehouseProcessing[" <b>Warehouse<br>Processing</b><br><i>Status = 5</i>"]

    %% -- Abnormal (from any processing state) --
    Confirmed -->|"WMS sync fails<br>or error"| Abnormal["⚠️ <b>Abnormal</b><br><i>Status = 9</i>"]
    WarehouseProcessing -->|"Processing error"| Abnormal

    %% -- Hold --
    Confirmed -->|"Hold by WMS<br>or operator"| Holded["⏸️ <b>Holded</b><br><i>Status = 100</i>"]
    WarehouseProcessing -->|"Hold by WMS<br>or operator"| Holded

    %% -- Shipping --
    WarehouseProcessing -->|"Goods shipped<br>(WMS callback<br>or AdminApi)"| Shipped["🚚 <b>Shipped</b><br><i>Status = 7</i>"]
    %% -- Delivery --
    Shipped -->|"Carrier confirms<br>delivery<br>(AdminApi)"| Delivered["📦 <b>Delivered</b><br><i>Status = 10</i>"]

    %% -- Completion --
    Shipped -->|"Operator completes<br>(AdminApi)"| Completed["✅ <b>Completed</b><br><i>Status = 8</i>"]
    Delivered -->|"Operator completes<br>(AdminApi)"| Completed

    %% -- Cancellation (from multiple states) --
    Requested -->|"POST /orders/{id}/cancel"| Canceled["❌ <b>Canceled</b><br><i>Status = 255</i>"]
    Confirmed -->|"POST /orders/{id}/cancel"| Canceled
    Rejected -->|"POST /orders/{id}/cancel"| Canceled
    WarehouseProcessing -->|"POST /orders/{id}/cancel"| Canceled

    %% -- Deletion --
    New -->|"DELETE /orders/{id}"| Deleted(["🗑️ <b>Deleted</b>"])
    Canceled -->|"DELETE /orders/{id}"| Deleted

    %% -- Styling --
    style New fill:#e3f2fd,stroke:#1565c0,color:#000
    style Requested fill:#fff3e0,stroke:#e65100,color:#000
    style Rejected fill:#ffebee,stroke:#c62828,color:#000
    style Confirmed fill:#e8f5e9,stroke:#2e7d32,color:#000
    style WarehouseProcessing fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style Shipped fill:#e0f7fa,stroke:#00695c,color:#000
    style Delivered fill:#f1f8e9,stroke:#33691e,color:#000
    style Completed fill:#e8eaf6,stroke:#283593,color:#000
    style Abnormal fill:#fff8e1,stroke:#f57f17,color:#000
    style Holded fill:#eceff1,stroke:#37474f,color:#000
    style Canceled fill:#fce4ec,stroke:#880e4f,color:#000
    style Deleted fill:#eeeeee,stroke:#616161,color:#666
```

---

## Status Transition Rules

The table below shows which statuses an order can transition **from ? to**, and who triggers it.

| From | To | Triggered By | API / Mechanism |
|------|----|-------------|-----------------|
| Create/ Import | **New** | Merchant | `POST /v1/orders` or `POST /v1/orders/import` |
| New | **Requested** | Merchant | `POST /v1/orders/{id}/request` |
| Requested, Rejected | **New** | Operator | AdminApi: `POST /orders/{id}/to-new` |
| Requested | **Confirmed** | Operator or Auto | AdminApi: `POST /orders/{id}/confirm` or auto-confirm event |
| Requested | **Rejected** | Operator | AdminApi: `POST /orders/{id}/reject` |
| Confirmed | **WarehouseProcessing** | System / Operator | AdminApi: `POST /orders/{id}/stock-out-ordered` |
| WarehouseProcessing | **Shipped** | System / Operator | AdminApi: `POST /orders/{id}/shipped` |
| Shipped | **Delivered** | System / Operator | AdminApi: `POST /orders/{id}/delivered` |
| Shipped, Delivered | **Completed** | System | AdminApi: `POST /orders/{id}/complete` |
| Confirmed, WarehouseProcessing | **Holded** | WMS / Operator | WMS hold callback |
| Any (except Completed) | **Abnormal** | System | WMS sync failure, processing error |
| Requested, Confirmed, Rejected, WarehouseProcessing | **Canceled** | Merchant / Operator | `POST /v1/orders/{id}/cancel` |
| New, Canceled | **Deleted** | Merchant | `DELETE /v1/orders/{id}` |

---

## Notifications per Status Change

When an order transitions to a new status, the system may send notifications to the merchant (via webhooks, emails, or in-app alerts).

| Status Change | Notification Sent to Merchant |
|--------------|-------------------------------|
|  New | — |
|  Requested | Order submitted for review |
|  Rejected | Order rejected (with reason) |
|  Confirmed | Order confirmed and being prepared |
|  WarehouseProcessing | — |
|  Shipped | Order shipped |
|  Delivered | — |
|  Completed | Order completed |
|  Canceled | Order cancelled (with reason) |
|  Abnormal | Order anomaly detected (with reason) |
|  Holded | — |
| Info updated | Order information changed |

---

## Merchant (Public API) Actions Summary

These are the actions available to merchants through the Public API (`/v1/orders`):

```mermaid
flowchart TD
    subgraph MerchantActions[" Merchant Actions via Public API"]
        direction TB
        A1["<b>Create Order</b><br>POST /v1/orders<br><i> Status: New</i>"]
        A2["<b>Import Orders</b><br>POST /v1/orders/import<br><i> Status: New</i>"]
        A3["<b>Update Order Info</b><br>PUT /v1/orders/{id}<br><i>Only when: New</i>"]
        A4["<b>Manage Order Lines</b><br>POST/PUT/DELETE .../lines<br><i>Only when: New</i>"]
        A5["<b>Manage Documents</b><br>POST/DELETE .../documents<br><i>Any status</i>"]
        A6["<b>Submit Order</b><br>POST /v1/orders/{id}/request<br><i>Requested</i>"]
        A7["<b>Batch Submit</b><br>POST /v1/orders/request-batch<br><i>Requested</i>"]
        A8["<b>Cancel Order</b><br>POST /v1/orders/{id}/cancel<br><i>? Status: Canceled</i>"]
        A9["<b>Delete Order</b><br>DELETE /v1/orders/{id}<br><i>Only when: New or Canceled</i>"]
        A10["<b>Query Orders</b><br>GET /v1/orders<br>GET /v1/orders/{id}<br>GET /v1/orders/count-status"]
    end

    style MerchantActions fill:#f5f5f5,stroke:#424242
```

---

## Operator (Admin API) Actions Summary

These are additional actions available to operators through the Admin API:

| Action | Endpoint | Allowed From Status |
|--------|----------|-------------------|
| Confirm | `POST /orders/{id}/confirm` | Requested |
| Batch Confirm | `POST /orders/confirm-batch` | Requested |
| Reject | `POST /orders/{id}/reject` | Requested |
| Move to New | `POST /orders/{id}/to-new` | Rejected, Requested |
| Stock-Out Request | `POST /orders/{id}/stock-out-ordered` | Confirmed |
| Mark Shipped | `POST /orders/{id}/shipped` | WarehouseProcessing |
| Mark Delivered | `POST /orders/{id}/delivered` | Shipped |
| Complete | `POST /orders/{id}/complete` | Shipped, Delivered |
| Cancel | `POST /orders/{id}/cancel` | Requested, Confirmed, Rejected, WarehouseProcessing |
| Sync to WMS | `POST /orders/{id}/sync-wms` | Confirmed |
| Make Shipping Label | `POST /orders/{id}/make-shipping-label` | ECommerce + Warehouse vendor + WMS synced |
| Change Shipping | `POST /orders/{id}/shipping` | New ? Completed |

---

## End-to-End Happy Path (ECommerce Example)

```mermaid
sequenceDiagram
    participant M as 🧑‍💼 Merchant<br/>(Public API)
    participant S as ⚙️ System
    participant O as 👤 Operator<br/>(Admin API)

    M->>S: POST /v1/orders (ECommerce)
    S-->>M: 201 Created (status: New)

    M->>S: POST /orders/{id}/lines
    S-->>M: 201 Created (line added)

    M->>S: POST /orders/{id}/request
    S-->>M: 202 Accepted (status: Requested)

    alt Auto-Confirm enabled
        S->>S: Auto-confirm order
    else Manual Confirm
        S-->>O: 🔔 Notification: "New order pending review"
        O->>S: POST /orders/{id}/confirm
    end

    S-->>S: Status Confirmed
    S-->>M: Notification: "Order confirmed"
    S->>S: Sync to WMS + Make/Upload shipping label

    O->>S: POST /orders/{id}/stock-out-ordered
    S-->>S: Status WarehouseProcessing

    S-->>S: Warehouse ships goods
    S-->>S: Status Shipped
    S-->>M: Notification: "Order shipped"

    S-->>S: Carrier confirms delivery
    S-->>S: Status Delivered

    O->>S: POST /orders/{id}/complete
    S-->>S: Status Completed
    S-->>M: Notification: "Order completed"
```

---

## Cancellation Flow

```mermaid
sequenceDiagram
    participant M as 🧑‍💼 Merchant
    participant S as ⚙️ System

    M->>S: POST /v1/orders/{id}/cancel<br/>{ cancelReason: "..." }

    alt Order is cancellable (Requested/Confirmed/Rejected/WarehouseProcessing)
        S-->>M: 202 Accepted (status: Canceled)
        S-->>M: Notification: "Order cancelled"
    else Order is NOT cancellable (New/Shipped/Completed/...)
        S-->>M: 400 Bad Request
    end

```

---

## Order Types

The system supports three order types. The workflow is the same for all types; only the required fields differ:

| Field | ECommerce | FBA | Wholesale |
|-------|:---------:|:---:|:---------:|
| customerId | ✅ Required | — | ✅ Required |
| storeId | ✅ Required | — | — |
| shippingAddress | ✅ Required | — | ✅ Required |
| fbaWarehouseId | — | ✅ Required | — |
| shipmentType | — | ✅ Required | ✅ Required |
| hasRepalletization | — | ✅ Required | ✅ Required |
| shipmentVendor | ✅ Required | ✅ Required | ✅ Required |
