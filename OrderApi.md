[PublicApi-OrdersController-Documentation.md](https://github.com/user-attachments/files/26318230/PublicApi-OrdersController-Documentation.md)
# Order Apis Documentation

> Generated: March 24, 2026
> Base Route: /v{version:apiVersion}/orders
> API Version: 1.0
> Authentication: OAuth2 Bearer Token (required on all endpoints)
> Authorization: Client credentials with scope "public-api" + resource permissions
> Rate Limiting: All endpoints are rate-limited (read/write/import policies)
> Company Scope: The caller's company is resolved from the authenticated token.

---

## Design Principles

This API is designed for **external systems** (merchants, third-party platforms) to interact with our order management system. The following principles guide what is exposed:

1. **`orderId` is returned in responses** — External systems need this identifier to perform subsequent operations (update, cancel, add lines, etc.), just as we store `WMSReference` from warehouse systems like Soho/SevenOceans. This is your reference to the order in our system.
2. **Company context is implicit** — Company identity is derived from the authentication token. You cannot see or access another company's data.
3. **Internal operational data is not exposed** — Fields like WMS sync status, ERP sync info, billing/payment status, source type, and internal user IDs are omitted.
4. **Internal entity IDs for related resources are not exposed in responses** — You won't see raw `companyId`, `warehouseId`, or `storeId` values in GET responses. Warehouses, carriers, and stores are represented by their code/name only.
5. **Input fields use IDs when you need to reference existing entities** — When creating or updating orders, you pass `warehouseId`, `customerId`, `storeId`, etc. as input because you need to tell us which entity to use. These IDs are provided to you via separate lookup endpoints (e.g., GET /v1/warehouses, GET /v1/customers).

---

## Enum Reference

### OrderType
- 1 = ECommerce
- 3 = FBA
- 4 = Wholesale

### OrderStatus
- 1 = New
- 2 = Requested
- 3 = Rejected
- 4 = Confirmed
- 5 = WarehouseProcessing
- 7 = Shipped
- 8 = Completed
- 9 = Abnormal
- 10 = Delivered
- 100 = Holded
- 255 = Canceled

### ShipmentVendorType
- 1 = SelfPickup
- 2 = Warehouse
- 3 = ThirdParty

### ShipmentType
- 1 = Express
- 2 = TruckFreight

### CustomerAddressType
- 0 = Commercial
- 1 = Residential

---

## Common Success Response Format

All successful responses are wrapped in the same envelope. This is documented once here; each endpoint section will only describe the fields inside "data".

### Single Object Response (HTTP 200 / 201 / 202)

```json
{
  "success": true,
  "data": { },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| success | boolean | Always `true` for successful responses |
| data | object or null | The response payload. Contents vary per endpoint. |
| timestamp | string (ISO 8601) | Server UTC time when the response was generated |

### Paginated Response (HTTP 200)

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

| Field | Type | Description |
|-------|------|-------------|
| success | boolean | Always `true` |
| data | array | List of items for the current page |
| currentPage | int | Current page number (1-based) |
| pageCount | int | Total number of pages |
| pageSize | int | Number of items per page |
| rowCount | int | Total number of items across all pages |
| timestamp | string (ISO 8601) | Server UTC time |

### Empty Accepted Response (HTTP 202)

Used by state-change endpoints that return no payload.

```json
{
  "success": true,
  "data": {},
  "timestamp": "2026-03-24T10:00:00Z"
}
```

---

## Common Error Responses

### 401 Unauthorized
No body. Missing or invalid token.

### 403 Forbidden
No body. Authenticated but lacks required permission or scope.

### 429 Too Many Requests
Rate limit exceeded. Retry after the duration specified in the `Retry-After` header.

### 409 Conflict — Duplicate Request (Idempotency)
Returned when a request with the same `X-Idempotency-Key` has already been processed. The server returns the cached response from the original request.

### 422 Unprocessable Entity — Validation Error (errorCode: 01 or 98)

```json
{
  "success": false,
  "errorCode": "01",
  "error": [
    {
      "propertyName": "customerId",
      "messages": [
        "CustomerId is required for ECommerce orders."
      ]
    }
  ],
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### 400 Bad Request — Domain/Business Logic Error (errorCode: 99)

```json
{
  "success": false,
  "errorCode": "99",
  "error": "Order is not in the correct status to perform this action.",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### 404 Not Found (errorCode: 404)

```json
{
  "success": false,
  "errorCode": "404",
  "error": "Order not found.",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### 422 Unprocessable Entity — Database Constraint (application errorCode: 201-299)
Note: The HTTP status is always 422. The "errorCode" is a custom application code, NOT an HTTP status.

```json
{
  "success": false,
  "errorCode": "201",
  "error": "The provided data already exists.",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### 500 Internal Server Error (errorCode: 100)

```json
{
  "success": false,
  "errorCode": "100",
  "error": "An unexpected error occurred",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

---

## Authentication

All API endpoints require a valid **OAuth2 Bearer Token**. Tokens are issued by the **VantageId** identity provider using the **Resource Owner Password Credentials** grant type.

### Step 1 — Obtain a Token

**Request:**

```
POST {vantageid_host}/connect/token
Content-Type: application/x-www-form-urlencoded
```

**Form Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| client_id | Yes | Your registered client ID (provided by the platform administrator) |
| grant_type | Yes | Must be `password` |
| username | Yes | Your user account email/username |
| password | Yes | Your user account password |
| scope | Yes | Must include `global_sales_public_api` |

**Sample Request (cURL):**

```sh
curl -X POST https://id.example.com/connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
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
| access_token | string | The JWT token to use in API requests |
| expires_in | int | Token lifetime in seconds (typically 3600 = 1 hour) |
| token_type | string | Always `Bearer` |
| scope | string | The granted scope(s) |

### Step 2 — Call the API

Include the token in the `Authorization` header and your company ID in the `X-Cid` header on every request:

```
GET /v1/orders
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
X-Cid: 3fa85f64-5717-4562-b3fc-2c963f66afa6
```

**Sample Request (cURL):**

```sh
curl -X GET https://api.example.com/v1/orders \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIs..." \
  -H "X-Cid: 3fa85f64-5717-4562-b3fc-2c963f66afa6"
```

### Token Validation

The API validates the following claims in the JWT token:

| Claim | Validation |
|-------|-----------|
| `iss` (Issuer) | Must match the configured VantageId authority URL |
| `aud` (Audience) | Must match `global_sales_public_api` |
| `exp` (Expiry) | Token must not be expired (30-second clock skew allowed) |
| `client_id` | Must match the registered client ID for this API |
| `scope` | Must contain `global_sales_public_api` |
| `sub` / `NameIdentifier` | Used to identify the user and check resource permissions |

### Error Responses

| Scenario | HTTP Status | Description |
|----------|-------------|-------------|
| Missing or malformed token | 401 Unauthorized | No body. Include a valid `Authorization: Bearer <token>` header. |
| Token expired | 401 Unauthorized | No body. Request a new token. |
| Invalid scope or client_id | 403 Forbidden | No body. Token does not have the required scope or client. |
| User lacks permission | 403 Forbidden | No body. The authenticated user does not have access to the requested resource/action. |
| Missing `X-Cid` header | 400 Bad Request | Company ID is required. |

### Notes

- **Token caching:** Tokens are valid for the duration specified in `expires_in`. Cache and reuse the token until it nears expiry, then request a new one. Do not request a new token for every API call.

---

## Special Headers

| Header | Required | Applies To | Description |
|--------|----------|------------|-------------|
| Authorization | Yes | All endpoints | `Bearer <token>` — obtained from the token endpoint (see Authentication section above) |
| X-Cid | Yes | All endpoints | Your company identifier (UUID). Scopes all data to your company. |

---

## API Endpoints — Orders

---

### #1 — GET /v1/orders

Get a paginated list of orders.

- Permission: Orders.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| pageIndex | int | No | Page number (1-based). Default: 1 |
| pageSize | int | No | Items per page. Default: 20, Max: 100 |
| type | int | No | Filter by OrderType (1=ECommerce, 3=FBA, 4=Wholesale) |
| warehouseId | string (UUID) | No | Filter by warehouse |
| productId | string (UUID) | No | Filter orders containing this product |
| fromDate | string (date) | No | Order date range start (inclusive). Format: YYYY-MM-DD |
| toDate | string (date) | No | Order date range end (inclusive). Format: YYYY-MM-DD |
| status | int[] | No | Filter by one or more OrderStatus values. Pass multiple: `?status=1&status=4` |
| clientReference | string | No | Filter by your reference code |
| orderNo | string | No | Filter by system order number |
| query | string | No | Free-text search across order number, client reference, customer name |
| shipmentVendor | int | No | Filter by ShipmentVendorType (1, 2, or 3) |

**Request Body:** None

**Response — 200 OK (Paginated):**

data is an array of order summaries:

| Field | Type | Description |
|-------|------|-------------|
| orderId | string (UUID) | **Your reference to this order in our system.** Use this for all subsequent API calls. |
| orderNo | string | System-generated human-readable order number |
| orderDate | string (ISO 8601) | Date the order was placed |
| dueDate | string (ISO 8601) or null | Expected due date |
| orderType | int | OrderType value (1, 3, or 4) |
| warehouseName | string | Name of the assigned warehouse |
| clientReference | string or null | Your own reference code (the value you provided when creating) |
| customerName | string | Customer full name |
| shipTo | string | Shipping address (formatted string) |
| carrierCode | string or null | Shipping carrier code (e.g., "FEDEX") |
| carrierServiceCode | string or null | Carrier service code (e.g., "GROUND") |
| shipmentVendor | int | ShipmentVendorType value |
| trackingNo | string or null | Tracking number |
| stockedOutDate | string (ISO 8601) or null | Date stock was shipped out |
| status | int | OrderStatus value |
| totalQuantity | int | Total item quantity |
| totalPackageQuantity | int | Total package count |

**Sample:**

```json
{
  "success": true,
  "data": [
    {
      "orderId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "orderNo": "ORD-00001",
      "orderDate": "2026-03-20T00:00:00Z",
      "dueDate": "2026-04-01T00:00:00Z",
      "orderType": 1,
      "warehouseName": "Warehouse A",
      "clientReference": "MY-REF-001",
      "customerName": "John Doe",
      "shipTo": "123 Main St, CA 90210",
      "carrierCode": "FEDEX",
      "carrierServiceCode": "GROUND",
      "shipmentVendor": 2,
      "trackingNo": "TRACK123",
      "stockedOutDate": null,
      "status": 1,
      "totalQuantity": 50,
      "totalPackageQuantity": 5
    }
  ],
  "currentPage": 1,
  "pageCount": 5,
  "pageSize": 20,
  "rowCount": 100,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 429

---

### #2 — GET /v1/orders/count-status

Get order count grouped by status.

- Permission: Orders.Read
- Rate Limit: Read policy

**Query Parameters:** Same filters as GET /v1/orders (excluding pageIndex, pageSize).

**Request Body:** None

**Response — 200 OK:**

data is an array of status counts. Always returns all statuses, even those with 0 count:

| Field | Type | Description |
|-------|------|-------------|
| status | int | OrderStatus value |
| count | int | Number of orders in that status |

**Sample:**

```json
{
  "success": true,
  "data": [
    { "status": 1, "count": 25 },
    { "status": 2, "count": 10 },
    { "status": 3, "count": 3 },
    { "status": 4, "count": 12 },
    { "status": 5, "count": 8 },
    { "status": 7, "count": 15 },
    { "status": 8, "count": 50 },
    { "status": 9, "count": 2 },
    { "status": 10, "count": 5 },
    { "status": 100, "count": 0 },
    { "status": 255, "count": 7 }
  ],
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 429

---

### #3 — GET /v1/orders/{orderId}

Get full details of an order.

- Permission: Orders.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier (returned when you created the order) |

**Request Body:** None

**Response — 200 OK:**

data is an order detail object:

| Field | Type | Description |
|-------|------|-------------|
| orderId | string (UUID) | Your reference to this order |
| orderNo | string | System-generated human-readable order number |
| orderType | int | OrderType value |
| orderDate | string (ISO 8601) | Date the order was placed |
| clientReference | string or null | Your own reference code |
| trackingNo | string or null | Tracking number |
| dueDate | string (ISO 8601) or null | Expected due date |
| stockedOutDate | string (ISO 8601) or null | Date stock was shipped out |
| actualDate | string (ISO 8601) or null | Actual completion/delivery date |
| status | int | OrderStatus value |
| shipmentVendor | int | ShipmentVendorType value |
| shipmentType | int | ShipmentType value |
| palletQuantity | int or null | Number of pallets (FBA/Wholesale only) |
| hasRepalletization | boolean or null | Whether repalletization is needed (FBA/Wholesale only) |
| pickupNo | string or null | Pickup reference number |
| pickupTime | string (ISO 8601) or null | Scheduled pickup time |
| shippingCarrier | object or null | Carrier information (see below) |
| warehouse | object or null | Warehouse information (see below) |
| totalAmount | number | Total monetary amount |
| totalQuantity | int | Total item quantity |
| totalPackageQuantity | int | Total package count |
| totalWeight | number | Total weight |
| shippingAddress | object | Shipping destination (see below) |
| remark | string or null | Order remark |
| cancelReason | string or null | Cancellation reason (only populated if status = 255) |

**shippingCarrier (nested object, null if no carrier assigned):**

| Field | Type | Description |
|-------|------|-------------|
| code | string | Carrier code (e.g., "FEDEX") |
| name | string | Carrier display name |

**warehouse (nested object, null if no warehouse assigned):**

| Field | Type | Description |
|-------|------|-------------|
| code | string | Warehouse code |
| name | string | Warehouse name |

**shippingAddress (nested object):**

| Field | Type | Description |
|-------|------|-------------|
| fullName | string | Recipient full name |
| phoneNumber | string | Recipient phone number |
| adminUnitName | string | State/province name |
| adminUnitPathName | string | Full path (e.g., "US > California") |
| addressLine | string | Street address |
| postalCode | string | ZIP / postal code |
| note | string or null | Delivery note |
| customerAddressType | int | 0=Commercial, 1=Residential |

**Sample:**

```json
{
  "success": true,
  "data": {
    "orderId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "orderNo": "ORD-00001",
    "orderType": 1,
    "orderDate": "2026-03-20T00:00:00Z",
    "clientReference": "MY-REF-001",
    "trackingNo": "TRACK123",
    "dueDate": "2026-04-01T00:00:00Z",
    "stockedOutDate": null,
    "actualDate": null,
    "status": 1,
    "shipmentVendor": 2,
    "shipmentType": 0,
    "palletQuantity": null,
    "hasRepalletization": null,
    "pickupNo": null,
    "pickupTime": null,
    "shippingCarrier": {
      "code": "FEDEX",
      "name": "FedEx"
    },
    "warehouse": {
      "code": "WH-01",
      "name": "Warehouse A"
    },
    "totalAmount": 299.99,
    "totalQuantity": 50,
    "totalPackageQuantity": 5,
    "totalWeight": 12.5,
    "shippingAddress": {
      "fullName": "John Doe",
      "phoneNumber": "+1234567890",
      "adminUnitName": "California",
      "adminUnitPathName": "US > California",
      "addressLine": "123 Main St",
      "postalCode": "90210",
      "note": null,
      "customerAddressType": 0
    },
    "remark": "Handle with care",
    "cancelReason": null
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 404, 429

---

### #4 — POST /v1/orders

Create a new order.

- Permission: Orders.Create
- Rate Limit: Write policy
- Required Header: `X-Idempotency-Key` (UUID — prevents duplicate creation on retry)

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| orderType | int | Yes | 1 (ECommerce), 3 (FBA), or 4 (Wholesale). Other values return 400. |
| orderDate | string (date) or null | No | Format: YYYY-MM-DD. Defaults to today if omitted. |
| warehouseId | string (UUID) or null | No | Warehouse to fulfill from. Obtain from GET /v1/warehouses. |
| clientReference | string or null | No | Your own reference code for this order (e.g., your system's order ID) |
| remark | string or null | No | Free-text note |
| storeId | string (UUID) or null | ECommerce: Yes | Store. Required for ECommerce only. Obtain from GET /v1/stores. |
| salesmanId | string (UUID) or null | No | Assigned salesman |
| dueDate | string (date) or null | No | Expected due date. Format: YYYY-MM-DD |
| shippingCarrierId | string (UUID) or null | No | Shipping carrier. Obtain from GET /v1/carriers. |
| carrierServiceId | string (UUID) or null | No | Carrier service level |
| trackingNo | string or null | No | Tracking number (if already known) |
| shippingAddress | object or null | ECommerce, Wholesale: Yes | Shipping destination. See fields below. |
| customerId | string (UUID) or null | ECommerce, Wholesale: Yes | Customer. Obtain from GET /v1/customers. |
| shipmentVendor | int or null | ECommerce, FBA, Wholesale: Yes | ShipmentVendorType (1=SelfPickup, 2=Warehouse, 3=ThirdParty) |
| fbaWarehouseId | string (UUID) or null | FBA: Yes | Destination FBA warehouse |
| shipmentType | int or null | FBA, Wholesale: Yes | 1=Express, 2=TruckFreight |
| palletQuantity | int or null | No | Number of pallets (FBA/Wholesale) |
| hasRepalletization | boolean or null | FBA, Wholesale: Yes | Whether repalletization is needed |
| pickupTime | string (ISO 8601) or null | No | Scheduled pickup time (FBA/Wholesale) |
| pickupNo | string or null | No | Pickup reference number (FBA/Wholesale) |

**shippingAddress Input Fields (nested object):**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| fullName | string | Yes | Recipient full name |
| phoneNumber | string | Yes | Recipient phone number |
| adminUnitId | int | Yes | Administrative unit ID (state/province). Obtain from GET /v1/admin-units. |
| addressLine | string | Yes | Street address |
| postalCode | string | Yes | ZIP / postal code |
| note | string or null | No | Delivery note |
| isSave | boolean | No | Save this address to customer profile. Default: false |
| customerAddressType | int | No | 0=Commercial (default), 1=Residential |

**Response — 201 Created:**

data contains the full order detail (same structure as GET /v1/orders/{orderId}). The `orderId` in the response is your reference — store this in your system for all subsequent operations.

**Sample Request (ECommerce):**

```json
{
  "orderType": 1,
  "orderDate": "2026-03-24",
  "warehouseId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "clientReference": "MY-SYSTEM-ORDER-12345",
  "remark": "Handle with care",
  "storeId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "dueDate": "2026-04-01",
  "shippingCarrierId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "trackingNo": "TRACK123",
  "shippingAddress": {
    "fullName": "John Doe",
    "phoneNumber": "+1234567890",
    "adminUnitId": 1,
    "addressLine": "123 Main St",
    "postalCode": "90210",
    "note": null,
    "isSave": false,
    "customerAddressType": 0
  },
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "shipmentVendor": 2
}
```

**Sample Response:**

```json
{
  "success": true,
  "data": {
    "orderId": "9ab12c34-5678-9012-def3-456789abcdef",
    "orderNo": "ORD-00042",
    "orderType": 1,
    "orderDate": "2026-03-24T00:00:00Z",
    "clientReference": "MY-SYSTEM-ORDER-12345",
    "status": 1,
    "..."
  },
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 400 (unsupported OrderType, domain error), 409 (duplicate idempotency key), 422 (validation/DB), 429

---

### #5 — PUT /v1/orders/{orderId}

Update an existing order's information. Only allowed when the order is in a modifiable status (typically New).

- Permission: Orders.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| orderType | int | Yes | Must match the order's current type. Cannot change order type. |
| orderDate | string (date) or null | No | Updated order date |
| warehouseId | string (UUID) or null | No | Reassign warehouse |
| clientReference | string or null | No | Update your reference code |
| remark | string or null | No | Update remark |
| storeId | string (UUID) or null | ECommerce: Yes | Store |
| salesmanId | string (UUID) or null | No | Assigned salesman |
| dueDate | string (date) or null | No | Updated due date |
| shippingCarrierId | string (UUID) or null | No | Shipping carrier |
| carrierServiceId | string (UUID) or null | No | Carrier service level |
| trackingNo | string or null | No | Tracking number |
| shippingAddress | object or null | ECommerce, Wholesale: Yes | Same fields as Create |
| customerId | string (UUID) or null | ECommerce, Wholesale: Yes | Customer |
| fbaWarehouseId | string (UUID) or null | FBA: Yes | FBA warehouse |
| shipmentVendor | int or null | ECommerce, FBA, Wholesale: Yes | ShipmentVendorType |
| shipmentType | int or null | FBA, Wholesale: Yes | ShipmentType |
| palletQuantity | int or null | No | Pallet count |
| hasRepalletization | boolean or null | FBA, Wholesale: Yes | Repalletization needed |
| pickupTime | string (ISO 8601) or null | No | Pickup time |
| pickupNo | string or null | No | Pickup reference |

**Response — 200 OK:** Empty data object.

**Sample Request:**
```json
{
  "orderType": 1,
  "warehouseId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "clientReference": "MY-SYSTEM-ORDER-12345-UPDATED",
  "remark": "Updated remark",
  "storeId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "dueDate": "2026-04-05",
  "shippingCarrierId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "trackingNo": "TRACK456",
  "shippingAddress": {
    "fullName": "Jane Doe",
    "phoneNumber": "+0987654321",
    "adminUnitId": 2,
    "addressLine": "456 Oak Ave",
    "postalCode": "10001",
    "customerAddressType": 1
  },
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "shipmentVendor": 2
}
```

**Possible Errors:** 401, 403, 400 (domain/state error, unsupported type), 404, 422, 429

---

### #6 — POST /v1/orders/{orderId}/request

Submit an order for processing. Transitions the order from **New → Requested** status.

- Permission: Orders.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |

**Request Body:** None

**Response — 202 Accepted:** Empty data object.

**Sample:**

```json
{
  "success": true,
  "data": {},
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 400 (order must be in New status), 404, 429

---

### #7 — POST /v1/orders/request-batch

Submit multiple orders for processing in a single request.

- Permission: Orders.Modify
- Rate Limit: Write policy

**Request Body:** Array of order identifiers.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| (body) | string (UUID)[] | Yes | List of order identifiers to submit |

**Sample Request:**

```json
[
  "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "9ab12c34-5678-9012-def3-456789abcdef"
]
```

**Response — 202 Accepted:** data contains batch processing result.

**Possible Errors:** 401, 403, 429

---

### #8 — POST /v1/orders/{orderId}/cancel

Cancel an order.

- Permission: Orders.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| cancelReason | string | Yes | Reason for cancellation. Free text. |

**Response — 202 Accepted:** Empty data object.

**Sample Request:**

```json
{
  "cancelReason": "Customer changed their mind"
}
```

**Possible Errors:** 401, 403, 400 (order is not in a cancellable status), 404, 429

---

### #9 — DELETE /v1/orders/{orderId}

Soft-delete an order. The order will no longer appear in list queries.

- Permission: Orders.Delete
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |

**Request Body:** None

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400 (invalid state), 404, 429

---

### #10 — POST /v1/orders/import

Import orders from an Excel file.

- Permission: Orders.Create
- Rate Limit: Import policy
- Content-Type: multipart/form-data
- Required Header: `X-Idempotency-Key` (UUID)

**Form Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| orderType | int | Yes | OrderType (1, 3, or 4) |
| file | File | Yes | Excel file. Allowed: `.xls`, `.xlsx` only. Max size depends on server config. |

**Response — 200 OK:** data contains import result summary.

**Sample Error — Wrong file type:**

```json
{
  "success": false,
  "errorCode": "99",
  "error": "File type is not supported!",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 400 (file type/domain), 409 (duplicate idempotency key), 422, 429

---

## API Endpoints — Order Lines

---

### #11 — GET /v1/orders/{orderId}/lines

Get a paginated list of order lines for a given order.

- Permission: Orders.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |

**Request Body:** None

**Response — 200 OK (Paginated):**

data is an array of order line objects:

| Field | Type | Description |
|-------|------|-------------|
| orderLineId | string (UUID) | Your reference to this line. Use for update/delete operations. |
| orderNo | string | Parent order number |
| orderStatus | string | Parent order status (text label, e.g., "New", "Confirmed") |
| warehouseName | string | Warehouse name |
| quantity | int | Ordered quantity |
| stockedOut | int or null | Actual stocked-out package quantity (populated after warehouse processing) |
| sellingPrice | number | Selling price per unit |
| expectedPrice | number | Expected price per unit |
| remark | string or null | Line remark |
| lotNo | string or null | Lot/batch number |
| product | object | Product information (see below) |

**product (nested object):**

| Field | Type | Description |
|-------|------|-------------|
| code | string | Product SKU code |
| name | string | Product name |
| quantityPerPackage | int | Units per package |
| packageUnitCode | string | Package unit (e.g., "BOX") |
| unitCode | string | Base unit (e.g., "PCS") |

**Sample:**

```json
{
  "success": true,
  "data": [
    {
      "orderLineId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "orderNo": "ORD-00001",
      "orderStatus": "New",
      "warehouseName": "Warehouse A",
      "quantity": 10,
      "stockedOut": null,
      "sellingPrice": 29.99,
      "expectedPrice": 25,
      "remark": null,
      "lotNo": "LOT-001",
      "product": {
        "code": "SKU-001",
        "name": "Product A",
        "quantityPerPackage": 1,
        "packageUnitCode": "BOX",
        "unitCode": "PCS"
      }
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 20,
  "rowCount": 3,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 404, 429

---

### #12 — GET /v1/orders/{orderId}/lines/{orderLineId}

Get a specific order line.

- Permission: Orders.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |
| orderLineId | string (UUID) | Yes | The order line identifier |

**Request Body:** None

**Response — 200 OK:** data is a single order line object (same fields as #11).

**Possible Errors:** 401, 403, 404, 429

---

### #13 — POST /v1/orders/{orderId}/lines

Add a new line to an order.

- Permission: Orders.Create
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| productId | string (UUID) | Yes | Product to add. Obtain from GET /v1/products. |
| lotNo | string or null | No | Lot/batch number |
| quantity | int | Yes | Quantity. Must be greater than 0. |
| sellingPrice | number | Yes | Selling price per unit. Must be >= 0. |
| expectedPrice | number | Yes | Expected price per unit. Must be >= 0. |
| remark | string or null | No | Free-text remark |

**Response — 201 Created:** data is the created order line (same fields as GET response, including `orderLineId`).

**Sample Request:**

```json
{
  "productId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "lotNo": "LOT-001",
  "quantity": 10,
  "sellingPrice": 29.99,
  "expectedPrice": 25.00,
  "remark": "First batch"
}
```

**Possible Errors:** 401, 403, 400, 404, 422, 429

---

### #14 — POST /v1/orders/{orderId}/lines/import

Import order lines from an Excel file.

- Permission: Orders.Import
- Rate Limit: Import policy
- Content-Type: multipart/form-data

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |

**Form Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| file | File | Yes | Excel file. Allowed: `.xls`, `.xlsx` only. |

**Response — 200 OK:** data contains import result summary.

**Possible Errors:** 401, 403, 400 (file type/domain), 422, 429

---

### #15 — PUT /v1/orders/{orderId}/lines/{orderLineId}

Update an existing order line.

- Permission: Orders.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |
| orderLineId | string (UUID) | Yes | The order line identifier |

**Request Body Fields:** Same as POST /v1/orders/{orderId}/lines.

**Response — 200 OK:** data is the updated order line (same fields as GET response).

**Possible Errors:** 401, 403, 400, 404, 422, 429

---

### #16 — DELETE /v1/orders/{orderId}/lines/{orderLineId}

Remove an order line.

- Permission: Orders.Delete
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |
| orderLineId | string (UUID) | Yes | The order line identifier |

**Request Body:** None

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400, 404, 429

---

## API Endpoints — Order Documents

---

### #17 — GET /v1/orders/{orderId}/documents

Get a paginated list of documents for an order.

- Permission: Orders.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |

**Request Body:** None

**Response — 200 OK (Paginated):**

data is an array of document objects:

| Field | Type | Description |
|-------|------|-------------|
| documentId | string (UUID) | Your reference to this document. Use for download/delete. |
| documentTypeName | string | Type of document (e.g., "Invoice", "Packing List") |
| remark | string or null | Document remark |
| fileName | string | Original file name |
| createdDateTime | string (ISO 8601) | Upload timestamp |

**Sample:**

```json
{
  "success": true,
  "data": [
    {
      "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "documentTypeName": "Invoice",
      "remark": "Signed copy",
      "fileName": "invoice-001.pdf",
      "createdDateTime": "2026-03-20T08:30:00Z"
    }
  ],
  "currentPage": 1,
  "pageCount": 1,
  "pageSize": 20,
  "rowCount": 2,
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 404, 429

---

### #18 — GET /v1/orders/{orderId}/documents/{documentId}/download

Download an order document.

- Permission: Orders.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |
| documentId | string (UUID) | Yes | The document identifier |

**Request Body:** None

**Response — 200 OK:** Binary file stream. `Content-Type` matches the file MIME type (e.g., `application/pdf`). Header: `Content-Disposition: attachment; filename="document.pdf"`

**Possible Errors:** 401, 403, 404, 429

---

### #19 — POST /v1/orders/{orderId}/documents

Upload documents for an order.

- Permission: Orders.Create
- Rate Limit: Import policy
- Content-Type: multipart/form-data
- Required Header: `X-Idempotency-Key` (UUID)

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |

**Form Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| documentTypeId | string (UUID) | Yes | Type of document being uploaded. Obtain from GET /v1/document-types. |
| remark | string or null | No | Free-text remark |
| files | File[] | Yes | One or more files to upload |

**Response — 202 Accepted:** Empty data object.

**Possible Errors:** 401, 403, 400, 404, 409 (duplicate idempotency key), 429

---

### #20 — DELETE /v1/orders/{orderId}/documents/{documentId}

Delete (hide) an order document.

- Permission: Orders.Delete
- Rate Limit: Write policy  

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | The order identifier |
| documentId | string (UUID) | Yes | The document identifier |

**Request Body:** None

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400, 404, 429
