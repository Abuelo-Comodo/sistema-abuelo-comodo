# Alert System Configuration - Sistema Abuelo Comodo v2.0

## Overview

This document describes the automated failure notification system that monitors Edge Functions and sends email alerts when errors occur. The system uses Resend as the email delivery service integrated with Supabase Edge Functions.

---

## Architecture

```
+------------------+     +-------------------+     +------------------+
|                  |     |                   |     |                  |
|  Edge Functions  +---->+  send-alert-email +---->+   Resend API     |
|                  |     |                   |     |                  |
+------------------+     +-------------------+     +--------+---------+
                                                           |
        Monitored Functions:                               v
        - push-order-to-shopify                   +------------------+
        - shopify-webhook                         |                  |
        - shopify-order-fulfilled                 |  Email Recipients|
        - shopify-inventory-update                |                  |
                                                  +------------------+
```

---

## Components

### Email Service: Resend

| Property | Value |
|----------|-------|
| Provider | Resend (resend.com) |
| Plan | Free tier |
| Limit | 100 emails/day, 3,000/month |
| API Endpoint | https://api.resend.com/emails |

### Edge Function: send-alert-email

| Property | Value |
|----------|-------|
| Name | send-alert-email |
| URL | https://irmtfzphtgjigfztbhlv.supabase.co/functions/v1/send-alert-email |
| Trigger | Called by other Edge Functions on error |
| Authentication | Supabase service_role key |

### Alert Recipients

| Name | Email | Status |
|------|-------|--------|
| Gabriel Uzcategui | gauzcategui1@gmail.com | Active |
| Roberto Bolanos | rbrtou@gmail.com | Pending domain verification |
| Valeria | valriaurda1@gmail.com | Pending domain verification |

---

## Monitored Edge Functions

### 1. push-order-to-shopify

| Property | Value |
|----------|-------|
| Purpose | Sync phone orders from Supabase to Shopify |
| Alert Trigger | Any error during order sync |
| Alert Message | "Error al sincronizar orden con Shopify" |
| Error Details | order_id, error message, timestamp |

### 2. shopify-webhook

| Property | Value |
|----------|-------|
| Purpose | Process incoming Shopify orders |
| Alert Trigger | Any error during order processing |
| Alert Message | "Error al procesar orden de Shopify" |
| Error Details | shopify_order_id, order_number, error message, timestamp |

### 3. shopify-order-fulfilled

| Property | Value |
|----------|-------|
| Purpose | Process fulfillment updates from Shopify |
| Alert Trigger | Any error during fulfillment processing |
| Alert Message | "Error al procesar fulfillment de Shopify" |
| Error Details | shopify_order_id, error message, timestamp |

### 4. shopify-inventory-update

| Property | Value |
|----------|-------|
| Purpose | Sync inventory updates from Shopify |
| Alert Trigger | Any error during inventory update |
| Alert Message | "Error al actualizar inventario desde Shopify" |
| Error Details | inventory_item_id, error message, timestamp |

---

## Email Alert Format

When an error occurs, recipients receive an email with the following structure:

**Subject:** `[ALERTA] Falla en {function_name}`

**Body:**
```
Sistema Abuelo Comodo - Alerta de Falla

Funcion: {function_name}
Fecha/Hora: {timestamp}

Mensaje:
{error_message}

Detalles del error:
{
  "error": "error description",
  "order_id": "ABC-123",
  "timestamp": "2025-12-01T15:00:00Z"
}

Este es un correo automatico del sistema de monitoreo.
```

---

## Configuration

### Environment Variables

The following secrets must be configured in Supabase Edge Functions:

| Variable | Description | Location |
|----------|-------------|----------|
| RESEND_API_KEY | Resend API key (re_xxxxx) | Supabase > Edge Functions > Secrets |
| SUPABASE_URL | Supabase project URL | Auto-configured |
| SUPABASE_SERVICE_ROLE_KEY | Service role key | Auto-configured |

### Setting Up RESEND_API_KEY

1. Log in to https://resend.com
2. Navigate to **API Keys**
3. Click **Create API Key**
4. Name: `RESEND_API_KEY`
5. Permission: `Full access`
6. Copy the generated key (starts with `re_`)
7. In Supabase Dashboard:
   - Go to **Edge Functions** > **Secrets**
   - Add new secret: `RESEND_API_KEY` = `re_xxxxx...`

---

## Domain Verification

To send emails to all recipients (not just the account owner), the domain must be verified in Resend.

### Step 1: Add Domain in Resend

1. Log in to https://resend.com
2. Navigate to **Domains**
3. Click **Add Domain**
4. Enter: `abuelocomodo.com`
5. Click **Add**

### Step 2: DNS Records

Resend will provide DNS records to add. These typically include:

#### SPF Record (TXT)

| Type | Host/Name | Value |
|------|-----------|-------|
| TXT | @ or abuelocomodo.com | v=spf1 include:_spf.resend.com ~all |

#### DKIM Record (TXT)

| Type | Host/Name | Value |
|------|-----------|-------|
| TXT | resend._domainkey | (Provided by Resend - long string) |

### Step 3: Add DNS Records to Domain Registrar

The DNS records must be added in your domain registrar's DNS settings. Common registrars:

#### GoDaddy
1. Log in to GoDaddy
2. Go to **My Products** > **DNS**
3. Click **Add Record**
4. Select **TXT** type
5. Enter Host and Value from Resend
6. Save

#### Namecheap
1. Log in to Namecheap
2. Go to **Domain List** > **Manage**
3. Click **Advanced DNS**
4. Click **Add New Record**
5. Select **TXT Record**
6. Enter Host and Value from Resend
7. Save

#### Cloudflare
1. Log in to Cloudflare
2. Select your domain
3. Go to **DNS** > **Records**
4. Click **Add record**
5. Type: **TXT**
6. Enter Name and Content from Resend
7. Save

#### Google Domains
1. Log in to Google Domains
2. Select your domain
3. Go to **DNS**
4. Scroll to **Custom records**
5. Click **Manage custom records**
6. Add TXT records from Resend
7. Save

### Step 4: Verify in Resend

1. After adding DNS records, return to Resend
2. Go to **Domains**
3. Click **Verify** next to your domain
4. Verification can take 5-60 minutes
5. Status will change to "Verified" when complete

### Step 5: Update Edge Function

After domain verification, update `send-alert-email` to include all recipients:

```typescript
const ALERT_RECIPIENTS = [
  'gauzcategui1@gmail.com',
  'rbrtou@gmail.com',
  'valriaurda1@gmail.com'
];
```

And update the "from" address:

```typescript
from: 'Alertas AC <alertas@abuelocomodo.com>'
```

---

## Testing the Alert System

### Manual Test via Supabase Dashboard

1. Go to Supabase > **Edge Functions** > **send-alert-email**
2. Click **Test**
3. Enter request body:

```json
{
  "function_name": "test-alert",
  "message": "Prueba del sistema de alertas",
  "error_details": {
    "test": true,
    "timestamp": "2025-12-01T15:00:00Z"
  }
}
```

4. Click **Send Request**
5. Verify response is `{"success": true}`
6. Check recipient email inbox

### Manual Test via cURL

```bash
curl -X POST 'https://irmtfzphtgjigfztbhlv.supabase.co/functions/v1/send-alert-email' \
  -H 'Authorization: Bearer YOUR_SUPABASE_ANON_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "function_name": "test-alert",
    "message": "Prueba del sistema de alertas",
    "error_details": {"test": true}
  }'
```

### Verify Production Alerts

To verify alerts work in production, you can:

1. Create a phone order with invalid data to trigger `push-order-to-shopify` error
2. Check Edge Function logs in Supabase
3. Verify alert email is received

---

## Troubleshooting

### Alert Email Not Received

| Issue | Solution |
|-------|----------|
| "validation_error" 403 | Domain not verified. Only account owner email works. |
| "RESEND_API_KEY not configured" | Add API key to Supabase Edge Function secrets |
| Email in spam folder | Add sender to contacts or verify domain |
| No error in logs | Check Edge Function deployment is current |

### Checking Resend Logs

1. Log in to https://resend.com
2. Go to **Emails** or **Logs**
3. View sent/failed emails and error details

### Checking Edge Function Logs

1. Go to Supabase > **Edge Functions**
2. Select the function
3. Click **Logs** tab
4. Filter by time range or search for errors

---

## Alert Flow Diagram

```
+-------------------+
|   Order Created   |
|   (AppSheet/      |
|    Shopify)       |
+--------+----------+
         |
         v
+--------+----------+
|   Edge Function   |
|   Processing      |
+--------+----------+
         |
    +----+----+
    |         |
    v         v
+-------+  +-------+
|Success|  | Error |
+-------+  +---+---+
               |
               v
      +--------+--------+
      | send-alert-email|
      +--------+--------+
               |
               v
      +--------+--------+
      |   Resend API    |
      +--------+--------+
               |
               v
      +--------+--------+
      | Email Delivered |
      | to Recipients   |
      +-----------------+
```

---

## Maintenance

### Adding New Recipients

1. Verify domain is configured in Resend
2. Update `send-alert-email` Edge Function:

```typescript
const ALERT_RECIPIENTS = [
  'gauzcategui1@gmail.com',
  'rbrtou@gmail.com',
  'valriaurda1@gmail.com',
  'new-recipient@abuelocomodo.com'  // Add new email
];
```

3. Deploy updated function

### Removing Recipients

1. Edit `send-alert-email` Edge Function
2. Remove email from `ALERT_RECIPIENTS` array
3. Deploy updated function

### Monitoring Email Usage

1. Log in to https://resend.com
2. Go to **Metrics** or **Usage**
3. Monitor daily/monthly email count
4. Free tier: 100/day, 3,000/month

---

## Security Considerations

1. **API Key Protection**: RESEND_API_KEY is stored as Supabase secret, not in code
2. **Service Role Authentication**: Alert function called with service_role key
3. **No Sensitive Data in Emails**: Error details do not include passwords or API keys
4. **Rate Limiting**: Resend enforces rate limits to prevent abuse

---

## Resend Account Information

| Property | Value |
|----------|-------|
| Account Owner | gauzcategui1@gmail.com |
| Dashboard | https://resend.com |
| API Keys | https://resend.com/api-keys |
| Domains | https://resend.com/domains |
| Documentation | https://resend.com/docs |

---

## Related Documentation

- [06_SECURITY_CONFIGURATION.md](./06_SECURITY_CONFIGURATION.md) - RLS and access control
- [04_EDGE_FUNCTIONS.md](./04_EDGE_FUNCTIONS.md) - Edge Function reference
- [02_TECHNICAL_ARCHITECTURE.md](./02_TECHNICAL_ARCHITECTURE.md) - System architecture

---

Document Version: 1.0
Last Updated: 2025-12-01
Author: Ivan Duarte - ByteUp LLC
