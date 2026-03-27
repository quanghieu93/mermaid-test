# Order Workflow

> This document describes the complete order lifecycle — from creation through fulfillment — including API actions, status transitions, and events/notifications fired at each step.

---

## Order Status Overview

| Status | Value | Description |
|--------|-------|-------------|
| New | 1 | Order created, editable (add/edit lines, documents, info) |
| Requested | 2 | Submitted for operator review |
| Rejected | 3 | Operator rejected the order |
| Confirmed | 4 | Approved, triggers WMS sync and shipping label |
| WarehouseProcessing | 5 | Warehouse is picking/packing the order |
| Shipped | 7 | Goods shipped out from warehouse |
| Delivered | 10 | Carrier confirmed delivery |
| Completed | 8 | Order finalized and closed |
| Abnormal | 9 | Error occurred during processing (WMS sync fail, etc.) |
| Holded | 100 | Temporarily on hold by WMS or operator |
| Canceled | 255 | Order cancelled |

---

## Complete Order Lifecycle Flow

```
                                    +-------------------+
                                    |  Customer / API   |
                                    +--------+----------+
                                             |
                              POST /orders   |   POST /orders/import
                                             v
                     +----------------------------------------------+
                     |                                              |
              +------+------+     PUT /orders/{id}                  |
              |             |     POST/PUT/DELETE .../lines          |
              |  (1) New    |<--- POST .../documents                |
              |             |     (edit while New)                   |
              +------+------+                                       |
                     |                                              |
                     | POST /orders/{id}/request                    |
                     v                                              |
              +--------------+                                      |
              |              |----- Operator rejects -----+         |
              | (2) Requested|                            |         |
              |              |----- Auto/Manual ----+     |         |
              +--------------+      confirm         |     |         |
                     ^                              v     v         |
                     |                      +-------+-+ +----------+|
                     |                      |         | |          ||
                     +-- Operator moves ----|  (4)    | |   (3)   ||
                         back to New        |Confirmed| | Rejected||
                                            |         | |          ||
                                            +----+----+ +-----+---+|
                                                 |            |     |
                              Sync to WMS        |            |     |
                              + Make Label       |            |     |
                                                 v            |     |
                                          +------+-------+    |     |
                                          |              |    |     |
                                          |     (5)      |    |     |
                                          |  Warehouse   |    |     |
                                          |  Processing  |    |     |
                                          |              |    |     |
                                          +------+-------+    |     |
                                                 |            |     |
                                     WMS ships   |            |     |
                                                 v            |     |
                                          +------+-------+    |     |
                                          |              |    |     |
                                          | (7) Shipped  |    |     |
                                          |              |    |     |
                                          +---+-----+----+    |     |
                                              |     |         |     |
                              Carrier delivers|     |Complete  |     |
                                              v     |         |     |
                                       +------+--+  |         |     |
                                       |         |  |         |     |
                                       |  (10)   |  |         |     |
                                       |Delivered|  |         |     |
                                       |         |  |         |     |
                                       +----+----+  |         |     |
                                            |       |         |     |
                                   Complete |       |         |     |
                                            v       v         |     |
                                       +----+-------+----+    |     |
                                       |                 |    |     |
                                       | (8) Completed   |    |     |
                                       |                 |    |     |
                                       +-----------------+    |     |
                                                              |     |
                                                              |     |
   +----------------------------------------------------------+-----+
   | Cancel (from Requested, Confirmed, Rejected, WarehouseProcessing)
   v
+--+------------+          DELETE /orders/{id}       +--------------+
|               | ----(from New or Canceled)-------> |              |
| (255) Canceled|                                    | Soft-Deleted |
|               |                                    |              |
+---------------+                                    +--------------+


  Side statuses (can occur during processing):

  +----------------+   WMS sync fail / error    +--------------+
  | Confirmed or   | ------------------------> |              |
  | Warehouse      |                           | (9) Abnormal |
  | Processing     |                           |              |
  +----------------+                           +--------------+

  +----------------+   Hold by WMS/operator     +--------------+
  | Confirmed or   | ------------------------> |              |
  | Warehouse      |                           | (100) Holded |
  | Processing     |                           |              |
  +----------------+                           +--------------+
```

---

## Status Transition Rules

| From | To | Triggered By | API / Mechanism |
|------|----|-------------|-----------------|
| — | **New** | Merchant | `POST /v1/orders` or `POST /v1/orders/import` |
| New | **Requested** | Merchant | `POST /v1/orders/{id}/request` |
| Requested, Rejected | **New** | Operator | AdminApi: `POST /orders/{id}/to-new` |
| Requested | **Confirmed** | Operator or Auto | AdminApi: `POST /orders/{id}/confirm` or auto-confirm event |
| Requested | **Rejected** | Operator | AdminApi: `POST /orders/{id}/reject` |
| Confirmed | **WarehouseProcessing** | System / Operator | AdminApi: `POST /orders/{id}/stock-out-ordered` |
| WarehouseProcessing | **Shipped** | WMS callback / Operator | AdminApi: `POST /orders/{id}/shipped` |
| Shipped | **Delivered** | Carrier / Operator | AdminApi: `POST /orders/{id}/delivered` |
| Shipped, Delivered | **Completed** | Operator | AdminApi: `POST /orders/{id}/complete` |
| Confirmed, WarehouseProcessing | **Holded** | WMS / Operator | WMS hold callback |
| Any (except Completed) | **Abnormal** | System | WMS sync failure, processing error |
| Requested, Confirmed, Rejected, WarehouseProcessing | **Canceled** | Merchant / Operator | `POST /v1/orders/{id}/cancel` |
| New, Canceled | **Soft-Deleted** | Merchant | `DELETE /v1/orders/{id}` |

---

## Events and Notifications per Status Change

When an order transitions to a new status, domain events are raised. These are published to Azure Service Bus (`gs.order.events` topic) and dispatched as notifications (which can trigger webhooks, emails, or in-app alerts).

### Event Pipeline

```
+------------------+       +-------------------------+       +---------------------------+
|  Order Entity    |       |  Azure Service Bus      |       |  Notification             |
|  (Domain Events) | ----> |  Topic: gs.order.events | ----> |  Dispatcher (Consumers)   |
+------------------+       +-------------------------+       +---------------------------+
                                      |
                            Subscriptions:
                            +-- order-notification-sub
                            +-- order-auto-confirm-sub
                            +-- order-wms-sync-sub
                            +-- order-upload-label-sub
                            +-- order-make-label-sub
                            +-- order-store-platform-sub
```

### Event Details

| Status Change | Domain Event | Integration Event(s) | Notification Dispatched |
|--------------|-------------|---------------------|------------------------|
| to New | `OrderCreated` | — | — |
| to Requested | `OrderRequested` | `OrderRequestedNotification` (manual) / `ConfirmOrderRequest` (auto) | `NotifyOrderRequested` |
| to Rejected | `OrderRejected` | `OrderRejectedNotification` | `NotifyOrderRejected` |
| to Confirmed | `OrderConfirmed` | `OrderConfirmedNotification` / `OrderSyncToWmsRequested` | `NotifyOrderConfirmed` |
| WMS sync OK | `OrderToWMSSucceed` | `OrderUploadLabelToWmsRequested` or `OrderMakeLabelRequested` | `NotifyOrderSyncWMSSuccess` |
| WMS sync fail | `OrderToWMSFailed` | `OrderToWMSFailedNotification` | `NotifyOrderSyncWMSFailed` |
| to Shipped | `OrderShipped` | `OrderShippedNotification` | `NotifyOrderShipped` |
| to Delivered | `OrderDelivered` | — | — |
| to Completed | `OrderCompleted` | `OrderCompletedNotification` / `OrderCompleteToStore` | `NotifyOrderCompleted` |
| to Canceled | `OrderCancelled` | `OrderCancelledNotification` | `NotifyOrderCancelled` |
| to Abnormal | `OrderAbnormal` | `OrderAbnormalNotification` | `NotifyOrderAnomaly` |
| to Holded | `OrderHolded` | — | — |
| Info updated | `OrderInfoChanged` | `OrderInfoChangedNotification` | `NotifyOrderChanged` |
| Stocked out | `OrderStockedOut` | `OrderStockedOutNotification` | `NotifyOrderStockedOut` |
| Label make fail | `MakeWmsShippingLabelFailed` | `OrderMakeLabelFailedNotification` | `NotifyOrderMakeLabelFailed` |
| Label sync fail | `OrderSyncShippingLabelFailed` | `OrderLabelSyncFailedNotification` | `NotifyOrderLabelSyncFailed` |

---

## Merchant (Public API) Actions Summary

These are the actions available to merchants through the Public API (`/v1/orders`):

| Action | Method | Endpoint | Status Requirement |
|--------|--------|----------|-------------------|
| Create Order | POST | `/v1/orders` | — (creates New) |
| Import Orders | POST | `/v1/orders/import` | — (creates New) |
| Update Order Info | PUT | `/v1/orders/{id}` | New only |
| Add Order Line | POST | `/v1/orders/{id}/lines` | New only |
| Update Order Line | PUT | `/v1/orders/{id}/lines/{lineId}` | New only |
| Remove Order Line | DELETE | `/v1/orders/{id}/lines/{lineId}` | New only |
| Import Order Lines | POST | `/v1/orders/{id}/lines/import` | New only |
| Upload Documents | POST | `/v1/orders/{id}/documents` | Any status |
| Delete Document | DELETE | `/v1/orders/{id}/documents/{docId}` | Any status |
| Submit Order | POST | `/v1/orders/{id}/request` | New |
| Batch Submit | POST | `/v1/orders/request-batch` | New |
| Cancel Order | POST | `/v1/orders/{id}/cancel` | Requested, Confirmed, Rejected, WarehouseProcessing |
| Delete Order | DELETE | `/v1/orders/{id}` | New, Canceled |
| Get Orders | GET | `/v1/orders` | — |
| Get Order Detail | GET | `/v1/orders/{id}` | — |
| Get Status Count | GET | `/v1/orders/count-status` | — |
| Get Order Lines | GET | `/v1/orders/{id}/lines` | — |
| Get Order Line | GET | `/v1/orders/{id}/lines/{lineId}` | — |
| Get Documents | GET | `/v1/orders/{id}/documents` | — |
| Download Document | GET | `/v1/orders/{id}/documents/{docId}/download` | — |

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
| Change Shipping | `POST /orders/{id}/shipping` | New through Completed |

---

## End-to-End Happy Path (ECommerce Example)

```
 Merchant (Public API)          System            Operator (Admin API)         WMS            Service Bus
 =====================     ===============     ========================     =========     =================
        |                        |                       |                     |                  |
        | POST /v1/orders        |                       |                     |                  |
        |----------------------->|                       |                     |                  |
        |<-- 201 (New) ---------|                       |                     |                  |
        |                        |                       |                     |                  |
        | POST .../lines         |                       |                     |                  |
        |----------------------->|                       |                     |                  |
        |<-- 201 (line added) --|                       |                     |                  |
        |                        |                       |                     |                  |
        | POST .../request       |                       |                     |                  |
        |----------------------->|                       |                     |                  |
        |<-- 202 (Requested) ---|                       |                     |                  |
        |                        |-- OrderRequested ---->|--------------------->|                  |
        |                        |                       |                     |                  |
        |                        |     [If auto-confirm enabled]               |                  |
        |                        |<-- ConfirmOrderRequest|                     |                  |
        |                        |                       |                     |                  |
        |                        |     [If manual confirm]                     |                  |
        |                        |                  Notification ------------->|                  |
        |                        |                       |                     |                  |
        |                        |  POST .../confirm     |                     |                  |
        |                        |<----------------------|                     |                  |
        |                        |                       |                     |                  |
        |                        |-- Status: Confirmed   |                     |                  |
        |                        |-- OrderConfirmed ---->|--------------------->| OrderConfirmed   |
        |                        |                       |                     |                  |
        |  Notification:         |                       |                     |                  |
        |  "Order confirmed"     |                       |                     |                  |
        |<-----------------------|                       |                     |                  |
        |                        |                       |                     |                  |
        |                        |-- SyncToWMS -------->|--------------------->|                  |
        |                        |                       |                     |                  |
        |                        |<---------- WMS sync success / label --------|                  |
        |                        |                       |                     |                  |
        |                        |  POST .../stock-out-ordered                 |                  |
        |                        |<----------------------|                     |                  |
        |                        |-- Status: WarehouseProcessing               |                  |
        |                        |                       |                     |                  |
        |                        |<---------- Shipped callback (quantities) ---|                  |
        |                        |-- Status: Shipped     |                     |                  |
        |                        |-- OrderShipped ------>|--------------------->| OrderShipped     |
        |                        |                       |                     |                  |
        |  Notification:         |                       |                     |                  |
        |  "Order shipped"       |                       |                     |                  |
        |<-----------------------|                       |                     |                  |
        |                        |                       |                     |                  |
        |                        |<-- Delivery confirm --|---------------------|                  |
        |                        |-- Status: Delivered   |                     |                  |
        |                        |                       |                     |                  |
        |                        |  POST .../complete    |                     |                  |
        |                        |<----------------------|                     |                  |
        |                        |-- Status: Completed   |                     |                  |
        |                        |-- OrderCompleted ---->|--------------------->| OrderCompleted   |
        |                        |                       |                     |                  |
        |  Notification:         |                       |                     |                  |
        |  "Order completed"     |                       |                     |                  |
        |<-----------------------|                       |                     |                  |
```

---

## Cancellation Flow

```
 Merchant                        System                    Service Bus
 ========                     ============              =================
    |                              |                          |
    | POST /v1/orders/{id}/cancel  |                          |
    | { cancelReason: "..." }      |                          |
    |----------------------------->|                          |
    |                              |                          |
    |   [If cancellable: Requested, Confirmed,                |
    |    Rejected, WarehouseProcessing]                       |
    |                              |                          |
    |<-- 202 (Canceled) ----------|                          |
    |                              |-- OrderCancelled ------->|
    |                              |                          |
    |   [If NOT cancellable: New, Shipped, Completed, ...]    |
    |                              |                          |
    |<-- 400 Bad Request ---------|                          |
```

---

## Order Types

The system supports three order types. The workflow is the same for all types; only the required fields differ:

| Field | ECommerce | FBA | Wholesale |
|-------|:---------:|:---:|:---------:|
| customerId | Required | — | Required |
| storeId | Required | — | — |
| shippingAddress | Required | — | Required |
| fbaWarehouseId | — | Required | — |
| shipmentType | — | Required | Required |
| hasRepalletization | — | Required | Required |
| shipmentVendor | Required | Required | Required |
