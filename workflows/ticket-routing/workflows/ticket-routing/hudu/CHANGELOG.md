# Changelog - Hudu Expiration Router

All notable changes to the Hudu Expiration Router workflow will be documented in this file.

## [1.0.0] - 2025-12-26

### Added
- Initial production release
- Webhook endpoint with token security
- Token validation (401 on invalid token)
- Parse expiration data from Hudu webhook payload
- Hudu API integration to fetch Integration Identifiers
- Merge node to preserve expiration data through HTTP requests
- SuperOps Client ID lookup from Integration Identifier assets
- Dynamic category/subcategory mapping based on expiration type
- Priority calculation based on days until expiration
- GraphQL mutation to create SuperOps tickets
- HTML-formatted ticket descriptions with Hudu links
- Error handling for missing mappings and former clients
- Support for 4 expiration types: domain, ssl_certificate, warranty, asset_field

### Technical Details

**Webhook URL:**
```
https://n8n.evenstarmsp.com/webhook/hudu-expirations?token=657864970f6659b676531a334dce9fa2ec3ac22f759e8af2a32d097e720c1507
```

**Node Flow:**
1. Webhook (POST)
2. Validate Token
3. Parse Expiration Data
4. GET Integration Identifiers (Hudu API)
5. Merge (preserves data)
6. Find SuperOps ID
7. Check Mapping Exists
8. Has SuperOps ID?
9. Build Ticket Data
10. Build GraphQL Request
11. CREATE SuperOps Ticket
12. Return Success

**Category Mapping:**
- `domain` → Expiration / Domain Expiration
- `ssl_certificate` → Expiration / SSL Expiration
- `warranty` → Expiration / Warranty Expiration
- `asset_field` → Expiration / Other

**Priority Mapping:**
- ≤7 days: HIGH
- ≤14 days: MEDIUM
- >14 days: LOW

**Hudu Integration:**
- Configured 4 separate webhooks (one per expiration type)
- All use same payload format with Hudu variables
- All point to same webhook endpoint

### Lessons Learned

**Critical Pattern - Merge Node:**
HTTP Request nodes replace item data. Must use Merge node (Combine by Position) to preserve expiration details from Parse node while adding API response from GET Integration Identifiers.

**Production vs Test URLs:**
- Test URL (`/webhook-test/`): Only works with "Listen for Test Event"
- Production URL (`/webhook/`): Always active, for real webhooks

**Execution Logging:**
Ensure "Save Execution Progress" is enabled in workflow settings, otherwise executions won't show in logs even though workflow runs.

### Dependencies
- Hudu Integration Identifier asset layout (ID: 26)
- Cross-platform ID sync workflow (populates Integration Identifiers)
- Hudu API credentials (Header Auth)
- SuperOps MSP API credentials (Multiple Headers Auth)

### Testing
- ✅ Manual curl test with sample domain expiration
- ✅ Token validation (401 on invalid token)
- ✅ SuperOps ticket creation verified
- ✅ Category/subcategory mapping confirmed
- ✅ Priority calculation validated
- ✅ HTML formatting in ticket description
- ✅ Hudu webhook configured for all 4 expiration types

### Known Issues
None at this time.

### Future Enhancements
- Alert deduplication
- Renewal workflow integration
- Expiration digest (daily summary)
- Custom trigger days per type
- Hudu note update on ticket creation
