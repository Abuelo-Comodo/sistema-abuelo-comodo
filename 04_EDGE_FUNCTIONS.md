# Edge Functions Documentation

## Sistema Abuelo Cómodo v2.0

---

## The Serverless Gateway

Where Zapier once stood as the intermediary—transforming webhooks, routing data, introducing latency with each hop—Edge Functions now serve as the direct conduit between Shopify's event stream and the PostgreSQL core. These TypeScript functions execute on Supabase's Deno runtime, processing incoming webhooks with sub-second response times and transactional certainty.

This document details the implementation, configuration, and operational characteristics of each Edge Function deployed in the system.

---

## Architecture Overview

```
SHOPIFY WEBHOOK
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│                   SUPABASE EDGE RUNTIME                  │
│                        (Deno)                            │
│  ┌────────────────────┐  ┌────────────────────────────┐ │
│  │  shopify-webhook   │  │  shopify-order-fulfilled   │ │
│  │  (orders/create)   │  │  (orders/fulfilled)        │ │
│  └─────────┬──────────┘  └──────────────┬─────────────┘ │
│            │                             │               │
│            ▼                             ▼               │
│  ┌─────────────────────────────────────────────────────┐│
│  │              SUPABASE CLIENT (service role)         ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
      │
      ▼
POSTGRESQL (triggers execute automatically)
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
      │
2. PARSE JSON body
      │
3. CHECK idempotency (existing order by shopify_order_id?)
      │  ├── YES → Return 200 with "already processed"
      │  └── NO → Continue
      │
4. FIND OR CREATE customer
      │  ├── Search by shopify_customer_id
      │  ├── Search by email
      │  └── Create new if not found
      │
5. MAP location (Shopify location_id → internal location_id)
      │
6. INSERT order record
      │
7. FOR EACH line_item:
      │  ├── Validate product exists by SKU
      │  ├── INSERT order_item
      │  │     └── TRIGGERS EXECUTE:
      │  │         ├── trigger_auto_process_recipes
      │  │         ├── trg_update_reserved_on_order
      │  │         └── trigger_generate_accounting
      │  └── Capture shopify_inventory_item_id if present
      │
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
      // Update with Shopify ID if we have it
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

#### Order Data Mapping

Shopify order fields map to internal schema:

| Shopify Field | Internal Field | Transformation |
|---------------|----------------|----------------|
| id | order_id, shopify_order_id | toString() |
| order_number | shopify_order_number | Direct |
| order_number | order_number | 'SHO-' + value |
| customer.id | customer_id | Lookup/create |
| location_id | location_id | Map to internal |
| subtotal_price | subtotal | parseFloat() |
| total_discounts | discount | parseFloat() |
| total_price | total_price | parseFloat() |
| financial_status | financial_status | Direct |
| created_at | order_timestamp | Direct |
| created_at | order_date | split('T')[0] |
| payment_gateway_names[0] | first_payment_method | Direct |
| shipping_lines[0].title | delivery_method | Direct |
| source_name | procedencia | Direct |

#### Order Item Processing

```typescript
for (const item of shopifyOrder.line_items) {
  // Skip items without SKU
  if (!item.sku) {
    console.warn(`Item "${item.name}" has no SKU, skipping...`);
    continue;
  }

  // Validate product exists
  const { data: product, error: productError } = await supabase
    .from('products')
    .select('item_id, name')
    .eq('item_id', item.sku)
    .single();

  if (productError || !product) {
    console.warn(`Product not found: ${item.sku}`);
    continue;
  }

  // Create order item
  const orderItem = {
    order_item_id: generateUid(12),
    order_id: order.order_id,
    item_id: item.sku,
    item_name: item.name,
    quantity: item.quantity,
    unit_price: parseFloat(item.price),
    total_price: parseFloat(item.price) * item.quantity,
    order_date: order.order_date,
    order_timestamp: order.order_timestamp
  };

  await supabase.from('order_items').insert(orderItem);
  
  // Capture inventory_item_id for future sync
  if (item.variant_inventory_management === 'shopify') {
    await supabase
      .from('products')
      .update({ shopify_inventory_item_id: item.inventory_item_id })
      .eq('item_id', item.sku);
  }
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

### Error Handling

| Scenario | Handling |
|----------|----------|
| Invalid JSON | Return 400 with parse error |
| Unknown location | Throw error, return 500 |
| Location not in database | Throw error, return 500 |
| Order insert failure | Throw error, return 500 |
| Product not found | Log warning, skip item, continue |
| Item insert failure | Log error, continue with remaining |

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
      │
2. PARSE JSON body
      │
3. FIND order by shopify_order_id
      │  └── NOT FOUND → Return 500 error
      │
4. UPDATE order status
      │  ├── fulfillment_status = 'fulfilled'
      │  └── warehouse_status = 'despachado'
      │
5. FOR EACH fulfillment:
      │
      FOR EACH line_item:
      │  ├── Find order_item by order_id + SKU
      │  ├── CREATE dispatch_record (main product)
      │  │
      │  ├── QUERY order_item_elements for components
      │  │
      │  FOR EACH component:
      │     └── CREATE dispatch_record (component)
      │
6. RETURN success response with dispatch counts
```

### Core Implementation

#### Order Status Update

```typescript
const { error: updateError } = await supabase
  .from('orders')
  .update({
    fulfillment_status: 'fulfilled',
    warehouse_status: 'despachado',
    updated_at: new Date().toISOString()
  })
  .eq('shopify_order_id', shopifyOrderId);
```

#### Dispatch Record Creation

For each fulfilled line item, the function creates dispatch records for both the main product and its components:

```typescript
// Main product dispatch
const dispatchRecord = {
  dispatch_id: generateUid(12),
  order_item_id: orderItem.order_item_id,
  order_id: orderData.order_id,
  item_id: line_item.sku,
  location_id: orderData.location_id,
  inventory_name: line_item.name,
  quantity_dispatched: line_item.quantity,
  dispatch_date: fulfillment.created_at.split('T')[0],
  motivo: 'Fulfillment Shopify',
  created_at: new Date().toISOString()
};

await supabase.from('dispatch_records').insert(dispatchRecord);

// Component dispatches
const { data: elements } = await supabase
  .from('order_item_elements')
  .select('element_item_id, element_name, quantity')
  .eq('order_item_id', orderItem.order_item_id);

for (const element of elements || []) {
  const elementDispatch = {
    dispatch_id: generateUid(12),
    order_item_id: orderItem.order_item_id,
    order_id: orderData.order_id,
    item_id: element.element_item_id,
    location_id: orderData.location_id,
    inventory_name: element.element_name,
    quantity_dispatched: element.quantity,
    dispatch_date: fulfillment.created_at.split('T')[0],
    motivo: `Component of ${line_item.sku}`,
    created_at: new Date().toISOString()
  };

  await supabase.from('dispatch_records').insert(elementDispatch);
}
```

### Response Schema

#### Success Response (200)

```json
{
  "success": true,
  "order_id": "7654321098765",
  "dispatch_records_created": 5,
  "component_records_created": 12,
  "processing_time_ms": 187
}
```

#### With Warnings (200)

```json
{
  "success": true,
  "order_id": "7654321098765",
  "dispatch_records_created": 4,
  "processing_time_ms": 203,
  "errors": [
    "Order item not found for SKU: UNKNOWN-SKU"
  ]
}
```

---

## Environment Variables

Both functions require the following environment variables configured in Supabase:

| Variable | Description | Source |
|----------|-------------|--------|
| SUPABASE_URL | Project API URL | Supabase Dashboard → Settings → API |
| SUPABASE_SERVICE_ROLE_KEY | Service role secret key | Supabase Dashboard → Settings → API |

### Configuration

```bash
# Set via Supabase CLI
supabase secrets set SUPABASE_URL=https://xxxxx.supabase.co
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...
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

# Deploy all functions
supabase functions deploy

# Deploy with specific environment
supabase functions deploy shopify-webhook --project-ref {project-id}
```

### Verification

```bash
# Check function status
supabase functions list

# View logs
supabase functions logs shopify-webhook
supabase functions logs shopify-order-fulfilled
```

---

## Monitoring and Debugging

### Log Patterns

Each function logs key events for debugging:

```typescript
console.log('=== Shopify Webhook Received ===');
console.log('Order ID:', shopifyOrder.id);
console.log('Order Number:', shopifyOrder.order_number);
console.log('Location ID:', shopifyOrder.location_id);
console.log('Customer:', shopifyOrder.customer?.email || 'Guest');
```

### Key Log Messages

| Message | Meaning |
|---------|---------|
| `Order already exists: {id}` | Idempotency check passed, duplicate webhook |
| `Found existing customer by Shopify ID` | Customer matched |
| `Created new customer: {id}` | New customer record |
| `Location mapped: PR → {location_id}` | Location resolution success |
| `✓ Order created: {order_number}` | Order insert success |
| `✓ Dispatch record created: {sku}` | Dispatch insert success |
| `Item "{name}" has no SKU, skipping` | Line item without SKU |
| `Product not found: {sku}` | SKU not in products table |

### Supabase Dashboard Monitoring

1. Navigate to Edge Functions in Supabase Dashboard
2. Select function to view invocation logs
3. Filter by time range or search log content
4. Monitor invocation counts and error rates

---

## Security Considerations

### Current State

The functions authenticate to the database using the service role key, which bypasses Row Level Security. This is appropriate for server-side webhook processing but requires that the webhook endpoint itself be secured.

### Pending: HMAC Validation

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

const shopifyOrder = JSON.parse(body);
```

**Implementation Priority**: High - should be implemented before production launch with significant order volume.

---

## Planned Functions

### inventory-sync (Not Yet Implemented)

Outbound synchronization of inventory levels to Shopify.

**Trigger**: inventory table UPDATE  
**Action**: Call Shopify Inventory API to update stock levels

### products-webhook (Not Yet Implemented)

Inbound synchronization of product changes from Shopify.

**Webhook Topic**: products/create, products/update  
**Action**: INSERT/UPDATE products table

### customers-webhook (Not Yet Implemented)

Inbound synchronization of customer data from Shopify.

**Webhook Topic**: customers/create, customers/update  
**Action**: INSERT/UPDATE customers table

---

## Testing

### Manual Webhook Testing

Use curl or Postman to simulate webhook calls:

```bash
curl -X POST \
  https://{project-ref}.supabase.co/functions/v1/shopify-webhook \
  -H "Content-Type: application/json" \
  -d '{
    "id": 9999999999999,
    "order_number": 9999,
    "location_id": "21449925",
    "email": "test@example.com",
    "total_price": "1500.00",
    "line_items": [
      {
        "sku": "TEST-SKU",
        "name": "Test Product",
        "quantity": 1,
        "price": "1500.00"
      }
    ]
  }'
```

### Shopify Test Orders

1. Enable test mode in Shopify Payments
2. Create test order through storefront
3. Monitor Edge Function logs for processing
4. Verify database records created correctly

---

*Document Version: 1.0*  
*System Version: 2.0*  
*Author: Ivan Duarte, Full Stack Developer*
