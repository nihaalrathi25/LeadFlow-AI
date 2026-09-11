# LeadFlow AI — Workflow Documentation

## 1. Workflow Overview

LeadFlow AI is an automated AI-powered lead qualification pipeline built using **n8n and Ollama**.

The workflow takes raw lead data, validates and cleans it, removes duplicates, uses an LLM to evaluate each lead, and routes leads based on their qualification score.

```text
CSV Input
   ↓
Extract CSV
   ↓
Validate Lead Data
   ↓
Deduplicate Leads
   ↓
Loop Through Leads
   ↓
Wait / Rate Control
   ↓
Ollama AI Scoring
   ↓
Parse AI Response
   ↓
Normalize Score
   ↓
Lead Classification
   ↓
 ┌────────────┬────────────┬────────────┐
 │            │            │            │
 HIGH        MEDIUM        LOW
 │            │            │
 ↓            ↓            ↓
Google       Google       Low Priority
Sheets       Sheets       Handling
 │
 ↓
AI Outreach Generation
```

---

# 2. Trigger

### Manual Trigger

The workflow starts with an **n8n Manual Trigger**.

This allows the workflow to be tested manually during development.

For production, this can be replaced with:

* Schedule Trigger
* Webhook
* CRM trigger
* Form submission
* Database trigger

---

# 3. CSV Input

The workflow reads the sample lead dataset from:

```text
/home/node/.n8n-files/sample_leads.csv
```

The CSV contains lead information such as:

```text
name
email
company
website
industry
country
employees
```

Example:

```text
Arjun Shah
arjun@dataflow.example
DataFlow
https://dataflow.example
Data Analytics
India
85
```

The CSV is then converted into individual n8n items for processing.

---

# 4. Lead Validation

Before sending leads to the AI model, the workflow validates the input data.

### Required Fields

The following fields are required:

```text
name
email
company
```

### Email Validation

A basic email pattern is used to identify incorrectly formatted email addresses.

### Website Validation

The website must begin with:

```text
http://
```

or

```text
https://
```

### Employee Validation

If the employee count is provided, it must contain a numeric value.

Invalid records are removed before further processing.

---

# 5. Lead Deduplication

Duplicate leads are removed to prevent the same company or prospect from being processed multiple times.

The workflow checks:

```text
Email
Company Name
```

If a duplicate email or company is detected, the duplicate record is excluded.

This reduces:

* Duplicate AI processing
* Unnecessary API/model calls
* Duplicate entries in Google Sheets
* Repeated outreach

---

# 6. Lead Processing Loop

After validation and deduplication, the workflow processes leads individually using an **n8n Loop Over Items** node.

Each lead is processed through the AI qualification pipeline.

A short delay is also included between processing operations to control the request rate.

---

# 7. AI Lead Scoring

The workflow sends each lead to **Ollama** using an HTTP request.

```text
n8n
 ↓
HTTP Request
 ↓
Ollama
 ↓
Llama 3.2:3b
```

The model evaluates the lead using five criteria.

| Criteria             | Maximum |
| -------------------- | ------: |
| Industry Fit         |      20 |
| Company Fit          |      15 |
| Automation Potential |      25 |
| Pain Point Strength  |      20 |
| Buying Intent        |      20 |
| **Total**            | **100** |

---

# 8. AI Scoring Rules

The model is instructed to use only information explicitly available in the lead data.

It should **not invent**:

* Pain points
* Buying signals
* Company information
* Industry information
* Business requirements

This makes the qualification process more controlled and reduces unsupported AI assumptions.

---

# 9. Structured AI Response

Ollama is configured to return a structured JSON response.

A typical response contains information such as:

```json
{
  "industry_fit": 18,
  "company_fit": 13,
  "automation_potential": 22,
  "pain_point_strength": 16,
  "buying_intent": 15,
  "total_score": 84,
  "qualification": "HIGH"
}
```

The response is then parsed inside n8n.

---

# 10. Score Normalization

The AI response is passed through a normalization step.

This ensures that the score is treated consistently as a numeric value and that the individual scoring fields can be used by later workflow nodes.

The maximum possible score is:

```text
20 + 15 + 25 + 20 + 20 = 100
```

---

# 11. Lead Classification

An IF/conditional routing system categorizes the lead.

### HIGH Priority

```text
Score >= 80
```

The lead is classified as:

```text
HIGH_PRIORITY
```

### MEDIUM Priority

```text
60 <= Score < 80
```

The lead is classified as:

```text
MEDIUM_QUALIFIED
```

### LOW Priority

```text
Score < 60
```

The lead is classified as:

```text
LOW
```

---

# 12. HIGH Priority Flow

High-scoring leads are considered the most valuable prospects.

The workflow:

```text
HIGH
 ↓
Set Lead Status
 ↓
Add Processing Timestamp
 ↓
Google Sheets
 ↓
Generate Outreach
```

The lead is stored in Google Sheets along with its qualification information.

The workflow then sends the lead through an additional AI step for outreach generation.

---

# 13. MEDIUM Priority Flow

Medium-qualified leads are also stored in Google Sheets.

The workflow assigns:

```text
MEDIUM_QUALIFIED
```

and records the processing timestamp.

These leads can later be reviewed or moved into a separate nurturing workflow.

---

# 14. LOW Priority Flow

Low-scoring leads are separated from the higher-priority prospects.

These leads can be used for:

* Future review
* Nurturing campaigns
* Additional qualification
* Filtering
* Separate storage

This prevents low-quality prospects from receiving the same immediate attention as high-priority leads.

---

# 15. Google Sheets Integration

Google Sheets acts as the output destination for qualified leads.

The workflow can store information such as:

```text
Name
Email
Company
Website
Industry
Country
Employees
Score
Qualification
Lead Status
Processed At
```

This creates a simple lead qualification database that can be accessed by a sales or business-development team.

---

# 16. AI Outreach Generation

High-priority leads can be passed to another Ollama request.

The purpose of this step is to generate an initial outreach message that can be used by a sales workflow.

The generated message is then combined with lead information before being prepared for outreach.

This component can later be connected to:

* Gmail
* Outlook
* SMTP
* CRM
* LinkedIn automation
* Slack
* Other messaging systems

---

# 17. Error Handling

The workflow performs validation before expensive AI processing.

```text
Raw Data
   ↓
Validation
   ↓
Only Valid Leads
   ↓
AI Processing
```

This prevents obviously invalid records from reaching the LLM.

Additional error-handling improvements can include:

* Retry failed Ollama requests
* Log failed leads
* Handle malformed JSON
* Add timeout handling
* Send failed records to a separate error sheet

---

# 18. Data Flow Summary

The complete data flow is:

```text
Lead CSV
   ↓
CSV Extraction
   ↓
Validation
   ↓
Deduplication
   ↓
Loop Over Leads
   ↓
Ollama / Llama 3.2
   ↓
AI Qualification
   ↓
Score Normalization
   ↓
Conditional Routing
   ↓
 ┌─────────┬──────────┬─────────┐
 HIGH     MEDIUM       LOW
  ↓          ↓           ↓
Sheets     Sheets      Review
  ↓
Outreach Generation
```

---

# 19. Why This Architecture?

The workflow separates the process into independent stages:

### Data Processing

Validation and deduplication ensure that the AI receives cleaner input.

### AI Layer

Ollama performs the reasoning and lead qualification.

### Business Logic

n8n handles score thresholds and routing.

### Storage Layer

Google Sheets provides an accessible destination for qualified leads.

### Outreach Layer

AI-generated messages can be used as the starting point for automated sales communication.

This separation makes the workflow easier to modify and extend.

---

# 20. Future Improvements

Potential improvements include:

### Lead Enrichment

Automatically retrieve additional company information from websites or APIs.

### CRM Integration

Send qualified leads directly to:

```text
HubSpot
Salesforce
Pipedrive
```

### Automated Email

Automatically send personalized emails to HIGH-priority leads.

### Follow-Up Automation

Create scheduled follow-ups based on lead status.

### Better AI Models

Replace Llama 3.2:3b with a larger local or cloud model when more reasoning accuracy is required.

### Analytics

Add dashboards showing:

* Total leads processed
* Validation rate
* Duplicate rate
* HIGH/MEDIUM/LOW distribution
* Average lead score
* Outreach volume

---

# 21. End-to-End Result

LeadFlow AI transforms a raw lead list into an automatically qualified sales pipeline.

```text
Raw Leads
    ↓
Clean Data
    ↓
Remove Duplicates
    ↓
AI Qualification
    ↓
Score 0–100
    ↓
Prioritize Leads
    ↓
Store Qualified Leads
    ↓
Prepare AI Outreach
```

The workflow demonstrates how **AI + workflow automation + business rules** can reduce repetitive manual lead qualification and create a scalable foundation for automated sales operations.
