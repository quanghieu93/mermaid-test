# Global Sales — Public API Documentation

> **Base URL:** `https://api.example.com`
> **API Version:** 1.0
> **Authentication:** OAuth2 Bearer Token
> **Rate Limiting:** All endpoints are rate-limited

---

## Table of Contents

- [1. Authentication](#1-authentication)
- [2. Common Headers](#2-common-headers)
- [3. Common Success Response Format](#3-common-success-response-format)
- [4. Common Error Responses](#4-common-error-responses)
- [5. API Reference](#5-api-reference)
  - [5.1 Product Categories](#51-product-categories)
  - [5.2 Products](#52-products)
  - [5.3 Warehouses](#53-warehouses)
  - [5.4 Inventories](#54-inventories)
  - [5.5 Shipments](#55-shipments)
  - [5.6 Orders](#56-orders)
  - [5.7 Billings](#57-billings)
  - [5.8 Invoices](#58-invoices)
- [6. Webhooks / Event Notifications](#6-webhooks--event-notifications)
- [7. Complete Workflow](#7-complete-workflow)

---

## 1. Authentication

All endpoints require a valid **OAuth2 Bearer Token** issued by the **VantageId** identity provider using the **Resource Owner Password Credentials** grant.

### Get a Token

```
POST {vantageid_host}/connect/token
Content-Type: application/x-www-form-urlencoded
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| client_id | string | Yes | Your registered client ID |
| client_secret | string | Yes | Your client secret key |
| grant_type | string | Yes | Must be `password` |
| username | string | Yes | Your account email/username |
| password | string | Yes | Your account password |
| scope | string | Yes | Must be `global_sales_public_api` |

**Sample Request:**

```sh
curl -X POST https://id.example.com/connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "grant_type=password" \
  -d "username=YOUR_USERNAME" \
  -d "password=YOUR_PASSWORD" \
  -d "scope=global_sales_public_api"
```

**Sample Response (200 OK):**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "scope": "global_sales_public_api"
}
```

| Field | Type | Description |
|-------|------|-------------|
| access_token | string | JWT token for API requests |
| expires_in | int | Lifetime in seconds |
| token_type | string | Always `Bearer` |

> **Note:** Cache and reuse the token until it nears expiry. Do not request a new token for every API call.

---

## 2. Common Headers

| Header | Required | Applies To | Description |
|--------|----------|------------|-------------|
| Authorization | Yes | All endpoints | `Bearer <token>` |
| X-Cid | Yes | All endpoints | Your company identifier (UUID) |
| X-Idempotency-Key | Yes | POST create/import | Unique key (UUID) to prevent duplicates on retry |

---

## 3. Common Success Response Format

### Single Object (200 / 201 / 202)

```json
{
  "success": true,
  "data": { },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### Paginated List (200)

```json
{
  "success": true,
  "data": [],
  "currentPage": 1,
  "pageCount": 5,
  "pageSize": 20,
  "rowCount": 100,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### Empty Accepted (202)

```json
{
  "success": true,
  "data": {},
  "timestamp": "2026-03-24T10:00:00Z"
}
```

---

## 4. Common Error Responses

### 400 Bad Request — Business Logic Error

```json
{
  "success": false,
  "errorCode": "99",
  "error": "Order is not in the correct status to perform this action.",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### 401 Unauthorized
No body. Missing or invalid token.

### 403 Forbidden
No body. Authenticated but lacks required permission or scope.

### 404 Not Found

```json
{
  "success": false,
  "errorCode": "404",
  "error": "Resource not found.",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### 409 Conflict — Duplicate Idempotency Key
Same `X-Idempotency-Key` already processed. Returns cached response.

### 422 Unprocessable Entity — Validation Error

```json
{
  "success": false,
  "errorCode": "01",
  "error": [
    { "propertyName": "code", "messages": ["Code is required."] }
  ],
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### 429 Too Many Requests
Rate limit exceeded. Retry after `Retry-After` header.

### 500 Internal Server Error

```json
{
  "success": false,
  "errorCode": "100",
  "error": "An unexpected error occurred",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

---

## 5. API Reference

---

### 5.1 Product Categories

Categories are used to classify products. You need category IDs when creating products.

---

#### GET /v1/categories

Get a paginated list of categories.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Page number (1-based). Default: 1 |
| pageSize | int | No | Items per page. Default: 20, Max: 100 |
| name | string | No | Filter by category name |
| parentId | string (UUID) | No | Filter by parent category |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/categories?pageIndex=1&pageSize=10" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "a1b2c3d4-0000-0000-0000-000000000001",
      "name": "Electronics",
      "parentId": null
    },
    {
      "id": "a1b2c3d4-0000-0000-0000-000000000002",
      "name": "Accessories",
      "parentId": "a1b2c3d4-0000-0000-0000-000000000001"
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 10,
  "rowCount": 2,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### GET /v1/categories/{categoryId}

Get a category by ID.

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/categories/a1b2c3d4-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-0000-0000-0000-000000000001",
    "name": "Electronics",
    "parentId": null
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 404, 429

---

### 5.2 Products

Manage your product catalog. Product IDs are required when creating shipment packings and order lines.

**Enums:**

| Enum | Values |
|------|--------|
| ProductStatus | 1 = Active, 2 = Inactive |

---

#### GET /v1/products

Get a paginated list of products.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| code | string | No | Filter by product code |
| name | string | No | Filter by product name |
| query | string | No | Free-text search |
| categoryIds | UUID[] | No | Filter by categories |
| status | int | No | 1=Active, 2=Inactive |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/products?pageIndex=1&pageSize=10&status=1" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "code": "SKU-001",
      "name": "Wireless Mouse",
      "status": 1,
      "quantityPerPackage": 50,
      "grossWeight": 0.15,
      "length": 12.0,
      "width": 6.5,
      "height": 3.5
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 10,
  "rowCount": 1,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### GET /v1/products/{productId}

Get a product by ID.

**Sample Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "code": "SKU-001",
    "name": "Wireless Mouse",
    "asin": "B08N5WRWNW",
    "labelName": "Mouse Wireless",
    "hsCode": "8471.60",
    "status": 1,
    "netWeight": 0.10,
    "grossWeight": 0.15,
    "length": 12.0,
    "width": 6.5,
    "height": 3.5,
    "quantityPerPackage": 50,
    "minStock": 100,
    "containBattery": true,
    "declaredValue": "15.00"
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 404, 429

---

#### POST /v1/products

Create a new product.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| code | string | Yes | Unique product code/SKU |
| name | string | Yes | Product name |
| categoryIds | UUID[] | No | Category IDs |
| asin | string | No | Amazon ASIN |
| labelName | string | No | Label/display name |
| hsCode | string | No | Harmonized system code |
| productUnitId | UUID | Yes | Unit of measure ID |
| netWeight | decimal | No | Net weight |
| grossWeight | decimal | Yes | Gross weight |
| weightUnitId | UUID | Yes | Weight unit ID |
| length | decimal | Yes | Length |
| width | decimal | Yes | Width |
| height | decimal | Yes | Height |
| dimensionUnitId | UUID | Yes | Dimension unit ID |
| packageLength | decimal | Yes | Package length |
| packageWidth | decimal | Yes | Package width |
| packageHeight | decimal | Yes | Package height |
| packagingUnitId | UUID | Yes | Packaging unit ID |
| cubicUnitId | UUID | Yes | Cubic unit ID |
| packageNetWeight | decimal | No | Package net weight |
| packageGrossWeight | decimal | No | Package gross weight |
| packageWeightUnitId | UUID | Yes | Package weight unit ID |
| quantityPerPackage | int | Yes | Items per package |
| minStock | int | Yes | Minimum stock level |
| notes | string | No | Notes |
| status | int | Yes | 1=Active, 2=Inactive |
| containBattery | bool | Yes | Contains battery? |
| declaredValue | string | Yes | Declared customs value |

**Sample Request:**

```json
{
  "code": "SKU-001",
  "name": "Wireless Mouse",
  "categoryIds": ["a1b2c3d4-0000-0000-0000-000000000001"],
  "productUnitId": "00000000-0000-0000-0000-000000000001",
  "grossWeight": 0.15,
  "weightUnitId": "00000000-0000-0000-0000-000000000002",
  "length": 12.0,
  "width": 6.5,
  "height": 3.5,
  "dimensionUnitId": "00000000-0000-0000-0000-000000000003",
  "packageLength": 15.0,
  "packageWidth": 10.0,
  "packageHeight": 5.0,
  "packagingUnitId": "00000000-0000-0000-0000-000000000003",
  "cubicUnitId": "00000000-0000-0000-0000-000000000004",
  "packageWeightUnitId": "00000000-0000-0000-0000-000000000002",
  "quantityPerPackage": 50,
  "minStock": 100,
  "status": 1,
  "containBattery": true,
  "declaredValue": "15.00"
}
```

**Sample Response (201):** Returns the full product object (same as GET by ID).

**Errors:** 400, 401, 403, 422, 429

---

#### PUT /v1/products/{productId}

Full update of a product. Same body as POST.

**Sample Response (202):** Returns the updated product object.

**Errors:** 400, 401, 403, 404, 422, 429

---

#### DELETE /v1/products/{productId}

Delete a product.

**Sample Response:** 204 No Content (no body).

**Errors:** 400, 401, 403, 404, 429

---

### 5.3 Warehouses

Read-only. Warehouse IDs are required when creating shipments, orders, and checking inventory.

---

#### GET /v1/warehouses

Get a paginated list of warehouses.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| name | string | No | Filter by name |
| code | string | No | Filter by code |
| query | string | No | Free-text search |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/warehouses?pageIndex=1&pageSize=10" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "b2c3d4e5-1111-2222-3333-444444444444",
      "code": "WH-US-01",
      "name": "US Main Warehouse",
      "address": "123 Warehouse St, CA 90001"
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 10,
  "rowCount": 1,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### GET /v1/warehouses/all

Get all warehouses (no pagination).

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    { "id": "...", "code": "WH-US-01", "name": "US Main Warehouse" },
    { "id": "...", "code": "WH-US-02", "name": "US Secondary Warehouse" }
  ],
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

### 5.4 Inventories

Check product stock at warehouses.

---

#### GET /v1/inventories

Get a paginated list of inventories (stock summary per product per warehouse).

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| warehouseIds | UUID[] | No | Filter by warehouses. Pass multiple: `?warehouseIds=id1&warehouseIds=id2` |
| productId | UUID | No | Filter by product |
| showEmpty | bool | No | Include zero-stock. Default: false |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/inventories?warehouseIds=b2c3d4e5-1111-2222-3333-444444444444&pageSize=10" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "productId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "productCode": "SKU-001",
      "productName": "Wireless Mouse",
      "warehouseId": "b2c3d4e5-1111-2222-3333-444444444444",
      "warehouseName": "US Main Warehouse",
      "quantity": 500,
      "availableQuantity": 480,
      "reservedQuantity": 20
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 10,
  "rowCount": 1,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### GET /v1/inventories/products

Get inventory for a specific product at a specific warehouse.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| warehouseId | UUID | Yes | Warehouse ID |
| productId | UUID | Yes | Product ID |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/inventories/products?warehouseId=b2c3...&productId=f47a..." \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": {
    "productId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "productCode": "SKU-001",
    "productName": "Wireless Mouse",
    "warehouseId": "b2c3d4e5-1111-2222-3333-444444444444",
    "warehouseName": "US Main Warehouse",
    "quantity": 500,
    "availableQuantity": 480,
    "reservedQuantity": 20
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### GET /v1/inventories/details

Get lot-level inventory breakdown.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| warehouseIds | UUID[] | No | Filter by warehouses |
| productId | UUID | No | Filter by product |
| showEmptyLot | bool | No | Include empty lots. Default: false |
| fromDate | date | No | Filter lots from date |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/inventories/details?productId=f47a...&warehouseIds=b2c3..." \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "productId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "productCode": "SKU-001",
      "warehouseName": "US Main Warehouse",
      "lotNo": "LOT-2026-001",
      "expiredDate": "2027-12-31T00:00:00Z",
      "quantity": 200,
      "availableQuantity": 190,
      "reservedQuantity": 10
    }
  ],
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

### 5.5 Shipments

Inbound shipments — send goods to the warehouse.

**Enums:**

| Enum | Values |
|------|--------|
| ShipmentStatus | 1=New, 2=Confirmed, 3=Onway, 4=StockedIn, 5=Completed, 6=Review, 7=Rejected, 8=Canceled, 9=PartStockedIn |
| Service | 1=USWarehousing, 2=Fullfillment, 3=FBA |
| ShipmentPackingType | 0=None, 1=AllPallet, 2=AllCarton, 3=Mix |

**Available Endpoints (overview):**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /v1/shipments | List shipments (paginated) |
| GET | /v1/shipments/{shipmentId} | Get shipment detail |
| GET | /v1/shipments/count-status | Count by status |
| POST | /v1/shipments | **Create shipment** (combined: info + packings + documents) |
| PUT | /v1/shipments/{shipmentId} | Update shipment info |
| POST | /v1/shipments/{shipmentId}/request | Submit for review |
| POST | /v1/shipments/{shipmentId}/renew | Renew (from Rejected/Canceled) |
| POST | /v1/shipments/{shipmentId}/cancel | Cancel |
| DELETE | /v1/shipments/{shipmentId} | Delete |

---

#### GET /v1/shipments

Get a paginated list of shipments.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| warehouseId | UUID | No | Filter by warehouse |
| productId | UUID | No | Filter by product in packings |
| fromDate | date | No | Shipment date range start |
| toDate | date | No | Shipment date range end |
| status | int[] | No | Filter by status. Pass multiple: `?status=1&status=6` |
| clientRef | string | No | Filter by your reference |
| shipmentNo | string | No | Filter by shipment number |
| query | string | No | Free-text search |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/shipments?pageIndex=1&pageSize=10&status=1&status=6" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "c3d4e5f6-2222-3333-4444-555555555555",
      "shipmentNo": "SH-20260401-001",
      "clientRef": "SHIP-2026-001",
      "status": 1,
      "transportationMode": 1,
      "service": 1,
      "cargoReadyDate": "2026-04-01T00:00:00Z",
      "expectedToWH": "2026-04-15T00:00:00Z",
      "warehouseName": "US Main Warehouse",
      "shipmentPackingType": 1,
      "totalPallet": 5,
      "totalCarton": null,
      "totalLooseCarton": null,
      "enableGoodsCounting": false,
      "totalQuantity": 150,
      "totalActualQuantity": null
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 10,
  "rowCount": 1,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### GET /v1/shipments/{shipmentId}

Get full details of a shipment.

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/shipments/c3d4e5f6-2222-3333-4444-555555555555" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "c3d4e5f6-2222-3333-4444-555555555555",
    "shipmentNo": "SH-20260401-001",
    "clientRef": "SHIP-2026-001",
    "status": 1,
    "transportationMode": 1,
    "service": 1,
    "cargoReadyDate": "2026-04-01T00:00:00Z",
    "expectedToWH": "2026-04-15T00:00:00Z",
    "warehouseId": "b2c3d4e5-1111-2222-3333-444444444444",
    "remark": "Handle with care",
    "enableGoodsCounting": false,
    "shipmentPackingType": 1,
    "totalPallet": 5,
    "totalCarton": null,
    "totalLooseCarton": null,
    "warehouse": {
      "id": "b2c3d4e5-1111-2222-3333-444444444444",
      "name": "US Main Warehouse"
    },
    "shipmentPackingSummaries": [
      {
        "productCode": "SKU-001",
        "productName": "Wireless Mouse",
        "quantity": 100,
        "actualQuantity": null
      }
    ]
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 404, 429

---

#### GET /v1/shipments/count-status

Get shipment count grouped by status.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| warehouseId | UUID | No | Filter by warehouse |
| productId | UUID | No | Filter by product |
| fromDate | date | No | Date range start |
| toDate | date | No | Date range end |

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    { "id": 1, "count": 10 },
    { "id": 6, "count": 5 },
    { "id": 2, "count": 3 },
    { "id": 8, "count": 2 }
  ],
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### Combined Create — POST /v1/shipments

Create a complete shipment in a single call (shipment info + packing lines + documents).

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| clientRef | string | Yes | Your reference code |
| transportationMode | int | Yes | Transportation mode |
| service | int | Yes | 1=USWarehousing, 2=Fullfillment, 3=FBA |
| cargoReadyDate | datetime | Yes | Cargo ready date |
| expectedToWH | datetime | Yes | Expected warehouse arrival |
| warehouseId | UUID | No | Destination warehouse |
| remark | string | No | Free-text remark |
| enableGoodsCounting | bool | No | Default: false |
| shipmentPackingType | int | No | 0=None, 1=AllPallet, 2=AllCarton, 3=Mix |
| totalPallet | int | No | Total pallets |
| totalCarton | int | No | Total cartons |
| totalLooseCarton | int | No | Total loose cartons |
| packings | array | No | Packing lines (see below) |
| documents | array | No | Documents to attach (see below) |

**packings[] item:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productId | UUID | Yes | Product ID |
| lotNo | string | No | Lot/batch number |
| quantity | int | Yes | Quantity (> 0) |
| goodsValue | decimal | No | Declared value. Default: 0 |
| expiredDate | datetime | No | Expiry date |
| grossWeight | decimal | Yes | Gross weight |
| measurement | decimal | Yes | Volume/measurement |
| remark | string | No | Line remark |

**documents[] item:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| documentTypeId | UUID | Yes | Document type ID |
| remark | string | No | Document remark |
| files | File[] | Yes | Files to upload |

**Sample Request (JSON body, without files):**

```json
{
  "clientRef": "SHIP-2026-001",
  "transportationMode": 1,
  "service": 1,
  "cargoReadyDate": "2026-04-01T00:00:00Z",
  "expectedToWH": "2026-04-15T00:00:00Z",
  "warehouseId": "b2c3d4e5-1111-2222-3333-444444444444",
  "shipmentPackingType": 1,
  "totalPallet": 5,
  "packings": [
    {
      "productId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "lotNo": "LOT-2026-001",
      "quantity": 100,
      "goodsValue": 1500.00,
      "grossWeight": 250.5,
      "measurement": 12.3
    },
    {
      "productId": "f47ac10b-58cc-4372-a567-0e02b2c3d480",
      "quantity": 50,
      "grossWeight": 120.0,
      "measurement": 6.0
    }
  ]
}
```

**Sample Response (201):**

```json
{
  "success": true,
  "data": {
    "id": "c3d4e5f6-2222-3333-4444-555555555555",
    "shipmentNo": "SH-20260401-001",
    "clientRef": "SHIP-2026-001",
    "status": 1,
    "transportationMode": 1,
    "service": 1,
    "cargoReadyDate": "2026-04-01T00:00:00Z",
    "expectedToWH": "2026-04-15T00:00:00Z"
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 400, 401, 403, 409, 422, 429

---

#### PUT /v1/shipments/{shipmentId}

Update a shipment's information. Only allowed when status is **New**.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| clientRef | string | Yes | Your reference code |
| transportationMode | int | Yes | Transportation mode |
| service | int | Yes | 1=USWarehousing, 2=Fullfillment, 3=FBA |
| cargoReadyDate | datetime | Yes | Cargo ready date |
| expectedToWH | datetime | Yes | Expected warehouse arrival |
| warehouseId | UUID | No | Destination warehouse |
| remark | string | No | Remark |
| enableGoodsCounting | bool | No | Enable goods counting |
| shipmentPackingType | int | No | 0=None, 1=AllPallet, 2=AllCarton, 3=Mix |
| totalPallet | int | No | Total pallets |
| totalCarton | int | No | Total cartons |
| totalLooseCarton | int | No | Total loose cartons |

**Sample Request:**

```json
{
  "clientRef": "SHIP-2026-001-UPDATED",
  "transportationMode": 1,
  "service": 1,
  "cargoReadyDate": "2026-04-05T00:00:00Z",
  "expectedToWH": "2026-04-20T00:00:00Z",
  "warehouseId": "b2c3d4e5-1111-2222-3333-444444444444",
  "shipmentPackingType": 1,
  "totalPallet": 6
}
```

**Sample Response (200):** Returns updated shipment detail (same as GET by ID).

**Errors:** 400 (not in editable status), 401, 403, 404, 422, 429

---

#### POST /v1/shipments/{shipmentId}/request

Submit a shipment for review. Transitions **New → Review**.

**Request Body:** None

**Sample Request:**

```sh
curl -X POST "https://api.example.com/v1/shipments/c3d4e5f6-.../request" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response:** 202 Accepted (empty data).

**Errors:** 400 (must be in New status), 401, 403, 404, 429

---

#### POST /v1/shipments/{shipmentId}/renew

Renew a rejected or cancelled shipment back to **New** status.

**Request Body:** None

**Sample Response:** 202 Accepted (empty data).

**Errors:** 400 (must be in Rejected or Canceled status), 401, 403, 404, 429

---

#### POST /v1/shipments/{shipmentId}/cancel

Cancel a shipment.

**Request Body:** None

**Sample Response:** 202 Accepted (empty data).

**Errors:** 400 (must be in Review or Confirmed status), 401, 403, 404, 429

---

#### DELETE /v1/shipments/{shipmentId}

Soft-delete a shipment. Only allowed when status is **New**, **Canceled**, or **Rejected**.

**Sample Response:** 204 No Content (no body).

**Errors:** 400 (not in deletable status), 401, 403, 404, 429

---

### 5.6 Orders

Outbound orders — fulfill customer orders from warehouse stock.

**Enums:**

| Enum | Values |
|------|--------|
| OrderType | 1=ECommerce, 3=FBA, 4=Wholesale |
| OrderStatus | 1=New, 2=Requested, 3=Rejected, 4=Confirmed, 5=WarehouseProcessing, 7=Shipped, 8=Completed, 9=Abnormal, 10=Delivered, 100=Holded, 255=Canceled |
| ShipmentVendorType | 1=SelfPickup, 2=Warehouse, 3=ThirdParty |
| ShipmentType | 1=Express, 2=TruckFreight |
| CustomerAddressType | 0=Commercial, 1=Residential |

**Available Endpoints (overview):**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /v1/orders | List orders (paginated) |
| GET | /v1/orders/{orderId} | Get order detail |
| GET | /v1/orders/count-status | Count by status |
| POST | /v1/orders | **Create order** (combined: info + lines + documents) |
| PUT | /v1/orders/{orderId} | Update order info |
| POST | /v1/orders/{orderId}/request | Submit for processing |
| POST | /v1/orders/request-batch | Batch submit |
| POST | /v1/orders/{orderId}/cancel | Cancel |
| DELETE | /v1/orders/{orderId} | Delete |

---

#### GET /v1/orders

Get a paginated list of orders.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| type | int | No | OrderType: 1=ECommerce, 3=FBA, 4=Wholesale |
| warehouseId | UUID | No | Filter by warehouse |
| productId | UUID | No | Filter by product in lines |
| fromDate | date | No | Order date range start |
| toDate | date | No | Order date range end |
| status | int[] | No | Filter by status. Pass multiple: `?status=1&status=2` |
| clientReference | string | No | Filter by your reference |
| orderNo | string | No | Filter by order number |
| shipmentVendor | int | No | 1=SelfPickup, 2=Warehouse, 3=ThirdParty |
| query | string | No | Free-text search |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/orders?pageIndex=1&pageSize=10&type=1&status=1" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "e6f7a8b9-5555-6666-7777-888888888888",
      "orderNo": "ORD-20260401-001",
      "orderType": 1,
      "status": 1,
      "clientReference": "ORD-2026-001",
      "orderDate": "2026-04-01T00:00:00Z",
      "dueDate": "2026-04-10T00:00:00Z",
      "warehouseName": "US Main Warehouse",
      "customerName": "John Doe",
      "storeName": "My Shopify Store",
      "trackingNo": null,
      "shipmentVendor": 2
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 10,
  "rowCount": 1,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### GET /v1/orders/{orderId}

Get full details of an order.

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/orders/e6f7a8b9-5555-6666-7777-888888888888" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "e6f7a8b9-5555-6666-7777-888888888888",
    "orderNo": "ORD-20260401-001",
    "orderType": 1,
    "status": 1,
    "clientReference": "ORD-2026-001",
    "orderDate": "2026-04-01T00:00:00Z",
    "dueDate": "2026-04-10T00:00:00Z",
    "remark": "Rush order",
    "warehouseId": "b2c3d4e5-1111-2222-3333-444444444444",
    "warehouseName": "US Main Warehouse",
    "customerId": "d4e5f6a7-3333-4444-5555-666666666666",
    "customerName": "John Doe",
    "storeId": "e5f6a7b8-4444-5555-6666-777777777777",
    "storeName": "My Shopify Store",
    "shipmentVendor": 2,
    "trackingNo": null,
    "shippingAddress": {
      "fullName": "John Doe",
      "phoneNumber": "+1-555-0100",
      "addressLine": "123 Main St, Apt 4B",
      "postalCode": "90001",
      "customerAddressType": 1
    },
    "orderLines": [
      {
        "id": "11111111-aaaa-bbbb-cccc-dddddddddddd",
        "productId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
        "productCode": "SKU-001",
        "productName": "Wireless Mouse",
        "quantity": 10,
        "sellingPrice": 25.00,
        "expectedPrice": 15.00
      }
    ]
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 404, 429

---

#### GET /v1/orders/count-status

Get order count grouped by status.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| type | int | No | OrderType filter |
| warehouseId | UUID | No | Filter by warehouse |
| fromDate | date | No | Date range start |
| toDate | date | No | Date range end |

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    { "id": 1, "count": 12 },
    { "id": 2, "count": 5 },
    { "id": 4, "count": 8 },
    { "id": 7, "count": 20 },
    { "id": 255, "count": 3 }
  ],
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### Combined Create — POST /v1/orders

Create a complete order in a single call (order info + lines + documents). The `status` field controls whether the order is created as **New** (editable) or immediately submitted as **Requested** (for processing).

**Request Body (common fields):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderType | int | Yes | 1=ECommerce, 3=FBA, 4=Wholesale |
| status | int | Yes | **1=New** (draft, editable) or **2=Requested** (submitted immediately) |
| warehouseId | UUID | No | Warehouse to fulfill from |
| clientReference | string | No | Your reference code |
| remark | string | No | Free-text remark |
| orderDate | datetime | No | Order date |
| dueDate | datetime | No | Due date |
| salesmanId | UUID | No | Salesman ID |
| shippingCarrierId | UUID | No | Carrier ID |
| carrierServiceId | UUID | No | Carrier service ID |
| trackingNo | string | No | Tracking number |
| shipmentVendor | int | Yes | 1=SelfPickup, 2=Warehouse, 3=ThirdParty |
| lines | array | No | Order lines (see below) |
| documents | array | No | Documents to attach (see below) |

**Additional fields by OrderType:**

| Field | Type | ECommerce | FBA | Wholesale |
|-------|------|:---------:|:---:|:---------:|
| customerId | UUID | Required | — | Required |
| storeId | UUID | Required | — | — |
| shippingAddress | object | Required | — | Required |
| fbaWarehouseId | UUID | — | Required | — |
| shipmentType | int | — | Required | Required |
| hasRepalletization | bool | — | Required | Required |
| palletQuantity | int | — | No | No |
| pickupTime | datetime | — | No | No |
| pickupNo | string | — | No | No |

**shippingAddress object:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| fullName | string | Yes | Recipient name |
| phoneNumber | string | Yes | Phone number |
| adminUnitId | int | Yes | Administrative unit ID |
| addressLine | string | Yes | Street address |
| postalCode | string | Yes | Postal/ZIP code |
| note | string | No | Delivery note |
| customerAddressType | int | No | 0=Commercial, 1=Residential |

**lines[] item:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productId | UUID | Yes | Product ID |
| lotNo | string | No | Lot/batch number |
| quantity | int | Yes | Quantity (> 0) |
| sellingPrice | decimal | Yes | Selling price per unit |
| expectedPrice | decimal | Yes | Expected/cost price |
| remark | string | No | Line remark |

**documents[] item:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| documentTypeId | UUID | Yes | Document type ID |
| remark | string | No | Document remark |
| files | File[] | Yes | Files to upload |

**Sample Request (ECommerce, status=New):**

```json
{
  "orderType": 1,
  "status": 1,
  "customerId": "d4e5f6a7-3333-4444-5555-666666666666",
  "storeId": "e5f6a7b8-4444-5555-6666-777777777777",
  "warehouseId": "b2c3d4e5-1111-2222-3333-444444444444",
  "clientReference": "ORD-2026-001",
  "shipmentVendor": 2,
  "shippingAddress": {
    "fullName": "John Doe",
    "phoneNumber": "+1-555-0100",
    "adminUnitId": 12345,
    "addressLine": "123 Main St, Apt 4B",
    "postalCode": "90001",
    "customerAddressType": 1
  },
  "lines": [
    {
      "productId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "quantity": 10,
      "sellingPrice": 25.00,
      "expectedPrice": 15.00
    },
    {
      "productId": "f47ac10b-58cc-4372-a567-0e02b2c3d480",
      "quantity": 5,
      "sellingPrice": 50.00,
      "expectedPrice": 30.00
    }
  ]
}
```

**Sample Request (ECommerce, status=Requested — auto-submit):**

```json
{
  "orderType": 1,
  "status": 2,
  "customerId": "d4e5f6a7-3333-4444-5555-666666666666",
  "storeId": "e5f6a7b8-4444-5555-6666-777777777777",
  "shipmentVendor": 2,
  "shippingAddress": { "..." : "..." },
  "lines": [
    { "productId": "...", "quantity": 10, "sellingPrice": 25.00, "expectedPrice": 15.00 }
  ]
}
```

**Sample Response (201):**

```json
{
  "success": true,
  "data": {
    "id": "e6f7a8b9-5555-6666-7777-888888888888",
    "orderNo": "ORD-20260401-001",
    "orderType": 1,
    "status": 1,
    "clientReference": "ORD-2026-001",
    "orderDate": "2026-04-01T00:00:00Z"
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 400, 401, 403, 409, 422, 429

---

#### PUT /v1/orders/{orderId}

Update an order's information. Only allowed when status is **New**.

**Request Body:** Same fields as POST /v1/orders (except `status` and `lines`/`documents` are not used here — use this endpoint only to change order header info).

**Sample Request:**

```json
{
  "orderType": 1,
  "customerId": "d4e5f6a7-3333-4444-5555-666666666666",
  "storeId": "e5f6a7b8-4444-5555-6666-777777777777",
  "clientReference": "ORD-2026-001-UPDATED",
  "shipmentVendor": 2,
  "shippingAddress": {
    "fullName": "John Doe",
    "phoneNumber": "+1-555-0100",
    "adminUnitId": 12345,
    "addressLine": "456 Updated St",
    "postalCode": "90002",
    "customerAddressType": 1
  }
}
```

**Sample Response (200):** Empty success.

**Errors:** 400 (not in editable status / invalid order type), 401, 403, 404, 422, 429

---

#### POST /v1/orders/{orderId}/request

Submit an order for processing. Transitions **New → Requested**.

**Request Body:** None

**Sample Request:**

```sh
curl -X POST "https://api.example.com/v1/orders/e6f7a8b9-.../request" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response:** 202 Accepted (empty data).

**Errors:** 400 (must be in New status), 401, 403, 404, 429

---

#### POST /v1/orders/request-batch

Submit multiple orders for processing in batch.

**Request Body:** Array of order IDs.

```json
[
  "e6f7a8b9-5555-6666-7777-888888888888",
  "f7a8b9c0-6666-7777-8888-999999999999"
]
```

**Sample Response (202):** Returns results per order (success/failure for each).

```json
{
  "success": true,
  "data": [
    { "orderId": "e6f7a8b9-...", "success": true },
    { "orderId": "f7a8b9c0-...", "success": false, "error": "Order is not in New status" }
  ],
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### POST /v1/orders/{orderId}/cancel

Cancel an order.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| cancelReason | string | Yes | Reason for cancellation |

**Sample Request:**

```json
{
  "cancelReason": "Customer changed their mind"
}
```

**Sample Response:** 202 Accepted (empty data).

**Errors:** 400 (must be in Requested, Confirmed, Rejected, or WarehouseProcessing status), 401, 403, 404, 429

---

#### DELETE /v1/orders/{orderId}

Soft-delete an order. Only allowed when status is **New** or **Canceled**.

**Sample Response:** 204 No Content (no body).

**Errors:** 400 (not in deletable status), 401, 403, 404, 429

---

### 5.7 Billings

Read-only. View billing records linked to your orders and shipments.

**Enums:**

| Enum | Values |
|------|--------|
| BillingStatus | 0=New, 1=Pending, 2=Invoiced, 3=Cancelled, 4=Completed |
| BillingDocumentType | 1=Order, 2=Shipment, 3=Warehouse, 4=StockTransaction |

---

#### GET /v1/billings

Get a paginated list of billings.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| billingNo | string | No | Filter by billing number |
| fromDate | date | No | Billing date range start |
| toDate | date | No | Billing date range end |
| status | int | No | BillingStatus value |
| documentType | int | No | 1=Order, 2=Shipment, etc. |
| invoiceNo | string | No | Filter by invoice number |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/billings?pageIndex=1&pageSize=10&status=1" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "a1b2c3d4-aaaa-bbbb-cccc-dddddddddddd",
      "billingNo": "BIL-20260401-001",
      "status": 1,
      "documentType": 1,
      "totalAmount": 350.00,
      "invoiceNo": null,
      "billingDate": "2026-04-01T00:00:00Z"
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 10,
  "rowCount": 1,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### GET /v1/billings/{billingId}

Get billing detail by ID.

**Sample Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-aaaa-bbbb-cccc-dddddddddddd",
    "billingNo": "BIL-20260401-001",
    "status": 1,
    "documentType": 1,
    "totalAmount": 350.00,
    "billingDate": "2026-04-01T00:00:00Z",
    "lines": [
      { "description": "Fulfillment fee", "amount": 250.00 },
      { "description": "Shipping fee", "amount": 100.00 }
    ]
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 404, 429

---

#### GET /v1/billings/find-by-document

Find billing by linked document (order or shipment).

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| type | int | Yes | 1=Order, 2=Shipment |
| id | UUID | Yes | The order or shipment ID |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/billings/find-by-document?type=1&id=e6f7a8b9-5555-6666-7777-888888888888" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):** Same as GET by ID.

**Response (204):** No billing found for the document.

**Errors:** 401, 403, 429

---

### 5.8 Invoices

Read-only. View invoices for your billing records.

**Enums:**

| Enum | Values |
|------|--------|
| InvoiceStatus | 0=Draft, 1=PendingApproval, 2=Approved, 3=Sent, 4=Canceled, 5=Refunded, 6=Failed, 7=Close |
| PaymentStatus | 0=Unpaid, 1=Pending, 2=Paid, 3=Refunded, 5=Overdue, 6=PartiallyPaid |

---

#### GET /v1/invoices

Get a paginated list of invoices.

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| invoiceNo | string | No | Filter by invoice number |
| fromDate | date | No | Date range start |
| toDate | date | No | Date range end |
| status | int | No | InvoiceStatus value |
| paymentStatus | int | No | PaymentStatus value |

**Sample Request:**

```sh
curl -X GET "https://api.example.com/v1/invoices?pageIndex=1&pageSize=10" \
  -H "Authorization: Bearer {token}" \
  -H "X-Cid: {companyId}"
```

**Sample Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "b2c3d4e5-bbbb-cccc-dddd-eeeeeeeeeeee",
      "invoiceNo": "INV-20260401-001",
      "status": 3,
      "paymentStatus": 0,
      "totalAmount": 350.00,
      "invoiceDate": "2026-04-05T00:00:00Z",
      "dueDate": "2026-05-05T00:00:00Z"
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 10,
  "rowCount": 1,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 429

---

#### GET /v1/invoices/{invoiceId}

Get invoice detail by ID.

**Sample Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "b2c3d4e5-bbbb-cccc-dddd-eeeeeeeeeeee",
    "invoiceNo": "INV-20260401-001",
    "status": 3,
    "paymentStatus": 0,
    "totalAmount": 350.00,
    "invoiceDate": "2026-04-05T00:00:00Z",
    "dueDate": "2026-05-05T00:00:00Z",
    "billings": [
      { "billingNo": "BIL-20260401-001", "amount": 350.00 }
    ]
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Errors:** 401, 403, 404, 429

---

## 6. Webhooks / Event Notifications

When key status changes occur, the system sends webhook notifications to your registered URL. This replaces the need to poll the API for updates.

### Webhook Delivery

- **Method:** POST to your registered URL
- **Content-Type:** application/json
- **Expected response:** 200 OK within 5 seconds
- **Retry:** Up to 3 retries with exponential backoff on failure

### Event Types

| Event | When It Fires | Key Payload Fields |
|-------|--------------|-------------------|
| `order.confirmed` | Order approved by operator | orderId, orderNo, status |
| `order.rejected` | Order rejected by operator | orderId, orderNo, status, reason |
| `order.shipped` | Order shipped from warehouse | orderId, orderNo, status, trackingNo |
| `order.completed` | Order finalized | orderId, orderNo, status |
| `order.cancelled` | Order cancelled | orderId, orderNo, status, reason |
| `order.abnormal` | Processing error occurred | orderId, orderNo, status, reason |
| `shipment.confirmed` | Shipment approved | shipmentId, shipmentNo, status |
| `shipment.rejected` | Shipment rejected | shipmentId, shipmentNo, status, reason |
| `shipment.stocked_in` | Goods received at warehouse | shipmentId, shipmentNo, status |
| `shipment.completed` | Shipment finalized | shipmentId, shipmentNo, status |

### Sample Webhook Payload

```json
{
  "event": "order.shipped",
  "timestamp": "2026-04-10T14:30:00Z",
  "data": {
    "orderId": "e6f7a8b9-5555-6666-7777-888888888888",
    "orderNo": "ORD-20260401-001",
    "status": 7,
    "trackingNo": "1Z999AA10123456784"
  }
}
```

---

## 7. Complete Workflow

This is the typical end-to-end flow for a merchant using the Public API.

### Step 1 — Initial Setup (one-time)

```
1. Get token            POST {vantageid}/connect/token
2. List categories      GET  /v1/categories           → get category IDs
3. Create products      POST /v1/products             → register your SKUs
4. List warehouses      GET  /v1/warehouses           → get warehouse IDs
```

### Step 2 — Inbound (send stock to warehouse)

```
1. Check inventory      GET  /v1/inventories          → see current stock
2. Create shipment      POST /v1/shipments            → single call with packings
3. Submit for review    POST /v1/shipments/{id}/request
4. Wait for webhook     → "shipment.confirmed" / "shipment.rejected"
5. Track progress       GET  /v1/shipments/{id}       → check status
6. Webhook arrives      → "shipment.stocked_in" (goods received)
7. Webhook arrives      → "shipment.completed" (finalized)
```

### Step 3 — Outbound (fulfill customer orders)

```
1. Check inventory      GET  /v1/inventories          → verify stock
2. Create order         POST /v1/orders               → single call with lines
                         ├── status=1 (New) → edit later, then submit
                         └── status=2 (Requested) → auto-submit immediately
3. (If New) Submit      POST /v1/orders/{id}/request
4. Wait for webhook     → "order.confirmed" / "order.rejected"
5. Webhook arrives      → "order.shipped" (with tracking number)
6. Webhook arrives      → "order.completed" (finalized)
```

### Step 4 — Billing & Payment

```
1. Check billings       GET  /v1/billings             → view charges
2. Find by order        GET  /v1/billings/find-by-document?type=1&id={orderId}
3. View invoices        GET  /v1/invoices             → payment overview
4. Invoice detail       GET  /v1/invoices/{id}        → breakdown
```

### Visual Flow

```
  [Categories] ──→ [Products] ──→ [Warehouses]
       │                │               │
       │                ▼               │
       │         ┌─────────────┐        │
       │         │ Inventories │←───────┘
       │         └──────┬──────┘
       │                │
       ├────────────────┼────────────────┐
       │                │                │
       ▼                ▼                ▼
  ┌─────────┐    ┌───────────┐    ┌───────────┐
  │Shipments│    │  Orders   │    │           │
  │(Inbound)│    │(Outbound) │    │           │
  └────┬────┘    └─────┬─────┘    │           │
       │               │          │           │
       │    Webhooks   │          │           │
       │  ◄─────────►  │          │           │
       │               │          │           │
       ▼               ▼          │           │
  ┌─────────────────────────┐     │           │
  │       Billings          │─────┘           │
  └────────────┬────────────┘                 │
               │                              │
               ▼                              │
  ┌─────────────────────────┐                 │
  │       Invoices          │─────────────────┘
  └─────────────────────────┘
```
