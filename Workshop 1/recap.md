# Building and Deploying a RAG AI Agent with Google ADK, Streamlit, and Cloud Run

> Learn how to build a Retrieval-Augmented Generation (RAG) AI assistant using Google's Agent Development Kit (ADK), Gemini 3.5 Flash, Streamlit, and Cloud Run. We'll start with a local JSON knowledge base and then upgrade the application to use Firestore Vector Search for production-ready semantic retrieval.

---

# Overview

Traditional Large Language Models generate responses from their training data, which can lead to hallucinations when asked about domain-specific information. Retrieval-Augmented Generation (RAG) solves this problem by retrieving trusted data before the model generates a response.

In this tutorial, we'll build an **AI Coffee Barista** that recommends drinks and pastries from a predefined menu. Initially, the menu is stored in a local JSON file, making the application simple to understand. Later, we'll migrate the knowledge base to Firestore with Vector Search, allowing semantic retrieval and dynamic menu updates.

---

# What You'll Build

By the end of this guide, you'll have:

- A Google ADK-powered AI agent
- Gemini 3.5 Flash as the underlying LLM
- A Streamlit chat application
- Local RAG using a JSON dataset
- Deployment to Google Cloud Run
- Optional Firestore Vector Search integration for production

---

# Prerequisites

Before starting, make sure you have:

- A Google Cloud Project
- Billing enabled
- Google Cloud CLI installed
- Cloud Shell access
- Python 3

---

# Create a Google Cloud Project

If you don't already have one:

1. Open the Google Cloud Console.
2. Create or select a project.
3. Enable billing.

---

# Start Cloud Shell

Open **Cloud Shell** from the Google Cloud Console.

Verify authentication:

```bash
gcloud auth list
```

Verify the active project:

```bash
gcloud config get project
```

If necessary, switch projects:

```bash
gcloud config set project YOUR_PROJECT_ID
```

---

# Enable Required APIs

Enable the Cloud Run, Vertex AI, and Cloud Build APIs.

```bash
gcloud services enable \
run.googleapis.com \
aiplatform.googleapis.com \
cloudbuild.googleapis.com
```

Verify the APIs:

```bash
gcloud services list --enabled
```

Wait a few minutes until the services appear in the output.

---

# Configure the Project

Export the project ID:

```bash
export PROJECT_ID=$(gcloud config get-value project)
```

Choose your closest region:

```bash
export REGION=asia-south1
```

Create the project folder:

```bash
mkdir coffee-barista-agent
cd coffee-barista-agent
```

---

# Creating the Knowledge Base

A RAG system needs a trusted knowledge source.

For this tutorial, we'll use a simple **menu.json** file containing coffee items, prices, descriptions, tags, and allergens.

Example structure:

```json
{
  "name": "Espresso Solo",
  "description": "A single shot of rich, bold espresso.",
  "price": 2.50,
  "tags": [
    "strong",
    "hot"
  ],
  "allergens": []
}
```

After creating the file, validate it:

```bash
cat menu.json | python3 -m json.tool
```

If everything is correct, you'll see:

```
Valid JSON!
```

---

# Why Use a JSON File?

Using a local JSON file keeps the tutorial simple.

Advantages:

- No database setup
- Easy to understand
- Fast prototyping
- Perfect for demos

However, production systems should use databases like:

- Firestore
- AlloyDB
- Cloud SQL

These allow updating menu items without redeploying the application.

---

# Installing Dependencies

Create a `requirements.txt` file.

```text
google-adk==2.2.0
streamlit==1.56.0
```

---

# Building the ADK Agent

The core of the application is the AI agent.

The agent uses:

- Gemini 3.5 Flash
- Google ADK
- A custom retrieval tool

Instead of embedding the entire menu into the prompt, we expose a Python function called `get_menu()`.

The function simply loads the JSON file and returns its contents whenever the model requests it.

```python
def get_menu():
    ...
```

The tool is then attached to an `LlmAgent`.

The system prompt instructs the agent to:

- Recommend only menu items
- Never hallucinate
- Ask one clarification question when needed
- Respect allergens
- Use only retrieved menu data

This grounding dramatically improves response reliability.

---

# Why Use Tools Instead of Long Prompts?

Suppose your menu grows from 8 items to 800.

Embedding the entire menu into every prompt means:

- More prompt tokens
- Higher cost
- Increased latency

Instead, ADK calls the retrieval tool only when necessary.

Benefits:

- Lower token usage
- Faster responses
- Reduced API costs
- Easier maintenance

---

# Building the Streamlit Frontend

Next, create `app.py`.

The Streamlit application provides:

- Premium coffee-themed UI
- Sidebar menu
- Chat interface
- Conversation history
- ADK integration

The application loads `menu.json` once and displays every item in the sidebar.

When users send messages:

1. Streamlit receives the prompt.
2. The ADK Runner processes it.
3. Gemini calls `get_menu()` if needed.
4. The grounded response is displayed.

---

# Managing Conversation State

Since Streamlit is stateless, conversations are stored in:

```python
st.session_state
```

This stores:

- Session ID
- Chat history
- ADK Runner

However, this state only exists while the browser tab remains open.

Refreshing the page clears everything.

Production applications should instead use:

- Firestore
- Redis
- ADK SessionService

to persist conversations.

---

# Deploying to Cloud Run

Rather than using the default Compute Engine service account, create a dedicated service account.

```bash
gcloud iam service-accounts create barista-agent-sa
```

Grant the required Vertex AI permission.

```bash
gcloud projects add-iam-policy-binding \
$PROJECT_ID \
--member="serviceAccount:barista-agent-sa@$PROJECT_ID.iam.gserviceaccount.com" \
--role="roles/aiplatform.user"
```

Now deploy directly from source.

```bash
gcloud run deploy coffee-barista \
--source .
```

Cloud Run automatically detects:

- Python
- requirements.txt
- Streamlit

and builds the container using Google Buildpacks.

No Dockerfile is required.

---

# Why Deploy From Source?

Cloud Run Buildpacks automatically:

- Detect Python
- Install dependencies
- Build the container
- Configure execution

Advantages:

- Less boilerplate
- Faster deployments
- Easier maintenance

Dockerfiles are still useful when custom system packages or runtime configurations are required.

---

# Why Create a Custom Service Account?

Using the default Compute Engine service account grants excessive permissions.

Instead, we assign only:

```
roles/aiplatform.user
```

This follows the **Principle of Least Privilege**, reducing security risks.

---

# Testing the Agent

Once deployed, open the Cloud Run URL and try a few prompts.

### Coffee Recommendation

> Recommend something strong and warm.

Expected:

- Espresso Solo

---

### Out-of-Menu Request

> Do you have Matcha Frappuccino?

Expected:

- Politely declines.
- Doesn't invent products.

---

### Allergen Check

> I'm lactose intolerant.

Expected:

Only recommends:

- Espresso Solo
- Oat Milk Honey Latte
- Cold Brew Coffee
- Nitro Cold Brew

---

# Moving to Production with Firestore Vector Search

Reading an entire JSON file works well for small demos but doesn't scale.

Instead, migrate your data to Firestore.

Benefits include:

- Dynamic updates
- No redeployment
- Scalable storage
- Semantic retrieval

---

# Enable Firestore

```bash
gcloud services enable firestore.googleapis.com
```

Create a Firestore database.

```bash
gcloud firestore databases create \
--database="coffee-menu" \
--location=$REGION
```

---

# Seed Firestore

Install the required libraries.

```bash
pip3 install \
google-cloud-firestore \
google-genai
```

Create a script that:

- Reads menu.json
- Generates embeddings using `text-embedding-004`
- Stores documents
- Saves embedding vectors

Run:

```bash
python3 seed.py
```

Now every menu item has its own vector embedding.

---

# Create the Vector Index

Create a Firestore vector index.

```bash
gcloud firestore indexes composite create \
...
```

The index enables nearest-neighbor search using cosine similarity.

---

# Grant Firestore Permissions

Allow Cloud Run to access Firestore.

```bash
gcloud projects add-iam-policy-binding \
...
--role="roles/datastore.user"
```

---

# Updating the Retrieval Tool

Replace the JSON-based retrieval function with a Firestore implementation.

The updated workflow becomes:

1. Generate an embedding for the user's query.
2. Perform vector search.
3. Retrieve the three nearest menu items.
4. Remove embeddings.
5. Return the results to Gemini.

This means the model receives only relevant documents instead of the entire menu.

---

# Updating the Sidebar

Instead of reading:

```python
menu.json
```

the sidebar now streams menu items directly from Firestore.

This ensures that any newly added products appear immediately after refreshing the application.

---

# Redeploy the Application

Deploy again:

```bash
gcloud run deploy coffee-barista \
--source .
```

No additional infrastructure changes are required.

---

# Verify Firestore Integration

Add a new menu item directly to Firestore.

For example:

```
Matcha Green Tea Latte
```

Generate its embedding and store it.

Refresh the Streamlit application.

You should notice:

- Matcha Latte appears in the sidebar.
- The chatbot recommends it when asked about matcha drinks.

No code changes or redeployment are necessary.

This demonstrates that the agent is now grounded in a live database.

---

# Cleaning Up

To avoid unnecessary cloud costs, delete the resources created during this tutorial.

Delete Cloud Run:

```bash
gcloud run services delete coffee-barista
```

Delete the service account:

```bash
gcloud iam service-accounts delete \
barista-agent-sa@$PROJECT_ID.iam.gserviceaccount.com
```

Delete Firestore (optional):

```bash
gcloud firestore databases delete \
--database="coffee-menu"
```

Finally, delete the entire Google Cloud project if it was created solely for this lab.

---

# Conclusion

In this tutorial, you built a complete Retrieval-Augmented Generation (RAG) application using Google's Agent Development Kit, Gemini 3.5 Flash, Streamlit, and Cloud Run.

Starting with a local JSON file allowed you to quickly prototype an AI-powered coffee assistant without managing databases. You then upgraded the application to use Firestore Vector Search, enabling semantic retrieval, dynamic menu updates, and production-ready scalability.

This architecture demonstrates how modern AI applications combine retrieval systems with large language models to deliver grounded, accurate, and context-aware responses while minimizing hallucinations and optimizing token usage.

As your datasets grow, replacing local files with vector databases becomes a natural progression, allowing your AI assistants to remain both efficient and reliable in real-world deployments.