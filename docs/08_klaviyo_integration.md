# Klaviyo Integration - Sistema Abuelo Comodo v2.0 (Updated)

## Overview

This document describes the automatic synchronization between Supabase customer data and Klaviyo marketing platform. When customer profiles are created or updated in Supabase, the changes are automatically synced to Klaviyo.

---

## Architecture

```
+------------------+     +-------------------+     +------------------+
|                  |     |                   |     |                  |
|    Supabase      +---->+  sync-klaviyo-    +---->+   Klaviyo API    |
|   (customers)    |     |     profile       |     |                  |
+------------------+     +-------------------+     +------------------+
        |                                                   |
        |  Database Trigger                                 v
        |  (INSERT/UPDATE)                         +------------------+
        |                                          |                  |
        +----------------------------------------->+  Klaviyo Profile |
                                                   |                  |
                                                   +------------------+
```

---

## Components

### Database Trigger

| Property | Value |
|----------|-------|
| Name | trg_sync_klaviyo_profile |
| Table | customers |
| Events | INSERT, UPDATE |
| Function | notify_klaviyo_on_customer_change() |

### Edge Function

| Property | Value |
|----------|-------|
| Name | sync-klaviyo-profile |
| URL | https://irmtfzphtgjigfztbhlv.supabase.co/functions/v1/sync-klaviyo-profile |
| Trigger | Called by database trigger on customer change |
| API Endpoint | https://a.klaviyo.com/api/profile-import |

---

## Data Mapping

### Supabase to Klaviyo Field Mapping

| Supabase Field | Klaviyo Field | Notes |
|----------------|---------------|-------|
| email | email | Primary identifier |
| first_name | first_name | Customer first name |
| last_name | last_name | Customer last name |
| phone | phone_number | E.164 format |
| customer_id | properties.customer_id | Custom property |
| - | properties.source | Always "Supabase" |
| - | properties.last_synced | Timestamp of sync |

---

## API Configuration

### Klaviyo API Details

| Property | Value |
|----------|-------|
| Base URL | https://a.klaviyo.com |
| Endpoint | /api/profile-import |
| Method | POST |
| Authentication | Klaviyo-API-Key header |

### Required Headers

```
Authorization: Klaviyo-API-Key {KLAVIYO_API_KEY}
accept: application/vnd.api+json
content-type: application/vnd.api+json
revision: 2025-04-15
```

### Request Payload Structure

```json
{
  "data": {
    "type": "profile",
    "attributes": {
      "email": "customer@example.com",
      "first_name": "John",
      "last_name": "Doe",
      "phone_number": "+525512345678",
      "properties": {
        "customer_id": "uuid-here",
        "source": "Supabase",
        "last_synced": "2025-12-01T18:41:04.685Z"
      }
    }
  }
}
```

### Response Codes

| Code | Meaning | Action |
|------|---------|--------|
| 200 | Profile updated | Success - existing profile updated |
| 201 | Profile created | Success - new profile created |
| 400 | Bad request | Check payload format |
| 401 | Unauthorized | Check API key |
| 409 | Duplicate | Profile exists (should not occur with upsert) |

---

## Environment Variables

### Supabase Edge Function Secrets

| Variable | Description | Location |
|----------|-------------|----------|
| KLAVIYO_API_KEY | Klaviyo private API key (pk_xxx) | Supabase > Edge Functions > Secrets |

### Setting Up KLAVIYO_API_KEY

1. Log in to https://klaviyo.com
2. Navigate to **Settings** > **API Keys**
3. Click **Create Private API Key**
4. Name: `Supabase Integration`
5. Permissions: `Full Access` or at minimum `Profiles: Write`
6. Copy the generated key (starts with `pk_`)
7. In Supabase Dashboard:
   - Go to **Edge Functions** > **Secrets**
   - Add new secret: `KLAVIYO_API_KEY` = `pk_xxxxx...`

---

## Database Trigger Function

```sql
CREATE OR REPLACE FUNCTION notify_klaviyo_on_customer_change()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.email IS NOT NULL THEN
    PERFORM net.http_post(
      url := 'https://irmtfzphtgjigfztbhlv.supabase.co/functions/v1/sync-klaviyo-profile',
      headers := jsonb_build_object(
        'Content-Type', 'application/json',
        'Authorization', 'Bearer {SUPABASE_ANON_KEY}'
      ),
      body := jsonb_build_object('record', row_to_json(NEW))
    );
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## Edge Function Code

```typescript
// sync-klaviyo-profile
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';

serve(async (req) => {
  try {
    const { record } = await req.json();
    
    const KLAVIYO_API_KEY = Deno.env.get('KLAVIYO_API_KEY');
    
    if (!KLAVIYO_API_KEY) {
      throw new Error('KLAVIYO_API_KEY not configured');
    }

    console.log('Syncing profile:', record.email);

    const response = await fetch('https://a.klaviyo.com/api/profile-import', {
      method: 'POST',
      headers: {
        'Authorization': `Klaviyo-API-Key ${KLAVIYO_API_KEY}`,
        'accept': 'application/vnd.api+json',
        'content-type': 'application/vnd.api+json',
        'revision': '2025-04-15'
      },
      body: JSON.stringify({
        data: {
          type: 'profile',
          attributes: {
            email: record.email,
            first_name: record.first_name,
            last_name: record.last_name,
            phone_number: record.phone,
            properties: {
              customer_id: record.customer_id,
              source: 'Supabase',
              last_synced: new Date().toISOString()
            }
          }
        }
      })
    });

    const responseText = await response.text();
    console.log('Status:', response.status, 'Response:', responseText);

    if (!response.ok) {
      throw new Error(`Klaviyo API error: ${responseText}`);
    }

    const result = JSON.parse(responseText);

    return new Response(JSON.stringify({ 
      success: true, 
      email: record.email,
      klaviyo_id: result.data?.id,
      action: response.status === 201 ? 'created' : 'updated'
    }), {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    });

  } catch (error) {
    console.error('Klaviyo sync failed:', error.message);
    
    return new Response(JSON.stringify({ 
      success: false, 
      error: error.message 
    }), {
      status: 500,
      headers: { 'Content-Type': 'application/json' }
    });
  }
});
```

---

## Testing

### Manual Test via Supabase Dashboard

1. Go to Supabase > **Edge Functions** > **sync-klaviyo-profile**
2. Click **Test**
3. Enter request body:

```json
{
  "record": {
    "customer_id": "test-123",
    "email": "test@example.com",
    "first_name": "Test",
    "last_name": "Customer",
    "phone": "+525512345678"
  }
}
```

4. Click **Send Request**
5. Verify response shows `"success": true`
6. Check Klaviyo > Audience > Profiles for the new/updated profile

### Test via Database Update

```sql
UPDATE customers 
SET first_name = 'Test Update', updated_at = NOW()
WHERE email = 'existing@example.com';
```

Then check:
1. Edge Function logs in Supabase
2. Profile in Klaviyo for updated data

---

## Troubleshooting

### Sync Not Working

| Issue | Solution |
|-------|----------|
| No logs in Edge Function | Check trigger is enabled: `SELECT tgenabled FROM pg_trigger WHERE tgname = 'trg_sync_klaviyo_profile'` |
| 401 Unauthorized | Verify KLAVIYO_API_KEY is set correctly in Supabase secrets |
| 400 Bad Request | Check payload format matches Klaviyo API spec |
| Profile not updating | Ensure email matches existing Klaviyo profile |

### Checking Trigger Status

```sql
SELECT 
  tgname, 
  tgenabled,
  tgrelid::regclass as table_name
FROM pg_trigger 
WHERE tgname = 'trg_sync_klaviyo_profile';
```

- `O` = Origin (enabled)
- `D` = Disabled
- `A` = Always

### Viewing Edge Function Logs

1. Go to Supabase > **Edge Functions**
2. Select **sync-klaviyo-profile**
3. Click **Logs** tab
4. Filter by time range or search for errors

---

## Klaviyo Account Information

| Property | Value |
|----------|-------|
| Account | Abuelo Comodo |
| Public API Key (Site ID) | Esek7g |
| Dashboard | https://klaviyo.com |
| API Keys | Settings > API Keys |
| Profiles | Audience > Profiles |

---

## Sync Behavior

### Create vs Update (Upsert)

The `/api/profile-import` endpoint performs an **upsert** operation:

- If email **does not exist** in Klaviyo: Creates new profile (201)
- If email **exists** in Klaviyo: Updates existing profile (200)

### Fields That Trigger Sync

Any INSERT or UPDATE on the `customers` table triggers sync, provided the record has a non-null email.

### Fields Synced

- email (identifier)
- first_name
- last_name
- phone
- customer_id (as custom property)

---

## Maintenance

### Updating API Key

1. Generate new key in Klaviyo > Settings > API Keys
2. Update secret in Supabase > Edge Functions > Secrets
3. Delete old key in Klaviyo

### Adding New Fields to Sync

1. Update Edge Function to include new field in payload
2. Redeploy function
3. Map field in `properties` object for custom fields

### Disabling Sync Temporarily

```sql
ALTER TABLE customers DISABLE TRIGGER trg_sync_klaviyo_profile;
```

### Re-enabling Sync

```sql
ALTER TABLE customers ENABLE TRIGGER trg_sync_klaviyo_profile;
```

---

## Related Documentation

- [06_SECURITY_CONFIGURATION.md](./06_SECURITY_CONFIGURATION.md) - Security settings
- [07_ALERT_SYSTEM.md](./07_ALERT_SYSTEM.md) - Email alerts
- [04_EDGE_FUNCTIONS.md](./04_EDGE_FUNCTIONS.md) - All Edge Functions

---

Document Version: 1.0
Last Updated: 2025-12-01
Author: Ivan Duarte 
