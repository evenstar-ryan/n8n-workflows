# Evenstar MSP - n8n Workflow Repository

This repository contains all n8n automation workflows for Evenstar MSP's operations.

## Purpose

Maintain version-controlled, documented, and transferable automation workflows that support business operations independently of individual team members.

## Repository Structure
```
n8n-workflows/
├── workflows/          # Production workflows
│   ├── platform-comparison/
│   ├── id-mapping/
│   └── ticket-routing/
├── credentials/        # Credential documentation (NO SECRETS)
├── docs/              # Architecture and troubleshooting guides
└── README.md          # This file
```

## Workflows

### Platform Comparison
**Status:** Production  
**Purpose:** Query all platforms (SuperOps, Hudu, NinjaOne, Huntress, DNS Filter) and generate comparison report showing which organizations exist where.  
**Output:** HTML table + CSV file  
**Location:** `workflows/platform-comparison/`

### ID Mapping
**Status:** In Development  
**Purpose:** Populate organization IDs across platforms (Hudu Integration Identifiers + SuperOps custom fields)  
**Dependencies:** Platform Comparison workflow  
**Location:** `workflows/id-mapping/`

### Ticket Routing
**Status:** Planned  
**Purpose:** Parse NOC alert emails and route to correct client in SuperOps  
**Dependencies:** ID Mapping workflow  
**Location:** `workflows/ticket-routing/`

## Deployment

All workflows run on: `mithrandir.evenstar.local` (DigitalOcean droplet)  
n8n accessible at: `https://n8n.evenstarmsp.com` (Cloudflare Access protected)

## Disaster Recovery

1. Fresh n8n install via Docker Compose
2. Import workflow JSON files from this repository
3. Recreate credentials (documented in `credentials/README.md`)
4. Test each workflow

## Credentials Required

See `credentials/README.md` for list of required credentials (names only, no secrets).

## Contributing

When modifying workflows:
1. Make changes in n8n UI
2. Test thoroughly
3. Export workflow as JSON
4. Update workflow's CHANGELOG.md
5. Commit with descriptive message
6. Push to GitHub

## Support

Contact: Ryan McKee (ryan@evenstarmsp.com)
