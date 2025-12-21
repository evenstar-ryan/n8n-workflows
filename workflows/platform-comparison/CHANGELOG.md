# Changelog - Platform Comparison Workflow

All notable changes to this workflow will be documented in this file.

## [1.0.0] - 2024-12-20

### Added
- Initial release of Platform Comparison workflow
- Support for 5 platforms: SuperOps, Hudu, NinjaOne, Huntress, DNS Filter
- Fuzzy matching with 85% similarity threshold using Levenshtein distance
- HTML table output with color-coded match status
- CSV export for manual review
- Parallel API execution for performance
- Standardized data format across all platforms

### Technical Details
- **SuperOps:** GraphQL query via POST to /msp endpoint
- **Hudu:** REST API with page_size=250
- **NinjaOne:** OAuth2 authentication with monitoring+management scopes
- **Huntress:** Basic auth with limit=500
- **DNS Filter:** API key auth, extracts from nested data structure

### Known Limitations
- No pagination handling (assumes <250 Hudu companies, <500 Huntress orgs)
- Fuzzy matching may miss some variations (e.g., "1st" vs "First")
- Manual review required for low platform count entries
- No filtering for active/inactive companies

### Dependencies
- n8n version: 2.0.3+
- Required credentials: 5 (see credentials/README.md)
- Estimated execution time: 15-30 seconds
