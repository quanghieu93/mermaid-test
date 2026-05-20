# Query Params Migration

> All text search params use case-insensitive partial match (`ILIKE %value%`).
> When both params are provided they are combined with **OR** (broader search).
> When only one is provided it filters on that field only.

---

## 1. `GET /v1/carriers` · `GET /v1/carriers` (Portal)

| Removed | Added | Searches |
|---|---|---|
| `query` | `code` | `Code` |
| `query` | `name` | `Name` |

**Combined:** `code` + `name` → `Code ILIKE OR Name ILIKE`

---

## 2. `GET /v1/countries` · `GET /v1/countries` (Portal)

| Removed | Added | Searches |
|---|---|---|
| `query` | `code` | `Code`, `Alpha2Code`, `Alpha3Code` |
| `query` | `name` | `Name` |

**Combined:** `code` + `name` → `Code ILIKE OR Alpha2Code ILIKE OR Alpha3Code ILIKE OR Name ILIKE`

---

## 3. `GET /v1/ports` · `GET /v1/ports` (Portal)

| Removed | Added | Searches |
|---|---|---|
| `query` | `code` | `Code` |
| `query` | `name` | `Name` |

**Combined:** `code` + `name` → `Code ILIKE OR Name ILIKE`

---

## 4. `GET /v1/commodities` · `GET /v1/commodities` (Portal)

| Removed | Added | Searches |
|---|---|---|
| `query` | `name` | `Name` |

---

## 5. `GET /v1/administrative-units` · `GET /v1/administrative-units` (Portal)

| Removed | Added | Searches |
|---|---|---|
| `query` | `name` | Full-text search on `Name` (PostgreSQL `phraseto_tsquery`) |

---

## 6. `GET /v1/companies`

| Removed | Added | Searches |
|---|---|---|
| `query` (text search) | `code` | `Code` |
| `query` (text search) | `name` | `Name` |
| `query` (text search) | `shortName` | `ShortName` |
| `query` (`@gmail.com`) | `emailDomain` | Member email contains `@{domain}` |

> `emailDomain` and text fields (`code`, `name`, `shortName`) are mutually exclusive.  
> When `emailDomain` is provided, text filters are ignored.

---

## 7. `GET /v1/units` · `GET /v1/units` (Portal)

| Removed | Added | Searches |
|---|---|---|
| `query` | `code` | `Code` |
| `query` | `name` | `Name` |

**Combined:** `code` + `name` → `Code ILIKE OR Name ILIKE`

---

## 8. `GET /v1/contacts` · `GET /v1/contacts/count-by-roles`

| Removed | Added | Searches |
|---|---|---|
| `firstName` | `name` | `FirstName`, `LastName` |
| `lastName` | `name` | `FirstName`, `LastName` |
| *(unchanged)* | `email` | `Email` |

**Combined:** `name` + `email` → `FirstName ILIKE OR LastName ILIKE OR Email ILIKE`

---

## 9. `GET /v1/contacts/subscriptions`

| Removed | Added | Searches |
|---|---|---|
| `query` | `name` | `FirstName`, `LastName` |
| `query` | `email` | `Email` |

**Combined:** `name` + `email` → `FirstName ILIKE OR LastName ILIKE OR Email ILIKE`

---

## 10. `GET /v1/contact-suggestions` · `GET /v1/contact-suggestions/count-by-status`

| Removed | Added | Searches |
|---|---|---|
| `query` | `name` | JSON contact `name` field |
| `query` | `email` | JSON contact `email` field |

**Combined:** `name` + `email` → JSON `name ILIKE OR email ILIKE`

