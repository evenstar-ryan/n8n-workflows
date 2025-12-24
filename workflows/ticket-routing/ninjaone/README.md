# NinjaOne Webhook → SuperOps Ticket Routing

**Status:** ✅ Production  
**Method:** Webhook (native support)  
**Latency:** ~1 second  
**Last Updated:** 2024-12-23

---

## Overview

Receives webhook notifications from NinjaOne RMM for device alerts and creates corresponding tickets in SuperOps with proper client assignment, priority mapping, and formatted alert details.

**Flow:**
```
NinjaOne Alert
  ↓ (webhook POST)
Webhook Endpoint (token validation)
  ↓
Parse Alert Data
  ↓
Hudu Lookup (NinjaOne Org ID → SuperOps Client ID)
  ↓
Build Ticket Data (severity → priority mapping)
  ↓
Create SuperOps Ticket (GraphQL mutation)
  ↓
Return Success
```

---

## Webhook Configuration

### Endpoint Details
**URL:** `https://n8n.evenstarmsp.com/webhook/ninjaone-alerts?token={TOKEN}`  
**Method:** POST  
**Content-Type:** application/json

### Security Token
**Token:** `d01175a75245b14e12e09fca7f1bdd6161aa8d4d3cd59c09485044165e231ba4`

**CRITICAL:** Token is validated on every request. Invalid tokens return 401 Unauthorized.

### Cloudflare Access Configuration
**Bypass Policy Required:**
- **Policy Name:** "n8n Webhooks Bypass"
- **Action:** Bypass
- **Include Rule:** Everyone
- **Why:** Cloudflare Access normally requires authentication, which blocks external webhooks

---

## NinjaOne Setup

### 1. Create Notification Channel
1. Go to **Administration → Notifications → Notification Channels**
2. Click **Add Notification Channel**
3. Select **Webhook**
4. **Name:** "n8n Ticket Router"
5. **URL:** `https://n8n.evenstarmsp.com/webhook/ninjaone-alerts?token=d01175a75245b14e12e09fca7f1bdd6161aa8d4d3cd59c09485044165e231ba4`
6. **Method:** POST
7. **Content Type:** application/json
8. **Payload:** (use NinjaOne default)
9. Save

### 2. Create Notification Rule
1. Go to **Administration → Notifications → Notification Rules**
2. Click **Add Notification Rule**
3. **Name:** "Critical Alerts to n8n"
4. **Trigger:** Device Alert
5. **Conditions:**
   - Severity: CRITICAL, HIGH (or all severities)
   - Organization: Start with 1-2 test orgs
6. **Channel:** n8n Ticket Router
7. Save

### 3. Production Rollout
- Monitor test orgs for 1 week
- Verify tickets created correctly
- Expand to all organizations

---

## Expected Webhook Payload

NinjaOne sends webhook payloads with the following structure:

```json
{
  "organizationId": 41,
  "deviceName": "SERVER-01",
  "deviceId": 12345,
  "alertId": "alert-abc-123",
  "severity": "CRITICAL",
  "status": "TRIGGERED",
  "message": "High CPU usage detected: 95% for 10 minutes",
  "timestamp": "2024-12-23T22:30:00Z"
}
```

**Key Fields:**
- `organizationId` - Used to lookup SuperOps Client ID in Hudu
- `deviceName` - Included in ticket subject
- `severity` - Mapped to SuperOps priority
- `status` - Included in ticket subject (TRIGGERED, CLEARED, etc.)
- `message` - Full alert description

---

## Workflow Components

### Node 1: Webhook
- **Type:** Webhook
- **Method:** POST
- **Path:** `ninjaone-alerts`
- **Authentication:** None (handled by Validate Token node)
- **Response:** Returns 200 OK with ticket ID on success

### Node 2: Validate Token
- **Type:** IF
- **Condition:** `{{ $json.query.token }}` equals `d01175a75245b14e12e09fca7f1bdd6161aa8d4d3cd59c09485044165e231ba4`
- **TRUE:** Continue to Parse Webhook
- **FALSE:** Return 401 Unauthorized

### Node 3: Parse Webhook
- **Type:** Code
- **Purpose:** Extract and normalize webhook data
- **Output:**
```javascript
{
  ninjaone_org_id: "41",
  device_name: "SERVER-01",
  device_id: "12345",
  alert_id: "alert-abc-123",
  severity: "CRITICAL",
  status: "TRIGGERED",
  message: "High CPU usage...",
  timestamp: "2024-12-23T22:30:00Z"
}
```

### Node 4: GET Integration Identifiers
- **Type:** HTTP Request
- **Method:** GET
- **URL:** `https://evenstar.huducloud.com/api/v1/assets?asset_layout_id=26&page_size=250`
- **Auth:** Hudu API (x-api-key header)
- **Purpose:** Fetch all Integration Identifier assets

### Node 5: Merge
- **Type:** Merge
- **Mode:** Combine
- **Combine By:** Position
- **Purpose:** Preserve Parse Webhook data + add Hudu assets array
- **Critical:** Without this, Find SuperOps ID loses access to ninjaone_org_id

### Node 6: Find SuperOps ID
- **Type:** Code
- **Purpose:** Search assets for matching NinjaOne Org ID, extract SuperOps Client ID
- **Logic:**
```javascript
const assets = $input.item.json.assets || [];
const ninjaoneOrgId = $input.item.json.ninjaone_org_id;

const matchingAsset = assets.find(asset => {
  const fields = asset.fields || [];
  const ninjaField = fields.find(f => f.label === 'NinjaOne Org ID');
  return ninjaField && ninjaField.value === ninjaoneOrgId;
});

const superopsField = matchingAsset?.fields.find(f => f.label === 'SuperOps Client ID');
const superopsId = superopsField?.value || null;
```

### Node 7: Check Mapping Exists
- **Type:** IF
- **Condition:** `{{ $json.mapping_found }}` is true
- **TRUE:** Continue to Has SuperOps ID?
- **FALSE:** Log Error → Return 200 (No Mapping)

### Node 8: Has SuperOps ID?
- **Type:** IF
- **Condition:** `{{ $json.superops_id }}` is not empty
- **TRUE:** Continue to Build Ticket Data
- **FALSE:** Return 200 (Former Client - no ticket)
- **Why:** Former clients have Hudu records but no active SuperOps account

### Node 9: Build Ticket Data
- **Type:** Code
- **Purpose:** Map severity → priority, format subject and description
- **Priority Mapping:**
```javascript
const priorityMap = {
  'CRITICAL': 'HIGH',
  'HIGH': 'HIGH',
  'MEDIUM': 'MEDIUM',
  'LOW': 'LOW',
  'UNKNOWN': 'MEDIUM'
};
```
- **Ticket Format:**
  - **Subject:** `[CRITICAL] SERVER-01: TRIGGERED`
  - **Description:** HTML-formatted with device details, severity, message, metadata

### Node 10: Build GraphQL Request
- **Type:** Code
- **Purpose:** Construct GraphQL mutation with proper escaping
- **Escaping:** Handles newlines, quotes, backslashes for GraphQL string safety
- **Output:**
```json
{
  "query": "mutation { createTicket(input: {...}) { ticketId displayId subject status } }"
}
```

### Node 11: CREATE SuperOps Ticket
- **Type:** HTTP Request
- **Method:** POST
- **URL:** `https://api.superops.ai/msp`
- **Auth:** SuperOps MSP API (Bearer token + X-Subdomain header)
- **Body:** `{{ $json }}`
- **Response:** Ticket ID and details

### Node 12: Return Success
- **Type:** Respond to Webhook
- **Status:** 200 OK
- **Body:** `{ "success": true, "ticketId": "..." }`

---

## SuperOps Ticket Details

### Ticket Fields
- **Client:** Mapped via SuperOps Client ID from Hudu
- **Subject:** `[{severity}] {device_name}: {status}`
- **Description:** HTML-formatted alert details
- **Type:** INCIDENT
- **Source:** INTEGRATION (enum, not string)
- **Priority:** HIGH, MEDIUM, or LOW (string)
- **Status:** OPEN
- **Category:** Hardware
- **Subcategory:** Server

**Note:** Technician assignment removed (dependent validation failed). Tickets created unassigned.

### GraphQL Mutation Structure
```graphql
mutation {
  createTicket(input: {
    client: { accountId: "3602625627970560000" }
    subject: "[CRITICAL] SERVER-01: TRIGGERED"
    description: "<strong>🚨 NinjaOne Alert</strong><br>..."
    ticketType: "INCIDENT"
    source: INTEGRATION
    priority: "HIGH"
    status: "OPEN"
    category: "Hardware"
    subcategory: "Server"
  }) {
    ticketId
    displayId
    subject
    status
  }
}
```

**Critical Field Notes:**
- `ticketType`: **string** (`"INCIDENT"`)
- `source`: **enum** (`INTEGRATION` - no quotes)
- `priority`: **string** (`"HIGH"`)
- Technician assignment causes `dependent_validation_failed` error

---

## Testing

### Manual Webhook Test
```bash
curl -X POST "https://n8n.evenstarmsp.com/webhook/ninjaone-alerts?token=d01175a75245b14e12e09fca7f1bdd6161aa8d4d3cd59c09485044165e231ba4" \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": 41,
    "deviceName": "TEST-DEVICE",
    "deviceId": 12345,
    "alertId": "test-alert-001",
    "severity": "CRITICAL",
    "status": "TRIGGERED",
    "message": "Test alert from curl",
    "timestamp": "2024-12-23T22:30:00Z"
  }'
```

**Expected Result:**
- n8n execution shows in Executions tab
- SuperOps ticket created for Evenstar MSP (accountId: 3602625627970560000)
- Ticket subject: `[CRITICAL] TEST-DEVICE: TRIGGERED`
- Ticket priority: HIGH

### Test Data (Evenstar MSP)
- **NinjaOne Org ID:** 41
- **Hudu Company ID:** 293
- **SuperOps Client ID:** 3602625627970560000
- **Huntress Org ID:** 457086
- **DNS Filter Org ID:** 8380324

---

## Troubleshooting

### Issue: Webhook returns 401
**Cause:** Invalid or missing token  
**Solution:** Verify token in URL matches: `d01175a75245b14e12e09fca7f1bdd6161aa8d4d3cd59c09485044165e231ba4`

### Issue: Webhook returns 302 redirect
**Cause:** Cloudflare Access blocking webhook  
**Solution:** Verify Cloudflare Access bypass policy exists for n8n webhooks

### Issue: "No mapping found" error
**Cause:** Integration Identifier asset doesn't exist or NinjaOne Org ID not populated  
**Solution:** 
1. Check Hudu for Integration Identifier asset for this company
2. Verify "NinjaOne Org ID" field matches webhook organizationId
3. Run cross-platform ID sync workflow

### Issue: Ticket created for wrong client
**Cause:** SuperOps Client ID incorrect in Hudu  
**Solution:** Verify "SuperOps Client ID" field in Integration Identifier matches SuperOps accountId

### Issue: Missing webhook data in Find SuperOps ID
**Cause:** Merge node not combining data correctly  
**Solution:** Verify Merge node connects Parse Webhook (Input 1) + GET Integration Identifiers (Input 2)

### Issue: GraphQL 400 Bad Request
**Cause:** Invalid mutation syntax or field types  
**Solution:**
- Verify `ticketType` is string: `"INCIDENT"`
- Verify `source` is enum: `INTEGRATION` (no quotes)
- Verify `priority` is string: `"HIGH"`, `"MEDIUM"`, or `"LOW"`
- Check escapeGraphQL function handles newlines/quotes correctly

### Issue: Former client getting tickets
**Cause:** Integration Identifier has SuperOps Client ID for inactive client  
**Solution:** Remove SuperOps Client ID from Hudu Integration Identifier (workflow will return 200 without creating ticket)

---

## Monitoring

### n8n Executions
- Navigate to **Executions** tab in n8n
- Filter to "NinjaOne Webhook Router" workflow
- Review execution data for each alert
- Check for failed executions (red indicators)

### SuperOps Tickets
- Filter tickets by Category: Hardware, Subcategory: Server
- Verify client assignment correct
- Check priority matches alert severity
- Confirm description formatting readable

### NinjaOne Notifications
- Monitor **Administration → Notifications → Notification History**
- Verify webhooks sent successfully (200 OK responses)
- Check for failed deliveries (retry attempts)

---

## Performance Metrics

**Expected Metrics (Production):**
- **Alerts/Week:** 50-100 (varies by client activity)
- **Execution Time:** <3 seconds per alert
- **Success Rate:** >95% (failures due to missing mappings)
- **Latency:** Alert → Ticket in <5 seconds

**Actual Metrics (To Be Measured):**
- TBD after 1 week of production monitoring

---

## Future Enhancements

- [ ] Auto-resolve tickets when alert status changes to CLEARED
- [ ] Alert deduplication (suppress repeated alerts within 30 minutes)
- [ ] Custom category/subcategory per organization
- [ ] Technician auto-assignment based on organization or on-call schedule
- [ ] Slack notifications for CRITICAL alerts
- [ ] Attach NinjaOne device link to ticket

---

## Version History

See [CHANGELOG.md](./CHANGELOG.md) for detailed version history.

---

**Maintainer:** Ryan McKee (ryan@evenstarmsp.com)  
**NinjaOne Docs:** https://app.ninjaone.com/apidocs/  
**SuperOps Docs:** https://developer.superops.com/msp
