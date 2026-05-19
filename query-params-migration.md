
## 2. `GET /api/port-relationships`

| Removed | Added |
|---------|-------|
| `query` | `code` — partial match on `Code` |
| `query` | `name` — partial match on `Name` |


---

## 3. `GET /api/campaigns`

| Removed | Added |
|---------|-------|
| `query` | `name` — partial match on `Name` and `Description` |


---

## 4. `GET /api/campaigns/{campaignId}/assigned-users`

| Removed | Added |
|---------|-------|
| `query` | `email` — partial match on `Contact.Email` |
| `query` | `name` — partial match on `Contact.FirstName` / `Contact.LastName` |


---

## 5. `GET /api/campaigns/{campaignId}/users`

| Removed | Added |
|---------|-------|
| `query` | `email` — partial match on `Email` |
| `query` | `name` — partial match on `FirstName` / `LastName` |


---

## 6. `GET /api/campaigns/{campaignId}/emails`

| Removed | Added |
|---------|-------|
| `query` | `name` — partial match on `Description` and `CampaignEmailNo` |


---

## 7. `GET /api/campaigns/{campaignId}/emails/{campaignEmailId}/contacts`

| Removed | Added |
|---------|-------|
| `query` | `email` — partial match on `Contact.Email` |
| `query` | `name` — partial match on `Contact.FirstName` / `Contact.LastName` |

---

## 8. `GET /api/campaigns/{campaignId}/emails/{campaignEmailId}/recipients`

| Removed | Added |
|---------|-------|
| `query` | `email` — partial match on `Contact.Email` |
| `query` | `name` — partial match on `Contact.FirstName` / `Contact.LastName` |


---

## 9. `GET /api/campaigns/{campaignId}/emails/{campaignEmailId}/logs`

| Removed | Added |
|---------|-------|
| `query` | `email` — partial match on `Sender` and `Receiver` |

---

## 10. `GET /api/companies/{companyId}/caring-teams` (CompanyCaringTeam list)

| Removed | Added |
|---------|-------|
| `query` | `email` — partial match on `Contact.Email` |
| `query` | `name` — partial match on `Contact.FirstName` / `Contact.LastName` |


---

## 11. `GET /api/companies/{companyId}/caring-teams` (CaringTeam info)

| Removed | Added |
|---------|-------|
| `query` | `email` — partial match on `Contact.Email` |


---

## 12. `GET /api/caring-teams` (companies for update caring team)

| Removed | Added |
|---------|-------|
| `query` | `code` — partial match on `Code` |
| `query` | `name` — partial match on `Name` / `ShortName` |


---

## 14. `GET /api/freight-catchers/{tradelaneId}/ports`

| Removed | Added |
|---------|-------|
| `query` | `code` — partial match on `Code` |
| `query` | `name` — partial match on `Name` |


---

## 15. `GET /api/freight-catchers/catcher-carriers`

| Removed | Added |
|---------|-------|
| `query` | `code` — partial match on `Carrier.Code` |
| `query` | `name` — partial match on `Carrier.Name` |


---

## 16. `GET /api/freight-catchers/{catcherCarrierId}/trade-lanes`

| Removed | Added |
|---------|-------|
| `query` | `code` — partial match on `TradeLane.Code` |
| `query` | `name` — partial match on `TradeLane.Name` |


---

## 17. `GET /api/media-libraries`

| Removed | Added |
|---------|-------|
| `query` | `filePath` — partial match on `FileStorage.FilePath` |


---

## 18. `GET /api/services`

| Removed | Added |
|---------|-------|
| `query` | `code` — partial match on `Code` |
| `query` | `name` — partial match on `Name` |


---

## 19. `GET /api/tags`

| Removed | Added |
|---------|-------|
| `query` | `name` — partial match on `Name` and `GroupTag.Name` |


---

## 20. `GET /api/teams` (tree teams)

| Removed | Added |
|---------|-------|
| `query` | `name` — partial match on `Name` |


---

## 21. `GET /api/teams`

| Removed | Added |
|---------|-------|
| `query` | `name` — partial match on `Name` |

---

## 22. `GET /api/template-signatures`

| Removed | Added |
|---------|-------|
| `query` | `description` — partial match on `Description` |


---

## 23. `GET /api/usertemplates/{templateId}/companies`

| Removed | Added |
|---------|-------|
| `query` | `code` — partial match on `Code` |
| `query` | `name` — partial match on `Name` |


---

## 24. `GET /api/usertemplates`

| Removed | Added |
|---------|-------|
| `query` | `description` — partial match on `Description` |

---

## 27. `GET /api/business-stages`

| Removed | Added |
|---------|-------|
| `query` | `code` — prefix match on `Code` (`value%`) |
| `query` | `name` — partial match on `Name` |

---

## 28. `GET /api/quotations`

| Removed | Added |
|---------|-------|
| `query` | `name` — partial match on `Receiver` and `Verifier` |


---

## 29. `GET /api/users` (export)

| Removed | Added |
|---------|-------|
| `query` | `email` — partial match on `Email` |
| `query` | `name` — partial match on `FirstName` / `LastName` |

---

## 30. `GET /api/users`

| Removed | Added |
|---------|-------|
| `query` | `email` — partial match on `User.Email` |
| `query` | `name` — partial match on `User.FirstName` / `User.LastName` |

---

## 31. Route-related endpoints

### `GET /api/routes`
| Removed | Added |
|---------|-------|
| `query` | `name` — partial match on `Name`, `FromAddress.Name`, `ToAddress.Name` |

---

### `GET /api/routechains`
| Removed | Added |
|---------|-------|
| `query` | `name` — partial match on `Name`, `FromAddress.Name`, `ToAddress.Name` |


---

### `GET /api/routegroups`
| Removed | Added |
|---------|-------|
| `query` | `code` — partial match on `Code` |
| `query` | `name` — partial match on `Name` |

