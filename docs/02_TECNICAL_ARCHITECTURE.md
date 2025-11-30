# Technical Architecture Documentation

## Sistema Abuelo Cómodo v2.0

---

## The Architectural Transformation

Beyond the spreadsheet paradigms and polling-based synchronization that defined the legacy infrastructure, a new architectural foundation has emerged—one built on event-driven principles, relational integrity, and real-time data processing. This document details the technical landscape of the modernized inventory management system.

The transformation represents more than a technology swap. It embodies a fundamental shift from reactive batch processing to proactive event handling, from implicit data relationships to explicit referential constraints, from timing-dependent synchronization to transactional certainty.

---

## System Context

### External Entities

The system operates within an ecosystem of interconnected platforms, each serving distinct operational functions:

**Shopify** serves as the commercial frontend, managing three distinct sales channels: POS Playa Regatas, POS Tienda Pilares, and the online web store. Orders originate here, triggering the downstream processing pipeline through webhook notifications.

**AppSheet** provides the operational interface layer, connecting warehouse staff, sales advisors, and administrative users to the underlying data. It reads from and writes to Supabase PostgreSQL directly, replacing the previous Google Sheets dependency.

**Vercel** hosts the label generation application, a React-based utility that produces shipping labels by querying order data from the central database.

### Core Platform

**Supabase** anchors the architecture, providing PostgreSQL database services, Edge Function runtime for serverless processing, and Row Level Security for access control. The platform eliminates the need for separate database hosting, API development, and authentication infrastructure.

---

## Data Flow Architecture

### Order Creation Pipeline

When a customer completes a purchase on Shopify, the platform dispatches a webhook to the Supabase Edge Function endpoint. The processing sequence unfolds as follows:

```
SHOPIFY (orders/create webhook)
    │
    ▼
EDGE FUNCTION: shopify-webhook
    │
    ├── Validate payload structure
    ├── Check idempotency (existing order?)
    ├── Find or create customer record
    ├── Map Shopify location to internal location_id
    ├── INSERT into orders table
    │
    ▼
FOR EACH line_item:
    │
    ├── Validate product exists (by SKU)
    ├── INSERT into order_items table
    │       │
    │       ▼
    │   TRIGGER: trigger_auto_process_recipes
    │       │
    │       ├── Query product_recipes for components
    │       ├── INSERT into order_item_elements (per component)
    │       │
    │       ▼
    │   TRIGGER: trg_update_reserved_on_order
    │       │
    │       ├── UPDATE inventory (reserve main product)
    │       ├── UPDATE inventory (reserve each component)
    │       │
    │       ▼
    │   TRIGGER: trigger_generate_accounting
    │       │
    │       ├── Determine order type and category
    │       ├── Query matching accounting_rules
    │       ├── INSERT into accounting_entries (per rule)
    │
    ▼
RESPONSE: { success: true, order_id, items_processed }
```

The entire pipeline executes within a single database transaction. Either all operations succeed, or none persist—eliminating the partial-state failures that plagued the polling-based legacy system.

### Order Fulfillment Pipeline

Shopify's fulfillment notification triggers a parallel processing path:

```
SHOPIFY (orders/fulfilled webhook)
    │
    ▼
EDGE FUNCTION: shopify-order-fulfilled
    │
    ├── Locate order by shopify_order_id
    ├── UPDATE orders (fulfillment_status, warehouse_status)
    │
    ▼
FOR EACH fulfillment.line_item:
    │
    ├── Find order_item_id by SKU
    ├── INSERT dispatch_record (main product)
    ├── Query order_item_elements for components
    │
    ▼
    FOR EACH component:
        │
        └── INSERT dispatch_record (component)
    │
    ▼
RESPONSE: { success: true, dispatch_records_created }
```

Dispatch records document what left the warehouse, providing the audit trail necessary for inventory reconciliation and accounting verification.

---

## Trigger Architecture

PostgreSQL triggers constitute the business logic layer, executing automatically in response to data changes. This pattern centralizes logic within the database, ensuring consistency regardless of the data entry point—whether from Edge Functions, AppSheet, or direct SQL operations.

### Trigger Execution Model

Triggers operate in defined sequences based on timing (BEFORE/AFTER) and event type (INSERT/UPDATE/DELETE):

| Table | Event | Timing | Trigger | Function |
|-------|-------|--------|---------|----------|
| orders | INSERT | BEFORE | trg_generate_order_id | generate_order_id() |
| orders | INSERT | AFTER | trg_update_customer_metrics | update_customer_metrics() |
| orders | UPDATE | BEFORE | trg_process_order_dispatch | process_order_dispatch() |
| orders | UPDATE | AFTER | trg_update_customer_metrics | update_customer_metrics() |
| order_items | INSERT | AFTER | trigger_auto_process_recipes | trg_process_recipes_on_insert() |
| order_items | INSERT | AFTER | trg_update_reserved_on_order | update_reserved_on_order() |
| order_items | INSERT | AFTER | trigger_generate_accounting | trg_generate_accounting_entries() |
| inventory_movements | INSERT | BEFORE | trg_update_inventory_from_movement | update_inventory_from_movement() |
| purchase_order_items | INSERT | AFTER | trg_update_in_transit_on_po | update_in_transit_on_po() |
| purchase_order_items | UPDATE | AFTER | trg_process_purchase_receipt | process_purchase_receipt() |
| transfer_items | INSERT | BEFORE | trg_validate_transfer_stock | validate_transfer_stock() |
| transfers | UPDATE | BEFORE | trg_process_transfer_receipt | process_transfer_receipt() |
| prospect_payments | INSERT/UPDATE | BEFORE | trg_validate_payments | validate_prospect_payments() |
| prospect_order_items | INSERT/UPDATE/DELETE | AFTER | trg_recalc_order_* | calculate_prospect_order_total() |
| prospects | UPDATE | BEFORE | trg_auto_convert_prospect_to_customer | auto_convert_prospect_to_customer() |

### Key Business Logic Functions

**generate_order_id()**: Generates sequential order identifiers for phone orders based on advisor prefix. Pattern: `{ADVISOR_PREFIX}-TLF-{SEQUENCE}`. Shopify orders retain their original Shopify ID.

**trg_process_recipes_on_insert()**: Expands composite products into component elements. Queries product_recipes where product_item_id matches the ordered item, filters by recipe_type = 'SALIDA' and location, then inserts corresponding records into order_item_elements with calculated quantities.

**trg_generate_accounting_entries()**: Applies accounting rules based on order type (Venta, Garantía, Intercambio, Obsequio), product category, and location (sucursal). Supports multiple entries per order item when multiple rules match.

**process_order_dispatch()**: Executes when warehouse_status changes to 'despachado'. Creates inventory_movements for the main product and all components, updates reserved quantities, sets dispatch_date.

---

## Location Architecture

The system manages inventory across two physical locations, each mapped to Shopify's location system:

| Location | Internal ID | Shopify ID | Code | Purpose |
|----------|-------------|------------|------|---------|
| Playa Regatas | 21449925 | 21449925 | PR | Main warehouse |
| Tienda Pilares | 63600984166 | 63600984166 | TP | Retail store |

Location-aware operations include:
- Inventory tracking (quantity_on_hand, quantity_reserved, quantity_incoming per location)
- Recipe selection (some products have location-specific component lists)
- Accounting rules (different cuenta_contable mappings per sucursal)
- Order assignment (location_id determines which warehouse fulfills)

---

## Recipe Processing Model

Composite products—those assembled from multiple components—rely on the product_recipes table for decomposition. The model supports three recipe types:

| Type | Purpose | Trigger Context |
|------|---------|-----------------|
| SALIDA | Components deducted from inventory on sale | order_items INSERT |
| ENTRADA | Components added to inventory on receipt | purchase_order_items UPDATE |
| DESCUENTO | Promotional or discount calculations | Accounting rules |

### Recipe Structure

```
product_recipes
├── recipe_id (UUID)
├── product_item_id (FK → products.item_id)  -- The composite product
├── element_item_id (FK → products.item_id)  -- The component
├── quantity_per_product (NUMERIC)           -- How many per unit sold
├── recipe_type (VARCHAR)                    -- SALIDA, ENTRADA, DESCUENTO
├── location (VARCHAR)                       -- PR, TP, or NULL (all locations)
└── is_active (BOOLEAN)
```

When an order_item is inserted, the trigger queries:

```sql
SELECT * FROM product_recipes
WHERE product_item_id = NEW.item_id
  AND UPPER(recipe_type) = 'SALIDA'
  AND (location = v_location OR location IS NULL);
```

For products with 15-19 components (common in the catalog), this single INSERT triggers 15-19 corresponding INSERT operations into order_item_elements, each with calculated quantities based on the ordered quantity multiplied by quantity_per_product.

---

## Inventory State Model

Inventory tracking extends beyond simple stock counts. The model captures multiple quantity dimensions:

| Field | Purpose | Modified By |
|-------|---------|-------------|
| quantity_on_hand | Physical stock available | inventory_movements |
| quantity_reserved | Allocated to pending orders | order_items INSERT, order dispatch |
| quantity_incoming | Expected from purchase orders | purchase_order_items INSERT |
| virtual_quantity | Calculated available (on_hand - reserved + incoming) | Derived |

### Inventory Movement Types

| Type | Quantity Sign | Triggered By |
|------|---------------|--------------|
| entrada | Positive | Purchase order receipt |
| salida | Negative | Order dispatch |
| transfer_in | Positive | Transfer receipt (destination) |
| transfer_out | Negative | Transfer receipt (origin) |
| ajuste | +/- | Manual adjustment |

---

## Edge Function Architecture

Edge Functions execute on Supabase's Deno runtime, processing incoming webhooks with sub-second latency. The architecture emphasizes idempotency, error handling, and comprehensive logging.

### Common Patterns

**Idempotency Check**: Every webhook handler first verifies whether the event has already been processed:

```typescript
const { data: existingOrder } = await supabase
  .from('orders')
  .select('order_id')
  .eq('shopify_order_id', shopifyOrderId)
  .single();

if (existingOrder) {
  return new Response(JSON.stringify({
    success: true,
    message: 'Order already processed',
    order_id: existingOrder.order_id
  }), { status: 200 });
}
```

**Location Mapping**: Shopify location IDs translate to internal identifiers:

```typescript
const LOCATION_MAP = {
  '21449925': 'PR',
  '63600984166': 'TP'
};
```

**UID Generation**: Unique identifiers follow a consistent pattern—uppercase letter prefix followed by alphanumeric characters:

```typescript
function generateUid(length) {
  const alphabet = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  const alphabet2 = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
  let result = alphabet2.charAt(Math.floor(Math.random() * alphabet2.length));
  for (let i = 0; i < length; i++) {
    result += alphabet.charAt(Math.floor(Math.random() * alphabet.length));
  }
  return result;
}
```

---

## Security Considerations

### Current Implementation

- Supabase service role key authenticates Edge Functions
- Row Level Security policies control AppSheet access
- HTTPS encryption for all data transmission

### Pending Implementation

**HMAC Webhook Validation**: Shopify signs webhooks with HMAC-SHA256. Implementing validation ensures only legitimate Shopify requests trigger processing:

```typescript
import { createHmac } from 'crypto';

function verifyShopifyWebhook(body: string, signature: string, secret: string): boolean {
  const hash = createHmac('sha256', secret)
    .update(body, 'utf8')
    .digest('base64');
  return hash === signature;
}
```

This should be implemented before production deployment with significant order volume.

---

## Performance Characteristics

### Latency Benchmarks

| Operation | Legacy System | New System |
|-----------|---------------|------------|
| Order processing (end-to-end) | 5-15 minutes | < 2 seconds |
| Recipe expansion | 5 minutes (batch) | < 100ms (per item) |
| Inventory reservation | 5 minutes (batch) | < 50ms (per item) |
| Accounting entry generation | 5 minutes (batch) | < 100ms (per item) |

### Scalability Vectors

PostgreSQL handles the current data volume (44,638 records) with negligible resource consumption. The architecture scales linearly with:

- **Concurrent orders**: Edge Functions scale horizontally on Supabase's infrastructure
- **Data volume**: PostgreSQL efficiently manages millions of records with proper indexing
- **Trigger complexity**: PL/pgSQL execution remains fast for current logic complexity

---

## Integration Points

### Shopify → Supabase

| Webhook Topic | Edge Function | Tables Affected |
|---------------|---------------|-----------------|
| orders/create | shopify-webhook | orders, order_items, order_item_elements, customers, inventory, accounting_entries |
| orders/fulfilled | shopify-order-fulfilled | orders, dispatch_records |
| inventory_levels/update | (planned) | inventory |
| products/create | (planned) | products |
| customers/create | (planned) | customers |

### Supabase → Shopify (Planned)

| Trigger | API Call | Purpose |
|---------|----------|---------|
| inventory UPDATE | Inventory API | Sync stock levels to storefront |
| products UPDATE | Products API | Sync product data changes |

### AppSheet → Supabase

AppSheet connects via PostgreSQL connector, reading from views and writing to tables with appropriate RLS policies. Key interfaces:

- Prospect management (CRUD on prospects table)
- Order creation for phone sales (INSERT to orders, order_items)
- Inventory adjustments (INSERT to inventory_movements)
- Dispatch confirmation (UPDATE orders.warehouse_status)

---

## The Road Ahead

The architecture establishes extensibility as a core principle. New webhooks follow established patterns. Additional business logic integrates as triggers. Frontend interfaces connect through the same PostgreSQL layer that powers AppSheet.

Immediate enhancement opportunities include HMAC validation for webhook security, bidirectional Shopify synchronization for inventory levels, and the Next.js administrative dashboard outlined in the system design. Each builds upon the foundation without requiring architectural revision.

For engineers extending this system: the patterns are deliberate. Edge Functions handle external boundaries, triggers manage internal consistency, and PostgreSQL serves as the single source of truth. Respect these boundaries, and the system will scale with the business.

---

*Document Version: 1.0*  
*System Version: 2.0*  
*Author: Ivan Duarte, Full Stack Developer*
