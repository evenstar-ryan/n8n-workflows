# Hudu Expiration Router

**Status:** Production  
**Version:** 1.0.0  
**Last Updated:** 2025-12-26

Automated routing of Hudu expiration alerts to SuperOps tickets with proper client assignment.

---

## Overview

This workflow receives webhook alerts from Hudu when expirations are approaching (domains, SSL certificates, warranties, custom asset fields) and automatically creates tickets in SuperOps with the correct client assignment and categorization.

### What It Does

1. Receives expiration webhook from Hudu
2. Validates security token
3. Looks up SuperOps Client ID from Hudu Integration Identifiers
4. Creates properly categorized ticket in SuperOps
5. Sets priority based on days until expiration
6. Routes to correct subcategory (Domain/SSL/Warranty/Other)

### Key Features

- **Token-secured webhook** - Prevents unauthorized access
- **Automatic client assignment** - Uses Hudu → SuperOps ID mapping
- **Dynamic priority** - HIGH (≤7 days), MEDIUM (≤14 days), LOW (>14 days)
- **Category mapping** - Routes to correct Expiration subcategory
- **Former client handling** - Skips companies without active SuperOps accounts

---

## Webhook Configuration

### Webhook URL

```
https://n8n.evenstarmsp.com/webhook/hudu-expirations?token=657864970f6659b676531a334dce9fa2ec3ac22f759e8af2a32d097e720c1507
```

**Security Token:** `657864970f6659b676531a334dce9fa2ec3ac22f759e8af2a32d097e720c1507`

### Hudu Webhook Setup

**Location:** Admin → Integrations → Webhooks

Configure **4 separate webhooks** (one per expiration type):

#### 1. Domain Expiration
- **Event Type:** Single Expiration
- **Expiration Type:** Domain Registration
- **Trigger:** X days before expiration (e.g., 30 days)
- **URL:** [webhook URL above]
- **Payload:** See below

#### 2. SSL Certificate Expiration
- **Event Type:** Single Expiration
- **Expiration Type:** SSL Certificate
- **Trigger:** X days before expiration (e.g., 14 days)
- **URL:** [webhook URL above]
- **Payload:** See below

#### 3. Warranty Expiration
- **Event Type:** Single Expiration
- **Expiration Type:** Warranty
- **Trigger:** X days before expiration (e.g., 30 days)
- **URL:** [webhook URL above]
- **Payload:** See below

#### 4. Asset Field Expiration
- **Event Type:** Single Expiration
- **Expiration Type:** Asset Field
- **Trigger:** X days before expiration (e.g., 30 days)
- **URL:** [webhook URL above]
- **Payload:** See below

### Payload (Same for All 4 Webhooks)

```json
{
  "company_id": "$COMPANY_ID",
  "company_name": "$COMPANY_NAME",
  "expiration_type": "$EXPIRATION_TYPE",
  "record_name": "$RECORD_NAME",
  "expiration_date": "$EXPIRATION_DATE",
  "trigger_days": "$TRIGGER_DAYS",
  "hudu_url": "$HUDU_URL",
  "record_id": "$RECORD_ID"
}
```

**Hudu Variables:**
- `$COMPANY_ID` - Hudu company ID
- `$COMPANY_NAME` - Company name
- `$EXPIRATION_TYPE` - domain, ssl_certificate, warranty, asset_field
- `$RECORD_NAME` - What's expiring (e.g., "evenstarmsp.com")
- `$EXPIRATION_DATE` - Expiration date
- `$TRIGGER_DAYS` - Days until expiration
- `$HUDU_URL` - Link to record in Hudu
- `$RECORD_ID` - Record ID in Hudu

---

## Workflow Architecture

```
Webhook (POST)
  ↓
Validate Token
  ├─ FALSE → Return 401 Unauthorized
  └─ TRUE → Continue
      ↓
Parse Expiration Data
  ↓
GET Integration Identifiers (Hudu API)
  ↓
Merge (Combine expiration data + API response)
  ↓
Find SuperOps ID
  ↓
Check Mapping Exists
  ├─ FALSE → Return 200 (No Mapping)
  └─ TRUE → Continue
      ↓
Has SuperOps ID?
  ├─ FALSE → Return 200 (Former Client)
  └─ TRUE → Continue
      ↓
Build Ticket Data
  ↓
Build GraphQL Request
  ↓
CREATE SuperOps Ticket (GraphQL)
  ↓
Return Success
```

### Critical Pattern: Merge Node

**Why it's needed:** HTTP Request nodes replace item data. The Merge node preserves expiration details from "Parse Expiration Data" while adding the assets array from "GET Integration Identifiers".

**Configuration:**
- Mode: **Combine**
- Combine By: **Position**
- Input 1: Parse Expiration Data
- Input 2: GET Integration Identifiers

Without this, all ticket fields show "Unknown".

---

## Category & Priority Mapping

### Expiration Type → Subcategory

| Hudu Expiration Type | SuperOps Category | SuperOps Subcategory |
|---------------------|-------------------|---------------------|
| `domain` | Expiration | Domain Expiration |
| `ssl_certificate` | Expiration | SSL Expiration |
| `warranty` | Expiration | Warranty Expiration |
| `asset_field` | Expiration | Other |

### Days Remaining → Priority

| Days Until Expiration | Priority |
|----------------------|----------|
| ≤ 7 days | HIGH |
| ≤ 14 days | MEDIUM |
| > 14 days | LOW |

---

## Ticket Format

### Subject
```
[EXPIRATION] {type_label}: {record_name} ({trigger_days} days)
```

**Example:** `[EXPIRATION] Domain Registration: evenstarmsp.com (30 days)`

### Description (HTML)

```html
<strong>🗓️ Hudu Expiration Alert</strong><br>
<br>
<strong>Type:</strong> Domain Registration<br>
<strong>Item:</strong> evenstarmsp.com<br>
<strong>Expires:</strong> 2025-02-15<br>
<strong>Days Remaining:</strong> 30<br>
<strong>Company:</strong> Evenstar MSP<br>
<br>
<strong>Action Required:</strong><br>
Review and renew before expiration date.<br>
<br>
<hr>
<small>
<strong>Hudu Link:</strong> <a href="https://evenstar.huducloud.com/companies/293/websites/12345">https://evenstar.huducloud.com/companies/293/websites/12345</a><br>
<strong>Record ID:</strong> 12345<br>
<strong>Hudu Company ID:</strong> 293
</small>
```

---

## Dependencies

### Hudu Integration Identifiers

**Asset Layout ID:** 26  
**Purpose:** Maps Hudu Company ID → SuperOps Client ID

**Custom Fields:**
- SuperOps Client ID
- NinjaOne Org ID
- Huntress Org ID
- DNS Filter Org ID

**Query Method:**
```
GET /api/v1/assets?asset_layout_id=26&company_id={hudu_company_id}
```

**Canonical Source:** Cross-platform ID sync workflow populates these assets.

### Credentials

| Platform | Credential Type | Used For |
|----------|----------------|----------|
| Hudu | Header Auth (x-api-key) | Querying Integration Identifiers |
| SuperOps | Multiple Headers Auth | Creating tickets via GraphQL |

---

## Error Handling

### No Integration Identifier Asset
**Response:** 200 OK (silent skip)  
**Reason:** Company not yet mapped across platforms  
**Action:** Run cross-platform ID sync workflow

### No SuperOps Client ID
**Response:** 200 OK (silent skip)  
**Reason:** Former client (documentation-only in Hudu)  
**Action:** None - intentional behavior

### Invalid Token
**Response:** 401 Unauthorized  
**Reason:** Token mismatch or missing  
**Action:** Check webhook URL includes correct token parameter

### SuperOps GraphQL Error
**Behavior:** Workflow fails, execution logged  
**Action:** Check SuperOps API status, verify credentials

---

## Testing

### Manual Webhook Test

```bash
curl -X POST "https://n8n.evenstarmsp.com/webhook/hudu-expirations?token=657864970f6659b676531a334dce9fa2ec3ac22f759e8af2a32d097e720c1507" \
  -H "Content-Type: application/json" \
  -d '{
    "company_id": "293",
    "company_name": "Evenstar MSP",
    "expiration_type": "domain",
    "record_name": "evenstarmsp.com",
    "expiration_date": "2025-02-15",
    "trigger_days": "30",
    "hudu_url": "https://evenstar.huducloud.com/companies/293/websites/12345",
    "record_id": "12345"
  }'
```

**Expected Response:**
```json
{
  "success": true,
  "message": "Expiration ticket created successfully"
}
```

**Verify in SuperOps:**
- Ticket created under Evenstar MSP
- Category: Expiration
- Subcategory: Domain Expiration
- Priority: LOW (30 days out)
- Subject: `[EXPIRATION] Domain Registration: evenstarmsp.com (30 days)`

---

## Troubleshooting

### Webhook not triggering
1. Verify workflow is **Active** (not just saved)
2. Check executions are enabled (Settings → Save Execution Progress)
3. Use production URL (`/webhook/`), not test URL (`/webhook-test/`)
4. Verify Cloudflare Access bypass policy includes webhook path

### Ticket shows "Unknown" for all fields
1. Check Merge node exists and is wired correctly
2. Verify Parse Expiration Data output has all fields
3. Check GET Integration Identifiers returns assets array

### No ticket created (200 response but nothing in SuperOps)
1. Verify Integration Identifier asset exists for company
2. Check SuperOps Client ID field is populated
3. Review execution log for GraphQL errors

---

## Future Enhancements

- [ ] Alert deduplication (prevent multiple tickets for same expiration)
- [ ] Renewal workflow integration (auto-create subtasks)
- [ ] Expiration digest (daily summary of upcoming expirations)
- [ ] Custom trigger days per expiration type
- [ ] Hudu note update on ticket creation

---

## Related Workflows

- **Cross-Platform ID Sync:** Populates Integration Identifier assets
- **NinjaOne Ticket Router:** Same architecture for RMM alerts

---

**Maintainer:** Ryan McKee (ryan@evenstarmsp.com)  
**Repository:** github.com/evenstar-ryan/n8n-workflows  
**Infrastructure:** mithrandir (DigitalOcean)
