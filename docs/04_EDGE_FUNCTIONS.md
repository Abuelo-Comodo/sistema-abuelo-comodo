# Edge Functions Documentation

## Sistema Abuelo Comodo v2.0

---

## The Serverless Gateway

Where Zapier once stood as the intermediary—transforming webhooks, routing data, introducing latency with each hop—Edge Functions now serve as the direct conduit between Shopify's event stream and the PostgreSQL core. These TypeScript functions execute on Supabase's Deno runtime, processing incoming webhooks with sub-second response times and transactional certainty.

This document details the implementation, configuration, and operational characteristics of each Edge Function deployed in the system.

---

## Architecture Overview

```
SHOPIFY WEBHOOKS                             POSTGRESQL TRIGGERS
      |                                             |
      v                                             v
+-----------------------------------------------------------+
|                   SUPABASE EDGE RUNTIME                    |
|                        (Deno)                              |
|  +------------------+  +------------------------+          |
|  | shopify-webhook  |  | shopify-order-fulfilled|          |
|  | (orders/create)  |  | (orders/fulfilled)     |          |
|  +------------------+  +------------------------+          |
|                                                            |
|  +------------------+  +------------------------+          |
|  | shopify-         |  | push-order-to-shopify  |          |
|  | inventory-update |  | (phone orders out)     |          |
|  +------------------+  +------------------------+          |
|                                                            |
|  +------------------+  +------------------------+          |
|  | hyper-processor  |  | sync-klaviyo-profile   |          |
|  | (products/*)     |  | (customer sync)        |          |
|  +------------------+  +------------------------+          |
|                                                            |
|  +------------------+                                      |
|  | send-alert-email |                                      |
|  | (notifications)  |                                      |
|  +------------------+                                      |
|            |                       |                       |
|            v                       v                       |
|  +---------------------------------------------------+    |
|  |          SUPABASE CLIENT (service role)           |    |
|  +---------------------------------------------------+    |
+-----------------------------------------------------------+
      |                                             |
      v                                             v
POSTGRESQL (triggers execute)              EXTERNAL APIs
                                    (Shopify, Klaviyo, Resend)
```

### Runtime Environment

| Property | Value |
|----------|-------|
| Runtime | Deno (TypeScript) |
| Execution Model | Serverless, per-request |
| Timeout | 60 seconds default |
| Memory | 256MB default |
| Cold Start | ~100ms |
| Warm Execution | <50ms |

### Dependencies

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
```

---

## Function Inventory

| Function | Direction | Webhook/Trigger | Purpose |
|----------|-----------|-----------------|---------|
| shopify-webhook | Inbound | orders/create | Process new Shopify orders |
| shopify-order-fulfilled | Inbound | orders/fulfilled | Process fulfillment updates |
| shopify-inventory-update | Inbound | inventory_levels/update | Sync inventory changes |
| hyper-processor | Inbound | products/* | Sync product catalog |
| push-order-to-shopify | Outbound | Database trigger | Push phone orders to Shopify |
| sync-klaviyo-profile | Outbound | Database trigger | Sync customers to Klaviyo |
| send-alert-email | Internal | Called by functions | Send failure notifications |

---

## Function: shopify-webhook

### Purpose

Processes Shopify `orders/create` webhook events, creating corresponding records in the database and triggering downstream business logic through PostgreSQL triggers.

### Endpoint

```
POST https://{project-ref}.supabase.co/functions/v1/shopify-webhook
```

### Shopify Configuration

| Setting | Value |
|---------|-------|
| Webhook Topic | orders/create |
| Format | JSON |
| API Version | 2024-01 (or current) |

### Request Flow

```
1. RECEIVE webhook payload from Shopify
      |
2. PARSE JSON body
      |
3. CHECK idempotency (existing order by shopify_order_id?)
      |  |-- YES: Return 200 with "already processed"
      |  |-- NO: Continue
      |
4. FIND OR CREATE customer
      |  |-- Search by shopify_customer_id
      |  |-- Search by email
      |  |-- Create new if not found
      |
5. MAP location (Shopify location_id to internal location_id)
      |
6. INSERT order record
      |
7. FOR EACH line_item:
      |  |-- Validate product exists by SKU
      |  |-- INSERT order_item
      |  |     |-- TRIGGERS EXECUTE:
      |  |         |-- trigger_auto_process_recipes
      |  |         |-- trg_update_reserved_on_order
      |  |         |-- trigger_generate_accounting
      |  |-- Capture shopify_inventory_item_id if present
      |
8. RETURN success response with processing summary
```

### Core Implementation

#### Location Mapping

```typescript
const LOCATION_MAP = {
  '21449925': 'PR',      // Playa Regatas
  '63600984166': 'TP'    // Tienda Pilares
};
```

#### UID Generation

All generated identifiers follow a consistent pattern: uppercase letter prefix followed by alphanumeric characters. This ensures visual distinction and reduces collision probability.

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

#### Customer Resolution

The function implements a hierarchical customer lookup strategy:

```typescript
async function findOrCreateCustomer(supabase, shopifyOrder) {
  const shopifyCustomerId = shopifyOrder.customer?.id?.toString();
  const email = shopifyOrder.email || shopifyOrder.customer?.email;
  const phone = shopifyOrder.phone || shopifyOrder.customer?.phone;

  // Priority 1: Match by Shopify customer ID
  if (shopifyCustomerId) {
    const { data: existing } = await supabase
      .from('customers')
      .select('customer_id')
      .eq('shopify_customer_id', shopifyCustomerId)
      .single();
    
    if (existing) return existing.customer_id;
  }

  // Priority 2: Match by email
  if (email) {
    const { data: existing } = await supabase
      .from('customers')
      .select('customer_id')
      .eq('email', email)
      .single();
    
    if (existing) {
      if (shopifyCustomerId) {
        await supabase
          .from('customers')
          .update({ shopify_customer_id: shopifyCustomerId })
          .eq('customer_id', existing.customer_id);
      }
      return existing.customer_id;
    }
  }

  // Priority 3: Create new customer
  const newCustomer = {
    customer_id: crypto.randomUUID(),
    shopify_customer_id: shopifyCustomerId || null,
    first_name: shopifyOrder.customer?.first_name || 'Cliente',
    last_name: shopifyOrder.customer?.last_name || 'Shopify',
    email: email || null,
    phone: phone || null,
    accepts_marketing: shopifyOrder.customer?.accepts_marketing || false,
    created_at: new Date().toISOString()
  };

  const { data: created } = await supabase
    .from('customers')
    .insert(newCustomer)
    .select('customer_id')
    .single();

  return created?.customer_id;
}
```

### Response Schema

#### Success Response (200)

```json
{
  "success": true,
  "order_id": "7654321098765",
  "order_number": "SHO-1234",
  "customer_id": "uuid-here",
  "items_processed": 3,
  "items_skipped": 0,
  "processing_time_ms": 234
}
```

#### Already Processed (200)

```json
{
  "success": true,
  "message": "Order already processed",
  "order_id": "7654321098765"
}
```

#### Error Response (500)

```json
{
  "success": false,
  "error": "Error message here",
  "processing_time_ms": 45
}
```

---

## Function: shopify-order-fulfilled

### Purpose

Processes Shopify `orders/fulfilled` webhook events, updating order status and creating dispatch records for inventory tracking.

### Endpoint

```
POST https://{project-ref}.supabase.co/functions/v1/shopify-order-fulfilled
```

### Shopify Configuration

| Setting | Value |
|---------|-------|
| Webhook Topic | orders/fulfilled |
| Format | JSON |
| API Version | 2024-01 (or current) |

### Request Flow

```
1. RECEIVE webhook payload from Shopify
      |
2. PARSE JSON body
      |
3. FIND order by shopify_order_id
      |  |-- NOT FOUND: Return 500 error
      |
4. UPDATE order status
      |  |-- fulfillment_status = 'fulfilled'
      |  |-- warehouse_status = 'despachado'
      |
5. FOR EACH fulfillment:
      |
      FOR EACH line_item:
      |  |-- Find order_item by order_id + SKU
      |  |-- CREATE dispatch_record (main product)
      |  |
      |  |-- QUERY order_item_elements for components
      |  |
      |  FOR EACH component:
      |     |-- CREATE dispatch_record (component)
      |
6. RETURN success response with dispatch counts
```

### Response Schema

#### Success Response (200)

```json
{
  "success": true,
  "order_id": "7654321098765",
  "dispatch_records_created": 5,
  "processing_time_ms": 156
}
```

#### Order Not Found (500)

```json
{
  "success": false,
  "error": "Order not found: 7654321098765",
  "processing_time_ms": 23
}
```

---

## Function: hyper-processor

### Purpose

Processes Shopify product webhooks (create, update, delete), maintaining synchronization between the Shopify catalog and the Supabase products table. This eliminates the manual product entry workflow that previously required creating entries in Google Sheets and AppSheet.

### Endpoint

```
POST https://{project-ref}.supabase.co/functions/v1/hyper-processor
```

### Shopify Configuration

| Setting | Value |
|---------|-------|
| Webhook Topics | products/create, products/update, products/delete |
| Format | JSON |
| API Version | 2024-01 (or current) |
| JWT Verification | DISABLED (Shopify does not send JWT) |

### Request Flow

```
1. RECEIVE webhook payload from Shopify
      |
2. PARSE JSON body
      |
3. DETECT event type
      |  |-- Has variants and title: CREATE or UPDATE
      |  |-- No variants, no title: DELETE
      |
4. FOR EACH variant:
      |
      IF SKU missing:
      |  |-- Log warning, skip variant
      |  |-- Continue to next variant
      |
      IF CREATE/UPDATE:
      |  |-- Strip HTML from description
      |  |-- Map Shopify fields to Supabase columns
      |  |-- UPSERT into products table
      |  |-- Initialize inventory for PR and TP locations
      |
      IF DELETE:
      |  |-- UPDATE products SET status = 'Inactivo'
      |  |-- Preserve referential integrity
      |
5. RETURN success response with processing summary
```

### Data Mapping

#### Shopify to Supabase Field Mapping

| Shopify Field | Supabase Column | Notes |
|---------------|-----------------|-------|
| variant.sku | item_id | Primary identifier (required) |
| variant.sku | sku | Duplicate for compatibility |
| title + variant.title | name | Combined if variant title exists |
| title + variant.title | inventory_name | Same as name |
| body_html | description | HTML stripped, max 500 chars |
| variant.price | sale_price | Decimal |
| variant.compare_at_price | purchase_cost | Decimal |
| status | status | 'active' to 'Activo', else 'Inactivo' |
| image.src | image_url | First image URL |
| id | shopify_product_id | Shopify product ID |
| variant.id | shopify_variant_id | Shopify variant ID |
| variant.inventory_item_id | shopify_inventory_item_id | For inventory sync |

#### Auto-Generated Fields

| Field | Value |
|-------|-------|
| is_composite | false |
| reorder_point | 5 |
| reorder_quantity | 10 |
| created_at | Current timestamp |
| updated_at | Current timestamp |

#### Inventory Initialization

When a product is created, inventory records are automatically initialized:

| Location | location_id | Initial Quantity |
|----------|-------------|------------------|
| Playa Regatas (PR) | 21449925 | 0 |
| Tienda Pilares (TP) | 63600984166 | 0 |

### Soft Delete Implementation

Products are never hard-deleted from the database. When Shopify sends a delete webhook:

1. Webhook payload contains only product ID (no variants, no title)
2. Function detects delete event by absence of variant data
3. All products matching `shopify_product_id` update to `status = 'Inactivo'`
4. Historical data remains intact for orders, inventory, accounting

### Response Schema

#### Success Response (200)

```json
{
  "success": true,
  "event_type": "create",
  "products_processed": 3,
  "products_skipped": 1,
  "skipped_reasons": ["Variant missing SKU"],
  "processing_time_ms": 234
}
```

#### Delete Response (200)

```json
{
  "success": true,
  "event_type": "delete",
  "shopify_product_id": "8234567890123",
  "products_deactivated": 2,
  "processing_time_ms": 89
}
```

---

## Function: push-order-to-shopify

### Purpose

Pushes phone orders created in AppSheet to Shopify Admin API, enabling unified order management regardless of sales channel origin.

### Endpoint

```
POST https://{project-ref}.supabase.co/functions/v1/push-order-to-shopify
```

### Trigger

Database trigger `trg_auto_push_on_item_insert` on `order_items` table, using pg_net for async HTTP calls.

### Request Flow

```
1. RECEIVE order_id from trigger
      |
2. VALIDATE order exists and has items
      |  |-- NO: Return error
      |
3. CHECK historical safeguard
      |  |-- Order before 2025-11-28: Reject
      |
4. CHECK already uploaded
      |  |-- YES: Return "already uploaded"
      |
5. BUILD Shopify order payload
      |  |-- Format phone number with +52
      |  |-- Build line items (variant_id or custom)
      |  |-- Include shipping address
      |  |-- Add tags and notes
      |
6. POST to Shopify Admin API
      |
7. UPDATE orders table
      |  |-- shopify_order_id
      |  |-- shopify_order_number
      |  |-- order_number = 'SHO-' + number
      |  |-- uploaded_to_shopify = TRUE
      |
8. RETURN success response
```

### Phone Number Formatting

Mexican phone numbers require specific formatting for Shopify API acceptance:

```typescript
const cleanPhone = rawPhone.replace(/[^0-9]/g, '');
let validPhone = undefined;

if (cleanPhone.length === 10) {
  validPhone = '+52' + cleanPhone;
} else if (cleanPhone.length === 12 && cleanPhone.startsWith('52')) {
  validPhone = '+' + cleanPhone;
} else if (cleanPhone.length === 13 && cleanPhone.startsWith('521')) {
  validPhone = '+' + cleanPhone;
} else if (cleanPhone.length > 10) {
  validPhone = cleanPhone.startsWith('+') ? cleanPhone : '+' + cleanPhone;
}
```

### Response Schema

#### Success Response (200)

```json
{
  "success": true,
  "order_id": "LAU-TLF-213",
  "shopify_order_id": "6286978220134",
  "shopify_order_number": 16062,
  "order_number": "SHO-16062",
  "items_count": 1,
  "processing_time_ms": 2854
}
```

#### Already Uploaded (200)

```json
{
  "success": true,
  "message": "Order already uploaded to Shopify",
  "order_id": "LAU-TLF-213",
  "shopify_order_id": "6286978220134"
}
```

#### Historical Order Rejected (400)

```json
{
  "success": false,
  "error": "Historical orders cannot be pushed to Shopify. Only orders created after 2025-11-28 are allowed.",
  "order_id": "OLD-TLF-001"
}
```

---

## Function: sync-klaviyo-profile

### Purpose

Synchronizes customer profile data to Klaviyo for marketing automation. Triggered by database changes on the customers table.

### Endpoint

```
POST https://{project-ref}.supabase.co/functions/v1/sync-klaviyo-profile
```

### Trigger

Database trigger `trg_sync_klaviyo_profile` on `customers` table.

### Klaviyo API Requirements

| Header | Value |
|--------|-------|
| Authorization | Klaviyo-API-Key {api_key} |
| Content-Type | application/vnd.api+json |
| Accept | application/vnd.api+json |
| Revision | 2025-04-15 |

### Response Handling

| Status | Meaning | Action |
|--------|---------|--------|
| 200/201 | Success | Profile created/updated |
| 409 | Duplicate | Profile exists, handled by upsert |
| 401 | Unauthorized | Check API key |

---

## Function: send-alert-email

### Purpose

Sends failure notification emails when Edge Functions encounter errors. Called by other functions in catch blocks.

### Endpoint

```
POST https://{project-ref}.supabase.co/functions/v1/send-alert-email
```

### Request Payload

```json
{
  "function_name": "shopify-webhook",
  "message": "Error al procesar orden de Shopify",
  "error_details": {
    "error": "Column 'ciudad_colonia' not found",
    "shopify_order_id": "6299069349990",
    "timestamp": "2025-12-02T16:11:34.650Z"
  }
}
```

### Email Service

Uses Resend API for email delivery. Without domain verification, only the account owner email receives alerts.

---

## Environment Variables

All functions require the following environment variables configured in Supabase:

| Variable | Description | Source |
|----------|-------------|--------|
| SUPABASE_URL | Project API URL | Supabase Dashboard: Settings > API |
| SUPABASE_SERVICE_ROLE_KEY | Service role secret key | Supabase Dashboard: Settings > API |
| SHOPIFY_DOMAIN | Shopify admin domain | abuelo-comodo.myshopify.com |
| SHOPIFY_ACCESS_TOKEN | Shopify Admin API token | Shopify Admin: Apps > Develop apps |
| KLAVIYO_API_KEY | Klaviyo private API key | Klaviyo: Settings > API Keys |
| RESEND_API_KEY | Resend email API key | Resend Dashboard |

### Configuration Commands

```bash
supabase secrets set SUPABASE_URL=https://xxxxx.supabase.co
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...
supabase secrets set SHOPIFY_DOMAIN=abuelo-comodo.myshopify.com
supabase secrets set SHOPIFY_ACCESS_TOKEN=shpat_xxxxx
supabase secrets set KLAVIYO_API_KEY=pk_xxxxx
supabase secrets set RESEND_API_KEY=re_xxxxx
```

---

## Deployment

### Prerequisites

- Supabase CLI installed and authenticated
- Project linked: `supabase link --project-ref {project-id}`

### Deploy Commands

```bash
# Deploy individual function
supabase functions deploy shopify-webhook
supabase functions deploy shopify-order-fulfilled
supabase functions deploy shopify-inventory-update
supabase functions deploy hyper-processor
supabase functions deploy push-order-to-shopify
supabase functions deploy sync-klaviyo-profile
supabase functions deploy send-alert-email

# Deploy all functions
supabase functions deploy

# Verify deployment
supabase functions list
```

---

## Shopify Webhook Configuration

### Required Webhooks

Configure in Shopify Admin > Settings > Notifications > Webhooks:

| Event | URL | Format |
|-------|-----|--------|
| orders/create | https://{ref}.supabase.co/functions/v1/shopify-webhook | JSON |
| orders/fulfilled | https://{ref}.supabase.co/functions/v1/shopify-order-fulfilled | JSON |
| inventory_levels/update | https://{ref}.supabase.co/functions/v1/shopify-inventory-update | JSON |
| products/create | https://{ref}.supabase.co/functions/v1/hyper-processor | JSON |
| products/update | https://{ref}.supabase.co/functions/v1/hyper-processor | JSON |
| products/delete | https://{ref}.supabase.co/functions/v1/hyper-processor | JSON |

---

## Monitoring and Debugging

### Log Patterns

Each function logs key events for debugging:

```typescript
console.log('=== Shopify Webhook Received ===');
console.log('Order ID:', shopifyOrder.id);
console.log('Order Number:', shopifyOrder.order_number);
console.log('Location ID:', shopifyOrder.location_id);
```

### Key Log Messages

| Message | Meaning |
|---------|---------|
| `Order already exists: {id}` | Idempotency check passed |
| `Found existing customer by Shopify ID` | Customer matched |
| `Created new customer: {id}` | New customer record |
| `Location mapped: PR -> {location_id}` | Location resolution success |
| `Order created: {order_number}` | Order insert success |
| `Product upserted: {sku}` | Product sync success |
| `Variant missing SKU, skipping` | Product skipped |
| `Soft delete: {product_id}` | Product deactivated |

### pg_net Request Logs

Monitor trigger-initiated HTTP calls:

```sql
SELECT id, status_code, content, created 
FROM net._http_response 
ORDER BY created DESC 
LIMIT 10;
```

### Supabase Dashboard Monitoring

1. Navigate to Edge Functions in Supabase Dashboard
2. Select function to view invocation logs
3. Filter by time range or search log content
4. Monitor invocation counts and error rates

---

## Security Considerations

### Current State

- Service role key authenticates Edge Functions to database
- Anon key used for trigger-initiated calls (server-side only)
- HTTPS encryption for all data transmission
- Historical order safeguard prevents accidental sync
- JWT verification disabled for Shopify webhooks

### Pending: HMAC Webhook Validation

Shopify signs webhooks with HMAC-SHA256. Implementing validation ensures only legitimate Shopify requests are processed:

```typescript
import { createHmac } from 'crypto';

function verifyShopifyWebhook(
  body: string,
  signature: string,
  secret: string
): boolean {
  const hash = createHmac('sha256', secret)
    .update(body, 'utf8')
    .digest('base64');
  return hash === signature;
}

// Usage in handler
const signature = req.headers.get('X-Shopify-Hmac-SHA256');
const body = await req.text();

if (!verifyShopifyWebhook(body, signature, SHOPIFY_WEBHOOK_SECRET)) {
  return new Response('Unauthorized', { status: 401 });
}
```

**Implementation Priority**: High - should be implemented before production launch with significant order volume.

---

## Error Handling Reference

### Common Errors and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `Schema "net" does not exist` | pg_net extension not enabled | `CREATE EXTENSION IF NOT EXISTS pg_net;` |
| `Phone is invalid` | Phone number wrong format | Ensure 10+ digits, function adds +52 |
| `Column not found in schema cache` | Schema mismatch | Verify column names match table |
| `No order items found` | Trigger fired before items | Use order_items trigger, not orders |
| `Historical orders cannot be pushed` | Pre-go-live order | Expected behavior for legacy data |
| `Variant missing SKU` | Product has no SKU in Shopify | Add SKU in Shopify before sync |
| `409 Conflict` (Klaviyo) | Profile exists | Normal - upsert handles this |

---

## Testing

### Manual Webhook Testing

```bash
# Test shopify-webhook
curl -X POST \
  https://{project-ref}.supabase.co/functions/v1/shopify-webhook \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {anon_key}" \
  -d '{"id": 9999999999, "order_number": "TEST", ...}'

# Test hyper-processor (product create)
curl -X POST \
  https://{project-ref}.supabase.co/functions/v1/hyper-processor \
  -H "Content-Type: application/json" \
  -d '{"id": 123456789, "title": "Test Product", "status": "active", "variants": [{"id": 111, "sku": "TEST-001", "price": "99.00"}]}'
```

### Verify Trigger Configuration

```sql
SELECT tgname, tgrelid::regclass, tgenabled
FROM pg_trigger 
WHERE tgname LIKE '%push%' OR tgname LIKE '%shopify%' OR tgname LIKE '%klaviyo%';
```

---

*Document Version: 1.2*
*System Version: 2.0*
*Last Updated: December 3, 2025*
*Author: Ivan Duarte, Full Stack Developer*
