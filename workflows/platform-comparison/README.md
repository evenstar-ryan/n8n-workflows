# Platform Comparison Workflow

## Overview

Queries all MSP platforms and generates a comparison report showing which organizations exist in which systems, with fuzzy name matching to identify the same company across platforms.

## Purpose

**Business Problem:** Organizations are managed across 5+ different platforms with inconsistent naming. No single source shows which clients exist where, making it difficult to:
- Identify gaps in platform coverage
- Spot orphaned/stale entries
- Ensure consistent client setup across all systems

**Solution:** Automated comparison with fuzzy matching that produces actionable reports.

## Platforms Queried

1. **SuperOps** (PSA) - Client accounts
2. **Hudu** (Documentation) - Company records
3. **NinjaOne** (RMM) - Organizations
4. **Huntress** (Security) - Organizations
5. **DNS Filter** (Security) - Organizations

## Output

### HTML Report
- Color-coded table showing match status:
  - 🟢 Green: Organization exists in all 5 platforms
  - 🟡 Yellow: Organization exists in 2-4 platforms
  - 🔴 Red: Organization exists in only 1 platform
- Sortable by platform count
- Shows organization IDs for each platform

### CSV Export
- Same data in spreadsheet format
- Used for manual review and corrections
- Can be imported into other tools

## Workflow Steps
```
1. Manual Trigger
   ↓
2. Parallel API Calls (5 branches)
   ├─→ SuperOps GraphQL (getClientList)
   ├─→ Hudu REST API (/companies)
   ├─→ NinjaOne REST API (/organizations)
   ├─→ Huntress REST API (/organizations)
   └─→ DNS Filter REST API (/organizations/all)
   ↓
3. Extract & Standardize Data
   Transform each platform's response to: {platform, id, name}
   ↓
4. Merge All Platforms
   Combine into single dataset (~400-500 items)
   ↓
5. Fuzzy Match Organizations
   - Normalize names (lowercase, remove punctuation)
   - Group by 85% similarity threshold
   - Use Levenshtein distance algorithm
   ↓
6. Format Outputs
   ├─→ HTML Table (with styling and color coding)
   └─→ CSV File (for Excel/Sheets)
```

## Dependencies

### Required Credentials
- SuperOps MSP API (Multiple Headers Auth)
- Hudu API (Header Auth)
- NinjaOne API (OAuth2)
- Huntress API (Basic Auth)
- DNS Filter API (Header Auth)

See `/credentials/README.md` for setup details.

### n8n Nodes Used
- Manual Trigger
- HTTP Request (5x - one per platform)
- Code (7x - for data extraction and formatting)
- Merge (1x - to combine platform data)

## Configuration

### Fuzzy Matching Threshold
Located in "Fuzzy Match Organizations" Code node:
```javascript
const threshold = 0.85; // 85% similarity required
```

**Adjust if:**
- Too many false positives → Increase (e.g., 0.90)
- Missing obvious matches → Decrease (e.g., 0.80)

### Pagination Limits
- **Hudu:** `page_size=250` (increase if >250 companies)
- **Huntress:** `limit=500` (increase if >500 orgs)
- **SuperOps, NinjaOne, DNS Filter:** Return all by default

## Usage

### Running the Workflow

1. Open workflow in n8n
2. Click "Execute Workflow" on Manual Trigger node
3. Wait ~10-30 seconds (depends on data volume)
4. Download outputs from final nodes:
   - HTML: Click "Binary" tab → Download
   - CSV: Click "Binary" tab → Download

### Interpreting Results

**Platform Count = 5:** ✅ Fully synced across all platforms  
**Platform Count = 3-4:** ⚠️ Missing from some platforms (investigate)  
**Platform Count = 1-2:** 🔴 Likely orphaned or needs cleanup  

### Common Issues

**Issue:** Organization appears as separate entries despite same name  
**Cause:** Name variation exceeds 85% similarity threshold  
**Fix:** Manually standardize names in source platforms, or adjust threshold

**Issue:** Fewer results than expected from a platform  
**Cause:** Pagination limit reached  
**Fix:** Increase `page_size` or `limit` parameter for that platform

**Issue:** Workflow times out  
**Cause:** Too many organizations (>1000 total)  
**Fix:** Add pagination handling or run during off-hours

## Maintenance

### When to Update

- **New platform added:** Add new HTTP Request + Extract Code nodes
- **Platform API changes:** Update corresponding HTTP Request node
- **Different matching logic needed:** Modify fuzzy matching threshold or algorithm
- **Additional output formats needed:** Add new formatter Code node

### Testing After Changes

1. Run workflow with known test data
2. Verify all platforms return data
3. Check fuzzy matching catches known duplicates
4. Confirm outputs are generated correctly

## Version History

See `CHANGELOG.md` in this directory.

## Related Workflows

- **ID Mapping:** Uses output from this workflow to populate cross-platform IDs
- **Ticket Routing:** Depends on ID Mapping being complete

## Notes

- Workflow execution time: ~15-30 seconds
- Data is not persisted - run on-demand when needed
- Outputs are binary files, not stored in n8n
- All API calls use production credentials (no test mode)

## Troubleshooting

**No output from a platform:**
1. Check credential is configured correctly
2. Verify API endpoint hasn't changed
3. Test API call manually via Postman
4. Check API rate limits

**Fuzzy matching too aggressive/conservative:**
1. Review CSV output for examples
2. Adjust threshold in Code node
3. Test with sample data
4. Document threshold change in CHANGELOG

**Memory errors with large datasets:**
1. Reduce page sizes
2. Add filtering (e.g., active clients only)
3. Split into multiple workflows by platform
