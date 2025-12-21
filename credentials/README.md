# n8n Credentials Documentation

This file documents the credentials required for all workflows. **NO ACTUAL SECRETS ARE STORED HERE.**

## Required Credentials

### SuperOps MSP API
- **Type:** Multiple Headers Auth
- **Headers:**
  - `Authorization: Bearer {api_token}`
  - `X-Subdomain: evenstar`
- **Used By:** Platform Comparison, ID Mapping, Ticket Routing
- **How to Generate:** SuperOps → Settings → API → Generate Token
- **Scopes:** Full access required

### Hudu API
- **Type:** Header Auth
- **Header:** `x-api-key: {api_key}`
- **Used By:** Platform Comparison, ID Mapping
- **How to Generate:** Hudu Admin → Basic Information → API Keys
- **Scopes:** Export data (minimum)

### NinjaOne API
- **Type:** OAuth2
- **Grant Type:** Client Credentials
- **Token URL:** `https://app.ninjarmm.com/ws/oauth/token`
- **Scopes:** `monitoring management`
- **Used By:** Platform Comparison, ID Mapping
- **How to Generate:** NinjaOne → Administration → Apps → API → Create OAuth app

### Huntress API
- **Type:** Basic Auth
- **Username:** API Key (hk_...)
- **Password:** API Secret
- **Used By:** Platform Comparison, ID Mapping
- **How to Generate:** Huntress → Account Settings → API Keys

### DNS Filter API
- **Type:** Header Auth
- **Header:** `Authorization: {api_key}`
- **Used By:** Platform Comparison, ID Mapping
- **How to Generate:** DNS Filter → Account Settings → API Key

## Credential Storage

Credentials are stored securely in n8n's encrypted database. They are NOT exported with workflows.

After disaster recovery, credentials must be manually recreated in n8n using the information above.
