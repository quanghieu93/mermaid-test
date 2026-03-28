# Product Apis Documentation

> Base Route: /v{version:apiVersion}/products
> API Version: 1.0
> Authentication: OAuth2 Bearer Token (required on all endpoints)
> Authorization: Client credentials with scope "public-api" + resource permissions
> Rate Limiting: All endpoints are rate-limited (read/write/import policies)
> Company Scope: The caller's company is resolved from the authenticated token.

---

## Design Principles

This API is designed for **external systems** (merchants, third-party platforms) to manage their product catalog. The following principles guide what is exposed:

1. **`productId` is returned in responses** — External systems need this identifier to perform subsequent operations (update, delete, change dimensions, etc.). This is your reference to the product in our system.
2. **Company context is implicit** — Company identity is derived from the authentication token. You cannot see or access another company's data.
3. **Internal operational data is not exposed** — Fields like WMS sync status, source type, and internal user IDs are omitted from documentation.
4. **Input fields use IDs when you need to reference existing entities** — When creating or updating products, you pass `productUnitId`, `weightUnitId`, `dimensionUnitId`, `categoryIds`, etc. as input because you need to tell us which entity to use. These IDs are provided to you via separate lookup endpoints (e.g., GET /v1/units, GET /v1/categories).

---

## Enum Reference

### ProductStatus
- 1 = Active
- 2 = Inactive

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
      "propertyName": "code",
      "messages": [
        "Code is required."
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
  "error": "Product code already exists.",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

### 404 Not Found (errorCode: 404)

```json
{
  "success": false,
  "errorCode": "404",
  "error": "Product not found.",
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

## API Endpoints

---

### #1 — GET /v1/products

Get a paginated list of products.

- Permission: Products.Read
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| pageIndex | int | No | Page number (1-based). Default: 1 |
| pageSize | int | No | Items per page. Default: 20, Max: 100 |
| code | string | No | Filter by product code (SKU) |
| name | string | No | Filter by product name |
| query | string | No | Free-text search across code and name |
| categoryIds | string (UUID)[] | No | Filter by one or more category IDs. Pass multiple: `?categoryIds=uuid1&categoryIds=uuid2` |
| status | int | No | Filter by ProductStatus (1=Active, 2=Inactive) |
| isValid | boolean | No | Filter by validation status |
| isDeleted | boolean | No | Include deleted products |

> **Note:** Do **not** include `companyId` in the query parameters. It is automatically set from your authenticated context.

**Request Body:** None

**Response — 200 OK (Paginated):**

data is an array of product summaries.

**Possible Errors:** 401, 403, 429

---

### #2 — GET /v1/products/{productId}

Get full details of a product.

- Permission: Products.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productId | string (UUID) | Yes | The product identifier |

**Request Body:** None

**Response — 200 OK:**

data is a product detail object.

**Possible Errors:** 401, 403, 404, 429

---

### #3 — POST /v1/products

Create a new product.

- Permission: Products.Create
- Rate Limit: Write policy

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| categoryIds | string (UUID)[] or null | No | Category IDs to assign the product to |
| code | string | Yes | Product code (SKU). Must be unique within the company. |
| name | string | Yes | Product name |
| asin | string or null | No | Amazon Standard Identification Number |
| labelName | string or null | No | Label / display name |
| hsCode | string or null | No | Harmonized System code for customs |
| productUnitId | string (UUID) | Yes | Base unit of measure. Obtain from GET /v1/units. |
| netWeight | number or null | No | Net weight per unit |
| grossWeight | number | Yes | Gross weight per unit |
| weightUnitId | string (UUID) | Yes | Weight unit. Obtain from GET /v1/units. |
| length | number | Yes | Product length |
| width | number | Yes | Product width |
| height | number | Yes | Product height |
| dimensionUnitId | string (UUID) | Yes | Dimension unit. Obtain from GET /v1/units. |
| thick | number or null | No | Thickness |
| thickUnitId | string (UUID) or null | No | Thickness unit |
| color | string or null | No | Product color |
| size | string or null | No | Product size label (e.g., "XL", "10x20") |
| packageLength | number | Yes | Package length |
| packageWidth | number | Yes | Package width |
| packageHeight | number | Yes | Package height |
| packagingUnitId | string (UUID) | Yes | Packaging dimension unit. Obtain from GET /v1/units. |
| cubicUnitId | string (UUID) | Yes | Cubic/volume unit. Obtain from GET /v1/units. |
| packageNetWeight | number or null | No | Package net weight |
| packageGrossWeight | number or null | No | Package gross weight |
| packageWeightUnitId | string (UUID) | Yes | Package weight unit. Obtain from GET /v1/units. |
| quantityPerPackage | int | Yes | Number of units per package |
| minStock | int | Yes | Minimum stock level (reorder threshold) |
| notes | string or null | No | Free-text notes |
| status | int | Yes | ProductStatus (1=Active, 2=Inactive) |
| containBattery | boolean | Yes | Whether the product contains a battery (affects shipping/customs) |
| declaredValue | string | Yes | Declared customs value |

**Response — 201 Created:**

data contains the full product detail object. A `Location` header points to the new product.

**Sample Request:**

```json
{
  "categoryIds": ["3fa85f64-5717-4562-b3fc-2c963f66afa6"],
  "code": "SKU-001",
  "name": "Widget A",
  "asin": "B08N5WRWNW",
  "labelName": "Widget A - Blue",
  "hsCode": "8471.30",
  "productUnitId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "netWeight": 0.5,
  "grossWeight": 0.6,
  "weightUnitId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "length": 10,
  "width": 5,
  "height": 3,
  "dimensionUnitId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "thick": null,
  "thickUnitId": null,
  "color": "Blue",
  "size": "M",
  "packageLength": 12,
  "packageWidth": 7,
  "packageHeight": 5,
  "packagingUnitId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "cubicUnitId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "packageNetWeight": 0.55,
  "packageGrossWeight": 0.7,
  "packageWeightUnitId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "quantityPerPackage": 1,
  "minStock": 50,
  "notes": "Fragile",
  "status": 1,
  "containBattery": false,
  "declaredValue": "15.00"
}
```

**Possible Errors:** 401, 403, 400 (duplicate code, domain error), 422, 429

---

### #4 — PATCH /v1/products/{productId}

Partially update a product's basic information (label name, min stock, notes, status). Use this for quick status changes without sending the full product payload.

- Permission: Products.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productId | string (UUID) | Yes | The product identifier |

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| labelName | string or null | No | Updated label / display name |
| minStock | int | Yes | Updated minimum stock level |
| notes | string or null | No | Updated notes |
| status | int | Yes | ProductStatus (1=Active, 2=Inactive) |

**Response — 202 Accepted:**

data contains the updated product detail object.

**Sample Request:**

```json
{
  "labelName": "Widget A - Red (Updated)",
  "minStock": 100,
  "notes": "Updated notes",
  "status": 1
}
```

**Possible Errors:** 401, 403, 400 (domain error), 404, 422, 429

---

### #5 — PUT /v1/products/{productId}

Fully update a product. All fields must be provided.

- Permission: Products.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productId | string (UUID) | Yes | The product identifier |

**Request Body Fields:** Same as [Create Product](#3--post-v1products), with all the same requirements.

**Response — 202 Accepted:**

data contains the updated product detail object.

**Possible Errors:** 401, 403, 400 (duplicate code, domain error), 404, 422, 429

---

### #6 — DELETE /v1/products/{productId}

Delete a product.

- Permission: Products.Delete
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productId | string (UUID) | Yes | The product identifier |

**Request Body:** None

**Response — 204 No Content** (no body)

**Possible Errors:** 401, 403, 400 (product is in use), 404, 429

---

### #7 — POST /v1/products/{productId}/change-weight-size

Update only the weight and dimension fields of a product, without affecting other product information.

- Permission: Products.Modify
- Rate Limit: Write policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productId | string (UUID) | Yes | The product identifier |

**Request Body Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| netWeight | number or null | No | Updated net weight per unit |
| grossWeight | number | Yes | Updated gross weight per unit |
| weightUnitId | string (UUID) | Yes | Weight unit. Obtain from GET /v1/units. |
| length | number | Yes | Updated product length |
| width | number | Yes | Updated product width |
| height | number | Yes | Updated product height |
| dimensionUnitId | string (UUID) | Yes | Dimension unit. Obtain from GET /v1/units. |

**Response — 202 Accepted:**

data contains the updated product detail object.

**Sample Request:**

```json
{
  "netWeight": 0.55,
  "grossWeight": 0.65,
  "weightUnitId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "length": 11,
  "width": 5.5,
  "height": 3.2,
  "dimensionUnitId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

**Possible Errors:** 401, 403, 400 (domain error), 404, 422, 429

---

### #8 — POST /v1/products/import

Import products from an Excel file.

- Permission: Products.Import
- Rate Limit: Import policy
- Content-Type: multipart/form-data

**Form Fields:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| file | File | Yes | Excel file. Allowed: `.xls`, `.xlsx` only. |

> **Note:** `companyId` is **not** required. It is automatically resolved from your authenticated context.

**Response — 200 OK:** data contains the import result summary.

**Sample Error — Wrong file type:**

```json
{
  "success": false,
  "errorCode": "99",
  "error": "File type is not supported!",
  "timestamp": "2026-03-24T10:00:00Z"
}
```

**Possible Errors:** 401, 403, 400 (file type/domain), 422, 429

---

### #9 — GET /v1/products/export

Export products to an Excel file.

- Permission: Products.Export
- Rate Limit: Read policy

**Query Parameters:**

| Field | Type | Required | Description / Constraints |
|-------|------|----------|---------------------------|
| isValid | boolean | No | Filter by validation status |
| categoryIds | string (UUID)[] | No | Filter by category IDs. Pass multiple: `?categoryIds=uuid1&categoryIds=uuid2` |

**Request Body:** None

**Response — 200 OK:** File download (`.xlsx`). The response `Content-Type` will be set to the appropriate MIME type and the `Content-Disposition` header will include the filename.

**Possible Errors:** 401, 403, 400 (domain error), 429

---

### #10 — GET /v1/products/{productId}/dimension-history

Get a paginated list of dimension change history for a product. Tracks changes to weight and size over time.

- Permission: Products.Read
- Rate Limit: Read policy

**Route Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productId | string (UUID) | Yes | The product identifier |

**Query Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| pageIndex | int | No | Default: 1 |
| pageSize | int | No | Default: 20, Max: 100 |

**Request Body:** None

**Response — 200 OK (Paginated):**

data is an array of dimension history records.

**Possible Errors:** 401, 403, 404, 429
