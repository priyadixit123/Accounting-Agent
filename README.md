
# Intelligent Invoice Processing with AI Classification and XML Export

Automated n8n workflow that extracts data from PDF invoices, uses AI for intelligent categorization and anomaly detection, generates accounting-compatible XML, routes high-value invoices for approval, archives results, and sends notifications.

## Overview

This workflow monitors a Google Drive folder (or accepts manual webhook uploads) for PDF invoices, extracts and parses key fields, classifies them with an OpenAI-powered AI agent, converts the structured data to XML, and handles approval routing + notifications.

**Key Features**
- PDF text extraction and structured field parsing
- AI-powered expense categorization, GL code suggestion, confidence scoring, and anomaly detection
- XML export for accounting systems
- Conditional human approval workflow (high-value or low-confidence invoices)
- Multi-channel notifications (Gmail + Slack)
- Archiving to Google Sheets
- Dual triggers: Google Drive folder monitoring + manual webhook upload

## Processing Flow

1. **Detect** new PDF invoices (Google Drive trigger or webhook)
2. **Filter** for PDF files only
3. **Download** and **extract** text from the PDF
4. **Parse** invoice fields (invoice number, date, total, vendor, due date, etc.)
5. **AI Classification** – categorize, suggest GL code, detect anomalies, decide if approval is needed
6. **Convert** structured data to XML
7. **Route**:
   - High-value / low-confidence → approval email
   - Otherwise → Slack notification
8. **Archive** to Google Sheets
9. **Respond** to webhook (if used)

## Prerequisites

- n8n instance (self-hosted or cloud)
- OpenAI API key
- Google account with:
  - Google Drive
  - Google Sheets
  - Gmail
- Slack workspace with a bot

## Required Credentials

Configure the following credentials in n8n:

| Credential              | Purpose                              |
|-------------------------|--------------------------------------|
| Google Drive OAuth2     | Watch folder + download PDFs         |
| OpenAI API              | AI classification (gpt-4o-mini)      |
| Slack Bot Token / OAuth | Channel notifications                |
| Google Sheets OAuth2    | Archive processed invoices           |
| Gmail OAuth2            | Approval request emails              |

## Setup Instructions

1. **Import the workflow**
   - In n8n, go to **Workflows → Import from File** and select the provided JSON.

2. **Configure Google Drive Trigger**
   - Open the **New Invoice Trigger** node.
   - Select the folder to watch (recommended: a dedicated “Invoices/Incoming” folder).
   - Ensure the Google Drive credential is connected.

3. **Configure Google Sheets**
   - Open **Archive to Google Sheets**.
   - Select or create a spreadsheet and sheet for logging.
   - Map the columns as needed (or let n8n auto-map on first run).

4. **Configure Notifications**
   - **Request Approval Email** → set the recipient email address.
   - **Slack Notification** → set the target channel (default example: `#finance-notifications`).

5. **Configure OpenAI**
   - Ensure the **OpenAI Chat Model** node has a valid OpenAI credential.
   - Model defaults to `gpt-4o-mini` (cost-effective). You can change it to `gpt-4o` for higher accuracy.

6. **Webhook (optional)**
   - The **Manual Upload Trigger** is available at:
     ```
     POST /webhook/process-invoice
     ```
   - Useful for testing or integrating with other systems.

7. **Activate the workflow**.

## Node Overview

| Node                        | Type                        | Purpose                                      |
|-----------------------------|-----------------------------|----------------------------------------------|
| New Invoice Trigger         | Google Drive Trigger        | Watches folder for new files                 |
| Manual Upload Trigger       | Webhook                     | Accepts manual/API uploads                   |
| Filter PDF Files            | Filter                      | Keeps only `.pdf` files                      |
| Download Invoice PDF        | Google Drive                | Downloads the file                           |
| Extract PDF Text            | Extract From File           | Extracts text content                        |
| Parse Invoice Data          | Code                        | Regex + structured field extraction          |
| AI Invoice Classifier       | LangChain Agent             | Categorizes, scores, detects anomalies       |
| Parse AI Classification     | Code                        | Merges AI output with original data          |
| Convert to XML              | XML                         | JSON → XML conversion                        |
| Format XML Output           | Set                         | Prepares final XML + metadata                |
| Needs Approval?             | IF                          | Routes high-value / low-confidence invoices  |
| Request Approval Email      | Gmail                       | Sends approval request                       |
| Slack Notification          | Slack                       | Posts status update                          |
| Archive to Google Sheets    | Google Sheets               | Logs the processed invoice                   |
| Respond to Webhook          | Respond to Webhook          | Returns success response                     |

## AI Classification Output

The AI agent returns a strict JSON object:

```json
{
  "category": "Office Supplies | Software | Professional Services | Travel | Equipment | Marketing | Utilities | Other",
  "glCode": "6100",
  "confidence": 0.92,
  "requiresApproval": false,
  "anomalyDetected": false,
  "anomalyReason": null,
  "summary": "Monthly SaaS subscription for project management tool"
}
```

Approval is triggered when:
- `requiresApproval` is `true`, **or**
- Invoice amount > $5,000

## XML Output Example

```xml
<Invoice>
  <invoiceNumber>INV-2024-00123</invoiceNumber>
  <vendorName>Acme Supplies Ltd</vendorName>
  <totalAmount>1249.50</totalAmount>
  <invoiceDate>2024-08-15</invoiceDate>
  <classification>
    <category>Office Supplies</category>
    <glCode>6100</glCode>
    ...
  </classification>
  ...
</Invoice>
```

## Customization Tips

- **Approval threshold**: Change the `$5,000` value in the **Needs Approval?** node and in the AI prompt.
- **Categories / GL codes**: Edit the system prompt and expected JSON schema in the **AI Invoice Classifier** node.
- **Parsing patterns**: Improve the regex patterns in the **Parse Invoice Data** Code node for better field extraction from your specific invoice layouts.
- **Model**: Switch to a more powerful model if classification quality is insufficient.
- **Additional actions**: Add nodes after the merge (e.g., upload XML to SFTP, create accounting entries, etc.).

## Error Handling & Best Practices

- The workflow continues on certain errors (`onError: continueRegularOutput` on the webhook).
- AI response parsing is resilient — falls back to a safe “manual review” classification if JSON parsing fails.
- Keep the Google Drive watched folder clean (move or delete processed files if desired).
- Monitor OpenAI usage and costs, especially with high invoice volumes.
- Test thoroughly with sample invoices of varying formats and amounts.

