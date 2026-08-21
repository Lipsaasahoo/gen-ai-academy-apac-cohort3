# Run a Personal AI Agent on Cloud Run: Coffee Shop Manager Assistant

> Build and deploy a personal AI agent on Google Cloud Run that analyzes historical coffee shop POS data, identifies graduation-weekend demand patterns, recommends staffing and inventory changes, and—after human approval—writes those tasks to a Google Sheets TODO list.

---

## Introduction

AI agents become much more useful when they can work with real business data and take controlled actions.

In this project, we'll build a **Coffee Shop Manager Assistant** using **Gemini, Google ADK, Google Sheets, Cloud Run, and Cloud Run Sandbox**.

The agent can:

- Read historical POS data from Google Sheets.
- Analyze beverage, pastry, staffing, and wait-time patterns.
- Correlate demand spikes with graduation ceremonies.
- Identify potential bottlenecks.
- Recommend staffing and inventory actions.
- Ask the manager for approval before changing data.
- Create or update a `TODO-2026` Google Sheets tab.

---

# Architecture

The overall workflow looks like this:

```text
                    ┌──────────────────────┐
                    │       Manager        │
                    │   Natural Language   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Cloud Run Agent    │
                    │                      │
                    │    Google ADK +      │
                    │       Gemini         │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
       Google Sheets      Cloud Run        Sandbox
        POS-2025 Data      Service        Python/Shell
              │                │                │
              └────────────────┼────────────────┘
                               │
                               ▼
                    Business Analysis
                               │
                               ▼
                    Recommendations
                               │
                               ▼
                    Human Approval
                               │
                               ▼
                    TODO-2026 Sheet
```

The key design principle is **Human-in-the-Loop**: the agent can analyze data and prepare recommendations, but it does not modify the spreadsheet until the manager explicitly approves the proposed tasks.

---

# What You'll Learn

By the end of this tutorial, you'll know how to:

- Configure Google Cloud and Cloud Run.
- Create a dedicated service account.
- Give a Cloud Run service controlled Google Sheets access.
- Build an ADK-based business agent.
- Use Cloud Run Sandbox for analysis.
- Read and update Google Sheets.
- Deploy from source using Cloud Run Buildpacks.
- Design a Human-in-the-Loop workflow.

---

# 1. The Use Case

Imagine a coffee shop near a university.

Graduation weekend can create unusual demand:

- Large groups arrive after ceremonies.
- Cold Brew demand can increase sharply.
- Customers may request more espresso shots.
- Alternative milk consumption can spike.
- Wait times can become a problem.

Historical POS data can reveal these patterns.

Instead of manually studying the spreadsheet, the manager can ask:

> The 2026 graduation schedule is the same as last year. Can you review last year's POS data and help me prepare?

The agent investigates the historical data and converts the findings into actionable tasks.

---

# 2. Setup and Requirements

Set your project:

```bash
export GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
```

Set your Cloud Run region:

```bash
export REGION=YOUR_REGION
```

Configure Google Cloud:

```bash
gcloud config set project $GOOGLE_CLOUD_PROJECT
gcloud config set run/region $REGION
```

Create environment variables for the service account:

```bash
export SA_NAME=coffee-shop-agent-sa
export SERVICE_ACCOUNT_ADDRESS=$SA_NAME@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

Enable the required APIs:

```bash
gcloud services enable --project $GOOGLE_CLOUD_PROJECT \
    run.googleapis.com \
    cloudbuild.googleapis.com \
    artifactregistry.googleapis.com \
    sheets.googleapis.com \
    aiplatform.googleapis.com
```

---

# 3. Create a Dedicated Service Account

Create the service account:

```bash
gcloud iam service-accounts create $SA_NAME \
    --description="Service account for the Coffee Shop Agent Codelab" \
    --display-name="Coffee Shop Agent SA"
```

Grant the Agent Platform User role:

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member="serviceAccount:$SERVICE_ACCOUNT_ADDRESS" \
    --role="roles/aiplatform.user"
```

For local testing with this identity, allow your Google Cloud identity to impersonate the service account:

```bash
gcloud iam service-accounts add-iam-policy-binding \
    $SERVICE_ACCOUNT_ADDRESS \
    --member="user:$(gcloud config get-value account)" \
    --role="roles/iam.serviceAccountTokenCreator"
```

Using a dedicated service account avoids relying on a broad default identity.

---

# 4. Create the Sales Spreadsheet

Create a Google Sheet representing last year's graduation-weekend sales.

Paste this CSV:

```csv
Day,Time,Drip_Coffee,Cold_Brew,Extra_Espresso,Alt_Milk_Oz,Pastries,Cashiers_Working,Wait_Time_Minutes
Saturday,07:00:00,95,15,5,30,80,2,9
Saturday,08:00:00,80,25,10,35,60,2,7
Saturday,09:00:00,30,30,20,40,20,1,4
Saturday,10:00:00,40,130,95,50,25,2,4
Saturday,11:00:00,25,45,25,40,15,1,5
Saturday,12:00:00,30,35,15,45,20,1,3
Saturday,13:00:00,15,20,10,30,10,1,2
Saturday,14:00:00,45,60,30,65,30,2,8
Saturday,15:00:00,20,30,15,35,15,1,3
Saturday,16:00:00,25,35,10,50,20,1,4
Saturday,17:00:00,10,20,5,35,10,1,2
Saturday,18:00:00,20,40,10,190,65,2,12
Saturday,19:00:00,10,15,5,40,15,1,2
Sunday,07:00:00,90,10,5,35,75,2,8
Sunday,08:00:00,75,20,10,40,65,2,6
Sunday,09:00:00,25,25,15,35,15,1,3
Sunday,10:00:00,60,35,15,60,50,2,7
Sunday,11:00:00,20,30,10,35,10,1,3
Sunday,12:00:00,25,40,10,55,20,1,3
Sunday,13:00:00,15,25,5,40,10,1,2
Sunday,14:00:00,30,50,15,180,60,2,11
Sunday,15:00:00,15,25,10,45,15,1,3
Sunday,16:00:00,20,45,20,40,15,1,4
Sunday,17:00:00,10,25,10,30,10,1,2
Sunday,18:00:00,25,145,110,45,20,2,16
Sunday,19:00:00,15,30,15,35,10,1,4
```

In Google Sheets, select the data and choose:

**Data → Split text to columns**

## Share the Sheet

Display the service account email:

```bash
echo $SERVICE_ACCOUNT_ADDRESS
```

In Google Sheets:

1. Click **Share**.
2. Paste the service account email.
3. Give it **Editor** access.
4. Click **Share**.

## Get the Spreadsheet ID

From:

```text
https://docs.google.com/spreadsheets/d/YOUR_SPREADSHEET_ID/edit
```

set:

```bash
export SPREADSHEET_ID=YOUR_SPREADSHEET_ID
```

---

# 5. Build the ADK Agent

Create the application:

```bash
mkdir coffee-mgr-agent
cd coffee-mgr-agent
```

The project contains:

```text
coffee-mgr-agent/
├── main.py
├── requirements.txt
└── Dockerfile
```

## requirements.txt

```text
fastapi>=0.100.0
uvicorn>=0.22.0
google-adk>=1.27.1
google-auth
google-api-python-client
```

## Dockerfile

```dockerfile
FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 8080

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

The application uses FastAPI and Uvicorn while ADK provides the agent functionality. The original codelab's application structure uses these same core dependencies and a Python 3.11 container.

---

# 6. Agent Tools

The application exposes four important tools.

### 1. Sandbox Execution

The `execute_sandbox_command()` function allows the agent to run POSIX shell commands inside the Cloud Run sandbox.

This can be used to create temporary Python scripts and perform calculations on retrieved data.

### 2. Read Google Sheets

```python
read_spreadsheet_values(
    spreadsheet_id,
    range_name
)
```

Reads values from a specified spreadsheet range.

### 3. Update Google Sheets

```python
update_spreadsheet_values(
    spreadsheet_id,
    range_name,
    values
)
```

Writes approved tasks into the spreadsheet.

### 4. Create a Spreadsheet Tab

```python
create_spreadsheet_tab(
    spreadsheet_id,
    tab_name
)
```

Creates `TODO-2026` when the tab does not already exist.

The source implementation uses Google Application Default Credentials with Sheets and Cloud Platform scopes to create the Sheets API client.

---

# 7. Configure the Business Agent

The agent is configured as a business analyst for graduation weekend.

It receives the spreadsheet ID through:

```python
SPREADSHEET_ID = os.environ.get("SPREADSHEET_ID")
```

The agent is instructed to:

1. Read `POS-2025`.
2. Receive the current graduation schedule from the manager.
3. Correlate historical product spikes with ceremonies.
4. Map those patterns to the new schedule.
5. Identify wait-time bottlenecks.
6. Recommend staffing and inventory changes.
7. Ask for approval before modifying the spreadsheet.

The source agent specifically focuses on **Cold Brew, Alt Milk, and Extra Espresso** when diagnosing complex beverage demand.

---

# 8. Bottleneck Diagnostics

The agent follows a simple decision process.

If:

```text
Wait_Time_Minutes > 10
```

the agent examines cashier capacity.

### Fewer Than Two Cashiers

If:

```text
Cashiers_Working < 2
```

the agent recommends another cashier.

### Two Cashiers With Complex-Item Spikes

If:

```text
Cashiers_Working == 2
```

and Cold Brew, Extra Espresso, or Alt Milk demand spikes, the agent identifies the likely constraint as **barista output**, not cashier capacity.

The recommendation is to add a:

> **Support Barista**

---

# 9. Human-in-the-Loop

The agent does not immediately write recommendations into the spreadsheet.

Instead, it must:

1. Present its findings.
2. Explain bottlenecks.
3. Provide actionable staffing and inventory recommendations.
4. Ask for explicit approval.

The required approval question is:

```text
Would you like me to add these tasks to your 'TODO-2026' TODO list?
```

No spreadsheet modification occurs until the manager explicitly approves.

This creates a clear safety boundary:

```text
Analyze
   ↓
Recommend
   ↓
Ask Permission
   ↓
Human Approval
   ↓
Write to Sheet
   ↓
Confirm
```

The source implementation explicitly requires approval before modifying spreadsheet data.

---

# 10. Deploy to Cloud Run

Cloud Run can transform the source application into a production container using Buildpacks.

Deploy:

```bash
gcloud beta run deploy coffee-mgr-agent \
    --source=. \
    --region=$REGION \
    --sandbox-launcher \
    --max-instances=1 \
    --session-affinity \
    --allow-unauthenticated \
    --no-cpu-throttling \
    --labels dev-tutorial=codelab-cloud-run-personal-agent-coffee-shop \
    --set-env-vars GOOGLE_GENAI_USE_VERTEXAI=1,GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=global,SPREADSHEET_ID=$SPREADSHEET_ID \
    --service-account $SERVICE_ACCOUNT_ADDRESS
```

Important deployment options include:

- `--source=.` — deploy from source.
- `--sandbox-launcher` — use the Cloud Run sandbox launcher.
- `--max-instances=1` — limit the service to one instance.
- `--session-affinity` — maintain client affinity.
- `--no-cpu-throttling` — keep CPU available.
- `--service-account` — run with the dedicated service account.

---

# 11. Chat With the Agent

After deployment, Cloud Run provides a service URL similar to:

```text
https://coffee-mgr-agent-YOUR_PROJECT_ID.YOUR_REGION.run.app
```

Open it in a browser.

The application provides a coffee-themed chat interface. The source UI uses a WebSocket connection and renders agent responses as Markdown.

---

# 12. Ask a Real Business Question

Send:

```text
The 2026 graduation schedule was just posted. It's the same schedule as last year. Can you review last year's POS data to help me prepare for this year?

Saturday, June 13:
College of Business (8:30 a.m.)
College of Science and Mathematics (12:30 p.m.)
College of Liberal Arts (4:30 p.m.)

Sunday, June 14:
College of Agriculture (8:30 a.m.)
College of Architecture (12:30 p.m.)
College of Engineering (4:30 p.m.)
```

The agent should compare the new schedule with historical POS patterns.

---

# 13. What the Agent Should Discover

The supplied POS data contains several notable spikes.

## Saturday 10:00 a.m.

```text
Cold Brew:        130
Extra Espresso:    95
Cashiers:           2
Wait Time:          4 minutes
```

This is a major complex-beverage demand spike following the Saturday morning ceremony.

## Saturday 6:00 p.m.

```text
Alt Milk:         190 oz
Cashiers:           2
Wait Time:         12 minutes
```

The wait time exceeds 10 minutes while two cashiers are working, making barista output a likely bottleneck.

## Sunday 2:00 p.m.

```text
Alt Milk:         180 oz
Pastries:          60
Cashiers:           2
Wait Time:         11 minutes
```

This is another significant post-ceremony inventory and service-pressure window.

## Sunday 6:00 p.m.

```text
Cold Brew:        145
Extra Espresso:   110
Cashiers:           2
Wait Time:         16 minutes
```

This is the highest wait-time period in the supplied dataset and the strongest indicator of a barista-capacity bottleneck.

---

# 14. Example Recommendations

The agent can turn the findings into tasks such as:

### Staffing

Schedule a **Support Barista** during the Saturday morning rush.

### Staffing

Schedule a **Support Barista** around the Sunday evening Engineering ceremony rush.

### Inventory

Pre-stage additional alternative milk before the afternoon post-ceremony periods.

The important point is that recommendations are based on the historical POS data and the manager's supplied schedule.

---

# 15. Approve the Changes

After presenting the recommendations, the agent asks:

```text
Would you like me to add these tasks to your 'TODO-2026' TODO list?
```

Reply:

```text
Yes
```

The agent then:

1. Checks for `TODO-2026`.
2. Creates it if necessary.
3. Appends the approved tasks.
4. Confirms what was written.

---

# 16. TODO-2026

The sheet contains fields such as:

| Task | Category | Ceremony | Date_Added |
|---|---|---|---|
| Schedule a Support Barista role for Saturday morning rush | Staffing | College of Business | Date added |
| Schedule a Support Barista role for Sunday evening rush | Staffing | College of Engineering | Date added |
| Pre-stage extra Alt Milk cartons | Inventory | College of Science/Math | Date added |

This turns an AI-generated analysis into a concrete operational checklist.

---

# Why This Architecture Is Useful

This project demonstrates a practical pattern for AI business assistants:

```text
Business Data
     ↓
AI Analysis
     ↓
Operational Insight
     ↓
Recommended Action
     ↓
Human Approval
     ↓
Automated Update
```

The agent is not simply a chatbot. It can inspect data, reason about patterns, create recommendations, and interact with business systems while retaining a human approval checkpoint.

---

# 17. Clean Up

To avoid unnecessary charges, remove the resources when finished.

## Delete the Cloud Run Service

```bash
gcloud run services delete coffee-mgr-agent \
    --region $REGION
```

## Delete the Service Account

```bash
gcloud iam service-accounts delete \
    $SERVICE_ACCOUNT_ADDRESS
```

## Delete the Project

If the project was created specifically for this tutorial:

```bash
gcloud projects delete ${GOOGLE_CLOUD_PROJECT}
```

> **Warning:** Only delete the entire project if you are certain it does not contain resources you need.

---

# Conclusion

In this project, we built a **personal AI Coffee Shop Manager Assistant** and deployed it on Cloud Run.

The agent combines:

- **Gemini** for reasoning.
- **Google ADK** for agent development.
- **Google Sheets API** for business data.
- **Cloud Run** for deployment.
- **Cloud Run Sandbox** for controlled code execution.
- **Human-in-the-Loop approval** for safe spreadsheet updates.

Instead of simply answering questions, the agent can analyze historical POS data, identify demand patterns around graduation ceremonies, diagnose staffing bottlenecks, recommend inventory changes, and update an operational TODO list after receiving explicit approval.

This pattern can be extended beyond coffee shops to many business workflows where AI needs access to data and the ability to prepare actions without removing human control.

---

## Key Takeaways

- Cloud Run can host a personal AI agent.
- Dedicated service accounts provide controlled application identity.
- Google Sheets can act as a lightweight operational data source.
- ADK tools allow an agent to interact with external systems.
- Cloud Run Sandbox can support controlled analysis workflows.
- Historical POS data can be converted into actionable business recommendations.
- Human approval provides an important safety boundary before data modification.
- Cloud Run provides a managed deployment environment for the complete application.

---

## References

- Google Cloud Run
- Cloud Run Sandbox
- Google Agent Development Kit
- Google Sheets API
- Gemini / Vertex AI