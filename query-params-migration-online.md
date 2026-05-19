# Query Params Migration

## 1. `GET /v1/carriers` · `GET /v1/carriers` (Portal)
| Removed | Added |
|---|---|
| `query` | `code` — partial match on `Code` |
| `query` | `name` — partial match on `Name` |

---

## 2. `GET /v1/countries` · `GET /v1/countries` (Portal)
| Removed | Added |
|---|---|
| `query` | `code` — partial match on `Code`, `Alpha2Code`, `Alpha3Code` |
| `query` | `name` — partial match on `Name` |

---

## 3. `GET /v1/ports` · `GET /v1/ports` (Portal)
| Removed | Added |
|---|---|
| `query` | `code` — partial match on `Code` |
| `query` | `name` — partial match on `Name` |

---

## 4. `GET /v1/commodities` · `GET /v1/commodities` (Portal)
| Removed | Added |
|---|---|
| `query` | `name` — partial match on `Name` |

---

## 5. `GET /v1/administrative-units` · `GET /v1/administrative-units` (Portal)
| Removed | Added |
|---|---|
| `query` | `name` — full-text search on `Name` |

---

## 6. `GET /v1/companies`
| Removed | Added |
|---|---|
| `query` (text search) | `code` — partial match on `Code` |
| `query` (text search) | `name` — partial match on `Name` |
| `query` (text search) | `shortName` — partial match on `ShortName` |
| `query` (e.g. `@gmail.com`) | `emailDomain` — match companies whose member email contains `@{domain}` |

> `emailDomain` and text fields (`code`, `name`, `shortName`) are mutually exclusive. When `emailDomain` is provided, text filters are ignored.

---

## 7. `GET /v1/units` · `GET /v1/units` (Portal)
| Removed | Added |
|---|---|
| `query` | `code` — partial match on `Code` |
| `query` | `name` — partial match on `Name` |

---

## 8. `GET /v1/contacts` · `GET /v1/contacts/count-by-roles`
| Removed | Added |
|---|---|
| `firstName` | `name` — partial match on `FirstName` OR `LastName` |
| `lastName` | `name` — partial match on `FirstName` OR `LastName` |

> `email` is unchanged.
