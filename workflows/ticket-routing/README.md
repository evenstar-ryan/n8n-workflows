# Ticket Routing Automation

**Status:** In Production (NinjaOne)  
**Last Updated:** 2024-12-23

---

## Overview

Automated routing of NOC alerts from multiple monitoring/security platforms to SuperOps ticketing system. Each platform integration creates properly formatted tickets with correct client assignment, priority mapping, and detailed alert context.

**Business Value:**
- Eliminates manual ticket creation for NOC alerts
- Ensures consistent ticket formatting and priority assignment
- Maintains client context through cross-platform ID mapping
- Reduces alert → ticket latency from minutes to seconds

---

## Architecture Decision: Webhook > API > Email

Platforms are integrated using the most reliable method available:

| Priority | Method | Latency | Reliability | Examples |
|----------|--------|---------|-------------|----------|
| 1️⃣ | **Webhooks** | ~1 second | Highest | NinjaOne |
| 2️⃣ | **API Polling** | 1-5 minutes | High | Huntress |
| 3️⃣ | **Email Parsing** | 2-10 minutes | Medium | Hudu Expirations |

**Decision Framework:**
- Native webhook support? → Use webhooks
- No webhook, has API? → Poll API every 1-5 minutes
- No webhook or API? → Parse email notifications (last resort)

---

## Platform Status

| Platform | Method | Status | Alerts/Week | Directory |
|----------|--------|--------|-------------|-----------|
| **NinjaOne** | Webhook | ✅ Production | ~50-100 | `ninjaone/` |
| **Huntress** | API Polling | 📋 Planned | ~10-20 | `huntress/` |
| **Hudu** | Email Parsing | 📋 Planned | ~5-10 | `hudu-expiration-alerts/` |

---

## Prerequisites

### 1. Cross-Platform ID Sync
All ticket routing workflows depend on the [Cross-Platform ID Sync](../../cross-platform-id-sync/) workflow to map organization IDs across platforms.

**Required Hudu Data:**
- Integration Identifier assets (layout_id: 26) for all companies
- Custom fields: NinjaOne Org ID, SuperOps Client ID, Huntress Org ID, DNS Filter Org ID

**Required SuperOps Data:**
- Custom UDF fields populated with platform IDs (udf3text = Hudu, udf4text = NinjaOne, etc.)

### 2. Credentials
Each platform requires API credentials stored in n8n:
- **SuperOps MSP API** - Multiple Headers Auth (Authorization + X-Subdomain)
- **Hudu API** - Header Auth (x-api-key)
- **Platform APIs** - Platform-specific (see individual platform READMEs)

### 3. Security
- Webhook endpoints secured with query parameter tokens (SHA-256)
- n8n instance protected by Cloudflare Access with webhook bypass policy
- All credentials encrypted in n8n database

---

## Common Components

### Severity → Priority Mapping
All platforms use consistent severity-to-priority mapping for SuperOps tickets:

```javascript
const priorityMap = {
  'CRITICAL': 'HIGH',
  'HIGH': 'HIGH',
  'MEDIUM': 'MEDIUM',
  'LOW': 'LOW',
  'UNKNOWN': 'MEDIUM'
};
```

### Ticket Template
Standard ticket structure across all platforms:

**Subject:** `[{severity}] {device/asset_name}: {status}`

**Description:**
- HTML-formatted with `<br>` tags for line breaks
- Platform identifier (NinjaOne Alert, Huntress Detection, etc.)
- Device/asset details
- Alert metadata (IDs, timestamps)

### Error Handling
All workflows implement:
- **Invalid token:** Return 401 (webhooks only)
- **No Hudu mapping:** Log error, return 200 (prevent retry loops)
- **Former client (no SuperOps ID):** Return 200, no ticket created
- **SuperOps API failure:** Execution fails, platform may retry

---

## Testing Strategy

### 1. Unit Testing (Per Platform)
- Mock webhook payload or API response
- Verify Hudu lookup succeeds
- Confirm SuperOps ticket created with correct client/priority
- Check error handling for edge cases

### 2. Integration Testing
- Trigger real alert from platform
- Verify end-to-end flow (alert → Hudu → SuperOps)
- Validate ticket appears in correct client with proper formatting

### 3. Production Rollout
- Start with 1-2 test organizations
- Monitor for 1 week
- Expand to all organizations
- Document any platform-specific quirks

---

## Troubleshooting

### "No mapping found" Errors
**Cause:** Integration Identifier asset missing or NinjaOne/Huntress Org ID not populated  
**Solution:** Run cross-platform ID sync workflow, verify Hudu has correct org IDs

### Tickets Created with Wrong Client
**Cause:** SuperOps Client ID incorrect in Hudu Integration Identifier  
**Solution:** Check Hudu asset custom field matches SuperOps accountId

### Webhooks Not Triggering
**Cause:** Cloudflare Access blocking, incorrect token, or n8n workflow not published  
**Solution:** 
1. Verify Cloudflare Access bypass policy exists
2. Check webhook URL includes correct token parameter
3. Ensure workflow is saved and published (not just in test mode)

### Former Clients Getting Tickets
**Cause:** Integration Identifier has SuperOps Client ID populated for inactive client  
**Solution:** Clear SuperOps Client ID field in Hudu for former clients (workflow will skip them)

---

## Monitoring

**n8n Executions:**
- Check `Executions` tab for each workflow
- Failed executions indicate API issues or missing mappings
- Review execution data to see webhook payloads and responses

**SuperOps Tickets:**
- Verify tickets appearing in correct client accounts
- Check priority/category/subcategory are accurate
- Confirm ticket descriptions are readable

**Platform Alerts:**
- Ensure alerts are triggering in source platform
- Check notification rules are configured correctly
- Monitor for missed alerts (platform sent but no ticket)

---

## Future Enhancements

- [ ] Slack notifications for failed ticket creation
- [ ] Automatic ticket assignment based on on-call schedule
- [ ] Alert deduplication (suppress duplicate alerts within time window)
- [ ] Alert correlation (link related alerts across platforms)
- [ ] Ticket auto-resolution when alert clears
- [ ] Custom category/subcategory mapping per client
- [ ] Integration with SuperOps automation rules

---

## Related Documentation

- [Cross-Platform ID Sync](../../cross-platform-id-sync/) - Prerequisite for all ticket routing
- [Project Tracker](../../../../n8n-automation-project-tracker.md) - Overall automation roadmap
- Platform-specific docs in each subdirectory

---

**Maintainer:** Ryan McKee (ryan@evenstarmsp.com)  
**Repository:** github.com/evenstar-ryan/n8n-workflows
