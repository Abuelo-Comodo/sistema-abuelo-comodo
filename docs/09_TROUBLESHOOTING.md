# Troubleshooting Guide - Sistema Abuelo Comodo v2.0

## Overview

This document provides troubleshooting procedures for all system components including Supabase database, Shopify integration, Edge Functions, AppSheet, Klaviyo sync, and email alerts.

---

## Table of Contents

1. [Quick Diagnostic Commands](#quick-diagnostic-commands)
2. [Shopify Sync Issues](#shopify-sync-issues)
3. [Edge Function Errors](#edge-function-errors)
4. [Database Issues](#database-issues)
5. [AppSheet Problems](#appsheet-problems)
6. [Klaviyo Integration](#klaviyo-integration)
7. [Email Alerts (Resend)](#email-alerts-resend)
8. [Inventory Discrepancies](#inventory-discrepancies)
9. [Order Processing Failures](#order-processing-failures)
10. [Performance Issues](#performance-issues)
11. [Emergency Procedures](#emergency-procedures)

---

## Quick Diagnostic Commands

### Check System Health

```sql
-- Check recent orders
SELECT order_id, order_number, created_at, shopify_order_id, status
FROM orders 
ORDER BY created_at DESC 
LIMIT 10;

-- Check for orders missing Shopify sync
SELECT order_id, order_number, created_at, tipo_de_orden
FROM orders 
WHERE shopify_order_id IS NULL 
AND tipo_de_orden = 'telefónico'
AND created_at > NOW() - INTERVAL '24 hours';

-- Check inventory levels
SELECT product_id, product_name, qty_pr, qty_tp, qty_total
FROM products 
WHERE qty_total < 5
ORDER BY qty_total;

-- Check recent Edge Function activity (via pg_net)
SELECT * FROM net._http_response 
ORDER BY created DESC 
LIMIT 20;
```

### Check Trigger Status

```sql
-- All triggers in system
SELECT 
  tgname AS trigger_name,
  tgrelid::regclass AS table_name,
  tgenabled AS status
FROM pg_trigger 
WHERE tgname LIKE 'trg_%'
ORDER BY tgrelid::regclass;
```

### Check RLS Status

```sql
SELECT schemaname, tablename, rowsecurity
FROM pg_tables 
WHERE schemaname = 'public'
AND rowsecurity = false
ORDER BY tablename;
```

---

## Shopify Sync Issues

### Orders Not Syncing to Shopify

**Symptoms:**
- Phone orders created in AppSheet don't appear in Shopify
- `shopify_order_id` remains NULL

**Diagnostic Steps:**

1. Check if trigger exists:
```sql
SELECT * FROM pg_trigger WHERE tgname = 'trg_push_order_to_shopify';
```

2. Check Edge Function logs:
   - Supabase Dashboard > Edge Functions > push-order-to-shopify > Logs

3. Verify order has correct tipo_de_orden:
```sql
SELECT order_id, tipo_de_orden, shopify_order_id 
FROM orders 
WHERE order_number = 'YOUR_ORDER_NUMBER';
```

**Solutions:**

| Issue | Solution |
|-------|----------|
| Trigger disabled | `ALTER TABLE orders ENABLE TRIGGER trg_push_order_to_shopify;` |
| Missing Shopify credentials | Check Edge Function secrets: SHOPIFY_ACCESS_TOKEN, SHOPIFY_STORE_DOMAIN |
| Invalid phone number | Phone is now added to order notes, not payload - redeploy Edge Function |
| Product not in Shopify | Verify product has valid `shopify_variant_id` |

### Orders Not Syncing FROM Shopify

**Symptoms:**
- Shopify orders don't appear in Supabase
- Online orders missing

**Diagnostic Steps:**

1. Check webhook configuration in Shopify:
   - Shopify Admin > Settings > Notifications > Webhooks

2. Verify webhook URL is correct:
   - `https://irmtfzphtgjigfztbhlv.supabase.co/functions/v1/shopify-webhook`

3. Check Edge Function logs for shopify-webhook

**Solutions:**

| Issue | Solution |
|-------|----------|
| Webhook not registered | Re-register webhook in Shopify admin |
| URL incorrect | Update webhook URL in Shopify |
| Edge Function error | Check logs, redeploy function |

### Inventory Not Updating

**Symptoms:**
- Sales don't reduce inventory
- Manual adjustments not reflected

**Diagnostic Steps:**

1. Check inventory trigger:
```sql
SELECT * FROM pg_trigger WHERE tgname LIKE '%inventory%';
```

2. Verify product recipe:
```sql
SELECT * FROM product_recipes WHERE recipe_product_id = 'PRODUCT_ID';
```

3. Check inventory log:
```sql
SELECT * FROM inventory_log 
WHERE product_id = 'PRODUCT_ID' 
ORDER BY created_at DESC LIMIT 10;
```

---

## Edge Function Errors

### General Debugging

1. **Check Logs:**
   - Supabase Dashboard > Edge Functions > [Function Name] > Logs

2. **Test Function Manually:**
   - Edge Functions > [Function Name] > Test
   - Send test payload

3. **Verify Secrets:**
   - Edge Functions > Secrets
   - Ensure all required variables are set

### Common Error Codes

| Error | Meaning | Solution |
|-------|---------|----------|
| 401 | Unauthorized | Check API keys/tokens |
| 403 | Forbidden | Check permissions |
| 404 | Not found | Verify endpoint URL |
| 422 | Unprocessable | Check payload format |
| 429 | Rate limited | Wait and retry |
| 500 | Server error | Check function logs |

### Function-Specific Issues

#### push-order-to-shopify

| Error | Cause | Solution |
|-------|-------|----------|
| "phone is invalid" | Invalid phone format | Phone now in notes, not payload |
| "variant not found" | Product missing in Shopify | Check shopify_variant_id |
| "inventory not available" | Out of stock in Shopify | Sync inventory first |

#### shopify-webhook

| Error | Cause | Solution |
|-------|-------|----------|
| "duplicate order" | Order already exists | Normal - idempotent handling |
| "customer not found" | Email not in system | Creates new customer automatically |

#### sync-klaviyo-profile

| Error | Cause | Solution |
|-------|-------|----------|
| 409 Conflict | Profile exists | Normal - upsert handles this |
| "Invalid headers" | Wrong content-type | Use `application/vnd.api+json` |

---

## Database Issues

### Trigger Not Firing

**Check trigger status:**
```sql
SELECT tgname, tgenabled FROM pg_trigger WHERE tgname = 'trigger_name';
```

**Enable trigger:**
```sql
ALTER TABLE table_name ENABLE TRIGGER trigger_name;
```

### RLS Blocking Access

**Symptoms:**
- "permission denied" errors
- Empty results when data exists

**Check RLS policies:**
```sql
SELECT * FROM pg_policies WHERE tablename = 'table_name';
```

**Temporary bypass (testing only):**
```sql
SET ROLE service_role;
-- run query
RESET ROLE;
```

### Slow Queries

**Check for missing indexes:**
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE order_number = 'ABC-123';
```

**Add index if needed:**
```sql
CREATE INDEX idx_orders_order_number ON orders(order_number);
```

---

## AppSheet Problems

### Data Not Refreshing

1. Force sync in AppSheet: Settings > Sync > Sync Now
2. Check if table is in "Tables" list
3. Verify PostgreSQL connection string

### Formula Errors

**Common issues:**

| Error | Solution |
|-------|----------|
| #REF | Referenced column doesn't exist |
| #VALUE | Type mismatch in formula |
| Circular reference | Check LOOKUP chains |

### User Access Issues

**Check user_roles table:**
```sql
SELECT * FROM user_roles WHERE user_email = 'user@example.com';
```

---

## Klaviyo Integration

### Profile Not Syncing

**Diagnostic steps:**

1. Check trigger exists:
```sql
SELECT * FROM pg_trigger WHERE tgname = 'trg_sync_klaviyo_profile';
```

2. Check trigger function has correct anon key:
```sql
SELECT prosrc FROM pg_proc WHERE proname = 'notify_klaviyo_on_customer_change';
```

3. Test Edge Function directly in Supabase dashboard

**Common issues:**

| Issue | Solution |
|-------|----------|
| No logs appearing | Check anon key in trigger function |
| 401 error | Regenerate Klaviyo API key |
| 409 duplicate | Normal - upsert handles existing profiles |
| Wrong headers | Use `application/vnd.api+json` |

### Verify API Key

```bash
curl --request GET \
  --url 'https://a.klaviyo.com/api/profiles/?page[size]=1' \
  --header 'Authorization: Klaviyo-API-Key YOUR_KEY' \
  --header 'accept: application/vnd.api+json' \
  --header 'revision: 2025-04-15'
```

---

## Email Alerts (Resend)

### Alerts Not Sending

**Check configuration:**
1. Verify RESEND_API_KEY in Edge Function secrets
2. Test send-alert-email function manually
3. Check Resend dashboard for logs

**Domain verification:**
- Without verified domain, only account owner email works
- Verify domain: Resend > Domains > Add Domain > Add DNS records

### Test Alert System

```json
{
  "function_name": "test-alert",
  "message": "Prueba del sistema de alertas",
  "error_details": {"test": true}
}
```

---

## Inventory Discrepancies

### Diagnose Mismatch

```sql
-- Compare Supabase vs expected
SELECT 
  p.product_id,
  p.product_name,
  p.qty_pr AS supabase_pr,
  p.qty_tp AS supabase_tp
FROM products p
WHERE p.product_id = 'PRODUCT_ID';

-- Check inventory log
SELECT * FROM inventory_log 
WHERE product_id = 'PRODUCT_ID'
ORDER BY created_at DESC 
LIMIT 20;
```

### Force Inventory Recalculation

```sql
-- Update triggers recalculation
UPDATE products 
SET updated_at = NOW() 
WHERE product_id = 'PRODUCT_ID';
```

---

## Order Processing Failures

### Order Stuck in Processing

```sql
-- Check order status
SELECT order_id, status, shopify_order_id, fulfillment_status
FROM orders 
WHERE order_number = 'ORDER_NUMBER';

-- Check line items
SELECT * FROM lineas WHERE order_id = 'ORDER_ID';

-- Check accounting entries
SELECT * FROM asientos WHERE order_id = 'ORDER_ID';
```

### Missing Accounting Entries

```sql
-- Manually trigger accounting
UPDATE orders 
SET status = 'completado', updated_at = NOW()
WHERE order_id = 'ORDER_ID';
```

---

## Performance Issues

### Slow Dashboard Loading

1. Check for large unindexed queries
2. Review AppSheet slice filters
3. Consider archiving old data

### High Database Load

```sql
-- Check active connections
SELECT count(*) FROM pg_stat_activity;

-- Check long-running queries
SELECT pid, query, state, query_start 
FROM pg_stat_activity 
WHERE state != 'idle' 
ORDER BY query_start;
```

---

## Emergency Procedures

### Disable All Shopify Sync (Emergency)

```sql
-- Disable outbound sync
ALTER TABLE orders DISABLE TRIGGER trg_push_order_to_shopify;

-- Note: Inbound sync (webhooks) disabled by removing webhook in Shopify admin
```

### Re-enable Sync

```sql
ALTER TABLE orders ENABLE TRIGGER trg_push_order_to_shopify;
```

### Rollback RLS (If Blocking Access)

```sql
-- Disable RLS on specific table
ALTER TABLE table_name DISABLE ROW LEVEL SECURITY;

-- Or drop restrictive policy
DROP POLICY "policy_name" ON table_name;
```

### Database Backup

Supabase automatically backs up daily. For manual backup:
1. Supabase Dashboard > Settings > Database > Backups
2. Download latest backup

---

## Support Contacts

| System | Contact | Access |
|--------|---------|--------|
| Supabase | Supabase Dashboard | Admin |
| Shopify | Shopify Admin | Store Owner |
| Klaviyo | klaviyo.com | Marketing Team |
| Resend | resend.com | Developer |

---

## Log Locations

| Component | Log Location |
|-----------|--------------|
| Edge Functions | Supabase > Edge Functions > [Name] > Logs |
| Database | Supabase > Logs > Postgres |
| Webhooks | Shopify Admin > Settings > Notifications |
| Klaviyo | Klaviyo > Settings > API Keys (Last used) |
| Resend | Resend Dashboard > Emails |

---

## Version History

| Date | Version | Changes |
|------|---------|---------|
| 2025-12-01 | 1.0 | Initial troubleshooting guide |

---

Document Version: 1.0
Last Updated: 2025-12-01
Author: Ivan Duarte - ByteUp LLC
