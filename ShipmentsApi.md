[PublicApi-ShipmentsController-Documentation.md](https://github.com/user-attachments/files/26318857/PublicApi-ShipmentsController-Documentation.md)
# Shipment Apis Documentation

> Base Route: /v{version:apiVersion}/shipments
> API Version: 1.0
> Authentication: OAuth2 Bearer Token (required on all endpoints)
> Rate Limiting: All endpoints are rate-limited (read/write/import policies)
> Company Scope: The caller's company is resolved from the `X-Cid` header. All data is automatically scoped to the authenticated company.

---

## Enum Reference

### ShipmentStatus
- 0 = Unknown
- 1 = New
- 2 = Confirmed
- 3 = Onway
- 4 = StockedIn
- 5 = Completed
- 6 = Review
- 7 = Rejected
- 8 = Canceled
- 9 = PartStockedIn

### TransportationMode
- Numeric byte value representing the transportation method (e.g., sea, air, land). Obtain valid values from your platform administrator.

### Service
- 0 = Unknown
- 1 = USWarehousing
- 2 = Fullfillment
- 3 = FBA

### ShipmentPackingType
- 0 = None
- 1 = AllPallet
- 2 = AllCarton
- 3 = Mix

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
curl -X GET https://api.example.com/v1/shipments \
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

## API Endpoints — Shipments

---

### #1 — GET /v1/shipments

Get a paginated list of shipments.

- Permission: Shipments.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Page number (1-based). Default: 1 |
| pageSize | int | No | Items per page. Default: 20, Max: 100 |
| warehouseId | string (UUID) | No | Filter by warehouse |
| productId | string (UUID) | No | Filter shipments containing this product |
| fromDate | string (date) | No | Shipment date range start (inclusive). Format: YYYY-MM-DD |
| toDate | string (date) | No | Shipment date range end (inclusive). Format: YYYY-MM-DD |
| status | int[] | No | Filter by one or more ShipmentStatus values. Pass multiple: `?status=1&status=2` |
| clientRef | string | No | Filter by your reference code |
| shipmentNo | string | No | Filter by system shipment number |
| query | string | No | Free-text search across shipment number, client reference |

**Response — 200 OK (Paginated):**

data is an array of shipment summaries:

| Field | Type | Description |
|-------|------|-------------|
| id | string (UUID) | Shipment identifier |
| shipmentNo | string | System-generated shipment number |
| warehouseName | string or null | Warehouse name |
| shipmentMasterNo | string or null | Parent shipment master number |
| companyShortName | string | Company short name |
| shipmentDate | string (ISO 8601) | Date the shipment was created |
| clientRef | string | Your reference code |
| transportationMode | int | Transportation mode value |
| service | int | Service type value |
| status | int | ShipmentStatus value |
| cargoReadyDate | string (ISO 8601) | Cargo ready date |
| expectedToWH | string (ISO 8601) | Expected arrival at warehouse |
| eta | string (ISO 8601) or null | Estimated time of arrival |
| whata | string (ISO 8601) or null | Warehouse actual time of arrival |
| wheta | string (ISO 8601) or null | Warehouse estimated time of arrival |
| enableGoodsCounting | boolean | Whether goods counting is enabled |
| totalPallet | int or null | Total pallet count |
| totalCarton | int or null | Total carton count |
| totalLooseCarton | int or null | Total loose carton count |
| shipmentPackingType | int | ShipmentPackingType value |
| totalQuantity | int or null | Total item quantity (computed) |
| totalActualQuantity | int or null | Total actual quantity (computed) |
| totalPackage | int or null | Total package count (computed) |

**Possible Errors:** 401, 403, 429

---

### #2 — GET /v1/shipments/count-status

Get shipment count grouped by status.

- Permission: Shipments.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| warehouseId | string (UUID) | No | Filter by warehouse |
| productId | string (UUID) | No | Filter by product |
| fromDate | string (date) | No | Date range start |
| toDate | string (date) | No | Date range end |

**Response — 200 OK:**

data is an array of status counts:

| Field | Type | Description |
|-------|------|-------------|
| id | int | ShipmentStatus value |
| count | int | Number of shipments in that status |

**Sample:**

```json
{
  "success": true,
  "data": [
    { "id": 1, "count": 10 },
    { "id": 2, "count": 5 },
    { "id": 3, "count": 3 },
    { "id": 4, "count": 8 },
    { "id": 5, "count": 20 },
    { "id": 6, "count": 2 },
    { "id": 7, "count": 1 },
    { "id": 8, "count": 4 },
    { "id": 9, "count": 0 }
  ],
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 429

---

### #3 — GET /v1/shipments/{shipmentId}

Get full details of a shipment.

- Permission: Shipments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Response — 200 OK:**

data is a shipment detail object:

| Field | Type | Description |
|-------|------|-------------|
| id | string (UUID) | Shipment identifier |
| companyId | string (UUID) | Company identifier |
| warehouseId | string (UUID) or null | Warehouse identifier |
| shipmentNo | string | System-generated shipment number |
| shipmentDate | string (ISO 8601) | Date the shipment was created |
| clientRef | string | Your reference code |
| transportationMode | int | Transportation mode value |
| service | int | Service type value |
| cargoReadyDate | string (ISO 8601) | Cargo ready date |
| expectedToWH | string (ISO 8601) | Expected arrival at warehouse |
| remark | string or null | Free-text remark |
| status | int | ShipmentStatus value |
| enableGoodsCounting | boolean | Whether goods counting is enabled |
| totalPallet | int or null | Total pallet count |
| totalCarton | int or null | Total carton count |
| totalLooseCarton | int or null | Total loose carton count |
| shipmentPackingType | int | ShipmentPackingType value |
| warehouse | object or null | Warehouse information |
| shipmentMasterNo | string or null | Parent shipment master number |
| eta | string (ISO 8601) or null | Estimated time of arrival |
| etd | string (ISO 8601) or null | Estimated time of departure |
| wheta | string (ISO 8601) or null | Warehouse estimated time of arrival |
| stockedInDate | string (ISO 8601) or null | Actual stocked-in date |
| shipmentPackingSummaries | array | Packing summary breakdown |

**Possible Errors:** 401, 403, 404, 429

---

### #4 — POST /v1/shipments

Create a new shipment.

- Permission: Shipments.Create
- Rate Limit: Write policy

**Request Body Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| clientRef | string | Yes | Your reference code for this shipment |
| transportationMode | int | Yes | Transportation mode value |
| service | int | Yes | Service type (0=Unknown, 1=USWarehousing, 2=Fullfillment, 3=FBA) |
| cargoReadyDate | string (ISO 8601) | Yes | Cargo ready date |
| expectedToWH | string (ISO 8601) | Yes | Expected arrival at warehouse |
| warehouseId | string (UUID) or null | No | Destination warehouse. Obtain from GET /v1/warehouses. |
| remark | string or null | No | Free-text remark |
| enableGoodsCounting | boolean | No | Enable goods counting. Default: false |
| shipmentPackingType | int | No | 0=None, 1=AllPallet, 2=AllCarton, 3=Mix |
| totalPallet | int or null | No | Total pallet count (when PackingType is AllPallet or Mix) |
| totalCarton | int or null | No | Total carton count (when PackingType is AllCarton or Mix) |
| totalLooseCarton | int or null | No | Total loose carton count |

**Sample Request:**

```json
{
  "clientRef": "MY-SHIPMENT-001",
  "transportationMode": 1,
  "service": 1,
  "cargoReadyDate": "2026-04-01T00:00:00Z",
  "expectedToWH": "2026-04-15T00:00:00Z",
  "warehouseId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "remark": "Handle with care",
  "enableGoodsCounting": false,
  "shipmentPackingType": 1,
  "totalPallet": 5,
  "totalCarton": null,
  "totalLooseCarton": null
}
```

**Response — 201 Created:**

data contains the full shipment detail (same structure as GET /v1/shipments/{shipmentId}).

**Possible Errors:** 401, 403, 400, 422, 429

---

### #5 — PUT /v1/shipments/{shipmentId}

Update an existing shipment's information. Only allowed when the shipment is in a modifiable status (New).

- Permission: Shipments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Request Body Fields:** Same as POST /v1/shipments (all fields from create), plus additional fields for checked quantities:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| clientRef | string | Yes | Your reference code |
| transportationMode | int | Yes | Transportation mode |
| service | int | Yes | Service type |
| cargoReadyDate | string (ISO 8601) | Yes | Cargo ready date |
| expectedToWH | string (ISO 8601) | Yes | Expected arrival at warehouse |
| warehouseId | string (UUID) or null | No | Warehouse |
| remark | string or null | No | Remark |
| enableGoodsCounting | boolean | No | Enable goods counting |
| shipmentPackingType | int | No | Packing type |
| totalPallet | int or null | No | Total pallet count |
| totalCarton | int or null | No | Total carton count |
| totalLooseCarton | int or null | No | Total loose carton count |

**Response — 200 OK:**

data contains the updated shipment detail.

**Possible Errors:** 401, 403, 400 (domain/state error), 404, 422, 429

---

### #6 — POST /v1/shipments/{shipmentId}/request

Submit a shipment for operator review. Transitions the shipment from **New → Review** status.

- Permission: Shipments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Request Body:** None

**Response — 202 Accepted:** Empty data object.

**Possible Errors:** 401, 403, 400 (shipment must be in New status), 404, 429

---

### #7 — POST /v1/shipments/{shipmentId}/renew

Renew a rejected or cancelled shipment back to **New** status so it can be edited and resubmitted.

- Permission: Shipments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Request Body:** None

**Response — 202 Accepted:** Empty data object.

**Possible Errors:** 401, 403, 400 (shipment must be in Rejected or Canceled status), 404, 429

---

### #8 — POST /v1/shipments/{shipmentId}/cancel

Cancel a shipment.

- Permission: Shipments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Request Body:** None

**Response — 202 Accepted:** Empty data object.

**Possible Errors:** 401, 403, 400 (shipment must be in Review or Confirmed status), 404, 429

---

### #9 — DELETE /v1/shipments/{shipmentId}

Delete a shipment. The shipment will no longer appear in list queries.

- Permission: Shipments.Delete
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Request Body:** None

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400 (shipment must be in New, Canceled, or Rejected status), 404, 429

---

## API Endpoints — Shipment Packings

---

### #10 — GET /v1/shipments/{shipmentId}/packings

Get a paginated list of packings (product lines) for a shipment.

- Permission: Shipments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| query | string | No | Free-text search across product code/name |

**Response — 200 OK (Paginated):**

data is an array of packing objects:

| Field | Type | Description |
|-------|------|-------------|
| id | string (UUID) | Packing line identifier |
| productId | string (UUID) | Product identifier |
| lotNo | string or null | Lot/batch number |
| expiredDate | string (ISO 8601) or null | Expiry date |
| containerNo | string or null | Container number |
| containerSize | string or null | Container size |
| quantity | int | Planned quantity |
| packageQuantity | int or null | Computed package count |
| stockedIn | int or null | Actual stocked-in quantity |
| remark | string or null | Line remark |
| goodsValue | number | Declared goods value |
| grossWeight | number | Gross weight |
| measurement | number | Measurement / volume |
| product | object | Product information (code, name, quantityPerPackage, etc.) |

**Possible Errors:** 401, 403, 404, 429

---

### #11 — GET /v1/shipments/{shipmentId}/packings/{packingId}

Get a specific shipment packing by its ID.

- Permission: Shipments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |
| packingId | string (UUID) | Yes | The packing line identifier |

**Response — 200 OK:** data is a single packing object (same fields as #10).

**Possible Errors:** 401, 403, 404, 429

---

### #12 — POST /v1/shipments/{shipmentId}/packings

Add a new packing line to a shipment.

- Permission: Shipments.Create or Shipments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Request Body Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productId | string (UUID) | Yes | Product to add. Obtain from GET /v1/products. |
| lotNo | string or null | No | Lot/batch number |
| quantity | int | Yes | Quantity. Must be greater than 0. |
| goodsValue | number | No | Declared goods value. Default: 0. |
| expiredDate | string (ISO 8601) or null | No | Product expiry date |
| grossWeight | number | Yes | Gross weight |
| measurement | number | Yes | Measurement / volume |
| remark | string or null | No | Free-text remark |

**Sample Request:**

```json
{
  "productId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "lotNo": "LOT-2026-001",
  "quantity": 100,
  "goodsValue": 1500.00,
  "expiredDate": "2027-12-31T00:00:00Z",
  "grossWeight": 250.5,
  "measurement": 12.3,
  "remark": "First batch"
}
```

**Response — 201 Created:** data is the created packing (same fields as GET response, including `id`).

**Possible Errors:** 401, 403, 400, 404, 422, 429

---

### #13 — POST /v1/shipments/{shipmentId}/packings/import

Import shipment packings from an Excel file.

- Permission: Shipments.Create or Shipments.Modify
- Rate Limit: Import policy
- Content-Type: multipart/form-data

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Form Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| file | File | Yes | Excel file. Allowed: `.xls`, `.xlsx` only. |

**Response — 200 OK:** data contains import result summary.

**Possible Errors:** 401, 403, 400 (file type/domain), 422, 429

---

### #14 — PUT /v1/shipments/{shipmentId}/packings/{packingId}

Update an existing shipment packing.

- Permission: Shipments.Create or Shipments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |
| packingId | string (UUID) | Yes | The packing line identifier |

**Request Body Fields:** Same as POST /v1/shipments/{shipmentId}/packings.

**Response — 200 OK:** data is the updated packing object.

**Possible Errors:** 401, 403, 400, 404, 422, 429

---

### #15 — DELETE /v1/shipments/{shipmentId}/packings/{packingId}

Remove a shipment packing.

- Permission: Shipments.Delete
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |
| packingId | string (UUID) | Yes | The packing line identifier |

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400, 404, 429

---

## API Endpoints — Shipment Documents

---

### #16 — GET /v1/shipments/{shipmentId}/documents

Get a paginated list of documents for a shipment.

- Permission: Shipments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |

**Response — 200 OK (Paginated):**

data is an array of document objects:

| Field | Type | Description |
|-------|------|-------------|
| documentId | string (UUID) | Document identifier |
| documentTypeName | string | Type of document (e.g., "Packing List", "Invoice") |
| remark | string or null | Document remark |
| fileName | string | Original file name |
| createdDateTime | string (ISO 8601) | Upload timestamp |
| isMasterDoc | boolean | Whether this document belongs to the parent shipment master |

**Possible Errors:** 401, 403, 404, 429

---

### #17 — GET /v1/shipments/{shipmentId}/documents/{documentId}

Get a specific shipment document metadata.

- Permission: Shipments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |
| documentId | string (UUID) | Yes | The document identifier |

**Response — 200 OK:** data is a single document object (same fields as #16).

**Possible Errors:** 401, 403, 404, 429

---

### #18 — GET /v1/shipments/{shipmentId}/documents/{documentId}/download

Download a shipment document.

- Permission: Shipments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |
| documentId | string (UUID) | Yes | The document identifier |

**Response — 200 OK:** Binary file stream. `Content-Type` matches the file MIME type. Header: `Content-Disposition: attachment; filename="document.pdf"`

**Possible Errors:** 401, 403, 400, 404, 429

---

### #19 — POST /v1/shipments/{shipmentId}/documents

Upload documents for a shipment.

- Permission: Shipments.Create or Shipments.Modify
- Rate Limit: Import policy
- Content-Type: multipart/form-data

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |

**Form Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| documentTypeId | string (UUID) | Yes | Type of document being uploaded. Obtain from GET /v1/document-types. |
| remark | string or null | No | Free-text remark |
| files | File[] | Yes | One or more files to upload |

**Response — 202 Accepted:** Empty data object.

**Possible Errors:** 401, 403, 400, 404, 429

---

### #20 — DELETE /v1/shipments/{shipmentId}/documents/{documentId}

Delete (hide) a shipment document.

- Permission: Shipments.Delete
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | The shipment identifier |
| documentId | string (UUID) | Yes | The document identifier |

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400, 404, 429

---

## Shipment Status Transition Rules

| From | To | Triggered By | API |
|------|----|-------------|-----|
| — | **New** | Merchant | `POST /v1/shipments` |
| New | **Review** | Merchant | `POST /v1/shipments/{id}/request` |
| Review | **Confirmed** | Operator | AdminApi |
| Review | **Rejected** | Operator | AdminApi |
| Confirmed | **Onway** | Operator | AdminApi |
| Onway, PartStockedIn | **StockedIn** | Warehouse / Operator | AdminApi |
| StockedIn | **Completed** | Operator | AdminApi |
| Review, Confirmed | **Canceled** | Merchant | `POST /v1/shipments/{id}/cancel` |
| Rejected, Canceled | **New** | Merchant | `POST /v1/shipments/{id}/renew` |
| New, Canceled, Rejected | **Deleted** | Merchant | `DELETE /v1/shipments/{id}` |

### Shipment Lifecycle (Quick View)

```
                         +-------------------+
                         |  Customer / API   |
                         +--------+----------+
                                  |
                       POST /shipments
                                  v
                           +------+------+
                           |             |
                    +----->|  (1) New    |<------- POST .../renew
                    |      |             |         (from Rejected / Canceled)
                    |      +------+------+
                    |             |
                    |             | POST .../request
                    |             v
                    |      +------+------+
                    |      |             |
                    |      | (6) Review  |
                    |      |             |
                    |      +---+----+----+
                    |          |    |
                    |  Confirm |    | Reject
                    |          v    v
                    |   +------++ ++---------+
                    |   |       | |          |
                    |   |  (2)  | |   (7)    |
                    |   |Confirm| | Rejected |
                    |   |  -ed  | |          |
                    |   +---+---+ +----------+
                    |       |
                    |       v
                    |  +----+----+
                    |  |         |
                    |  |(3) Onway|
                    |  |         |
                    |  +----+----+
                    |       |
                    |       v
                    |  +----+-------+     +-------------+
                    |  |            |---->|             |
                    |  |(4) Stocked |     |(9) Part     |
                    |  |    In      |<----| Stocked In  |
                    |  +----+-------+     +-------------+
                    |       |
                    |       v
                    |  +----+-------+
                    |  |            |
                    |  |(5)Complete |
                    |  |    -ed     |
                    |  +------------+
                    |
    Cancel ---------+-------> (8) Canceled
    (from Review, Confirmed)

    Delete (from New, Canceled, Rejected) --> Deleted
```
