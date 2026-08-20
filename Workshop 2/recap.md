# Build and Deploy AI Agents with Gemini and BigQuery MCP Server on Cloud Run

> Build an AI data agent using Google's Agent Development Kit (ADK), Gemini in Agent Platform, the BigQuery MCP server, and Cloud Run. The agent can inspect and analyze structured NYC Citi Bike data in BigQuery using natural-language questions.

## Introduction

AI agents become much more useful when they can access real, up-to-date data instead of relying only on information contained in the language model.

In this project, we'll build a **data analysis agent** using Google's **Agent Development Kit (ADK)** and **Gemini**. The agent connects to structured data in **BigQuery** through the **BigQuery Model Context Protocol (MCP) server**.

The completed application runs on **Cloud Run**, Google's fully managed serverless platform for containerized applications.

### Technologies

- **Gemini** — reasoning and language model
- **Google ADK** — agent development framework
- **BigQuery MCP Server** — bridge between the agent and BigQuery
- **Cloud Run** — serverless deployment platform

---

## What You'll Learn

By the end of this tutorial, you'll know how to:

- Create an AI agent using Google ADK.
- Use Gemini in Agent Platform.
- Connect an ADK agent to BigQuery through MCP.
- Give an agent read-only access to structured data.
- Test the agent locally with the ADK Web interface.
- Deploy the agent to Cloud Run.
- Ask natural-language questions about BigQuery datasets.

---

# Understanding the Technologies

## Cloud Run

**Cloud Run** is a fully managed serverless compute platform that allows you to run containerized applications without managing the underlying infrastructure.

For this project, Cloud Run provides the production environment where our AI agent runs.

## Agent Development Kit

**Agent Development Kit (ADK)** is an open-source framework for building, debugging, and deploying AI agents.

ADK provides components such as:

- `LlmAgent`
- Tools
- MCP toolsets
- Agent runners
- Web development interfaces

Our agent uses `LlmAgent` together with the BigQuery MCP toolset.

## BigQuery

**BigQuery** is Google's fully managed, serverless enterprise data warehouse.

In this tutorial, the agent works with:

```text
bigquery-public-data.new_york_citibike
```

This dataset contains Citi Bike trips and station information from the NYC area.

## Model Context Protocol

**Model Context Protocol (MCP)** provides a standardized way for AI models and agents to connect to external data sources and tools.

The **BigQuery MCP Server** provides tools that allow an AI agent to inspect and query BigQuery data securely.

---

# Architecture

```text
User
  │
  ▼
ADK Web Interface
  │
  ▼
Google ADK LlmAgent
  │
  ▼
BigQuery MCP Toolset
  │
  ▼
BigQuery MCP Server
  │
  ▼
BigQuery Dataset
  │
  ▼
NYC Citi Bike Data
```

The agent receives a natural-language question, investigates the available BigQuery data, creates an appropriate SQL query, executes it through the MCP server, and uses the results to formulate a response.

---

# 1. Setup and Requirements

Start by setting the Google Cloud project:

```bash
gcloud config set project YOUR_PROJECT_ID
```

Configure the Cloud Run region:

```bash
gcloud config set run/region CLOUD-RUN-REGION
```

## Configure Environment Variables

```bash
export GOOGLE_CLOUD_PROJECT="${GOOGLE_CLOUD_PROJECT:-$(gcloud config get-value project -q)}"

export GOOGLE_CLOUD_REGION="${GOOGLE_CLOUD_REGION:-$(CR_REGION=$(gcloud config get-value run/region -q 2>/dev/null); echo "${CR_REGION:-us-central1}")}"

export GOOGLE_GENAI_USE_ENTERPRISE="True"

export GOOGLE_CLOUD_LOCATION="global"
```

You can save these variables in `env.sh` and run:

```bash
source env.sh
```

> **Security note:** Do not commit `env.sh` to version control.

## Enable Required APIs

```bash
gcloud services enable --project "${GOOGLE_CLOUD_PROJECT}"     run.googleapis.com     cloudbuild.googleapis.com     artifactregistry.googleapis.com     bigquery.googleapis.com     aiplatform.googleapis.com
```

---

# 2. Create the Data Agent

Create the application directory:

```bash
mkdir data_agent
```

The project will contain:

```text
data_agent/
    agent.py
    __init__.py
    requirements.txt
```

## Connect to the BigQuery MCP Server

The agent uses Google Application Default Credentials (ADC) for authentication.

```python
import os

from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool.mcp_toolset import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StreamableHTTPConnectionParams

import google.auth
from google.auth.transport.requests import Request

_application_default_credentials, project_id = google.auth.default()
_request = Request()
_application_default_credentials.refresh(_request)

project_id = os.getenv("GOOGLE_CLOUD_PROJECT", project_id)

if not project_id:
    raise ValueError("GOOGLE_CLOUD_PROJECT environment variable is not set.")
```

## Create the Authentication Provider

```python
def _adc_auth_header_provider(context=None) -> dict[str, str]:
    if not _application_default_credentials.valid:
        _application_default_credentials.refresh(_request)

    return {
        "Authorization": f"Bearer {_application_default_credentials.token}",
        "x-goog-user-project": project_id
    }
```

This supplies the Google Cloud access token to the MCP server.

## Configure the BigQuery MCP Toolset

```python
bigquery_toolset = McpToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="https://bigquery.googleapis.com/mcp",
        tool_filter=[
            "get_dataset_info",
            "list_table_ids",
            "get_table_info",
            "execute_sql_readonly",
        ]
    ),
    header_provider=_adc_auth_header_provider
)
```

The enabled tools are:

| Tool | Purpose |
|---|---|
| `get_dataset_info` | Inspect dataset information |
| `list_table_ids` | Discover available tables |
| `get_table_info` | Inspect table schemas |
| `execute_sql_readonly` | Execute read-only SQL |

Using `execute_sql_readonly` provides a security boundary that prevents accidental data modification.

---

# Configure the Agent

The agent is instructed to investigate the dataset before answering questions.

Its workflow is:

1. Analyze the dataset.
2. Inspect tables and schemas.
3. Investigate dimensions and values.
4. Understand the user's question.
5. Formulate a query plan.
6. Generate SQL.
7. Verify SQL correctness with a dry run.
8. Execute the query using `execute_sql_readonly`.
9. Return the answer in Markdown.

A key instruction is:

> **Do not make assumptions about the data. Always verify assumptions using the available tools.**

The agent works with:

```text
bigquery-public-data.new_york_citibike
```

## Create the LLM Agent

```python
root_agent = LlmAgent(
    model="gemini-3.6-flash",
    name="data_agent",
    instruction=system_instruction,
    description="A helpful assistant that can answer questions using NYC Citibike data.",
    tools=[bigquery_toolset]
)
```

---

# Required Deployment Files

Create `__init__.py`:

```bash
echo "from . import agent" > data_agent/__init__.py
```

Create `requirements.txt`:

```bash
echo -e "google-adk==2.4.*\nmcp==1.29.*" > data_agent/requirements.txt
```

Final structure:

```text
data_agent/
├── __init__.py
├── agent.py
└── requirements.txt
```

---

# 3. Test the Agent Locally

ADK provides a Web interface for interactive development and debugging.

From the directory containing `data_agent`, run:

```bash
uv tool run --with "mcp==1.29.*" --from "google-adk[mcp]==2.4.*" adk web --allow_origins="*" --port 8080 .
```

Open:

```text
http://localhost:8080/
```

In Cloud Shell, use **Web Preview → Preview on port 8080**.

Ask:

```text
What data do you have?
```

The agent should use the BigQuery MCP tools to explore the Citi Bike dataset and provide an overview of the available tables and fields.

---

# 4. Deploy the Agent to Cloud Run

Deploy the agent using the ADK CLI:

```bash
uv tool run --from google-adk==2.4.0     adk deploy cloud_run     --with_ui     --project $GOOGLE_CLOUD_PROJECT     --region $GOOGLE_CLOUD_REGION     --service_name bq-data-agent     --app_name data_agent     data_agent     --     --allow-unauthenticated     --max-instances 1     --labels dev-tutorial=codelab-cloud-run-adk-gemini-bq-mcp     --set-env-vars GOOGLE_GENAI_USE_ENTERPRISE=True,GOOGLE_CLOUD_PROJECT=${GOOGLE_CLOUD_PROJECT},GOOGLE_CLOUD_LOCATION=${GOOGLE_CLOUD_LOCATION}
```

The `--with_ui` option deploys the ADK Web interface with the agent.

## Get the Cloud Run URL

```bash
gcloud run services describe bq-data-agent     --project $GOOGLE_CLOUD_PROJECT     --region $GOOGLE_CLOUD_REGION     --format 'value(status.url)'
```

Open the returned URL in a browser.

---

# 5. Test the Deployed Agent

Try a real-world data analysis question:

```text
We have budget for 3 coffee trucks.
We want to find the best city bike stations to place our coffee trucks.
```

The agent should:

1. Explore the Citi Bike dataset.
2. Inspect available tables.
3. Understand the schema.
4. Determine which fields are useful.
5. Formulate a query strategy.
6. Generate BigQuery SQL.
7. Execute the SQL through the BigQuery MCP server.
8. Return three recommended stations.

This demonstrates how an AI agent can combine reasoning with real structured data.

---

# Understanding the Agent Workflow

```text
User Question
      │
      ▼
   Gemini
      │
      ▼
 Google ADK
      │
      ▼
 BigQuery MCP
      │
      ├── Discover tables
      ├── Inspect schemas
      ├── Analyze dimensions
      └── Execute read-only SQL
      │
      ▼
 BigQuery
      │
      ▼
 Citi Bike Dataset
      │
      ▼
 Query Results
      │
      ▼
 Gemini
      │
      ▼
 Final Answer
```

The important part of this workflow is that the model does not need to assume the structure of the dataset. It can inspect the available data and construct its query based on what it discovers.

---

# Why MCP Matters

Without MCP, an application would typically need custom integration code for every external data source.

MCP provides a standardized interface between the AI agent and external tools.

In this project:

```text
Gemini
   │
   ▼
ADK
   │
   ▼
MCP Toolset
   │
   ▼
BigQuery MCP Server
   │
   ▼
BigQuery
```

This makes it easier to build agents that can work with external services through standardized tools.

---

# Security Considerations

The agent uses Application Default Credentials to authenticate with Google Cloud.

The configured BigQuery toolset specifically exposes:

```text
execute_sql_readonly
```

instead of a general-purpose SQL execution tool.

This limits the agent to read-only data access and reduces the risk of accidental modifications.

The agent is also instructed to inspect schemas and values before querying the data rather than relying on assumptions.

---

# 6. Clean Up

To avoid unnecessary cloud charges, remove the resources when finished.

## Delete the Cloud Run Service

```bash
gcloud run services delete bq-data-agent     --project "${GOOGLE_CLOUD_PROJECT}"     --region "${GOOGLE_CLOUD_REGION}"     --quiet
```

## Delete the Entire Project

If the project was created specifically for this lab:

```bash
gcloud projects delete ${GOOGLE_CLOUD_PROJECT}
```

> **Warning:** Only delete the entire project if you are certain it does not contain resources you need.

---

# Conclusion

In this tutorial, we built and deployed an AI data agent using **Gemini**, **Google ADK**, **BigQuery MCP Server**, and **Cloud Run**.

The agent can interact with real structured data instead of relying only on the model's internal knowledge. Through MCP, BigQuery becomes an accessible tool that the agent can use to inspect schemas, investigate data, generate SQL, and retrieve results.

The final architecture combines:

- **Gemini** for reasoning
- **ADK** for agent development
- **MCP** for standardized tool integration
- **BigQuery** for structured data analysis
- **Cloud Run** for serverless deployment

This pattern is useful for building AI-powered data assistants that need secure, up-to-date access to enterprise datasets.

---

## Key Takeaways

- AI agents become more useful when they can access real data.
- Google ADK provides the framework for building the agent.
- MCP provides a standardized connection to external tools.
- BigQuery MCP allows agents to query structured BigQuery data.
- Read-only tools provide an important safety boundary.
- Cloud Run provides a managed serverless deployment environment.
- Schema discovery helps the agent avoid unsupported assumptions.
