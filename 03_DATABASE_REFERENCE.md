# Database Reference Guide

## Sistema Abuelo Cómodo v2.0

---

## The Relational Foundation

Where spreadsheets once held data in flat, denormalized rows—relationships implied rather than enforced, integrity maintained by application logic and human vigilance—a structured relational model now defines the data landscape. Twenty-nine tables interconnected by fifty-two foreign key constraints form the backbone of the inventory management system.

This reference documents every table, column, constraint, and function that comprises the PostgreSQL schema. It serves as the authoritative source for engineers extending, maintaining, or troubleshooting the system.

---

## Schema Overview

### Table Inventory by Domain

| Domain | Tables | Records | Purpose |
|--------|--------|---------|---------|
| Core Entities | 7 | 13,624 | Master data: customers, products, locations |
| Order Processing | 4 | 21,420 | Transactions and fulfillment |
| Inventory Management | 3 | 1,235 | Stock tracking and recipes |
| Prospects/CRM | 5 | 8,107 | Lead management pipeline |
| Purchasing | 4 | 0 | Vendor orders (pending migration) |
| Accounting | 2 | 226 | Financial entries and rules |
| Configuration | 4 | 47 | System settings and reference data |

---

## Core Entity Tables

### customers

The central customer registry, consolidating buyer information from Shopify transactions and phone sales.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| customer_id | UUID | NO | gen_random_uuid() | Primary key |
| shopify_customer_id | BIGINT | YES | - | Shopify integration reference |
| first_name | VARCHAR(100) | NO | - | Customer first name |
| last_name | VARCHAR(100) | YES | - | Customer last name |
| email | VARCHAR(255) | YES | - | Email address (unique when present) |
| phone | VARCHAR(20) | YES | - | Primary phone number |
| total_orders | INTEGER | YES | 0 | Aggregate order count |
| total_spent | NUMERIC | YES | 0 | Aggregate purchase value |
| accepts_marketing | BOOLEAN | YES | false | Marketing consent flag |
| tags | TEXT | YES | - | Shopify customer tags |
| note | TEXT | YES | - | Internal notes |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | YES | now() | Last modification timestamp |

**Indexes**: PRIMARY KEY (customer_id), UNIQUE (shopify_customer_id), UNIQUE (email)

**Triggers**: Updated by trg_update_customer_metrics on orders INSERT/UPDATE

---

### products

The product catalog, including both simple items and composite products with recipes.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| item_id | VARCHAR(50) | NO | - | Primary key (SKU) |
| product_id | UUID | YES | gen_random_uuid() | Secondary identifier |
| name | VARCHAR(255) | NO | - | Product display name |
| description | TEXT | YES | - | Product description |
| category_id | UUID | YES | - | FK to categories |
| vendor_id | UUID | YES | - | FK to vendors |
| purchase_cost | NUMERIC(10,2) | YES | 0 | Unit cost from supplier |
| sale_price | NUMERIC(10,2) | YES | 0 | Standard selling price |
| shopify_product_id | BIGINT | YES | - | Shopify product reference |
| shopify_variant_id | BIGINT | YES | - | Shopify variant reference |
| shopify_inventory_item_id | BIGINT | YES | - | Shopify inventory item ID |
| is_composite | BOOLEAN | YES | false | Has recipe components |
| is_active | BOOLEAN | YES | true | Available for sale |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | YES | now() | Last modification timestamp |

**Indexes**: PRIMARY KEY (item_id), INDEX on category_id, INDEX on vendor_id

**Foreign Keys**: category_id → categories.category_id, vendor_id → vendors.vendor_id

---

### categories

Hierarchical product categorization supporting parent-child relationships.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| category_id | UUID | NO | gen_random_uuid() | Primary key |
| name | VARCHAR(100) | NO | - | Category name (unique) |
| description | TEXT | YES | - | Category description |
| parent_category_id | UUID | YES | - | FK to parent category |
| icon | VARCHAR(255) | YES | - | Icon identifier for UI |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |

**Self-Reference**: parent_category_id → categories.category_id

---

### locations

Physical warehouse and store locations for inventory management.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| location_id | VARCHAR(20) | NO | - | Primary key |
| name | VARCHAR(100) | NO | - | Location display name |
| code | VARCHAR(10) | NO | - | Short code (PR, TP) |
| shopify_location_id | VARCHAR(50) | YES | - | Shopify location mapping |
| address | TEXT | YES | - | Physical address |
| is_active | BOOLEAN | YES | true | Location operational status |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |

**Current Data**:
- 21449925: Playa Regatas (PR)
- 63600984166: Tienda Pilares (TP)

---

### vendors

Supplier registry for purchase order management.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| vendor_id | UUID | NO | gen_random_uuid() | Primary key |
| name | VARCHAR(255) | NO | - | Vendor company name |
| contact_name | VARCHAR(100) | YES | - | Primary contact person |
| email | VARCHAR(255) | YES | - | Contact email |
| phone | VARCHAR(50) | YES | - | Contact phone |
| address | TEXT | YES | - | Vendor address |
| notes | TEXT | YES | - | Internal notes |
| is_active | BOOLEAN | YES | true | Active supplier flag |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |

---

### advisors

Sales team members who create phone orders.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| advisor_id | VARCHAR(20) | NO | - | Primary key |
| name | VARCHAR(100) | NO | - | Advisor full name |
| prefix | VARCHAR(10) | NO | - | Order ID prefix (e.g., ANA, GAB) |
| appsheet_name | VARCHAR(100) | YES | - | Name as displayed in AppSheet |
| is_active | BOOLEAN | YES | true | Currently active flag |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | YES | now() | Last modification timestamp |

**Usage**: Prefix used in order_id generation: `{prefix}-TLF-{sequence}`

---

### addresses

Normalized address storage for prospects and customers.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| address_id | VARCHAR(20) | NO | - | Primary key |
| calle | VARCHAR(100) | NO | - | Street address |
| colonia | VARCHAR(100) | NO | - | Neighborhood/Colony |
| ciudad | VARCHAR(100) | NO | - | City |
| estado | USER-DEFINED | NO | - | Mexican state (enum) |
| codigo_postal | VARCHAR(100) | NO | - | Postal code |
| punto_referencia | VARCHAR(100) | NO | - | Delivery reference point |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | YES | now() | Last modification timestamp |

---

## Order Processing Tables

### orders

The central transaction table capturing all sales across channels.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| order_id | VARCHAR(50) | NO | - | Primary key |
| shopify_order_id | VARCHAR(50) | YES | - | Shopify order reference |
| order_number | VARCHAR(50) | YES | - | Display order number |
| shopify_order_number | BIGINT | YES | - | Shopify numeric order number |
| customer_id | UUID | YES | - | FK to customers |
| prospect_id | VARCHAR(50) | YES | - | FK to prospects (phone orders) |
| advisor_id | VARCHAR(20) | YES | - | FK to advisors (phone orders) |
| location_id | VARCHAR(20) | YES | - | FK to fulfillment location |
| address_id | VARCHAR(20) | YES | - | FK to delivery address |
| customer_name | VARCHAR(255) | YES | - | Denormalized customer name |
| customer_email | VARCHAR(255) | YES | - | Denormalized email |
| customer_phone | VARCHAR(50) | YES | - | Denormalized phone |
| direccion | TEXT | YES | - | Delivery address (legacy) |
| ciudad_colonia | TEXT | YES | - | City/colony (legacy) |
| subtotal | NUMERIC(10,2) | YES | 0 | Pre-discount total |
| discount | NUMERIC(10,2) | YES | 0 | Discount amount |
| total_price | NUMERIC(10,2) | YES | 0 | Final order total |
| first_payment | NUMERIC(10,2) | YES | - | First payment amount |
| first_payment_date | DATE | YES | - | First payment date |
| first_payment_method | VARCHAR(50) | YES | - | FK to payment_methods |
| second_payment | NUMERIC(10,2) | YES | - | Second payment amount |
| second_payment_method | VARCHAR(50) | YES | - | FK to payment_methods |
| third_payment | NUMERIC(10,2) | YES | - | Third payment amount |
| third_payment_method | VARCHAR(50) | YES | - | FK to payment_methods |
| fourth_payment | NUMERIC(10,2) | YES | - | Fourth payment amount |
| fourth_payment_method | VARCHAR(50) | YES | - | FK to payment_methods |
| fifth_payment | NUMERIC(10,2) | YES | - | Fifth payment amount |
| fifth_payment_method | VARCHAR(50) | YES | - | FK to payment_methods |
| order_status | VARCHAR(50) | YES | - | Business status (Compra, Pendiente, etc.) |
| financial_status | VARCHAR(50) | YES | - | Payment status |
| fulfillment_status | VARCHAR(50) | YES | 'unfulfilled' | Shipping status |
| warehouse_status | VARCHAR(50) | YES | - | Warehouse processing status |
| order_type | VARCHAR(50) | YES | - | Venta, Garantía, Intercambio, Obsequio |
| procedencia | VARCHAR(100) | YES | - | Order origin |
| source_type | VARCHAR(50) | YES | - | Shopify, Phone, etc. |
| channel | VARCHAR(50) | YES | - | Sales channel |
| delivery_method | VARCHAR(100) | YES | - | Shipping method |
| order_date | DATE | YES | - | Order placement date |
| order_timestamp | TIMESTAMPTZ | YES | - | Precise order time |
| dispatch_date | DATE | YES | - | Fulfillment date |
| digit | INTEGER | YES | - | Sequence number for phone orders |
| app_id | VARCHAR(50) | YES | - | Shopify app identifier |
| order_source | VARCHAR(50) | YES | - | Source system |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | YES | now() | Last modification timestamp |

**Foreign Keys**: customer_id → customers, prospect_id → prospects, advisor_id → advisors, location_id → locations, address_id → addresses, payment methods → payment_methods

**Triggers**: 
- trg_generate_order_id (BEFORE INSERT)
- trg_update_customer_metrics (AFTER INSERT/UPDATE)
- trg_process_order_dispatch (BEFORE UPDATE)

---

### order_items

Line items within orders, one record per product per order.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| order_item_id | VARCHAR(50) | NO | - | Primary key |
| order_id | VARCHAR(50) | NO | - | FK to orders |
| item_id | VARCHAR(50) | NO | - | FK to products (SKU) |
| item_name | VARCHAR(255) | YES | - | Product name at time of order |
| quantity | INTEGER | NO | 1 | Quantity ordered |
| unit_price | NUMERIC(10,2) | YES | - | Price per unit |
| total_price | NUMERIC(10,2) | YES | - | Line total (qty × price) |
| unit_cost | NUMERIC(10,2) | YES | - | Cost per unit |
| total_item_cost | NUMERIC(10,2) | YES | - | Line cost (qty × cost) |
| total_paid | NUMERIC(10,2) | YES | - | Amount paid for this line |
| order_date | DATE | YES | - | Denormalized order date |
| order_timestamp | TIMESTAMPTZ | YES | - | Denormalized order time |
| computed_key | VARCHAR(100) | YES | - | Composite key for lookups |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |

**Foreign Keys**: order_id → orders, item_id → products

**Triggers** (all AFTER INSERT):
- trigger_auto_process_recipes → Expands composite products
- trg_update_reserved_on_order → Reserves inventory
- trigger_generate_accounting → Creates accounting entries

---

### order_item_elements

Component breakdown for composite products, generated by recipe trigger.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| element_id | VARCHAR(50) | NO | - | Primary key |
| order_item_id | VARCHAR(50) | NO | - | FK to parent order_items |
| order_id | VARCHAR(50) | NO | - | FK to orders |
| parent_item_id | VARCHAR(50) | NO | - | FK to composite product |
| element_item_id | VARCHAR(50) | NO | - | FK to component product |
| element_name | VARCHAR(255) | YES | - | Component product name |
| quantity | NUMERIC | NO | - | Component quantity |
| element_cost | NUMERIC(10,2) | YES | - | Unit cost of component |
| total_cost_element | NUMERIC(10,2) | YES | - | Total cost (qty × cost) |
| order_timestamp | TIMESTAMPTZ | YES | - | Order timestamp |
| order_date | DATE | YES | - | Order date |
| computed_key | VARCHAR(100) | YES | - | Composite key for lookups |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |

**Foreign Keys**: order_item_id → order_items, order_id → orders, parent_item_id → products, element_item_id → products

---

### dispatch_records

Fulfillment documentation tracking what left the warehouse.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| dispatch_id | VARCHAR | NO | - | Primary key |
| order_item_id | VARCHAR | YES | - | FK to order_items |
| order_id | VARCHAR | YES | - | FK to orders |
| item_id | VARCHAR | YES | - | FK to products |
| location_id | VARCHAR | YES | - | FK to fulfillment location |
| dispatch_date | DATE | NO | - | Dispatch date |
| quantity_dispatched | INTEGER | NO | - | Quantity shipped |
| inventory_name | VARCHAR | YES | - | Product name for reference |
| motivo | VARCHAR | YES | - | Dispatch reason/notes |
| unit_cost | NUMERIC | YES | - | Unit cost at dispatch |
| total_cost | NUMERIC | YES | - | Total cost dispatched |
| order_cancelled | BOOLEAN | YES | false | Cancellation flag |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |

**Foreign Keys**: order_item_id → order_items, order_id → orders, item_id → products, location_id → locations

---

## Inventory Management Tables

### inventory

Stock levels by product and location.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| inventory_id | UUID | NO | gen_random_uuid() | Primary key |
| item_id | VARCHAR(50) | NO | - | FK to products |
| location_id | VARCHAR(20) | NO | - | FK to locations |
| quantity_on_hand | NUMERIC | YES | 0 | Physical stock count |
| quantity_reserved | NUMERIC | YES | 0 | Allocated to orders |
| quantity_incoming | NUMERIC | YES | 0 | Expected from POs |
| virtual_quantity | NUMERIC | YES | 0 | Calculated available |
| last_updated | TIMESTAMPTZ | YES | now() | Last stock change |

**Constraints**: UNIQUE (item_id, location_id)

**Foreign Keys**: item_id → products, location_id → locations

---

### inventory_movements

Audit trail of all stock changes.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| movement_id | UUID | NO | gen_random_uuid() | Primary key |
| item_id | VARCHAR(50) | NO | - | FK to products |
| location_id | VARCHAR(20) | NO | - | FK to locations |
| movement_type | VARCHAR(20) | NO | - | entrada, salida, transfer_in, transfer_out, ajuste |
| quantity | NUMERIC | NO | - | Change amount (+/-) |
| previous_quantity | INTEGER | YES | - | Stock before movement |
| new_quantity | INTEGER | YES | - | Stock after movement |
| reference_type | VARCHAR(50) | YES | - | order, purchase_order, transfer |
| reference_id | VARCHAR(50) | YES | - | Related record ID |
| notes | TEXT | YES | - | Movement notes |
| user_id | VARCHAR(50) | YES | - | User who made change |
| created_at | TIMESTAMPTZ | YES | now() | Movement timestamp |

**Triggers**: trg_update_inventory_from_movement (BEFORE INSERT) → Updates inventory.quantity_on_hand

---

### product_recipes

Component definitions for composite products.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| recipe_id | UUID | NO | gen_random_uuid() | Primary key |
| product_item_id | VARCHAR(50) | NO | - | FK to composite product |
| element_item_id | VARCHAR(50) | NO | - | FK to component product |
| quantity_per_product | NUMERIC | NO | 1 | Components per unit |
| recipe_type | VARCHAR(20) | YES | 'SALIDA' | SALIDA, ENTRADA, DESCUENTO |
| location | VARCHAR(10) | YES | - | Location-specific (PR, TP, NULL=all) |
| is_active | BOOLEAN | YES | true | Recipe active flag |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |

**Foreign Keys**: product_item_id → products.item_id, element_item_id → products.item_id

---

## Prospect/CRM Tables

### prospects

Lead management for phone sales pipeline.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| prospect_id | VARCHAR(50) | NO | - | Primary key |
| linea_id | VARCHAR(20) | YES | - | FK to phone lines |
| address_id | VARCHAR(20) | YES | - | FK to addresses |
| order_id | VARCHAR(50) | YES | - | FK to converted order |
| nombres | VARCHAR(100) | YES | - | First name |
| apellidos | VARCHAR(100) | YES | - | Last name |
| correo | VARCHAR(255) | YES | - | Email |
| celular | VARCHAR(20) | YES | - | Phone |
| fecha | DATE | YES | - | Initial contact date |
| estado | VARCHAR(50) | YES | - | Mexican state |
| es_cliente | BOOLEAN | YES | false | Converted to customer |
| vendedor | VARCHAR(100) | YES | - | Assigned salesperson |
| producto_interes | TEXT | YES | - | Product of interest |
| nota | TEXT | YES | - | Notes |
| ... | ... | ... | ... | (50+ additional CRM fields) |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | YES | now() | Last modification timestamp |

**Triggers**: trg_auto_convert_prospect_to_customer (BEFORE UPDATE) → Creates customer record on conversion

---

### lineas

Phone lines used for prospect tracking.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| linea_id | VARCHAR(20) | NO | - | Primary key |
| nombre | VARCHAR(100) | NO | - | Line name/identifier |
| is_active | BOOLEAN | YES | true | Line active status |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |

---

## Accounting Tables

### accounting_entries

Generated accounting records for financial tracking.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| asiento_id | VARCHAR(20) | NO | - | Primary key |
| order_id | VARCHAR(50) | YES | - | FK to source order |
| tipo | VARCHAR(100) | NO | - | Entry type (Venta Camas, Garantía Accesorios, etc.) |
| sucursal | VARCHAR(50) | NO | - | Branch (General, Tienda) |
| item_referencia | VARCHAR(50) | YES | - | Product SKU reference |
| item_identificador | VARCHAR(50) | YES | - | Display identifier |
| item_nombre | VARCHAR(255) | YES | - | Product name |
| amount | NUMERIC | NO | - | Entry amount |
| cuenta_contable | VARCHAR(100) | NO | - | Account code |
| order_date | DATE | YES | - | Order date |
| entry_created_at | TIMESTAMPTZ | YES | - | Entry generation time |
| is_reconciled | BOOLEAN | YES | false | Reconciliation status |
| reconciled_at | TIMESTAMPTZ | YES | - | Reconciliation timestamp |
| reconciled_by | VARCHAR(100) | YES | - | Reconciling user |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |

---

### accounting_rules

Configuration rules for automatic entry generation.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| rule_id | UUID | NO | gen_random_uuid() | Primary key |
| tipo | VARCHAR(100) | NO | - | Entry type to match |
| sucursal | VARCHAR(50) | NO | - | Branch to match |
| monto_referencia | VARCHAR(20) | YES | - | Amount source (Precio, Costo) |
| item_identificador | VARCHAR(50) | YES | - | Override identifier |
| item_nombre | VARCHAR(255) | YES | - | Override name |
| item_amount | NUMERIC | NO | - | Multiplier or fixed amount |
| cuenta_contable | VARCHAR(100) | NO | - | Target account code |
| is_active | BOOLEAN | YES | true | Rule active status |
| created_at | TIMESTAMPTZ | YES | now() | Record creation timestamp |
| updated_at | TIMESTAMPTZ | YES | now() | Last modification timestamp |

**Constraints**: UNIQUE (tipo, sucursal, monto_referencia, cuenta_contable)

---

## Key Functions Reference

### generate_uid(length INTEGER)

Generates unique identifiers with uppercase letter prefix.

```sql
SELECT generate_uid(12);  -- Returns: 'Xa7bC9dE2fGh'
```

### generate_order_id()

Trigger function that generates order IDs for phone orders.

**Logic**:
1. Check if shopify_order_id is NULL and advisor_id is present
2. Get advisor prefix from advisors table
3. Find max digit for this advisor's phone orders
4. Generate: `{prefix}-TLF-{digit+1}`

### trg_process_recipes_on_insert()

Expands composite products into components.

**Logic**:
1. Get location_id from parent order
2. Map to location code (PR/TP)
3. Query product_recipes for matching product, recipe_type='SALIDA', location match
4. INSERT into order_item_elements for each component

### trg_generate_accounting_entries()

Creates accounting entries based on configured rules.

**Logic**:
1. Determine order type and category from order and product
2. Calculate tipo: `{order_type} {category}`
3. Determine sucursal from location_id
4. Query matching accounting_rules
5. INSERT into accounting_entries for each matching rule

---

## Utility Queries

### Record Counts by Table

```sql
SELECT 'orders' as tabla, COUNT(*) as registros FROM orders
UNION ALL SELECT 'order_items', COUNT(*) FROM order_items
UNION ALL SELECT 'customers', COUNT(*) FROM customers
-- ... extend as needed
ORDER BY registros DESC;
```

### Products with Recipes

```sql
SELECT 
  p.item_id,
  p.name,
  COUNT(pr.recipe_id) as component_count
FROM products p
JOIN product_recipes pr ON pr.product_item_id = p.item_id
WHERE pr.recipe_type = 'SALIDA'
GROUP BY p.item_id, p.name
ORDER BY component_count DESC;
```

### Inventory by Location

```sql
SELECT 
  l.name as location,
  COUNT(i.item_id) as products,
  SUM(i.quantity_on_hand) as total_stock,
  SUM(i.quantity_reserved) as reserved
FROM inventory i
JOIN locations l ON l.location_id = i.location_id
GROUP BY l.name;
```

### Orders Missing Recipe Processing

```sql
SELECT o.order_id, oi.item_id, oi.quantity
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
JOIN product_recipes pr ON pr.product_item_id = oi.item_id
LEFT JOIN order_item_elements oie ON oie.order_item_id = oi.order_item_id
WHERE oie.element_id IS NULL
  AND pr.recipe_type = 'SALIDA';
```

---

*Document Version: 1.0*  
*System Version: 2.0*  
*Author: Ivan Duarte, Full Stack Developer*
