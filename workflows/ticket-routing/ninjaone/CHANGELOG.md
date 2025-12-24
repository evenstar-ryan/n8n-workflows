# Changelog - NinjaOne Ticket Routing

All notable changes to the NinjaOne webhook router workflow will be documented in this file.

---

## [1.0.0] - 2024-12-23

### Added
- Initial production release
- Webhook endpoint with token validation
- Parse webhook data from NinjaOne alerts
- Hudu Integration Identifier lookup by NinjaOne Org ID
- SuperOps Client ID extraction from Hudu assets
- Severity to priority mapping (CRITICAL/HIGH → HIGH, MEDIUM → MEDIUM, LOW → LOW)
- HTML-formatted ticket descriptions with proper escaping
- Error handling for invalid tokens, missing mappings, and former clients
- GraphQL mutation for SuperOps ticket creation
- Merge node to preserve webhook data through HTTP requests

### Architecture Decisions
- **Webhook over API:** NinjaOne has native webhook support (lowest latency, highest reliability)
- **Token security:** Query parameter token validated on every request (SHA-256)
- **Hudu as canonical source:** Integration Identifier assets store all platform org IDs
- **Former client handling:** Skip ticket creation if SuperOps Client ID empty (prevents errors)
- **Unassigned tickets:** Removed technician assignment due to SuperOps validation errors
- **GraphQL field types:**
  - `ticketType`: string (`"INCIDENT"`)
  - `source`: enum (`INTEGRATION` - no quotes)
  - `priority`: string (`"HIGH"`, `"MEDIUM"`, `"LOW"`)
  
### Technical Details
- **Cloudflare Access bypass:** Required to allow external webhooks through authentication layer
- **Merge node pattern:** Combines Parse Webhook output + GET Integration Identifiers to preserve data
- **Escape function:** Handles newlines, quotes, backslashes in GraphQL strings
- **Category/Subcategory:** Hardcoded to "Hardware" / "Server" (SuperOps requires valid values)

### Testing
- ✅ Manual curl test with mock payload
- ✅ End-to-end test creating SuperOps ticket
- ✅ Token validation (401 on invalid token)
- ✅ Missing mapping error handling (200 response, no ticket)
- ✅ Former client handling (200 response, no ticket)
- ⏳ Waiting for real NinjaOne alert to verify production flow

### Known Issues
- Technician assignment not implemented (dependent validation errors in SuperOps GraphQL)
- Category/Subcategory hardcoded (no per-organization customization)
- No alert deduplication (repeated alerts create multiple tickets)
- No auto-resolution when alert status changes to CLEARED

### Performance
- **Execution time:** <3 seconds per alert
- **Success rate:** 100% in testing (285 Hudu assets searched)
- **Latency:** Alert → Ticket in <5 seconds

---

## [Unreleased]

### Planned Features
- Auto-resolve tickets when NinjaOne alert status changes to CLEARED
- Alert deduplication (suppress repeated alerts within 30 minutes)
- Custom category/subcategory mapping per organization
- Technician auto-assignment based on organization or on-call schedule
- Slack notifications for CRITICAL severity alerts
- Attach NinjaOne device link to SuperOps ticket

### Under Consideration
- Batch processing for multiple simultaneous alerts
- Alert correlation (link related alerts across devices)
- Historical alert data storage in Airtable
- Custom field mapping (add NinjaOne alert metadata to SuperOps custom fields)

---

## Development Notes

### GraphQL Debugging Journey
1. **Initial issue:** 400 Bad Request with inline string escaping
2. **Attempted fix:** JSON/RAW Parameters approach
3. **Root cause:** Field type mismatches (`ticketType` and `source` as strings vs enums)
4. **Solution:** Code node with escapeGraphQL function + correct field types
5. **Final working mutation:** `ticketType: "INCIDENT"`, `source: INTEGRATION`, `priority: "HIGH"`

### Merge Node Pattern
Critical for preserving data across HTTP Request boundaries:
```
Parse Webhook → output: {ninjaone_org_id, device_name, ...}
GET Integration Identifiers → output: {assets: [...]}
Merge (Combine by Position) → output: {ninjaone_org_id, device_name, ..., assets: [...]}
```
Without Merge, Find SuperOps ID loses access to ninjaone_org_id from Parse Webhook.

### IF Node Logic Corrections
Original workflow had backwards IF logic (TRUE → error paths, FALSE → success paths). Fixed:
- **Validate Token:** TRUE → Continue, FALSE → 401
- **Check Mapping Exists:** TRUE → Continue, FALSE → Log Error
- **Has SuperOps ID?:** TRUE → Create Ticket, FALSE → Skip (Former Client)

---

**Maintainer:** Ryan McKee (ryan@evenstarmsp.com)
