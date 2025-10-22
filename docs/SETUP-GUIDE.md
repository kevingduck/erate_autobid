# Detailed Setup Guide

This guide walks you through setting up the E-rate RFP automation system step-by-step.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Gmail Setup](#gmail-setup)
3. [Google Cloud Setup](#google-cloud-setup)
4. [Anthropic Setup](#anthropic-setup)
5. [n8n Configuration](#n8n-configuration)
6. [Workflow Import](#workflow-import)
7. [Testing](#testing)

## Prerequisites

### Required Software

Install Docker and Docker Compose:

**macOS:**
```bash
brew install docker docker-compose
```

**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
```

**Windows:**
Download Docker Desktop from https://www.docker.com/products/docker-desktop

### Required Accounts

- Gmail account (for receiving E-rate emails)
- Google Cloud Platform account
- Anthropic account with API access (for Claude)

## Gmail Setup

### 1. Enable IMAP Access

1. Go to Gmail Settings (gear icon → See all settings)
2. Click "Forwarding and POP/IMAP" tab
3. Enable IMAP
4. Click "Save Changes"

### 2. Create App Password

Google requires App Passwords for third-party app access:

1. Go to Google Account settings: https://myaccount.google.com/
2. Click "Security" in left sidebar
3. Under "How you sign in to Google", enable "2-Step Verification" if not already enabled
4. Go back to Security settings
5. Click "App passwords"
6. Select "Mail" and "Other (Custom name)"
7. Name it "n8n E-rate Automation"
8. Click "Generate"
9. **Copy the 16-character password** - you'll need this for `.env`

### 3. Add Email Filter (Optional)

To ensure E-rate emails are easy to find:

1. Search for: `from:no-reply@erateprofitworks.com`
2. Click the dropdown menu (⋮) → "Filter messages like these"
3. Click "Create filter"
4. Select "Apply label" and create label "E-rate/Daily Leads"
5. Click "Create filter"

## Google Cloud Setup

### 1. Create Project

1. Go to Google Cloud Console: https://console.cloud.google.com/
2. Click project dropdown → "New Project"
3. Name: "E-rate Automation"
4. Click "Create"

### 2. Enable Google Docs API

1. Go to "APIs & Services" → "Library"
2. Search for "Google Docs API"
3. Click on it and click "Enable"
4. Also enable "Google Drive API" (needed for creating docs)

### 3. Create OAuth Credentials

1. Go to "APIs & Services" → "Credentials"
2. Click "Create Credentials" → "OAuth client ID"
3. If prompted, configure OAuth consent screen:
   - User Type: External
   - App name: "E-rate Automation"
   - User support email: Your email
   - Developer contact: Your email
   - Click "Save and Continue"
   - Scopes: Add `https://www.googleapis.com/auth/documents` and `https://www.googleapis.com/auth/drive.file`
   - Test users: Add your email
   - Click "Save and Continue"

4. Create OAuth Client ID:
   - Application type: "Web application"
   - Name: "n8n E-rate"
   - Authorized redirect URIs: `http://localhost:5678/rest/oauth2-credential/callback`
   - Click "Create"

5. **Copy Client ID and Client Secret** - you'll need these for n8n

### 4. Download Credentials

1. Click the download icon next to your new OAuth client
2. Save the JSON file
3. Keep it secure - you'll use it in n8n

## Anthropic Setup

### 1. Create Account

1. Go to https://console.anthropic.com/
2. Sign up or log in
3. Add payment method (required for API access)

### 2. Generate API Key

1. Go to https://console.anthropic.com/settings/keys
2. Click "Create Key"
3. Name it "E-rate Automation"
4. **Copy the key immediately** - you won't see it again
5. Store it securely

### 3. Set Usage Limits (Recommended)

1. Go to "Settings" → "Limits"
2. Set a monthly budget (e.g., $30)
3. Set email notifications at 50% and 75%
4. Note: Claude Haiku is much cheaper than GPT-4 (~$0.25 per million input tokens)

## n8n Configuration

### 1. Initial Setup

```bash
# Clone repository
cd ~/
git clone <your-repo-url> erate_autobid
cd erate_autobid

# Run setup
npm run setup
```

### 2. Configure Environment

```bash
# Copy example environment file
cp .env.example .env

# Edit with your preferred editor
nano .env
```

Fill in the following:

```bash
# n8n Configuration
N8N_USER=admin
N8N_PASSWORD=YourSecurePassword123!  # Change this!
N8N_HOST=localhost
N8N_ENCRYPTION_KEY=  # Generate with: openssl rand -hex 32
TIMEZONE=America/Chicago  # Or your timezone

# Email Configuration
EMAIL_USER=kevin@convergednetworks.com
EMAIL_PASSWORD=your_16_char_app_password  # From Gmail App Password
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_IMAP_PORT=993

# Google Drive Configuration
GOOGLE_CLIENT_ID=your_client_id_here
GOOGLE_CLIENT_SECRET=your_client_secret_here
GOOGLE_REDIRECT_URI=http://localhost:5678/rest/oauth2-credential/callback

# Anthropic API (Claude Haiku)
ANTHROPIC_API_KEY=sk-ant-your-key-here
```

Generate encryption key:
```bash
openssl rand -hex 32
```

### 3. Start n8n

```bash
npm start
```

Wait for n8n to start (about 30 seconds), then access it at: http://localhost:5678

### 4. First Login

1. Open http://localhost:5678
2. Login with credentials from `.env`:
   - Username: `admin`
   - Password: Your N8N_PASSWORD

## Workflow Import

### 1. Import Workflows

Import in this specific order:

1. Click "+" → "Import from File"
2. Select `workflows/1-email-parser.json`
3. Click "Import"
4. Repeat for:
   - `workflows/2-rfp-scraper.json`
   - `workflows/3-pdf-analyzer.json`
   - `workflows/4-bom-generator.json`
   - `workflows/5-proposal-generator.json`
   - `workflows/0-main-orchestrator.json` (last)

### 2. Configure Credentials

For each workflow, configure the required credentials:

#### Gmail IMAP Credential

1. Click on "Email Trigger (IMAP)" node
2. Click "Create New Credential"
3. Fill in:
   - Host: `imap.gmail.com`
   - Port: `993`
   - SSL/TLS: Enabled
   - User: Your Gmail address
   - Password: Your App Password (16 characters)
4. Click "Create"

#### Anthropic Credential

1. Click on "AI Analysis - Claude Haiku" node
2. Click "Create New Credential"
3. Select "Anthropic API" (or use HTTP Header Auth)
   - Name: `x-api-key`
   - Value: Your Anthropic API key
4. Click "Create"

#### Google Docs OAuth2 Credential

1. Click on "Create Google Doc" node
2. Click "Create New Credential"
3. Select "Google Docs OAuth2 API"
4. Fill in:
   - Client ID: From Google Cloud
   - Client Secret: From Google Cloud
   - Scopes: (should be pre-filled)
5. Click "Connect my account"
6. Authorize in popup window
7. Click "Create"

#### SMTP Credential (for notifications)

1. Click on "Send Notification Email" node
2. Click "Create New Credential"
3. Fill in:
   - Host: `smtp.gmail.com`
   - Port: `587`
   - SSL/TLS: Enabled (STARTTLS)
   - User: Your Gmail address
   - Password: Your App Password
4. Click "Create"

### 3. Update Webhook URLs

In the main orchestrator workflow:
1. Check that all webhook URLs point to `http://localhost:5678`
2. If running on a different port or host, update accordingly

### 4. Activate Workflows

1. Open each workflow
2. Toggle the "Active" switch in top right
3. Ensure all 6 workflows are active

## Testing

### Test 1: Manual Email Check

1. Open workflow "1. E-rate Email Parser"
2. Click "Execute Workflow"
3. Check if it finds your E-rate emails
4. View output in "Parse Email HTML" node

### Test 2: Manual Application Processing

Use the manual trigger:

```bash
curl -X POST http://localhost:5678/webhook/manual-trigger \
  -H "Content-Type: application/json" \
  -d '{
    "applicationNumber": "260003459",
    "applicantName": "Test School",
    "state": "NJ",
    "applicationUrl": "http://app.erateprofitworks.com/ext/?bn=9308&app=260003459"
  }'
```

Or use n8n's webhook node test feature:
1. Open "0. Main Orchestrator"
2. Click on "Manual Trigger Webhook" node
3. Click "Listen for Test Event"
4. Use curl command or Postman to send test data
5. Watch execution progress

### Test 3: End-to-End Test

Wait for the next scheduled run (8:30 AM weekday) or:

1. Edit the schedule in "0. Main Orchestrator"
2. Change cron expression to run in 5 minutes
3. Wait for execution
4. Check email for summary
5. Check Google Drive for generated proposals

### Verify Success

Successful setup shows:
- ✅ No errors in workflow executions
- ✅ PDFs downloaded to `/pdfs` directory
- ✅ Google Doc created in your Drive
- ✅ Email notification received

## Common Setup Issues

### "Authentication failed" - Gmail

**Solution:**
- Verify App Password is correct (16 characters, no spaces)
- Ensure IMAP is enabled in Gmail settings
- Check 2-Step Verification is enabled

### "Invalid credentials" - Google Docs

**Solution:**
- Verify Client ID and Secret are correct
- Check redirect URI matches exactly: `http://localhost:5678/rest/oauth2-credential/callback`
- Re-authorize the connection in n8n
- Ensure APIs are enabled in Google Cloud Console

### "Insufficient quota" - Anthropic

**Solution:**
- Add payment method to Anthropic account
- Check API usage limits (Claude Haiku is very affordable)
- Verify API key is active

### "Cannot connect to webhook"

**Solution:**
- Ensure n8n is running: `docker ps`
- Check port 5678 is not in use: `lsof -i :5678`
- Verify firewall allows connections
- Check webhook URLs use correct host/port

### Docker Permission Issues

**Linux only:**
```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Logout and login again
```

## Next Steps

After successful setup:

1. **Customize Product Catalog**: Edit `data/product-catalog.json` with your pricing
2. **Adjust Schedule**: Modify cron expression in main orchestrator if needed
3. **Setup Monitoring**: Configure additional email notifications
4. **Backup**: Run `npm run backup` weekly
5. **Review Proposals**: Check first few generated proposals for accuracy

## Getting Help

If you encounter issues:

1. Check n8n execution logs
2. Review Docker logs: `docker-compose logs -f`
3. Verify all credentials are correct
4. Test each workflow individually
5. Check the troubleshooting section in main README.md

## Security Checklist

Before going to production:

- [ ] Changed default n8n password
- [ ] Stored `.env` file securely (not in git)
- [ ] Limited Anthropic API usage/budget
- [ ] Reviewed Google Cloud quotas
- [ ] Set up HTTPS with reverse proxy
- [ ] Configured firewall rules
- [ ] Set up regular backups
- [ ] Tested failure scenarios

Congratulations! Your E-rate automation system is now ready to use! 🎉
