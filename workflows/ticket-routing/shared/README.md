# Shared Concepts - Ticket Routing

This directory contains documentation for concepts shared across all ticket routing workflows.

---

## Cross-Platform ID Mapping

All ticket routing workflows depend on the **Cross-Platform ID Sync** workflow to maintain mappings between platform organization IDs.

**Canonical Source:** Hudu Integration Identifier assets (layout_id: 26)

**Stored IDs:**
- Hudu Company ID (implicit - asset lives under this company)
- SuperOps Client ID (custom field)
- NinjaOne Org ID (custom field)
- Huntress Org ID (custom field)
- DNS Filter Org ID (custom field)
- Avanan Org ID (custom field - reserved for future)

**Lookup Pattern:**
```javascript
// Get all Integration Identifier assets
const assets = await GET('https://evenstar.huducloud.com/api/v1/assets?asset_layout_id=26&page_size=250');

// Find matching asset by platform org ID
const matchingAsset = assets.find(asset => {
  const fields = asset.fields || [];
  const platformField = fields.find(f => f.label === 'NinjaOne Org ID'); // or Huntress, etc.
  return platformField && platformField.value === incomingOrgId;
});

// Extract SuperOps Client ID
const superopsField = matchingAsset?.fields.find(f => f.label === 'SuperOps Client ID');
const superopsId = superopsField?.value || null;
```

---

## Token Security

All webhook endpoints use query parameter tokens for authentication.

**Token Format:** SHA-256 hash (64 characters)  
**Validation:** Every request validates token before processing  
**Invalid Token Response:** 401 Unauthorized

**Example:**
```
https://n8n.evenstarmsp.com/webhook/ninjaone-alerts?token=d01175a75245b14e12e09fca7f1bdd6161aa8d4d3cd59c09485044165e231ba4
```

**Token Generation:**
```bash
openssl rand -hex 32
```

**IF Node Validation:**
```javascript
// Field: {{ $json.query.token }}
// Operation: Equal
// Value: {expected_token}
// TRUE: Continue processing
// FALSE: Return 401 Unauthorized
```

---

## Cloudflare Access Configuration

n8n instance is protected by Cloudflare Access (AzureAD SSO), but webhooks from external platforms need to bypass authentication.

**Required Policy:**
- **Name:** "n8n Webhooks Bypass"
- **Action:** Bypass
- **Application:** n8n (n8n.evenstarmsp.com)
- **Include Rule:** Everyone
- **Path:** Not configurable in Cloudflare Access UI (policy applies to all paths)

**Why Needed:**
- External platforms (NinjaOne, Huntress, etc.) can't authenticate with AzureAD
- Token validation happens at n8n workflow level
- UI access still requires Cloudflare Access login

**Security:**
- Webhook endpoints validate tokens (401 on mismatch)
- Cloudflare Access protects n8n UI
- Hybrid approach: public webhooks, private UI

---

## Merge Node Pattern

Critical pattern for preserving data across HTTP Request boundaries.

**Problem:** HTTP Request nodes replace item data with API response, losing previous context.

**Solution:** Use Merge node to combine data streams.

**Configuration:**
- **Mode:** Combine
- **Combine By:** Position
- **Input 1:** Previous node data (e.g., Parse Webhook)
- **Input 2:** HTTP Request response (e.g., GET Integration Identifiers)
- **Output:** Combined object with both datasets

**Example:**
```
Parse Webhook
  → output: {ninjaone_org_id: "41", device_name: "SERVER-01", ...}
  ↓
GET Integration Identifiers
  → output: {assets: [{...}, {...}, ...]}
  ↓
Merge (Combine by Position)
  Input 1: Parse Webhook data
  Input 2: HTTP Request data
  → output: {ninjaone_org_id: "41", device_name: "SERVER-01", ..., assets: [...]}
```

Without Merge, downstream nodes lose access to ninjaone_org_id.

---

## Severity → Priority Mapping

Consistent mapping across all platforms:

```javascript
const priorityMap = {
  'CRITICAL': 'HIGH',
  'HIGH': 'HIGH',
  'MEDIUM': 'MEDIUM',
  'LOW': 'LOW',
  'UNKNOWN': 'MEDIUM'
};
```

**Rationale:**
- **CRITICAL → HIGH:** SuperOps has no CRITICAL priority
- **HIGH → HIGH:** Direct mapping
- **MEDIUM → MEDIUM:** Direct mapping
- **LOW → LOW:** Direct mapping
- **UNKNOWN → MEDIUM:** Conservative default

**Customization:**
Per-organization mapping possible in future (e.g., client A wants all alerts as HIGH).

---

## Ticket Format

Standard structure across all platforms:

### Subject Line
`[{severity}] {device/asset_name}: {status}`

**Examples:**
- `[CRITICAL] SERVER-01: TRIGGERED`
- `[HIGH] WORKSTATION-05: DETECTED`
- `[MEDIUM] FIREWALL-01: WARNING`

### Description (HTML)
```html
<strong>🚨 Platform Alert</strong><br>
<br>
<strong>Device/Asset:</strong> {name}<br>
<strong>Severity:</strong> {severity}<br>
<strong>Status:</strong> {status}<br>
<strong>Time:</strong> {timestamp}<br>
<br>
<strong>Message:</strong><br>
{message}<br>
<br>
<hr>
<small>
<strong>Alert ID:</strong> {alert_id}<br>
<strong>Platform Org ID:</strong> {platform_org_id}<br>
<strong>Device/Asset ID:</strong> {device_id}
</small>
```

**Formatting Notes:**
- SuperOps supports HTML in descriptions
- Use `<br>` for line breaks (not `\n`)
- Use `<strong>` for labels
- Use `<hr>` for section separators
- Use `<small>` for metadata

---

## Error Handling

Consistent error handling across all workflows:

### Invalid Token (Webhooks Only)
**Response:** 401 Unauthorized  
**Body:** `{ "error": "Invalid token" }`  
**Prevents:** Unauthorized access to workflow

### No Hudu Mapping Found
**Response:** 200 OK  
**Body:** `{ "success": false, "reason": "No mapping found" }`  
**Prevents:** Platform retry loops on missing data  
**Logs:** Error message to n8n execution for troubleshooting

### Former Client (No SuperOps ID)
**Response:** 200 OK  
**Body:** `{ "success": true, "reason": "Former client - no ticket created" }`  
**Prevents:** Errors trying to create tickets for inactive clients  
**Why:** Hudu has documentation for all companies (past and present), SuperOps only has active clients

### SuperOps API Failure
**Response:** Execution fails (no response to webhook)  
**Platform Behavior:** May retry webhook (varies by platform)  
**Logs:** Full error details in n8n execution

---

## SuperOps GraphQL Ticket Creation

All workflows use GraphQL mutation to create SuperOps tickets.

### Mutation Structure
```graphql
mutation {
  createTicket(input: {
    client: { accountId: "{superops_client_id}" }
    subject: "{subject}"
    description: "{description}"
    ticketType: "INCIDENT"
    source: INTEGRATION
    priority: "{priority}"
    status: "OPEN"
    category: "{category}"
    subcategory: "{subcategory}"
  }) {
    ticketId
    displayId
    subject
    status
  }
}
```

### Critical Field Types
- `ticketType`: **string** (`"INCIDENT"`)
- `source`: **enum** (`INTEGRATION` - no quotes)
- `priority`: **string** (`"HIGH"`, `"MEDIUM"`, `"LOW"`)
- `status`: **string** (`"OPEN"`)
- `category`: **string** (must exist in SuperOps)
- `subcategory`: **string** (must exist in SuperOps)

### String Escaping
Required for description field (contains HTML, newlines, quotes):

```javascript
function escapeGraphQL(str) {
  if (!str) return '';
  return str
    .replace(/\\/g, '\\\\')   // Escape backslashes
    .replace(/"/g, '\\"')      // Escape quotes
    .replace(/\n/g, '\\n')     // Escape newlines
    .replace(/\r/g, '\\r')     // Escape carriage returns
    .replace(/\t/g, '\\t');    // Escape tabs
}
```

### Common Errors

**400 Bad Request - WrongType:**
- `ticketType` sent as enum instead of string
- Solution: Use `ticketType: "INCIDENT"`

**400 Bad Request - Missing Required Field:**
- `source` field not provided
- Solution: Use `source: INTEGRATION` (enum, no quotes)

**400 Bad Request - Dependent Validation Failed:**
- `technician` field has dependency issues
- Solution: Remove technician field, create tickets unassigned

---

## Credentials

All workflows require these n8n credentials:

### SuperOps MSP API
- **Type:** Multiple Headers Auth
- **Headers:**
  - `Authorization: Bearer {SUPEROPS_TOKEN}`
  - `X-Subdomain: evenstar`

### Hudu API
- **Type:** Header Auth
- **Header Name:** `x-api-key`
- **Header Value:** `{HUDU_API_KEY}`

### Platform-Specific Credentials
See individual platform READMEs for additional credentials (NinjaOne, Huntress, etc.)

**Security:**
- All credentials stored in n8n encrypted database
- Credentials NOT exported with workflow JSON
- Disaster recovery: See main repo README for recreation instructions

---

## Monitoring

### n8n Executions
- **Location:** n8n UI → Executions tab
- **Filter:** By workflow name
- **Success Indicators:** Green checkmarks, 200 OK responses
- **Failure Indicators:** Red X, error messages in node outputs
- **Review:** Inspect execution data to see webhook payloads and API responses

### SuperOps Tickets
- **Location:** SuperOps → Tickets → All Tickets
- **Filter:** By category (Hardware, Security, etc.)
- **Validation:**
  - Correct client assignment
  - Proper priority (HIGH, MEDIUM, LOW)
  - Readable description formatting
  - Accurate subject line

### Platform Notification History
- **NinjaOne:** Administration → Notifications → Notification History
- **Huntress:** (API-based, no notification history)
- **Hudu:** (Email-based, check email logs)

**Success Indicators:**
- 200 OK responses
- No retry attempts
- Consistent delivery timing

---

## Related Documentation

- [Cross-Platform ID Sync](../../../cross-platform-id-sync/) - Prerequisite workflow
- [Project Tracker](../../../../../n8n-automation-project-tracker.md) - Overall automation roadmap
- Platform-specific docs in parent directories

---

**Maintainer:** Ryan McKee (ryan@evenstarmsp.com)
