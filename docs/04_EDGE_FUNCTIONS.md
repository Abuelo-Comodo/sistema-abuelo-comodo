# Edge Functions Documentation

## Sistema Abuelo Comodo v2.0

---

## The Serverless Gateway

Where Zapier once stood as the intermediary—transforming webhooks, routing data, introducing latency with each hop—Edge Functions now serve as the direct conduit between Shopify's event stream and the PostgreSQL core. These TypeScript functions execute on Supabase's Deno runtime, processing incoming webhooks with sub-second response times and transactional certainty.

This document details the implementation, configuration, and operational characteristics of each Edge Function deployed in the system.

---

## Architecture Overview

```
SHOPIFY WEBHOOK                              POSTGRESQL TRIGGER
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
|            |                       |                       |
|            v                       v                       |
|  +---------------------------------------------------+    |
|  |          SUPABASE CLIENT (service role)           |    |
|  +---------------------------------------------------+    |
+-----------------------------------------------------------+
      |                                             |
      v                                             v
POSTGRESQL (triggers execute)              SHOPIFY ADMIN API
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
  "component_records_created": 12,
  "processing_time_ms": 187
}
```

---

## Function: push-order-to-shopify

### Purpose

Pushes phone orders created in AppSheet/Supabase to Shopify, enabling unified order management and fulfillment tracking. This function completes the bidirectional synchronization between systems.

### Endpoint

```
POST https://{project-ref}.supabase.co/functions/v1/push-order-to-shopify
```

### Trigger Mechanism

This function is called automatically via PostgreSQL trigger when order items are inserted for phone orders:

```sql
-- Trigger on order_items table
CREATE TRIGGER trg_auto_push_on_item_insert
AFTER INSERT ON order_items
FOR EACH ROW
EXECUTE FUNCTION notify_push_to_shopify_on_items();
```

The trigger function uses pg_net for asynchronous HTTP calls:

```sql
CREATE OR REPLACE FUNCTION notify_push_to_shopify_on_items()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM 1 FROM orders o
  WHERE o.order_id = NEW.order_id
    AND o.source_type IN ('Telefonico', 'AppSheet')
    AND o.shopify_order_id IS NULL
    AND (o.vendedor IS NULL OR o.vendedor != 'Shopify');
  
  IF FOUND THEN
    PERFORM net.http_post(
      url := 'https://{project-ref}.supabase.co/functions/v1/push-order-to-shopify',
      headers := jsonb_build_object(
        'Content-Type', 'application/json',
        'Authorization', 'Bearer ' || '{anon_key}'
      ),
      body := jsonb_build_object('order_id', NEW.order_id)
    );
  END IF;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### Request Flow

```
1. RECEIVE order_id from trigger or manual call
      |
2. FETCH order from Supabase
      |
3. CHECK safeguards:
      |  |-- Already uploaded? Return success (idempotent)
      |  |-- Historical order (pre 2025-11-28)? Reject
      |  |-- No order items? Reject
      |
4. BUILD Shopify payload:
      |  |-- Format phone with +52 country code
      |  |-- Map line items (variant_id or custom)
      |  |-- Build shipping address
      |  |-- Add tags (order_id, vendedor, telefonico)
      |
5. POST to Shopify Admin API
      |
6. UPDATE Supabase order:
      |  |-- shopify_order_id
      |  |-- shopify_order_number
      |  |-- order_number = 'SHO-' + number
      |  |-- uploaded_to_shopify = TRUE
      |  |-- shopify_upload_date
      |
7. RETURN success response
```

### Core Implementation

#### Historical Order Safeguard

Prevents accidental sync of legacy orders that already exist in historical data:

```typescript
const GO_LIVE_DATE = new Date('2025-11-28T00:00:00Z');
const orderCreatedAt = new Date(order.created_at);

if (orderCreatedAt < GO_LIVE_DATE) {
  return new Response(JSON.stringify({
    success: false,
    error: 'Historical orders cannot be pushed to Shopify. Only orders created after 2025-11-28 are allowed.',
    order_id: orderId,
    created_at: order.created_at
  }), { status: 400 });
}
```

#### Phone Number Formatting

Mexican phone numbers require +52 country code for Shopify API acceptance:

```typescript
const rawPhone = order.customer_phone || order.receiver_phone || order.celular || '';
const cleanPhone = rawPhone.replace(/[^0-9]/g, '');
let validPhone = undefined;

if (cleanPhone.length === 10) {
  // Mexican 10-digit number - add country code
  validPhone = '+52' + cleanPhone;
} else if (cleanPhone.length === 12 && cleanPhone.startsWith('52')) {
  // Already has country code without +
  validPhone = '+' + cleanPhone;
} else if (cleanPhone.length === 13 && cleanPhone.startsWith('521')) {
  // Has country code with mobile prefix
  validPhone = '+' + cleanPhone;
} else if (cleanPhone.length > 10) {
  // Some other international format
  validPhone = cleanPhone.startsWith('+') ? cleanPhone : '+' + cleanPhone;
}
```

#### Line Item Building

Handles both Shopify-cataloged products and custom items:

```typescript
for (const item of orderItems) {
  const variantId = await getShopifyVariantId(shopifyDomain, accessToken, item.item_id);
  
  if (variantId) {
    // Product exists in Shopify - use variant_id
    lineItems.push({
      variant_id: variantId,
      quantity: item.quantity,
      price: (item.unit_price || 0).toString()
    });
  } else {
    // Product not in Shopify - use custom line item
    lineItems.push({
      title: item.item_name || item.item_id,
      quantity: item.quantity,
      price: (item.unit_price || 0).toString(),
      requires_shipping: true
    });
  }
}
```

#### Shopify Order Payload

```typescript
const shopifyPayload = {
  order: {
    line_items: lineItems,
    email: order.customer_email || order.receiver_email || undefined,
    phone: validPhone,
    financial_status: order.first_payment ? 'paid' : 'pending',
    fulfillment_status: null,
    shipping_address: shippingAddress,
    billing_address: shippingAddress,
    note: `${order.nota || ''}\nOrden telefonica: ${orderId}\nVendedor: ${order.vendedor || 'N/A'}`.trim(),
    tags: `telefonico, ${order.vendedor || ''}, ${orderId}`.replace(/,\s*$/, ''),
    source_name: 'phone',
    send_receipt: false,
    send_fulfillment_receipt: false
  }
};
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
  "customer_id": "uuid-here",
  "location_id": "21449925",
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
  "shopify_order_id": "6286978220134",
  "shopify_order_number": 16062
}
```

#### Historical Order Rejected (400)

```json
{
  "success": false,
  "error": "Historical orders cannot be pushed to Shopify. Only orders created after 2025-11-28 are allowed.",
  "order_id": "OLD-TLF-001",
  "created_at": "2025-09-15T10:30:00Z"
}
```

#### Error Response (500)

```json
{
  "success": false,
  "error": "No order items found for this order",
  "processing_time_ms": 239
}
```

---

## Environment Variables

All functions require the following environment variables configured in Supabase:

| Variable | Description | Source |
|----------|-------------|--------|
| SUPABASE_URL | Project API URL | Supabase Dashboard: Settings > API |
| SUPABASE_SERVICE_ROLE_KEY | Service role secret key | Supabase Dashboard: Settings > API |
| SHOPIFY_DOMAIN | Shopify admin domain | abuelo-comodo.myshopify.com |
| SHOPIFY_ACCESS_TOKEN | Shopify Admin API token | Shopify Admin: Apps > Develop apps |

### Shopify API Configuration

The push-order-to-shopify function requires a custom app in Shopify:

1. Shopify Admin > Settings > Apps and sales channels > Develop apps
2. Create app: "Supabase Integration"
3. Configure Admin API scopes:
   - read_products
   - write_orders
   - read_orders
   - write_customers
   - read_customers
4. Install app and copy Admin API access token

### Configuration Commands

```bash
# Set via Supabase CLI
supabase secrets set SUPABASE_URL=https://xxxxx.supabase.co
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...
supabase secrets set SHOPIFY_DOMAIN=abuelo-comodo.myshopify.com
supabase secrets set SHOPIFY_ACCESS_TOKEN=shpat_xxxxx
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
supabase functions deploy push-order-to-shopify

# Deploy all functions
supabase functions deploy

# Verify deployment
supabase functions list
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
```

### Key Log Messages

| Message | Meaning |
|---------|---------|
| `Order already exists: {id}` | Idempotency check passed, duplicate webhook |
| `Found existing customer by Shopify ID` | Customer matched |
| `Created new customer: {id}` | New customer record |
| `Location mapped: PR -> {location_id}` | Location resolution success |
| `Order created: {order_number}` | Order insert success |
| `Triggered push to Shopify for order` | Outbound sync initiated |
| `Rejecting historical order` | Pre-go-live order blocked |
| `SKU not found in Shopify, using custom line item` | Product fallback |

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
| `Phone has already been taken` | Duplicate customer phone | Remove phone from customer object |
| `No order items found` | Trigger fired before items added | Use order_items trigger, not orders |
| `Historical orders cannot be pushed` | Order predates go-live | Expected behavior for legacy data |

---

## Testing

### Manual Webhook Testing

```bash
curl -X POST \
  https://{project-ref}.supabase.co/functions/v1/push-order-to-shopify \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {anon_key}" \
  -d '{"order_id": "TEST-TLF-001"}'
```

### Verify Trigger Configuration

```sql
SELECT tgname, tgrelid::regclass, tgenabled
FROM pg_trigger 
WHERE tgname LIKE '%push%' OR tgname LIKE '%shopify%';
```

---

*Document Version: 1.1*
*System Version: 2.0*
*Last Updated: November 28, 2025*
*Author: Ivan Duarte, Full Stack Developer*
