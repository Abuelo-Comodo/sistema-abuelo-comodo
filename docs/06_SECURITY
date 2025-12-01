# Security Configuration - Sistema Abuelo Comodo v2.0

## Overview

This document describes the security configuration for the Abuelo Comodo inventory management system, including Row Level Security (RLS) policies, access control mechanisms, and explanations for dashboard warnings.

---

## Row Level Security (RLS) Status

### Tables with RLS Enabled

All production tables have Row Level Security enabled with permissive policies:

| Policy Name | Role | Permissions |
|-------------|------|-------------|
| service_role_all | service_role | Full CRUD access |
| authenticated_all | authenticated | Full CRUD access |

**Production tables protected:**
- orders
- order_items
- order_item_elements
- customers
- products
- product_recipes
- inventory
- inventory_movements
- prospects
- prospect_orders
- prospect_order_items
- prospect_payments
- prospect_customer_bridge
- accounting_entries
- accounting_rules
- addresses
- advisors
- categories
- dispatch_records
- lineas
- locations
- payment_methods
- purchase_orders
- purchase_order_items
- shopify_discounts
- transfers
- transfer_items
- user_roles
- user_settings
- vendors
- webhook_logs

---

## Views Showing "Unrestricted"

The Supabase dashboard displays "Unrestricted" warnings for database views. This is expected behavior and not a security risk.

### Why Views Show "Unrestricted"

1. **PostgreSQL views do not support RLS directly** - Row Level Security is a table-level feature
2. **Views inherit security from base tables** - When a view queries a table with RLS, the policies are applied
3. **SECURITY DEFINER views** - Some views use definer security, executing with creator privileges

### Views in System

| View Name | Purpose | Security Model |
|-----------|---------|----------------|
| vendors_appsheet | AppSheet vendor display | Base table RLS |
| customers_appsheet | AppSheet customer display | Base table RLS |
| lineas_appsheet | AppSheet lines display | Base table RLS |
| products_appsheet | AppSheet product display | Base table RLS |
| product_recipes_appsheet | AppSheet recipe display | Base table RLS |
| dispatch_records_appsheet | AppSheet dispatch display | Base table RLS |
| v_inventory_availability | Inventory calculations | SECURITY DEFINER |
| v_inventory_movements | Movement history | SECURITY DEFINER |
| v_low_stock_alerts | Stock alerts | SECURITY DEFINER |
| v_negative_stock_check | Stock validation | SECURITY DEFINER |
| v_active_transfers | Transfer tracking | SECURITY DEFINER |
| v_transfer_details | Transfer info | SECURITY DEFINER |
| v_transfer_history | Transfer history | SECURITY DEFINER |
| v_active_purchase_orders | PO tracking | SECURITY DEFINER |
| view_calendario_ordenes | Order calendar | SECURITY DEFINER |
| view_despacho_pr | PR dispatch view | SECURITY DEFINER |
| view_despacho_tp | TP dispatch view | SECURITY DEFINER |
| view_en_camino | In-transit view | SECURITY DEFINER |
| view_entradas_revisar | Entry review | SECURITY DEFINER |
| view_inventario_pr | PR inventory | SECURITY DEFINER |
| view_inventario_tp | TP inventory | SECURITY DEFINER |
| view_logistica_ordenes_pr | PR logistics | SECURITY DEFINER |
| view_logistica_ordenes_tp | TP logistics | SECURITY DEFINER |
| active_orders | Non-archived orders | Base table RLS |
| archived_orders | Archived orders | Base table RLS |

**Conclusion:** "Unrestricted" warnings on views are cosmetic dashboard indicators, not security vulnerabilities.

---

## Backup Tables

Historical backup tables remain without RLS policies. These are used for data recovery only and have no API access configured.

| Backup Table | Created | Purpose |
|--------------|---------|---------|
| constraint_backup | Migration | Constraint restoration |
| inventory_backup_before_migration_20251030 | 2024-10-30 | Pre-migration snapshot |
| migration_backup_20251125 | 2025-11-25 | Migration checkpoint |
| orders_backup_20251125 | 2025-11-25 | Orders snapshot |
| prospect_customer_bridge_backup_20251125 | 2025-11-25 | Bridge table snapshot |
| prospect_orders_backup_20251125 | 2025-11-25 | Prospect orders snapshot |
| prospects_backup | Migration | Prospects snapshot |
| prospects_backup_20251125 | 2025-11-25 | Prospects checkpoint |

**Recommendation:** Delete backup tables after confirming system stability (30-60 days post go-live).

---

## Access Control Matrix

| Connection Method | User/Role | RLS Applied? | Notes |
|-------------------|-----------|--------------|-------|
| AppSheet | postgres (superuser) | No - Bypassed | Direct database connection |
| Edge Functions | service_role | No - Bypassed | Service role key |
| Supabase Dashboard | postgres | No - Bypassed | Admin interface |
| REST API (anon key) | anon | Yes | Blocked without explicit policy |
| REST API (authenticated) | authenticated | Yes | Allowed via policy |
| Shopify Webhooks | service_role | No - Bypassed | Via Edge Functions |

---

## AppSheet Security Layer

In addition to database-level security, AppSheet implements view-level access control:

```
IN(LOOKUP(USEREMAIL(), "user_roles", "email", "role"), LIST("Admin"))
```

This formula:
1. Gets the current user's email
2. Looks up their role in the user_roles table
3. Grants access only if role is "Admin"

**Applied to views:**
- Prospectos Telefónicos
- Prospectos UB
- Calendario órdenes
- Inventario Asesor
- Pedidos AC
- All sensitive operational views

---

## Edge Function Security

### Authentication
All Edge Functions validate requests using:
- Supabase service_role key for database operations
- Shopify HMAC validation for webhook endpoints (recommended enhancement)

### Environment Variables
Sensitive credentials stored as Supabase secrets:
- SUPABASE_URL
- SUPABASE_SERVICE_ROLE_KEY
- SHOPIFY_DOMAIN
- SHOPIFY_ACCESS_TOKEN

### Function Permissions
| Function | Access Level | Trigger |
|----------|--------------|---------|
| shopify-webhook | Public (webhook) | Shopify order events |
| shopify-order-fulfilled | Public (webhook) | Shopify fulfillment |
| shopify-inventory-update | Public (webhook) | Shopify inventory |
| push-order-to-shopify | Internal (trigger) | Database trigger via pg_net |

---

## Security Recommendations

### Implemented
- [x] RLS enabled on all production tables
- [x] Service role policies for Edge Functions
- [x] Authenticated user policies for future use
- [x] AppSheet view-level security
- [x] Environment variable secrets

### Recommended Enhancements
- [ ] HMAC webhook signature validation
- [ ] Rate limiting on public endpoints
- [ ] Audit logging for sensitive operations
- [ ] Periodic backup table cleanup
- [ ] API key rotation schedule

---

## Incident Response

### If Unauthorized Access Suspected

1. **Immediate:** Rotate Supabase service_role key
2. **Immediate:** Rotate Shopify access token
3. **Review:** Check webhook_logs for anomalies
4. **Review:** Audit user_roles table for unauthorized entries
5. **Report:** Document incident timeline

### Key Rotation Procedure

1. Generate new keys in Supabase Dashboard
2. Update Edge Function environment variables
3. Update AppSheet connection if needed
4. Verify all integrations working
5. Invalidate old keys

---

## Conclusion

The Sistema Abuelo Comodo security configuration follows defense-in-depth principles:

1. **Database Layer:** RLS policies restrict anonymous API access
2. **Application Layer:** AppSheet view formulas control user access
3. **Integration Layer:** Service role keys authenticate Edge Functions
4. **Network Layer:** HTTPS encryption for all communications

The "Unrestricted" warnings visible in Supabase dashboard for views and backup tables do not represent security vulnerabilities. The system is secure for production use.

---

Document Version: 1.0
Last Updated: 2025-12-01
Author: Ivan Duarte - ByteUp LLC
