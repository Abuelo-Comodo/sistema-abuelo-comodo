# Technical Architecture Documentation

## Sistema Abuelo Comodo v2.0

---

## The Architectural Transformation

Beyond the spreadsheet paradigms and polling-based synchronization that defined the legacy infrastructure, a new architectural foundation has emerged—one built on event-driven principles, relational integrity, and real-time data processing. This document details the technical landscape of the modernized inventory management system.

The transformation represents more than a technology swap. It embodies a fundamental shift from reactive batch processing to proactive event handling, from implicit data relationships to explicit referential constraints, from timing-dependent synchronization to transactional certainty.

---

## System Context

### External Entities

The system operates within an ecosystem of interconnected platforms, each serving distinct operational functions:

**Shopify** serves as the commercial frontend, managing three distinct sales channels: POS Playa Regatas, POS Tienda Pilares, and the online web store. Orders originate here, triggering the downstream processing pipeline through webhook notifications. Product catalog changes also propagate bidirectionally between Shopify and the central database.

**AppSheet** provides the operational interface layer, connecting warehouse staff, sales advisors, and administrative users to the underlying data. It reads from and writes to Supabase PostgreSQL directly, replacing the previous Google Sheets dependency.

**Vercel** hosts the label generation application, a React-based utility that produces shipping labels by querying order data from the central database.

### Core Platform

**Supabase** anchors the architecture, providing PostgreSQL database services, Edge Function runtime for serverless processing, and Row Level Security for access control. The platform eliminates the need for separate database hosting, API development, and authentication infrastructure.

---

## Data Flow Architecture

### Order Creation Pipeline (Shopify to Supabase)

When a customer completes a purchase on Shopify, the platform dispatches a webhook to the Supabase Edge Function endpoint. The processing sequence unfolds as follows:

```
SHOPIFY (orders/create webhook)
    |
    v
EDGE FUNCTION: shopify-webhook
    |
    |-- Validate payload structure
    |-- Check idempotency (existing order?)
    |-- Find or create customer record
    |-- Map Shopify location to internal location_id
    |-- INSERT into orders table
    |
    v
FOR EACH line_item:
    |
    |-- Validate product exists (by SKU)
    |-- INSERT into order_items table
    |       |
    |       v
    |   TRIGGER: trigger_auto_process_recipes
    |       |
    |       |-- Query product_recipes for components
    |       |-- INSERT into order_item_elements (per component)
    |       |
    |       v
    |   TRIGGER: trg_update_reserved_on_order
    |       |
    |       |-- UPDATE inventory (reserve main product)
    |       |-- UPDATE inventory (reserve each component)
    |       |
    |       v
    |   TRIGGER: trigger_generate_accounting
    |       |
    |       |-- Determine order type and category
    |       |-- Query matching accounting_rules
    |       |-- INSERT into accounting_entries (per rule)
    |
    v
RESPONSE: { success: true, order_id, items_processed }
```

The entire pipeline executes within a single database transaction. Either all operations succeed, or none persist—eliminating the partial-state failures that plagued the polling-based legacy system.

### Phone Order Pipeline (Supabase to Shopify)

Phone orders created through AppSheet follow a reverse synchronization path, automatically pushing to Shopify for unified order management:

```
APPSHEET (Phone Order Creation)
    |
    v
INSERT into orders table
    |
    v
INSERT into order_items table
    |
    v
TRIGGER: trg_auto_push_on_item_insert
    |
    |-- Verify source_type IN ('Telefonico', 'AppSheet')
    |-- Verify shopify_order_id IS NULL
    |-- Verify vendedor != 'Shopify'
    |
    v
pg_net HTTP POST (async)
    |
    v
EDGE FUNCTION: push-order-to-shopify
    |
    |-- Validate order exists and has items
    |-- Check historical order safeguard (post 2025-11-28 only)
    |-- Build Shopify order payload
    |-- Format phone with +52 country code
    |-- Create line items (variant_id or custom)
    |-- POST to Shopify Admin API
    |
    v
UPDATE orders SET
    shopify_order_id = response.id,
    shopify_order_number = response.order_number,
    order_number = 'SHO-' || response.order_number,
    uploaded_to_shopify = TRUE
```

This bidirectional synchronization ensures all orders—regardless of origin—appear in Shopify for unified fulfillment management.

### Product Sync Pipeline (Shopify to Supabase)

Product catalog changes in Shopify automatically propagate to the Supabase database, eliminating the manual data entry that previously required creating entries in both Google Sheets and AppSheet:

```
SHOPIFY (products/create, products/update, products/delete)
    |
    v
EDGE FUNCTION: hyper-processor
    |
    |-- Detect event type (create/update/delete)
    |
    v
FOR EACH variant (products may have multiple):
    |
    |-- VALIDATE SKU exists (skip if missing)
    |
    v
    IF CREATE/UPDATE:
    |   |-- Strip HTML from description
    |   |-- Map Shopify fields to Supabase columns
    |   |-- UPSERT into products table
    |   |-- Initialize inventory records for both locations
    |
    v
    IF DELETE:
    |   |-- Soft delete: SET status = 'Inactivo'
    |   |-- Preserve historical data integrity
    |
    v
RESPONSE: { success: true, products_processed, skipped }
```

This pipeline enables single-source product management in Shopify while maintaining the relational structure required for inventory tracking and order processing.

### Order Fulfillment Pipeline

Shopify's fulfillment notification triggers a parallel processing path:

```
SHOPIFY (orders/fulfilled webhook)
    |
    v
EDGE FUNCTION: shopify-order-fulfilled
    |
    |-- Locate order by shopify_order_id
    |-- UPDATE orders (fulfillment_status, warehouse_status)
    |
    v
FOR EACH fulfillment.line_item:
    |
    |-- Find order_item_id by SKU
    |-- INSERT dispatch_record (main product)
    |-- Query order_item_elements for components
    |
    v
    FOR EACH component:
        |
        |-- INSERT dispatch_record (component)
    |
    v
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
| order_items | INSERT | AFTER | trg_auto_push_on_item_insert | notify_push_to_shopify_on_items() |
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

**trg_generate_accounting_entries()**: Applies accounting rules based on order type (Venta, Garantia, Intercambio, Obsequio), product category, and location (sucursal). Supports multiple entries per order item when multiple rules match.

**process_order_dispatch()**: Executes when warehouse_status changes to 'despachado'. Creates inventory_movements for the main product and all components, updates reserved quantities, sets dispatch_date.

**notify_push_to_shopify_on_items()**: Triggers asynchronous HTTP call to push-order-to-shopify Edge Function when phone order items are inserted. Uses pg_net extension for non-blocking execution.

---

## Location Architecture

The system manages inventory across two physical locations, each mapped to Shopify's location system:

| Location | Internal ID | Shopify ID | Code | Purpose |
|----------|-------------|------------|------|---------|
| Playa Regatas | 21449925 | 21449925 | PR | Primary warehouse and showroom |
| Tienda Pilares | 63600984166 | 63600984166 | TP | Secondary retail location |

Location determines:
- Which inventory records are affected by transactions
- Which recipe variants apply (some products have location-specific components)
- Which accounting rules generate entries
- Which Shopify POS channel processes the sale

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

### Function Inventory

| Function | Direction | Trigger | Purpose |
|----------|-----------|---------|---------|
| shopify-webhook | Inbound | Shopify orders/create | Process new Shopify orders |
| shopify-order-fulfilled | Inbound | Shopify orders/fulfilled | Process fulfillment updates |
| shopify-inventory-update | Inbound | Shopify inventory_levels/update | Sync inventory changes |
| push-order-to-shopify | Outbound | Database trigger | Push phone orders to Shopify |
| hyper-processor | Inbound | Shopify products/* | Sync product catalog changes |
| sync-klaviyo-profile | Outbound | Database trigger | Sync customer profiles to Klaviyo |
| send-alert-email | Internal | Edge Functions | Send failure notification emails |

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

**Phone Number Formatting**: Mexican phone numbers require country code for Shopify API:

```typescript
const cleanPhone = rawPhone.replace(/[^0-9]/g, '');
let validPhone = undefined;

if (cleanPhone.length === 10) {
  validPhone = '+52' + cleanPhone;  // Add Mexico country code
} else if (cleanPhone.length === 12 && cleanPhone.startsWith('52')) {
  validPhone = '+' + cleanPhone;
}
```

---

## Security Considerations

### Current Implementation

- Supabase service role key authenticates Edge Functions
- Row Level Security policies control AppSheet access
- HTTPS encryption for all data transmission
- Historical order safeguard prevents accidental sync of legacy data
- JWT verification disabled for Shopify webhooks (Shopify does not send JWT tokens)

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

## Soft Delete Architecture

The system implements soft delete patterns for data preservation and audit compliance:

### Orders Table Extensions

| Column | Type | Purpose |
|--------|------|---------|
| archived_at | TIMESTAMPTZ | Soft delete timestamp |
| archived_reason | TEXT | Deletion justification |
| archived_by | TEXT | User who archived |
| uploaded_to_shopify | BOOLEAN | Sync status flag |
| shopify_upload_date | TIMESTAMPTZ | Sync timestamp |

### Products Table Soft Delete

Products deleted in Shopify are not removed from the database. Instead, their status is set to 'Inactivo', preserving:
- Order history referencing the product
- Inventory movement records
- Accounting entries
- Recipe relationships

### Helper Functions

```sql
-- Archive an order (soft delete)
SELECT archive_order('ERI-TLF-277', 'Test order', 'Ivan Duarte');

-- Restore an archived order
SELECT restore_order('ERI-TLF-277');
```

### Views

- `active_orders`: Filters to non-archived orders only
- `archived_orders`: Shows archived orders for audit

---

## Performance Characteristics

### Latency Benchmarks

| Operation | Legacy System | New System |
|-----------|---------------|------------|
| Order processing (end-to-end) | 5-15 minutes | < 2 seconds |
| Recipe expansion | 5 minutes (batch) | < 100ms (per item) |
| Inventory reservation | 5 minutes (batch) | < 50ms (per item) |
| Accounting entry generation | 5 minutes (batch) | < 100ms (per item) |
| Phone order push to Shopify | N/A (manual) | < 3 seconds (auto) |
| Product sync from Shopify | N/A (manual) | < 1 second (auto) |

### Scalability Vectors

PostgreSQL handles the current data volume (44,638 records) with negligible resource consumption. The architecture scales linearly with:

- **Concurrent orders**: Edge Functions scale horizontally on Supabase's infrastructure
- **Data volume**: PostgreSQL efficiently manages millions of records with proper indexing
- **Trigger complexity**: PL/pgSQL execution remains fast for current logic complexity

---

## Integration Points

### Shopify to Supabase (Inbound)

| Webhook Topic | Edge Function | Tables Affected |
|---------------|---------------|-----------------|
| orders/create | shopify-webhook | orders, order_items, order_item_elements, customers, inventory, accounting_entries |
| orders/fulfilled | shopify-order-fulfilled | orders, dispatch_records |
| inventory_levels/update | shopify-inventory-update | inventory |
| products/create | hyper-processor | products, inventory |
| products/update | hyper-processor | products |
| products/delete | hyper-processor | products (soft delete) |

### Supabase to Shopify (Outbound)

| Trigger | Edge Function | Purpose |
|---------|---------------|---------|
| trg_auto_push_on_item_insert | push-order-to-shopify | Sync phone orders to Shopify |

### Supabase to Klaviyo (Outbound)

| Trigger | Edge Function | Purpose |
|---------|---------------|---------|
| trg_sync_klaviyo_profile | sync-klaviyo-profile | Sync customer profile changes |

### AppSheet to Supabase

AppSheet connects via PostgreSQL connector, reading from views and writing to tables with appropriate RLS policies. Key interfaces:

- Prospect management (CRUD on prospects table)
- Order creation for phone sales (INSERT to orders, order_items)
- Inventory adjustments (INSERT to inventory_movements)
- Dispatch confirmation (UPDATE orders.warehouse_status)

---

## The Road Ahead

The architecture establishes extensibility as a core principle. New webhooks follow established patterns. Additional business logic integrates as triggers. Frontend interfaces connect through the same PostgreSQL layer that powers AppSheet.

Immediate enhancement opportunities include HMAC validation for webhook security, order update synchronization, and the Next.js administrative dashboard outlined in the system design. Each builds upon the foundation without requiring architectural revision.

For engineers extending this system: the patterns are deliberate. Edge Functions handle external boundaries, triggers manage internal consistency, and PostgreSQL serves as the single source of truth. Respect these boundaries, and the system will scale with the business.

---

*Document Version: 1.2*
*System Version: 2.0*
*Last Updated: December 3, 2025*
*Author: Ivan Duarte, Full Stack Developer*
