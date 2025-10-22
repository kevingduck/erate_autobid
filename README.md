# E-rate RFP Automation System

Automated end-to-end solution for processing E-rate RFP emails, extracting requirements, generating bills of materials, and creating proposal documents for Converged Networks.

## Overview

This system automates the entire E-rate RFP response process:

1. **Email Monitoring** - Monitors inbox for daily E-rate ProfitWorks emails
2. **Application Scraping** - Extracts links and downloads RFP/470 PDFs
3. **PDF Analysis** - Uses AI to extract equipment requirements from PDFs
4. **BOM Generation** - Maps requirements to Ruckus, Extreme, and Ubiquiti products
5. **Proposal Creation** - Generates Google Docs proposals ready for pricing and submission

## Features

- ✅ Automated daily email processing
- ✅ Intelligent PDF parsing with AI
- ✅ Multi-manufacturer product catalog (Ruckus, Extreme, Ubiquiti)
- ✅ Automatic equivalency mapping
- ✅ Google Docs proposal generation
- ✅ Email notifications for completed proposals
- ✅ Manual trigger for individual applications

## Technology Stack

- **n8n** - Workflow automation platform
- **Docker** - Containerization
- **Claude Haiku 4.5** - Latest, fast, cost-effective PDF analysis and requirements extraction
- **Google Docs API** - Proposal document generation
- **IMAP/SMTP** - Email integration

## Prerequisites

- Docker and Docker Compose installed
- Gmail account with App Password (for email monitoring)
- Google Cloud Project with Docs API enabled
- Anthropic API key (for Claude)
- Access to E-rate ProfitWorks emails

## Quick Start

### 1. Clone and Setup

```bash
git clone <repository-url>
cd erate_autobid

# Run setup script to create directories and copy environment template
npm run setup
```

### 2. Configure Environment

Edit `.env` file with your credentials:

```bash
nano .env
```

Required configurations:
- Email credentials (Gmail IMAP)
- Anthropic API key (for Claude Haiku)
- Google OAuth credentials
- n8n encryption key (generate with `openssl rand -hex 32`)

### 3. Start n8n

```bash
# Start n8n in Docker
npm start

# Or manually with docker-compose
docker-compose up -d
```

### 4. Access n8n Interface

Open your browser to: `http://localhost:5678`

Default credentials (change these in .env):
- Username: `admin`
- Password: `changeme`

### 5. Import Workflows

1. Go to n8n web interface
2. Click "Import from File"
3. Import workflows in this order:
   - `workflows/0-main-orchestrator.json`
   - `workflows/1-email-parser.json`
   - `workflows/2-rfp-scraper.json`
   - `workflows/3-pdf-analyzer.json`
   - `workflows/4-bom-generator.json`
   - `workflows/5-proposal-generator.json`

### 6. Configure Credentials

In n8n, set up the following credentials:

#### Gmail IMAP
- Type: IMAP
- Host: `imap.gmail.com`
- Port: `993`
- User: Your Gmail address
- Password: App-specific password (not your regular password)

#### Anthropic (Claude)
- Type: Anthropic API
- API Key: Your Anthropic API key

#### Google Docs OAuth2
- Follow Google Cloud Console setup instructions
- Add authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`
- Download credentials JSON and add to n8n

#### SMTP (for notifications)
- Type: SMTP
- Host: `smtp.gmail.com`
- Port: `587`
- User: Your Gmail address
- Password: App-specific password

### 7. Activate Workflows

Enable all workflows in n8n by toggling the "Active" switch.

## Project Structure

```
erate_autobid/
├── docker-compose.yml          # Docker configuration
├── .env                        # Environment variables (not in git)
├── .env.example               # Environment template
├── package.json               # NPM scripts
├── README.md                  # This file
│
├── workflows/                 # n8n workflow definitions
│   ├── 0-main-orchestrator.json
│   ├── 1-email-parser.json
│   ├── 2-rfp-scraper.json
│   ├── 3-pdf-analyzer.json
│   ├── 4-bom-generator.json
│   └── 5-proposal-generator.json
│
├── data/                      # Product catalogs and configurations
│   └── product-catalog.json   # Ruckus/Extreme/Ubiquiti products
│
├── templates/                 # Proposal templates
│   └── proposal-template.json
│
├── pdfs/                      # Downloaded RFP documents (auto-created)
├── credentials/               # n8n credentials (auto-created, not in git)
├── n8n-data/                 # n8n database (auto-created, not in git)
└── docs/                     # Additional documentation
```

## Workflow Details

### 0. Main Orchestrator
**Trigger:** Daily at 8:30 AM (Monday-Friday) or manual webhook

Coordinates the entire automation pipeline:
1. Checks for new emails
2. Processes each lead sequentially
3. Calls other workflows via webhooks
4. Sends daily summary email

### 1. Email Parser
**Input:** E-rate ProfitWorks daily emails

Extracts:
- Application numbers
- Applicant names
- State and location
- Product requirements
- Links to application pages

### 2. RFP Scraper
**Input:** Application URLs from email parser

Actions:
- Loads application page HTML
- Extracts 470 form PDF link
- Extracts RFP document links (ignores funding history)
- Downloads all PDFs
- Extracts structured data from HTML (equipment, contacts, budget)

### 3. PDF Analyzer
**Input:** Downloaded PDFs

Uses Claude Haiku 4.5 to:
- Extract text from PDFs
- Identify equipment requirements
- Parse quantities and specifications
- Extract budget and deadline information
- Structure requirements for BOM generation

### 4. BOM Generator
**Input:** Extracted requirements

Actions:
- Loads product catalog
- Maps requirements to Ruckus products (primary)
- Generates Extreme alternatives
- Generates Ubiquiti alternatives
- Uses AI to validate BOM and suggest improvements

### 5. Proposal Generator
**Input:** BOM with alternatives

Actions:
- Creates new Google Doc
- Formats proposal with all sections
- Includes pricing tables for all manufacturers
- Adds contact information and terms
- Sends notification email with link

## Usage

### Automatic Mode (Default)

The system runs automatically every weekday at 8:30 AM:
1. Checks email for new E-rate leads
2. Processes all new applications
3. Generates proposals for each
4. Sends summary email when complete

### Manual Mode

To manually process a specific application:

```bash
curl -X POST http://localhost:5678/webhook/manual-trigger \
  -H "Content-Type: application/json" \
  -d '{
    "applicationNumber": "260003459",
    "applicantName": "Example School",
    "applicationUrl": "http://app.erateprofitworks.com/ext/?bn=9308&app=260003459"
  }'
```

Or use the webhook in n8n interface for testing.

### Monitoring

View execution logs in n8n:
1. Go to "Executions" tab
2. Click on any execution to see detailed logs
3. Check each node's output data

## Customization

### Product Catalog

Edit `data/product-catalog.json` to:
- Add new products
- Update pricing ranges
- Modify equivalency mappings
- Add new manufacturers

### Proposal Template

Edit `templates/proposal-template.json` to:
- Change document structure
- Update company information
- Modify boilerplate text
- Adjust formatting

### Email Filters

Modify the email filter in workflow 1 to:
- Change sender address
- Adjust subject line matching
- Add additional filters

### AI Analysis

Adjust AI prompts in workflows 3 and 4 to:
- Extract additional fields
- Change analysis approach
- Modify output format

## Troubleshooting

### Email Not Being Read

- Verify IMAP credentials in n8n
- Check Gmail security settings
- Ensure "Less secure app access" is enabled OR use App Password
- Verify email filter settings in workflow

### PDFs Not Downloading

- Check network connectivity
- Verify PDF URLs are accessible
- Ensure `/data/pdfs` directory exists and has write permissions
- Check Docker volume mounts

### AI Analysis Failing

- Verify Anthropic API key is valid
- Check API quota and billing
- Ensure PDFs contain readable text (not just images)
- Review error logs in n8n execution details

### Google Docs Not Creating

- Verify Google OAuth credentials
- Check API is enabled in Google Cloud Console
- Ensure redirect URI is correctly configured
- Re-authenticate if token expired

### General Debugging

```bash
# View n8n logs
npm run logs

# Or with docker-compose
docker-compose logs -f n8n

# Restart n8n
npm run restart

# Stop all services
npm run stop
```

## Maintenance

### Backup

```bash
# Create backup of workflows, templates, and PDFs
npm run backup

# This creates: backup-YYYYMMDD-HHMMSS.tar.gz
```

### Update n8n

```bash
# Pull latest n8n image
docker-compose pull n8n

# Restart with new version
npm run restart
```

### Clean Up Old PDFs

```bash
# Remove PDFs older than 30 days
find pdfs/ -name "*.pdf" -mtime +30 -delete
```

## Security Considerations

- ✅ All credentials stored in `.env` (not committed to git)
- ✅ n8n protected with basic authentication
- ✅ Google OAuth for secure API access
- ✅ Docker network isolation
- ⚠️ For production: Enable HTTPS with reverse proxy (nginx/Caddy)
- ⚠️ For production: Use stronger n8n authentication
- ⚠️ For production: Implement database backups

## Cost Estimates

### Anthropic API (Claude Haiku 4.5)
- ~$0.001-0.003 per PDF analyzed (much cheaper than GPT-4!)
- Daily cost: $0.001-0.003 × (number of PDFs)
- Estimated: $0.10-1/day depending on volume

### Google Cloud
- Docs API: Free tier covers typical usage
- Additional: ~$0-2/month

### Infrastructure
- Self-hosted (Docker): Free
- Or n8n Cloud: Starts at $20/month

**Total estimated cost: $3-30/month** depending on volume and hosting choice (10x cheaper than OpenAI!)

## Support

For issues or questions:
- Check n8n documentation: https://docs.n8n.io
- Review execution logs in n8n interface
- Check this README's troubleshooting section

## Roadmap

Future enhancements:
- [ ] Database integration for tracking proposals
- [ ] Advanced pricing API integration with distributors
- [ ] Automated bid submission
- [ ] Dashboard for proposal status
- [ ] Mobile notifications
- [ ] Multi-user support with role-based access
- [ ] Historical analytics and reporting
- [ ] Integration with CRM systems

## License

Private - Converged Networks, LLC

## Version History

- **v1.0.0** (2025-10-22) - Initial release
  - Email parsing
  - PDF analysis
  - BOM generation
  - Google Docs proposal creation

---

Built with ❤️ by Converged Networks for winning E-rate bids
