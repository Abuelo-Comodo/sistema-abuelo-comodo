# Abuelo Comodo Inventory Management System

A modern, event-driven inventory management platform built on Supabase, replacing legacy Google Sheets infrastructure with real-time PostgreSQL triggers and Edge Functions.

---

## The Technical Landscape

What began as a functional system built on interconnected tools—Shopify, Zapier, Google Sheets, Apps Script—had evolved into an architecture constrained by its own complexity. Orders traversed a five-step pipeline with polling intervals of five minutes, creating latencies that rippled through warehouse operations. Data lived in spreadsheets without normalization, relationships existed only through AppSheet's virtual references, and the entire system depended on timing-based synchronization rather than event-driven certainty.

This project represents a fundamental architectural shift: from polling to webhooks, from spreadsheets to PostgreSQL, from batch processing to real-time triggers. The result is a system that processes orders in under two seconds rather than fifteen minutes, with data integrity guaranteed by relational constraints rather than application-level validation.

## Architecture Overview

```
+---------------------------------------------------------------------+
|                         SHOPIFY                                      |
|                    (POS PR, POS TP, Web)                            |
+-----------------------------+---------------------------------------+
                              | webhooks (orders/create, orders/fulfilled)
                              v
+---------------------------------------------------------------------+
|                    SUPABASE EDGE FUNCTIONS                          |
|         shopify-webhook | shopify-order-fulfilled                   |
|         push-order-to-shopify (bidirectional sync)                  |
+-----------------------------+---------------------------------------+
                              | INSERT/UPDATE
                              v
+---------------------------------------------------------------------+
|                      POSTGRESQL TRIGGERS                            |
|  +-------------------+  +--------------------+  +------------------+ |
|  | Process Recipes   |  | Reserve Inventory  |  | Generate Acctg   | |
|  | (order_items)     |  | (order_items)      |  | (order_items)    | |
|  +-------------------+  +--------------------+  +------------------+ |
|  +-------------------+                                               |
|  | Push to Shopify   |  <- Phone orders auto-sync                   |
|  | (order_items)     |                                               |
|  +-------------------+                                               |
+-----------------------------+---------------------------------------+
                              |
                              v
+---------------------------------------------------------------------+
|                     POSTGRESQL TABLES (29)                          |
|   orders | order_items | order_item_elements | inventory | ...      |
+-----------------------------+---------------------------------------+
                              |
                              v
+---------------------------------------------------------------------+
|                         APPSHEET                                     |
|              (Operational UI for warehouse & sales teams)            |
+---------------------------------------------------------------------+
```

## Key Innovation Vectors

**Event-Driven Processing**: Shopify webhooks trigger Edge Functions that insert data directly into PostgreSQL. No intermediaries, no polling delays. Orders flow through the system in milliseconds rather than minutes.

**Trigger-Based Business Logic**: PostgreSQL triggers execute automatically on data changes. When an order item is inserted, the system processes product recipes, reserves inventory, and generates accounting entries—all within a single transaction with guaranteed consistency.

**Bidirectional Shopify Synchronization**: Phone orders created through AppSheet automatically push to Shopify within seconds via database triggers and Edge Functions. All orders—regardless of origin—appear in Shopify for unified fulfillment management.

**Normalized Data Model**: 29 tables with 52 foreign key constraints replace the flat spreadsheet structure. Composite products with up to 19 components are handled through proper relational design rather than string concatenation and formula-based lookups.

**Dual-Location Inventory**: Stock levels track independently across Playa Regatas (PR) and Tienda Pilares (TP), with transfer workflows designed for inter-location movement.

**Soft Delete Architecture**: Orders are archived rather than deleted, preserving audit trails and enabling recovery when needed.

## Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| Database | PostgreSQL (Supabase) | Primary data store with triggers and functions |
| Backend | Edge Functions (Deno/TypeScript) | Webhook processing and API integrations |
| Frontend | AppSheet | Operational interface |
| E-commerce | Shopify | Point of sale and online store |
| Hosting | Supabase Cloud | Managed PostgreSQL and Edge Functions |
| Extensions | pg_net | Async HTTP calls from triggers |

## Database Schema

The schema organizes into logical domains:

**Core Entities**: `customers`, `products`, `categories`, `locations`, `vendors`, `advisors`

**Order Processing**: `orders`, `order_items`, `order_item_elements`, `dispatch_records`

**Inventory Management**: `inventory`, `inventory_movements`, `product_recipes`

**Prospects/CRM**: `prospects`, `prospect_orders`, `prospect_order_items`, `prospect_payments`

**Purchasing**: `purchase_orders`, `purchase_order_items`, `transfers`, `transfer_items`

**Accounting**: `accounting_entries`, `accounting_rules`

## Active Triggers

| Trigger | Table | Event | Function |
|---------|-------|-------|----------|
| trigger_auto_process_recipes | order_items | INSERT | Expands composite products into components |
| trigger_generate_accounting | order_items | INSERT | Creates accounting entries by category/location |
| trg_update_reserved_on_order | order_items | INSERT | Reserves inventory for items and components |
| trg_auto_push_on_item_insert | order_items | INSERT | Pushes phone orders to Shopify |
| trg_generate_order_id | orders | INSERT | Generates sequential order IDs by advisor |
| trg_process_order_dispatch | orders | UPDATE | Processes inventory on fulfillment |
| trg_update_customer_metrics | orders | INSERT/UPDATE | Maintains customer aggregates |

## Edge Functions

| Function | Direction | Purpose |
|----------|-----------|---------|
| shopify-webhook | Inbound | Handles orders/create events from Shopify |
| shopify-order-fulfilled | Inbound | Handles orders/fulfilled events from Shopify |
| shopify-inventory-update | Inbound | Handles inventory_levels/update events |
| push-order-to-shopify | Outbound | Pushes phone orders to Shopify Admin API |

## Migration Metrics

| Domain | Records | Notes |
|--------|---------|-------|
| Customers | 13,099 | Deduplicated from legacy system |
| Orders | 4,685 | Historical transactions preserved |
| Order Items | 9,600 | Line item detail |
| Dispatch Records | 7,110 | Fulfillment history |
| Prospects | 8,106 | Sales pipeline |
| Products | 443 | Active catalog |
| Product Recipes | 562 | Composite product formulas |
| Inventory | 673 | Stock by product/location |
| **Total** | **44,638** | Across 20 active tables |

## Documentation

| Document | Audience | Description |
|----------|----------|-------------|
| 01_RESUMEN_EJECUTIVO.pdf | Stakeholders | Executive summary in Spanish |
| 02_TECHNICAL_ARCHITECTURE.md | Engineers | System architecture and data flows |
| 03_DATABASE_REFERENCE.md | Engineers | Schema, triggers, functions reference |
| 04_EDGE_FUNCTIONS.md | Engineers | Webhook and API implementation details |
| 05_GUIA_OPERACIONES.md | Operations | Common procedures and troubleshooting |

## Project Status

### Delivered

- Database schema design and implementation (29 tables, 52 FK constraints)
- PostgreSQL triggers for automated business logic (19 triggers)
- Edge Functions for Shopify webhook processing (4 functions)
- Bidirectional Shopify synchronization for phone orders
- Soft delete architecture with archive/restore functions
- Prospectos Telefonicos to Orders workflow
- Data migration from legacy system (44,638 records)
- AppSheet connectivity to Supabase

### Pending Implementation

- HMAC validation for webhook security
- Additional webhooks (products/create, customers/create)
- Order update synchronization (edits push to Shopify)
- Purchase orders data migration
- Inter-location transfers data migration
- Next.js admin dashboard (designed, not implemented)

## The Road Ahead

The architecture establishes a foundation designed for growth. PostgreSQL scales efficiently to millions of records, and the trigger-based approach ensures business logic remains consistent regardless of data entry point—whether from Shopify webhooks, AppSheet forms, or future API integrations.

Immediate priorities include implementing HMAC validation for webhook security and completing the bidirectional sync for order updates. The table structures for purchase orders and transfers are ready; they await data migration and interface implementation.

For teams extending this system: the patterns are consistent. Edge Functions handle external integrations, triggers manage internal business logic, and AppSheet provides the operational interface. New features should follow these established conventions.

---

## Development

### Prerequisites

- Supabase project with PostgreSQL database
- Shopify store with webhook permissions and custom app
- AppSheet connected to Supabase PostgreSQL
- pg_net extension enabled for async HTTP

### Environment Variables (Edge Functions)

```
SUPABASE_URL=your_supabase_url
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
SHOPIFY_DOMAIN=your-store.myshopify.com
SHOPIFY_ACCESS_TOKEN=shpat_xxxxx
```

### Deploying Edge Functions

```bash
supabase functions deploy shopify-webhook
supabase functions deploy shopify-order-fulfilled
supabase functions deploy push-order-to-shopify
```

### Enabling pg_net Extension

```sql
CREATE EXTENSION IF NOT EXISTS pg_net;
```

---

## Author

**Ivan Duarte**
Full Stack Developer

---

## License

Proprietary - Abuelo Comodo. All rights reserved.
