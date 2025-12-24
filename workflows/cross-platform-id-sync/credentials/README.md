# n8n Credentials Documentation

This file documents the credentials required for the Cross-Platform ID Sync workflow. **NO ACTUAL SECRETS ARE STORED HERE.**

## Required Credentials

### SuperOps MSP API
- **Type:** Multiple Headers Auth
- **Headers:**
  - `Authorization: Bearer {api_token}`
  - `X-Subdomain: evenstar`
- **Used By:** Platform queries, custom field updates
- **How to Generate:** SuperOps → Settings → API → Generate Token
- **Scopes:** Full access required (read clients, update clients)
- **API Endpoint:** `https://api.superops.ai/msp`
- **Documentation:** https://developer.superops.com/msp

### Hudu API
- **Type:** Header Auth
- **Header:** `x-api-key: {api_key}`
- **Used By:** Platform queries, Integration Identifier asset operations
- **How to Generate:** Hudu Admin → Basic Information → API Keys
- **Scopes:** Minimum required:
  - Read companies
  - Read/write assets
  - Export data
- **API Endpoint:** `https://evenstar.huducloud.com/api/v1`
- **Rate Limit:** 300 requests/minute

### NinjaOne API
- **Type:** OAuth2
- **Grant Type:** Client Credentials
- **Token URL:** `https://app.ninjarmm.com/ws/oauth/token`
- **Scopes:** `monitoring management`
- **Used By:** Platform queries only (read organizations)
- **How to Generate:** 
  1. NinjaOne → Administration → Apps → API
  2. Create new OAuth app
  3. Copy Client ID and Client Secret
- **API Endpoint:** `https://app.ninjarmm.com/api/v2`

### Huntress API
- **Type:** Basic Auth
- **Username:** API Key (format: `hk_...`)
- **Password:** API Secret
- **Used By:** Platform queries only (read organizations)
- **How to Generate:** 
  1. Huntress → Account Settings → API Keys
  2. Create new API key
  3. Copy key and secret immediately (secret only shown once)
- **API Endpoint:** `https://api.huntress.io/v1`
- **Documentation:** https://api.huntress.io/docs

### DNS Filter API
- **Type:** Header Auth
- **Header:** `Authorization: {api_key}`
- **Used By:** Platform queries only (read organizations)
- **How to Generate:** 
  1. DNS Filter → Account Settings → API Key
  2. Generate new key
- **API Endpoint:** `https://api.dnsfilter.com`
- **Documentation:** https://api.dnsfilter.com/docs

## Credential Storage

Credentials are stored securely in n8n's encrypted database. They are **NOT** exported with workflows.

After disaster recovery or new n8n installation:
1. Import `workflow.json` from this repository
2. Manually recreate each credential in n8n using the information above
3. Assign credentials to corresponding nodes in the workflow
4. Test workflow with a small batch before full execution

## Security Notes

- **Never commit actual API keys/secrets to Git**
- **Rotate credentials regularly** (quarterly recommended)
- **Use minimum required scopes** for each credential
- **Monitor API usage** for anomalies
- **Revoke credentials immediately** if compromised
- **Document credential rotation** in CHANGELOG.md

## Credential Assignment in Workflow

| Node Name | Credential Type | Credential Name in n8n |
|-----------|----------------|------------------------|
| SuperOps - Get Clients | Multiple Headers Auth | SuperOps MSP API |
| Hudu - Get Companies | Header Auth | Hudu API |
| NinjaOne - Get Organizations | OAuth2 | NinjaOne API |
| Huntress - Get Organizations | Basic Auth | Huntress API |
| DNS Filter - Get Organizations | Header Auth | DNS Filter API |
| Hudu - GET Integration Identifier Assets | Header Auth | Hudu API |
| Hudu - CREATE Integration Identifier Asset | Header Auth | Hudu API |
| Hudu - UPDATE Integration Identifier Asset | Header Auth | Hudu API |
| SuperOps - Update Custom Fields | Multiple Headers Auth | SuperOps MSP API |

## Testing Credentials

After recreating credentials:

1. Test each platform query node individually (click "Test step")
2. Verify data structure matches expected format
3. Check rate limits aren't exceeded
4. Run full workflow with `.slice(0, 1)` to test single company
5. Verify both Hudu and SuperOps updates work
6. Scale up to full execution

## Troubleshooting

**401 Unauthorized:**
- Credential is invalid or expired
- Check credential format matches requirements above
- Regenerate credential in source platform

**403 Forbidden:**
- Credential lacks required scopes/permissions
- Check scope requirements above
- Recreate credential with proper permissions

**429 Too Many Requests:**
- Rate limit exceeded
- Add Wait nodes between batches
- Reduce batch size or execution frequency
