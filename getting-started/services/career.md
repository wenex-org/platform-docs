---
title: "Career Service: Businesses, Staff and Stock"
description: "The Wenex career service: businesses, branches, employees, customers, products, services, stores and inventory stock, with query tips."
---

# Career

**Port:** REST `:3140` · gRPC `:5140`

The largest domain service, managing all business-side entities: businesses, branches, employees, customers, products, services, stores, and inventory stock.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Businesses | `/career/businesses` | Top-level business or organization |
| Branches | `/career/branches` | Business subdivisions |
| Employees | `/career/employees` | Staff records |
| Customers | `/career/customers` | Customer records |
| Products | `/career/products` | Product catalog items |
| Services | `/career/services` | Service offerings |
| Stores | `/career/stores` | Physical or logical stores / warehouses |
| Stocks | `/career/stocks` | Inventory records per product |

## `career/businesses`

Top-level business or organization record.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Business name |
| `type` | ✅ | `BusinessType` | `CHARITY`, `INSURANCE`, `GOVERNMENT`, `INDIVIDUAL`, `PARTNERSHIP`, `COOPERATIVE`, `CORPORATION`, `NONPROFIT` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |

### Common Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `code` | string | Short identifier |
| `alias` | string | Alternative name |
| `slogan` | string | Slogan |
| `website` | URL | Website URL |
| `address` | string | Address |
| `location` | MongoId | `logistic/locations` reference |
| `categories` | string[] | Category labels |
| `logo` / `cover` | MongoId | `special/files` references (raw, not populatable) |
| `foundation_date` | date | Foundation date |

### Population

| Path | Collection |
| --- | --- |
| `location` | `logistic/locations` |

## `career/branches`

Business subdivision — the business's origin branch or one of its other branches.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `BranchType` | `ORIGIN`, `BRANCH` |
| `business` | ✅ | MongoId | Parent business |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |

### Population

| Path | Collection |
| --- | --- |
| `parent` | `career/branches` |
| `manager` | `career/employees` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |

## `career/employees`

Staff record linked to a business, optionally to a branch, manager, and services.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `EmployeeType` | `CONTRACT`, `FULL_TIME`, `PART_TIME`, `TEMPORARY` |
| `job_title` | ✅ | string | Role / job title |
| `business` | ✅ | MongoId | Parent business |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |

### Population

| Path | Collection |
| --- | --- |
| `branch` | `career/branches` |
| `manager` | `career/employees` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |
| `currency` | `financial/currencies` |
| `profile` | `identity/profiles` |

## `career/customers`

Customer record associated with a business.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `CustomerType` | `VIP`, `LOYAL`, `REGULAR`, `OCCASIONAL` |
| `business` | ✅ | MongoId | Parent business |

### Population

| Path | Collection |
| --- | --- |
| `profile` | `identity/profiles` |
| `stores` | `career/stores` |
| `branch` | `career/branches` |
| `services` | `career/services` |
| `business` | `career/businesses` |
| `employees` | `career/employees` |
| `location` / `addresses` | `logistic/locations` |
| `documents` / `certifications` | `special/files` |

## `career/products`

Catalog item offered by a business, branch, or store.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Product name |

### Nested `features` Object

Products may have a `features` array of nested objects — each with `type` (`ProductFeatureType`: `TEXT`, `RANGE`, `ORDINAL`, `DISCRETE`, `CATEGORICAL`), `title` (e.g. `Color`), and `value` (boolean, number or string). Features are embedded objects, not referenced IDs.

### Population

| Path | Collection |
| --- | --- |
| `store` | `career/stores` |
| `branch` | `career/branches` |
| `business` | `career/businesses` |
| `cover` / `gallery` | `special/files` |

## `career/services`

Service offering attached to a business.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Service name |
| `type` | ✅ | `ServiceType` | `PERIODIC`, `ON_DEMAND` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |
| `business` | ✅ | MongoId | Parent business |

### Population

| Path | Collection |
| --- | --- |
| `branch` | `career/branches` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |

## `career/stores`

Physical or logical store / warehouse.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Store name |
| `type` | ✅ | `StoreType` | `RETAIL`, `WHOLESALE`, `FRANCHISE`, `MARKETPLACE`, `DISTRIBUTOR`, `DROPSHIPPING` |
| `fork` | ✅ | `StoreFork` | `ORIGIN`, `BRANCH` |
| `business` | ✅ | MongoId | Parent business |

### Population

| Path | Collection |
| --- | --- |
| `parent` | `career/stores` |
| `manager` | `career/employees` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |

## `career/stocks`

Inventory entry for a product.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `StockType` | `COLD`, `BULKY`, `NORMAL`, `FROZEN`, `FRAGILE`, `VALUABLE`, `HAZARDOUS` |
| `product` | ✅ | MongoId | Parent product |
| `inventory` | ✅ | number | Current inventory level |

### Population

| Path | Collection |
| --- | --- |
| `store` | `career/stores` |
| `branch` | `career/branches` |
| `product` | `career/products` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |
| `currency` | `financial/currencies` |

## Query Tips

- Scope most branch, employee, service, store, customer, product, and stock queries by `business`.
- Use `branch` for branch-local records.
- Use `store` for retail- or warehouse-local products and stocks.
