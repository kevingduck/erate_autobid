# E-rate Automation Workflow Diagram

## Complete Process Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     AUTOMATED DAILY PROCESS                          │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│  8:30 AM Daily   │
│  (Mon-Fri)       │
│  Schedule Trigger│
└────────┬─────────┘
         │
         ▼
┌────────────────────────────┐
│ 1. EMAIL PARSER            │
│ • Check Gmail IMAP         │
│ • Find E-rate emails       │
│ • Extract app links        │
│ • Parse requirements       │
└────────┬───────────────────┘
         │
         ▼
┌────────────────────────────┐
│ FOR EACH APPLICATION:      │
└────────┬───────────────────┘
         │
         ▼
┌────────────────────────────┐
│ 2. RFP SCRAPER             │
│ • Load app page            │
│ • Find 470 & RFP PDFs     │
│ • Download all docs        │
│ • Extract HTML data        │
│   - Equipment              │
│   - Budget                 │
│   - Contacts               │
└────────┬───────────────────┘
         │
         ▼
┌────────────────────────────┐
│ 3. PDF ANALYZER            │
│ • Extract text from PDFs   │
│ • AI analysis (Haiku 4.5)  │
│ • Parse requirements       │
│ • Structure data           │
└────────┬───────────────────┘
         │
         │
┌────────┴────────────────────────────────────────────────────────────┐
│                    MANUAL INTERVENTION POINT                         │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────┐
│ 3b. PRICING REQUEST        │
│ • Create Google Sheet      │
│ • Email to Ruckus team     │
│ • Email notification to    │
│   you                      │
│ • PAUSE AUTOMATION         │
└────────┬───────────────────┘
         │
         │ ⏸️  WORKFLOW PAUSES HERE
         │
         │ ✋ YOUR ACTIONS:
         │    1. Wait for Ruckus team email
         │    2. Fill in pricing sheet
         │    3. Review and adjust
         │    4. Call resume webhook
         │
         ▼
┌────────────────────────────┐
│ 3c. RESUME FROM PRICING    │
│ • Read pricing sheet       │
│ • Parse BOM data           │
│ • Validate pricing         │
│ • Format for proposal      │
└────────┬───────────────────┘
         │
         ▼
┌────────────────────────────┐
│ 5. PROPOSAL GENERATOR      │
│ • Create Google Doc        │
│ • Add all sections         │
│ • Format pricing tables    │
│ • Add company info         │
└────────┬───────────────────┘
         │
         ▼
┌────────────────────────────┐
│ NOTIFICATION EMAIL         │
│ • Link to proposal doc     │
│ • Summary of totals        │
│ • Ready for final review   │
└────────────────────────────┘

         │
         ▼
┌────────────────────────────┐
│ ✅ READY TO SUBMIT         │
│ • Review proposal          │
│ • Add any final touches    │
│ • Export and submit bid    │
└────────────────────────────┘
```

## Workflow Details

### Workflow 0: Main Orchestrator
**File:** `0-main-orchestrator.json`

**Purpose:** Coordinates the entire automation pipeline

**Triggers:**
- Schedule: Daily at 8:30 AM (Mon-Fri)
- Manual: POST to `/webhook/manual-trigger`

**Flow:**
1. Check for new emails
2. Process each lead sequentially
3. Pause at pricing step
4. Send summary email

---

### Workflow 1: Email Parser
**File:** `1-email-parser.json`

**Purpose:** Parse E-rate ProfitWorks daily emails

**Input:** IMAP email from `no-reply@erateprofitworks.com`

**Output:**
```json
{
  "applicationNumber": "260003459",
  "applicantName": "School Name",
  "state": "FL",
  "sites": 1,
  "products": [
    {
      "type": "Wireless Access Points",
      "manufacturer": "Ubiquiti",
      "quantity": 18
    }
  ],
  "applicationUrl": "http://app.erateprofitworks.com/..."
}
```

---

### Workflow 2: RFP Scraper
**File:** `2-rfp-scraper.json`

**Purpose:** Download PDFs and extract structured data

**Input:** Application URL

**Actions:**
1. Load ErateProfitWorks application page
2. Extract 470 form PDF link
3. Extract RFP document links (from RFP section only)
4. Download all PDFs
5. Extract equipment table from HTML
6. Extract contact info and budget

**Output:**
```json
{
  "applicationNumber": "260003459",
  "pdfFiles": [
    {
      "fileName": "260003459_470_form.pdf",
      "type": "470"
    },
    {
      "fileName": "260003459_rfp_1.pdf",
      "type": "RFP"
    }
  ],
  "structuredData": {
    "equipment": [...],
    "contacts": {...},
    "budget": "$39,910.86",
    "contractDate": "11/18/2025"
  }
}
```

---

### Workflow 3: PDF Analyzer
**File:** `3-pdf-analyzer.json`

**Purpose:** AI-powered PDF analysis

**Input:** Downloaded PDF files

**Actions:**
1. Read each PDF
2. Extract text
3. Send to Claude Haiku 4.5 with specialized prompt
4. Parse AI response
5. Structure requirements

**Output:**
```json
{
  "requirements": [
    {
      "category": "switches",
      "quantity": 9,
      "specifications": "24-port PoE+ Gigabit",
      "preferredManufacturer": "Other",
      "requiredFeatures": ["PoE+", "Managed"]
    }
  ],
  "budget": "$39,910.86",
  "deadline": "11/18/2025"
}
```

---

### Workflow 3b: Manual Pricing Request
**File:** `3b-manual-pricing-request.json`

**Purpose:** Create pricing request for Ruckus team

**Input:** Extracted requirements

**Actions:**
1. Create Google Sheet with requirements
2. Add columns for Ruckus/Extreme/Ubiquiti pricing
3. Generate email to Ruckus team
4. Send notification to you
5. Pause automation

**Google Sheet Format:**
| Category | Qty | Specifications | Required Features | Ruckus Part # | Ruckus Price | Ruckus Total | ... |
|----------|-----|----------------|-------------------|---------------|--------------|--------------|-----|
| Switches | 9   | 24-port PoE+   | PoE+, Managed     | _(fill in)_   | _(fill in)_  | =(formula)   | ... |

**Emails Sent:**
1. To Ruckus team with requirements and sheet link
2. To you with notification and resume instructions

---

### Workflow 3c: Resume from Pricing
**File:** `3c-resume-from-pricing.json`

**Purpose:** Continue automation after pricing is entered

**Trigger:** POST to `/webhook/resume-from-pricing`

**Input:**
```json
{
  "applicationNumber": "260003459",
  "spreadsheetId": "abc123..."
}
```

**Actions:**
1. Read pricing from Google Sheet
2. Parse all three manufacturer options
3. Calculate totals
4. Validate data
5. Trigger proposal generation

---

### Workflow 4: BOM Generator (Optional)
**File:** `4-bom-generator.json`

**Purpose:** Automated BOM generation (skipped with manual pricing)

**Note:** This workflow is bypassed when using manual pricing (3b/3c).
It's kept for future automation if APIs become available.

---

### Workflow 5: Proposal Generator
**File:** `5-proposal-generator.json`

**Purpose:** Create final Google Doc proposal

**Input:** BOM with pricing

**Actions:**
1. Create new Google Doc
2. Add cover page with applicant info
3. Add executive summary
4. Add requirements summary
5. Add pricing tables:
   - Primary: Ruckus
   - Alternative: Extreme
   - Alternative: Ubiquiti
6. Add totals and budget comparison
7. Add contact info and terms

**Output:**
- Google Doc URL
- Email notification with link
- Summary of totals

---

## Manual Intervention Steps

### When You Receive "Pricing Request" Email:

1. **Open the Google Sheet** linked in the email

2. **Review Requirements** extracted from PDFs

3. **Contact Ruckus Team** (email already sent, or follow up)

4. **Fill in Pricing:**
   - Ruckus part numbers
   - Ruckus unit prices
   - (Optionally) Extreme and Ubiquiti alternatives

5. **Review Totals:**
   - Check against budget
   - Ensure all items are covered
   - Add notes if needed

6. **Resume Automation:**
   ```bash
   curl -X POST http://localhost:5678/webhook/resume-from-pricing \
     -H "Content-Type: application/json" \
     -d '{
       "applicationNumber": "260003459",
       "spreadsheetId": "YOUR_SHEET_ID"
     }'
   ```

   Or use the link/command in the notification email

7. **Wait for Proposal:** You'll receive email when Google Doc is ready

8. **Review Proposal:**
   - Open Google Doc
   - Review all sections
   - Add any final touches
   - Export as PDF if needed

9. **Submit Bid** through E-rate portal

---

## Time Estimates

| Step | Automated Time | Manual Time |
|------|----------------|-------------|
| Email parsing | 10 seconds | - |
| PDF download | 30 seconds | - |
| PDF analysis | 60 seconds | - |
| Pricing request setup | 30 seconds | - |
| **→ Wait for Ruckus** | - | **1-24 hours** |
| **→ Review & fill pricing** | - | **15-30 minutes** |
| Resume automation | 5 seconds | - |
| Proposal generation | 30 seconds | - |
| **→ Final review** | - | **10-15 minutes** |
| **TOTAL** | ~3 minutes automated | ~30-45 minutes manual |

**Overall:** What used to take 2-3 hours is now 30-45 minutes of your time!

---

## Data Flow Diagram

```
┌─────────────┐
│   Email     │
│  (HTML)     │
└──────┬──────┘
       │
       │ Extract: app numbers, URLs, basic product info
       ▼
┌─────────────┐
│ Application │
│   URLs      │
└──────┬──────┘
       │
       │ Scrape: HTML tables, PDF links, contact info
       ▼
┌─────────────┐
│  PDF Files  │
└──────┬──────┘
       │
       │ AI Extract: detailed requirements, specs
       ▼
┌─────────────────┐
│  Requirements   │
│  (Structured)   │
└──────┬──────────┘
       │
       │ Create: Google Sheet with requirements
       ▼
┌───────────────────────┐
│  Google Sheet         │
│  (Empty Pricing)      │
└──────┬────────────────┘
       │
       │ ⏸️  MANUAL: Fill in pricing
       ▼
┌───────────────────────┐
│  Google Sheet         │
│  (With Pricing)       │
└──────┬────────────────┘
       │
       │ Read & Parse: Extract all pricing data
       ▼
┌─────────────┐
│  BOM with   │
│   Pricing   │
└──────┬──────┘
       │
       │ Generate: Format into proposal document
       ▼
┌─────────────┐
│  Google Doc │
│  Proposal   │
└─────────────┘
```

---

## Error Handling

Each workflow includes error handling:

- **Email not found:** Skip and try next day
- **PDF download failed:** Retry 3 times, then notify
- **AI analysis error:** Fall back to basic parsing
- **Pricing sheet empty:** Send error notification
- **Google Docs API error:** Retry with exponential backoff

---

## Monitoring & Notifications

You'll receive emails for:

✅ **Success:**
- Daily summary of processed applications
- Pricing request created (with action items)
- Proposal generated (with link)

⚠️ **Attention Needed:**
- Invalid pricing data when resuming
- Manual review required

❌ **Errors:**
- PDF download failures
- API errors
- Missing credentials

---

## Next Steps After Proposal Generation

1. **Open Google Doc** from notification email
2. **Review all sections** for accuracy
3. **Add/adjust pricing** if needed
4. **Add company-specific details:**
   - Certifications
   - References
   - Timeline details
5. **Format for submission:**
   - File → Download → PDF
   - Or share directly with client
6. **Submit through E-rate portal**

---

## Future Enhancements

Potential automations to add:

- [ ] Automatic bid submission to E-rate portal
- [ ] Price checking against distributor APIs
- [ ] Inventory availability checking
- [ ] Automatic follow-up emails
- [ ] Bid tracking dashboard
- [ ] Win/loss analytics
- [ ] Calendar integration for deadlines
