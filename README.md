# Building with Agentic AI — Article 5: RAG Systems in Practice

This repository contains the n8n workflow JSON files for Article 5 of the **Building with Agentic AI** series: *RAG Systems in Practice: From Simple Retrieval to Agentic and Graph RAG*.

The series is published on [Medium](https://medium.com/@sarasoleymani) and [LinkedIn](#).

---

## What's in this repo

| File | Description |
|------|-------------|
| `Ingestion Pipeline.json` | Workflow 1.1 — Ingestion pipeline. Watches a Google Drive folder and automatically chunks, embeds, and indexes new documents into Pinecone. |
| `RAG- Query pipeline.json` | Workflow 1.2 — Simple RAG query pipeline. Embeds user queries and retrieves grounded answers from the Pinecone knowledge base. |
| `Agentic RAG_ Competitive Intelligence Assistant.json` | Workflow 2 — Agentic RAG. Rewrites user queries before retrieval to improve semantic match, then synthesizes grounded answers. |
| `Graph Rag - Account relationship mapping.jsonn` | Workflow 3 — Graph RAG. Extracts entities from queries, traverses a Neo4j knowledge graph, and synthesizes relational insights. |
| `knowledge-base/` | Synthetic sales knowledge base used in the demo (plain .txt files). |

---

## The three RAG tiers

| Tier | How it works | Best for | n8n build |
|------|-------------|----------|-----------|
| Simple RAG | User query embedded directly, matched against vector store | FAQs, policy lookups, direct queries | Workflows 1.1 + 1.2 |
| Agentic RAG | Orchestrator rewrites query before retrieval | Conversational queries, competitive intelligence | Workflow 2 |
| Graph RAG | Entities extracted and traversed through a knowledge graph | Account relationships, network connections | Workflow 3 |

---

## Prerequisites

Before running these workflows you will need:

- **n8n** — cloud or self-hosted (these workflows were built on n8n cloud)
- **OpenAI API key** — used for embeddings (text-embedding-3-small) and LLM calls (GPT-4o)
- **Pinecone account** — vector store for Simple RAG and Agentic RAG
- **Google Drive** — source folder for document ingestion
- **Neo4j AuraDB** — graph database for Graph RAG (AuraDB Professional required for HTTP API access)

---

## Setup

### 1. Pinecone index

Create a new index in your Pinecone account with the following settings:

- Index name: `sales-knowledge-base`
- Dimensions: `512`
- Metric: `cosine`
- Type: Serverless
- Cloud/Region: AWS us-east-1

### 2. Google Drive folder

Create a folder in Google Drive named `sales-knowledge-base`. This is the folder the ingestion workflow watches for new documents.

Upload the three synthetic knowledge base files from the `knowledge-base/` folder in this repo:

- `Enterprise_Security_Compliance.txt`
- `Competitive_Battle_Cards.txt`
- `Pricing_Discount_Policy.txt`

> **Important**: upload files one at a time and wait for each ingestion workflow execution to complete before uploading the next. The Google Drive Trigger fires per file — bulk uploads can cause some files to be missed.

### 3. Neo4j AuraDB (Graph RAG only)

Create an AuraDB instance and run the following Cypher script to populate the knowledge graph:

```cypher
// Create Companies
CREATE (acme:Company {name: "Acme", type: "vendor"})
CREATE (databricks:Company {name: "Databricks", type: "prospect"})
CREATE (snowflake:Company {name: "Snowflake", type: "other"})
CREATE (acmecustomer:Company {name: "TechCorp", type: "customer"})

// Create Contacts
CREATE (jane:Contact {name: "Jane Smith", title: "CTO", email: "jane@techcorp.com"})
CREATE (mike:Contact {name: "Mike Johnson", title: "VP of Sales", email: "mike@databricks.com"})
CREATE (sara:Contact {name: "Sara Lee", title: "Account Executive", email: "sara@acme.com"})
CREATE (john:Contact {name: "John Park", title: "VP of Engineering", email: "john@databricks.com"})

// Create Deals
CREATE (deal1:Deal {name: "Databricks Enterprise Deal", stage: "Negotiation", value: 250000})

// Create Interactions
CREATE (int1:Interaction {type: "Demo", date: "2025-01-15", notes: "Showed enterprise features"})
CREATE (int2:Interaction {type: "Follow-up call", date: "2025-02-01", notes: "Discussed pricing"})

// Create Relationships
CREATE (jane)-[:WORKS_AT]->(acmecustomer)
CREATE (mike)-[:WORKS_AT]->(databricks)
CREATE (john)-[:WORKS_AT]->(databricks)
CREATE (sara)-[:WORKS_AT]->(acme)
CREATE (jane)-[:PREVIOUSLY_WORKED_AT]->(snowflake)
CREATE (mike)-[:PREVIOUSLY_WORKED_AT]->(snowflake)
CREATE (jane)-[:KNOWS]->(mike)
CREATE (databricks)-[:IS_PROSPECT]->(deal1)
CREATE (sara)-[:PARTICIPATED_IN]->(deal1)
CREATE (sara)-[:HAD_INTERACTION]->(int1)
CREATE (mike)-[:HAD_INTERACTION]->(int1)
CREATE (sara)-[:HAD_INTERACTION]->(int2)
```

This creates the key relationship the Graph RAG workflow surfaces: Jane Smith (CTO at TechCorp) and Mike Johnson (VP of Sales at Databricks) both previously worked at Snowflake — a hidden connection that vector search cannot find but graph traversal surfaces instantly.

---

## Workflow details

### Workflow 1.1— Ingestion pipeline

**Trigger**: Google Drive folder watch (fires on file create or update)

**Flow**: Google Drive Trigger → Google Drive Download → Pinecone Vector Store

**Key configuration**:
- Default Data Loader: Binary
- Recursive Character Text Splitter: Chunk Size 512, Chunk Overlap 50
- OpenAI Embeddings: text-embedding-3-small, 512 dimensions
- Pinecone namespace: `gtm-docs`

> **Note**: Do not use the Extract from File node in this pipeline. The Default Data Loader handles document parsing directly — adding Extract from File will break the ingestion.

---

### Workflow 1.2 — Simple RAG query pipeline

**Trigger**: Chat Trigger (replace with Webhook node for Slack or frontend integration)

**Flow**: Chat Trigger → Pinecone Vector Store (as tool) → AI Agent

**Key configuration**:
- Pinecone operation: Retrieve Documents (As Tool for AI Agent)
- OpenAI Embeddings: text-embedding-3-small, 512 dimensions — must match ingestion
- Pinecone namespace: `gtm-docs`
- AI Agent model: GPT-4o

**Test queries**:
- `What is our refund policy for annual contracts?`
- `What is our SOC 2 compliance status for enterprise accounts?`
- `How do we handle data residency for EU customers?`

---

### Workflow 2 — Agentic RAG

**Trigger**: Chat Trigger

**Flow**: Chat Trigger → Query Rewriter → Synthesis Agent (with Pinecone as tool)

**Key configuration**:
- Query Rewriter: basic OpenAI node (not an Agent), GPT-4o, rewrites raw query into retrieval-optimized language
- Synthesis Agent prompt: set to `{{ $json.output }}` to receive the rewritten query
- Pinecone tool description: must explicitly instruct the Agent to always call the tool before answering — the Agent routes based on the tool description
- Two separate LLM calls by design: rewriter and synthesizer have different system prompts and different jobs

**Test queries** (intentionally conversational to demonstrate query rewriting):
- `how do we beat Gong when the prospect lives in Salesforce?`
- `what's our story on security for big regulated companies?`
- `can we do a deal with a big team and give them a good price?`

---

### Workflow 3 — Graph RAG

**Trigger**: Chat Trigger

**Flow**: Chat Trigger → Entity Extractor → Edit Fields → HTTP Request (Neo4j) → Synthesis Agent

**Key configuration**:
- Entity Extractor: basic OpenAI node, GPT-4o, returns `{"company": "CompanyName"}`
- HTTP Request: POST to Neo4j AuraDB Query API v2
  - URL format: `https://YOUR_INSTANCE_ID.databases.neo4j.io/db/YOUR_INSTANCE_ID/query/v2`
  - Authentication: Basic Auth with AuraDB credentials
  - Headers: `Content-Type: application/json`, `Accept: application/json`
- Synthesis Agent: receives graph traversal results and synthesizes into narrative

**Cypher query used**:
```cypher
MATCH (c:Contact)-[:WORKS_AT|PREVIOUSLY_WORKED_AT]->(company:Company {name: 'Databricks'})
OPTIONAL MATCH (c)<-[:KNOWS]-(ourContact:Contact)
RETURN c.name as contactName, c.title as title, ourContact.name as connectedContact
```

**Test query**:
- `Who in our network has a connection to someone at Databricks?`

> **Disclaimer**: This is a simplified illustration of the Graph RAG pattern. Building a production-grade knowledge graph requires dedicated graph engineers and carefully designed ontologies. The workflow here demonstrates the architecture — not a production-ready implementation.

---

## The ontology

The Graph RAG workflow is built on a simple ontology defining the entity types and relationships in the knowledge graph:

```
Entities:     Contact, Company, Deal, Interaction

Relationships:
  Contact  ──[WORKS_AT]──────────────▶  Company
  Contact  ──[PREVIOUSLY_WORKED_AT]──▶  Company
  Contact  ──[KNOWS]─────────────────▶  Contact
  Contact  ──[PARTICIPATED_IN]────────▶  Deal
  Contact  ──[HAD_INTERACTION]────────▶  Interaction
  Company  ──[IS_PROSPECT]────────────▶  Deal
```

In production, this ontology would be populated from CRM data and maintained by graph engineers as contacts change roles and new relationships form.

---

## Lessons learned building these workflows

A few things that aren't obvious from the n8n docs:

**On ingestion**: the Default Data Loader stores binary blobs if the file type isn't supported — always verify vector content in the Pinecone console before assuming ingestion worked. Use plain `.txt` files for reliable parsing.

**On retrieval**: embedding model and dimensions must match exactly between ingestion and retrieval workflows. A mismatch produces zero results with no error message.

**On the Agentic RAG tool description**: the AI Agent routes to tools based on their description. A vague description means the Agent may skip the tool entirely and answer from training data. Be explicit — instruct the Agent to always call the tool before answering.

**On Graph RAG and AuraDB**: the Neo4j HTTP Query API (v2) is required for n8n integration. AuraDB Free tier may restrict HTTP API access — AuraDB Professional is recommended for this workflow.

---

## Series index

| Article | Topic |
|---------|-------|
| Week 1 | How to pick the right problem for AI agents |
| Week 2 | What is an agentic system, actually? |
| Week 4 | The AI Meeting Prep Assistant: from problem to full product |
| Week 5 | RAG systems in practice: Simple, Agentic, and Graph RAG ← you are here |
| Week 6 | Evaluations: how to know when your RAG system is ready for launch |

---

## About the series

**Building with Agentic AI** is a 10-week series covering practical agentic AI implementation for GTM, sales, and marketing teams. Each article pairs conceptual explanation with real GTM examples and hands-on n8n workflow builds.

Follow along on [Medium](#) and [LinkedIn](#).
