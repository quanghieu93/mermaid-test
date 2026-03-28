# PublicApi — InventoriesController API Documentation

> Controller: GlobalSales.PublicApi.Controllers.InventoriesController
> Base Route: /v{version:apiVersion}/inventories
> API Version: 1.0
> Authentication: OAuth2 Bearer Token (required on all endpoints)
> Authorization: Client credentials with scope "public-api" + resource permissions
> Rate Limiting: All endpoints are rate-limited (read/write/import policies)
> Company Scope: The caller's company is resolved from the authenticated token. There is no need to pass a company ID — all data is automatically scoped to the authenticated company.

---

## Design Principles

This API is designed for **external systems** (merchants, third-party platforms) to view inventory data and manage stock adjustments. The following principles guide what is exposed:

1. **`adjustmentId` is returned in responses** — External systems need this identifier to perform subsequent operations (update, request, add lines, etc.). This is your reference to the adjustment in our system.
2. **Company context is implicit** — Company identity is derived from the authentication token. You cannot see or access another company's data.
3. **Internal operational data is not exposed** — Fields like WMS sync status, source type, and internal user IDs are omitted from documentation.
4. **Input fields use IDs when you need to reference existing entities** — When creating adjustments or lines, you pass `warehouseId`, `productId`, `stockLineId`, etc. These IDs are provided via separate lookup endpoints.

---

## Enum Reference

### AdjustmentStatus
- 0 = Unknown
- 1 = New
- 2 = Requested
- 3 = Confirmed
- 4 = Rejected
- 5 = Adjusted

### AdjustReason
- 0 = None
- 1 = NaturalDepletion
- 2 = EroneousDataEntry
- 3 = SystemDataError

### StockStatus
- 1 = Good
- 2 = Defective
- 3 = Returned

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
      "propertyName": "warehouseId",
      "messages": ["WarehouseId is required."]
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
  "error": "Adjustment is not in the correct status to perform this action.",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### 404 Not Found (errorCode: 404)

```json
{
  "success": false,
  "errorCode": "404",
  "error": "Adjustment not found.",
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

## API Endpoints — Inventories

---

### #1 — GET /v1/inventories/products

Get inventory for a specific product at a specific warehouse.

- Permission: Inventories.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| warehouseId | string (UUID) | Yes | The warehouse to check |
| productId | string (UUID) | Yes | The product to check |

**Request Body:** None

**Response — 200 OK:** data contains product inventory details (on-hand, ordered, incoming quantities).

**Possible Errors:** 401, 403, 404, 429

---

### #2 — GET /v1/inventories

Get a paginated list of inventories.

- Permission: Inventories.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| pageIndex | int | No | Page number (1-based). Default: 1 |
| pageSize | int | No | Items per page. Default: 20, Max: 100 |
| warehouseIds | string (UUID)[] | No | Filter by warehouse(s). Pass multiple: `?warehouseIds=uuid1&warehouseIds=uuid2` |
| productId | string (UUID) | No | Filter by product |
| showEmpty | boolean | No | Include products with zero inventory. Default: false |
| orderBy | string | No | Sort field |

> **Note:** Do **not** include `companyId` in the query parameters. It is automatically set from your authenticated context.

**Request Body:** None

**Response — 200 OK (Paginated):** data is an array of inventory summaries.

**Possible Errors:** 401, 403, 429

---

### #3 — GET /v1/inventories/details

Get inventory details with lot-level breakdown.

- Permission: Inventories.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| warehouseIds | string (UUID)[] | No | Filter by warehouse(s) |
| productId | string (UUID) | No | Filter by product |
| showEmptyLot | boolean | No | Include lots with zero quantity. Default: false |
| fromDate | string (date) | No | Filter from date. Format: YYYY-MM-DD |

**Request Body:** None

**Response — 200 OK:** data contains detailed inventory breakdown by lot.

**Possible Errors:** 401, 403, 429

---

### #4 — GET /v1/inventories/histories/summary

Get inventory history summary (aggregated view).

- Permission: Inventories.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| warehouseId | string (UUID) | Yes | Warehouse to query |
| productId | string (UUID) | Yes | Product to query |
| status | int | No | StockStatus filter (1=Good, 2=Defective, 3=Returned) |
| fromDate | string (date) | Yes | Start date. Format: YYYY-MM-DD |
| toDate | string (date) | Yes | End date. Format: YYYY-MM-DD |
| lotNo | string | No | Filter by lot number |

**Request Body:** None

**Response — 200 OK:** data contains aggregated history summary.

**Possible Errors:** 401, 403, 429

---

### #5 — GET /v1/inventories/histories

Get a paginated list of inventory history records.

- Permission: Inventories.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| warehouseId | string (UUID) | Yes | Warehouse to query |
| productId | string (UUID) | Yes | Product to query |
| status | int | No | StockStatus filter |
| fromDate | string (date) | Yes | Start date |
| toDate | string (date) | Yes | End date |
| lotNo | string | No | Filter by lot number |

**Request Body:** None

**Response — 200 OK (Paginated):** data is an array of inventory history records.

**Possible Errors:** 401, 403, 429

---

### #6 — GET /v1/inventories/export

Export inventories to an Excel file.

- Permission: Inventories.Export
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| warehouseIds | string (UUID)[] | No | Filter by warehouse(s) |
| productId | string (UUID) | No | Filter by product |
| showEmpty | boolean | No | Include zero-inventory products |

**Request Body:** None

**Response — 200 OK:** File download (`.xlsx`). The `Content-Type` and `Content-Disposition` headers are set appropriately.

**Possible Errors:** 401, 403, 400, 429

---

### #7 — GET /v1/inventories/histories/export

Export inventory history to an Excel file.

- Permission: Inventories.Export
- Rate Limit: Read policy

**Query Parameters:** Same as [#5 — GET /v1/inventories/histories](#5--get-v1inventorieshistories).

**Request Body:** None

**Response — 200 OK:** File download (`.xlsx`).

**Possible Errors:** 401, 403, 400, 429

---

## API Endpoints — Adjustments

---

### #8 — GET /v1/inventories/adjustments/{adjustmentId}

Get an adjustment by its ID.

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |

**Request Body:** None

**Response — 200 OK:** data contains adjustment details.

**Possible Errors:** 401, 403, 404, 429

---

### #9 — GET /v1/inventories/adjustments

Get a paginated list of adjustments.

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| warehouseId | string (UUID) | No | Filter by warehouse |
| fromDate | string (date) | Yes | Start date |
| toDate | string (date) | Yes | End date |
| status | int[] | No | Filter by AdjustmentStatus values. Pass multiple: `?status=1&status=2` |
| documentRef | string | No | Filter by document reference |
| adjustSheetNo | string | No | Filter by adjustment sheet number |
| query | string | No | Free-text search |

**Request Body:** None

**Response — 200 OK (Paginated):** data is an array of adjustment summaries.

**Possible Errors:** 401, 403, 429

---

### #10 — POST /v1/inventories/adjustments

Create a new inventory adjustment.

- Permission: Adjustments.Create
- Rate Limit: Write policy

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| warehouseId | string (UUID) | Yes | Warehouse to adjust. Obtain from GET /v1/warehouses. |
| documentRef | string or null | No | External document reference |
| reason | int | Yes | AdjustReason (0=None, 1=NaturalDepletion, 2=EroneousDataEntry, 3=SystemDataError) |
| remark | string or null | No | Free-text note |
| stocktakingDate | string (date) | Yes | Date of stocktaking. Format: YYYY-MM-DD |
| stocktakingPIC | string | Yes | Person in charge of stocktaking |

**Response — 201 Created:** data contains the full adjustment detail object.

**Sample Request:**

```json
{
  "warehouseId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "documentRef": "ADJ-EXT-001",
  "reason": 2,
  "remark": "Cycle count correction",
  "stocktakingDate": "2026-03-20",
  "stocktakingPIC": "John Doe"
}
```

**Possible Errors:** 401, 403, 400, 422, 429

---

### #11 — PUT /v1/inventories/adjustments/{adjustmentId}

Update an existing adjustment's information. Only allowed when the adjustment is in a modifiable status (typically New).

- Permission: Adjustments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| documentRef | string or null | No | Updated document reference |
| reason | int | Yes | AdjustReason |
| remark | string or null | No | Updated remark |
| stocktakingDate | string (date) | Yes | Updated stocktaking date |
| stocktakingPIC | string | Yes | Updated person in charge |

**Response — 202 Accepted:** data contains the updated adjustment detail object.

**Possible Errors:** 401, 403, 400, 404, 422, 429

---

### #12 — GET /v1/inventories/adjustments/count-status

Get adjustment count grouped by status.

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| warehouseId | string (UUID) | No | Filter by warehouse |
| fromDate | string (date) | Yes | Start date |
| toDate | string (date) | Yes | End date |

**Request Body:** None

**Response — 200 OK:** data is an array of status/count pairs.

**Possible Errors:** 401, 403, 429

---

### #13 — PATCH /v1/inventories/adjustments/{adjustmentId}/request

Submit an adjustment for processing. Transitions the adjustment from **New → Requested** status.

- Permission: Adjustments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |

**Request Body:** None

**Response — 202 Accepted:** data contains the updated adjustment detail object.

**Possible Errors:** 401, 403, 400 (adjustment must be in New status), 404, 429

---

### #14 — PATCH /v1/inventories/adjustments/{adjustmentId}/renew

Renew a rejected adjustment back to New status for re-editing.

- Permission: Adjustments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |

**Request Body:** None

**Response — 202 Accepted:** data contains the updated adjustment detail object.

**Possible Errors:** 401, 403, 400 (adjustment must be in Rejected status), 404, 429

---

### #15 — GET /v1/inventories/adjustments/products

Get a paginated list of products with inventory at a specific warehouse (for use when building adjustment lines).

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| warehouseId | string (UUID) | Yes | Warehouse to query |
| code | string | No | Filter by product code |
| name | string | No | Filter by product name |

**Request Body:** None

**Response — 200 OK (Paginated):** data is an array of products with inventory quantities.

**Possible Errors:** 401, 403, 429

---

### #16 — GET /v1/inventories/adjustments/product

Get inventory summary for a specific product at a warehouse (for adjustment detail view).

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| warehouseId | string (UUID) | Yes | Warehouse to query |
| productId | string (UUID) | Yes | Product to query |
| stockLineId | string (UUID) | No | Filter to a specific stock line |
| status | int | No | StockStatus filter (1=Good, 2=Defective, 3=Returned) |

**Request Body:** None

**Response — 200 OK:** data contains product inventory summary with stock breakdown.

**Possible Errors:** 401, 403, 429

---

### #17 — GET /v1/inventories/adjustments/stock-lines

Get a paginated list of stock lines for a specific product at a warehouse.

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |
| warehouseId | string (UUID) | Yes | Warehouse to query |
| productId | string (UUID) | Yes | Product to query |
| status | int | No | StockStatus filter |
| referenceNo | string | No | Filter by reference number |
| query | string | No | Free-text search |

**Request Body:** None

**Response — 200 OK (Paginated):** data is an array of stock line records.

**Possible Errors:** 401, 403, 429

---

### #18 — DELETE /v1/inventories/adjustments/{adjustmentId}

Delete an adjustment.

- Permission: Adjustments.Delete
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |

**Request Body:** None

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400 (invalid state), 404, 429

---

## API Endpoints — Adjustment Lines

---

### #19 — GET /v1/inventories/adjustments/{adjustmentId}/lines/{adjustmentLineId}

Get a specific adjustment line.

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |
| adjustmentLineId | string (UUID) | Yes | The adjustment line identifier |

**Request Body:** None

**Response — 200 OK:** data contains adjustment line details.

**Possible Errors:** 401, 403, 404, 429

---

### #20 — GET /v1/inventories/adjustments/{adjustmentId}/lines

Get a paginated list of adjustment lines.

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |

**Request Body:** None

**Response — 200 OK (Paginated):** data is an array of adjustment line objects.

**Possible Errors:** 401, 403, 429

---

### #21 — POST /v1/inventories/adjustments/{adjustmentId}/lines

Add a new line to an adjustment.

- Permission: Adjustments.Create or Adjustments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| productId | string (UUID) | Yes | Product to adjust. Obtain from GET /v1/inventories/adjustments/products. |
| stockLineId | string (UUID) or null | No | Specific stock line to adjust. Obtain from GET /v1/inventories/adjustments/stock-lines. |
| status | int | Yes | StockStatus (1=Good, 2=Defective, 3=Returned) |
| mode | boolean | Yes | `true` = increase quantity, `false` = decrease quantity |
| quantity | int | Yes | Quantity to adjust (positive number) |

**Response — 201 Created:** data contains the created adjustment line details.

**Sample Request:**

```json
{
  "productId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "stockLineId": null,
  "status": 1,
  "mode": false,
  "quantity": 5
}
```

**Possible Errors:** 401, 403, 400, 422, 429

---

### #22 — PUT /v1/inventories/adjustments/{adjustmentId}/lines/{adjustmentLineId}

Update an existing adjustment line.

- Permission: Adjustments.Create or Adjustments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |
| adjustmentLineId | string (UUID) | Yes | The adjustment line identifier |

**Request Body Fields:** Same as [#21 — Create Adjustment Line](#21--post-v1inventoriesadjustmentsadjustmentidlines).

**Response — 202 Accepted:** data contains the updated adjustment line details.

**Possible Errors:** 401, 403, 400, 404, 422, 429

---

### #23 — DELETE /v1/inventories/adjustments/{adjustmentId}/lines/{adjustmentLineId}

Remove an adjustment line.

- Permission: Adjustments.Create or Adjustments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |
| adjustmentLineId | string (UUID) | Yes | The adjustment line identifier |

**Request Body:** None

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400, 404, 429

---

## API Endpoints — Adjustment Documents

---

### #24 — GET /v1/inventories/adjustments/{adjustmentId}/documents/{documentId}

Get a specific adjustment document.

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |
| documentId | string (UUID) | Yes | The document identifier |

**Request Body:** None

**Response — 200 OK:** data contains document metadata.

**Possible Errors:** 401, 403, 404, 429

---

### #25 — GET /v1/inventories/adjustments/{adjustmentId}/documents

Get a paginated list of documents for an adjustment.

- Permission: Adjustments.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |

**Request Body:** None

**Response — 200 OK (Paginated):** data is an array of document metadata objects.

**Possible Errors:** 401, 403, 429

---

### #26 — GET /v1/inventories/adjustments/{adjustmentId}/documents/{documentId}/download

Download an adjustment document file.

- Permission: Adjustments.Read or Adjustments.Create or Adjustments.Modify
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |
| documentId | string (UUID) | Yes | The document identifier |

**Request Body:** None

**Response — 200 OK:** File stream with the appropriate MIME type and download filename.

**Possible Errors:** 401, 403, 400, 404, 429

---

### #27 — POST /v1/inventories/adjustments/{adjustmentId}/documents

Upload documents for an adjustment.

- Permission: Adjustments.Create or Adjustments.Modify
- Rate Limit: Write policy
- Content-Type: multipart/form-data

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |

**Form Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| documentTypeId | string (UUID) | Yes | The document type |
| remark | string | No | Additional notes |
| files | file[] | Yes | One or more files to upload |

**Response — 200 OK**

**Possible Errors:** 401, 403, 400, 422, 429

---

### #28 — DELETE /v1/inventories/adjustments/{adjustmentId}/documents/{documentId}

Delete (hide) an adjustment document.

- Permission: Adjustments.Create or Adjustments.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| adjustmentId | string (UUID) | Yes | The adjustment identifier |
| documentId | string (UUID) | Yes | The document identifier |

**Request Body:** None

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400, 404, 429
