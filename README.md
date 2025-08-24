# Backend-Developer-Case-Study
# StockFlow – Take‑Home Submission



## Part 1 — Product Creation API

### Issues I found

* SKU uniqueness not enforced
* Product tied to one warehouse while it should support many
* Float for price (precision problems)
* No field validation, optional fields can break
* Two commits instead of one transaction
* Missing tenant checks (cross‑company issues)
* No error handling, retries create duplicates
* No audit trail for stock changes

### Impact

* Duplicates and inconsistent data in production
* Products not usable across multiple warehouses
* Wrong price calculations
* Orphan records if commit fails
* Security gaps across companies
* Reports and alerts not reliable

### Fixed version

* Enforced SKU uniqueness in DB + error handling
* Removed warehouse from Product model
* Used a single transaction for atomicity
* Added validations for inputs
* Added company scope check
* Allowed optional initial stock (default 0)
* Wrote stock movement to a ledger table

---

## Part 2 — Database Schema

### Design

* **Companies & Users**: multi‑tenant setup
* **Warehouses**: tied to companies
* **Products**: global SKU uniqueness, linked to product types
* **Product Types**: carry default low‑stock thresholds
* **Inventory**: per product per warehouse
* **Inventory Ledger**: records every stock movement
* **Suppliers & ProductSuppliers**: many‑to‑many, preferred flag
* **Bundles**: bundles link to component products
* **Sales Orders**: to track activity for alerts

### Decisions

* SKU globally unique (as stated)
* Numeric(12,2) for prices
* Ledger is immutable source of truth, inventory is a snapshot
* Indexes on hot paths (SKU, product+warehouse, ledger date)
* Threshold logic handled via product type or warehouse safety stock

### Questions I’d ask

* Should SKUs be global or per company long term?
* Currency handling per product or company?
* Thresholds: per type, per product, or per warehouse?
* How is “recent sales activity” defined?
* Bundles: stocked as kits or only virtual?
* Do we need reservations/allocations?

---

## Part 3 — Low‑Stock Alerts API

### Assumptions

* Threshold comes from product type or overridden by warehouse
* Recent sales = shipments in last 30 days
* Alerts only for products with activity
* ADS (average daily sales) = shipped\_qty / 30 days
* Supplier info included (preferred first)

### Endpoint

`GET /api/companies/{company_id}/alerts/low-stock`

Response format:

```json
{
  "alerts": [
    {
      "product_id": 123,
      "product_name": "Widget A",
      "sku": "WID-001",
      "warehouse_id": 456,
      "warehouse_name": "Main Warehouse",
      "current_stock": 5,
      "threshold": 20,
      "days_until_stockout": 12,
      "supplier": {
        "id": 789,
        "name": "Supplier Corp",
        "contact_email": "orders@supplier.com"
      }
    }
  ],
  "total_alerts": 1
}
```

### Edge cases handled

* No sales → no alert
* No supplier → still alert with null supplier
* Negative or zero stock → flagged
* Multi‑warehouse supported
* Scoped by company

---

