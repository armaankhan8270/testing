# Snowflake Cortex Complete Mastery Guide
## Cortex Search + Cortex Analyst + Cortex Agents + Document Storage — Zero Doubt Edition

---

# TABLE OF CONTENTS

```
PART 1:  Snowflake Fundamentals (Required Foundation)
PART 2:  What is Snowflake Cortex (The Umbrella)
PART 3:  How Documents Are Stored in Snowflake
PART 4:  Cortex Search — Complete Deep Dive
PART 5:  Cortex Analyst — Complete Deep Dive
PART 6:  Cortex Agents — Complete Deep Dive
PART 7:  The Complete RAG Architecture (All Pieces Together)
PART 8:  End-to-End Walkthrough Using Atlantic American Data
PART 9:  Query Lifecycle — What Happens When a User Asks a Question
```

---

# PART 1: SNOWFLAKE FUNDAMENTALS (Required Foundation)

Before Cortex makes any sense, you need to understand HOW Snowflake organizes data. Think of it as nested containers:

```
┌─────────────────────────────────────────────────────────────────┐
│                      SNOWFLAKE ACCOUNT                          │
│                  (your entire organization)                     │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                      DATABASE                           │   │
│   │              "ATLANTIC_AMERICAN_DB"                      │   │
│   │                                                         │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │                    SCHEMA                        │   │   │
│   │   │              "HEALTH_CLAIMS"                      │   │   │
│   │   │                                                   │   │   │
│   │   │   ┌─────────────┐  ┌─────────────┐  ┌─────────┐   │   │   │
│   │   │   │   TABLE     │  │   STAGE     │  │  VIEW   │   │   │   │
│   │   │   │ (structured │  │ (raw files/ │  │(virtual │   │   │   │
│   │   │   │  rows/cols) │  │  documents) │  │  table) │   │   │   │
│   │   │   └─────────────┘  └─────────────┘  └─────────┘   │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Terminology you must know:**

| Term | Meaning |
|---|---|
| **Database** | Top-level container (one per business/project — e.g., `ATLANTIC_AMERICAN_DB`) |
| **Schema** | Sub-folder inside a database (e.g., `HEALTH_CLAIMS`, `PC_CLAIMS`, `RAG_SYSTEM`) |
| **Table** | Structured rows/columns, just like SQL Server |
| **Stage** | A storage location (internal or pointing to Azure Blob/S3) where RAW FILES live (PDFs, images) |
| **Warehouse** | The "compute engine" — virtual servers that actually RUN your queries (you pay per second of usage) |
| **Role** | Security/permission groups (RBAC — Role-Based Access Control) |

### The Warehouse Concept (Critical — This is How You Pay)

```
A Warehouse is like renting a car engine ONLY while you're driving:

┌─────────────────────────────────────────────────────────────┐
│  WAREHOUSE: "RAG_COMPUTE_WH"                                 │
│  Size: MEDIUM (4 servers worth of compute)                   │
│  Auto-Suspend: 60 seconds of inactivity → turns OFF           │
│  Auto-Resume: Turns back ON instantly when a query arrives    │
│                                                               │
│  You are billed ONLY for the seconds it's actively running   │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Creating a warehouse for your RAG workloads
CREATE WAREHOUSE RAG_COMPUTE_WH
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE;
```

---

# PART 2: WHAT IS SNOWFLAKE CORTEX (The Umbrella)

**Snowflake Cortex** is Snowflake's built-in AI/ML layer. It means you NEVER have to send your data OUTSIDE Snowflake to a third-party AI service (like OpenAI directly) — everything happens INSIDE Snowflake's secure boundary. This is HUGE for Atlantic American because of HIPAA/PHI compliance — your sensitive EOB and medical data never leaves Snowflake's governed environment.

```
┌───────────────────────────────────────────────────────────────────────┐
│                        SNOWFLAKE CORTEX SUITE                        │
├─────────────────────┬─────────────────────┬───────────────────────────┤
│  CORTEX FUNCTIONS    │   CORTEX SEARCH     │   CORTEX ANALYST          │
│  (LLM primitives)    │   (Unstructured     │   (Structured data        │
│                      │    doc retrieval)    │    natural language Q&A) │
├─────────────────────┼─────────────────────┼───────────────────────────┤
│ COMPLETE()           │ Vector + Keyword     │ Text-to-SQL translation   │
│ EMBED_TEXT_768()     │ Hybrid Search        │ Semantic Model (YAML)     │
│ SUMMARIZE()          │ Re-ranking built-in  │ Verified Query Repository │
│ EXTRACT_ANSWER()     │ Auto-chunking        │ Metric/Dimension mapping  │
│ SENTIMENT()          │ Auto-embedding       │                          │
│ TRANSLATE()          │ Auto-refresh         │                          │
├─────────────────────┴─────────────────────┴───────────────────────────┤
│                         CORTEX AGENTS                                │
│         (Orchestrates BOTH Search + Analyst + Custom Tools            │
│          to answer complex, multi-step questions)                     │
└───────────────────────────────────────────────────────────────────────┘
```

### The Simple Way to Remember What Each Piece Does:

```
"What does the EOB document SAY?"          → CORTEX SEARCH (unstructured text)
"How MUCH did we pay across all claims?"   → CORTEX ANALYST (structured numbers)
"Compare what the policy says vs totals"   → CORTEX AGENT (combines both)
```

---

# PART 3: HOW DOCUMENTS ARE STORED IN SNOWFLAKE

This is the piece that connects your Azure Blob migration to Snowflake. There are TWO storage concepts you need to fully understand.

## 3.1 Stages — The "Landing Zone" for Raw Files

A **Stage** is a pointer to a storage location. It does NOT copy your 7TB into Snowflake — it just tells Snowflake "here's where the raw files live."

```
┌──────────────────────────────────────────────────────────────────┐
│                    TYPES OF STAGES                               │
├────────────────────┬───────────────────────────────────────────── │
│ Internal Stage      │ Files physically stored INSIDE Snowflake's   │
│                     │ own managed storage (Snowflake manages it)   │
├────────────────────┼───────────────────────────────────────────── │
│ External Stage      │ Files stay in YOUR Azure Blob Storage/S3 —  │
│ (YOUR USE CASE)     │ Snowflake just reads from it directly        │
└────────────────────┴───────────────────────────────────────────── │
```

### Setting Up an External Stage Pointing to Your Azure Blob

```sql
-- STEP 1: Create a Storage Integration (secure credential bridge to Azure)
CREATE STORAGE INTEGRATION azure_aame_integration
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = 'AZURE'
    ENABLED = TRUE
    AZURE_TENANT_ID = 'your-azure-tenant-id'
    STORAGE_ALLOWED_LOCATIONS = ('azure://aameimagerightmig.blob.core.windows.net/raw-documents/');

-- STEP 2: Grant Snowflake's service principal access in Azure 
-- (done in Azure Portal — you get a consent URL from DESC INTEGRATION)
DESC STORAGE INTEGRATION azure_aame_integration;

-- STEP 3: Create the External Stage pointing to your Blob container
CREATE STAGE AAME_RAW_DOCS_STAGE
    STORAGE_INTEGRATION = azure_aame_integration
    URL = 'azure://aameimagerightmig.blob.core.windows.net/raw-documents/'
    DIRECTORY = (ENABLE = TRUE)   -- enables file-listing metadata
    FILE_FORMAT = (TYPE = 'AUTO');

-- STEP 4: See what files are visible through the stage (no data copied yet!)
LIST @AAME_RAW_DOCS_STAGE;

-- STEP 5: Query the Directory Table (auto-generated file inventory)
SELECT RELATIVE_PATH, SIZE, LAST_MODIFIED
FROM DIRECTORY(@AAME_RAW_DOCS_STAGE)
LIMIT 100;
```

**What this gives you:**
```
RELATIVE_PATH                                                    SIZE      LAST_MODIFIED
drawer=HealthClaims/year=2024/file=CLM-88412/doc=EOB/page_001.pdf  245KB    2024-03-15
drawer=HealthClaims/year=2024/file=CLM-88412/doc=EOB/page_002.pdf  198KB    2024-03-15
drawer=PC_Claims/year=2024/file=CLM-99021/doc=FNOL/page_001.tif    1.2MB    2024-02-10
```

## 3.2 The FILE Data Type (Snowflake's Native Document Handling)

Snowflake has a special column type called `FILE` that lets you reference a document AND process it with built-in functions (OCR, text extraction) without ever manually downloading it.

```sql
-- Creating a table that references documents directly
CREATE TABLE RAW_DOCUMENT_REGISTRY (
    document_id         VARCHAR,
    drawer_name          VARCHAR,
    file_number           VARCHAR,      -- Claim/Policy Number
    doc_type              VARCHAR,       -- EOB, FNOL, CMS-1500, etc.
    page_number           INT,
    file_ref               FILE,          -- <-- Special FILE type!
    relative_path         VARCHAR,
    ingested_timestamp    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Populating it by referencing the stage directly
INSERT INTO RAW_DOCUMENT_REGISTRY (document_id, drawer_name, file_number, doc_type, page_number, file_ref, relative_path)
SELECT 
    MD5(relative_path) AS document_id,
    SPLIT_PART(relative_path, '/', 1) AS drawer_name,
    SPLIT_PART(relative_path, '/', 3) AS file_number,
    SPLIT_PART(relative_path, '/', 4) AS doc_type,
    1 AS page_number,
    TO_FILE('@AAME_RAW_DOCS_STAGE', relative_path) AS file_ref,
    relative_path
FROM DIRECTORY(@AAME_RAW_DOCS_STAGE);
```

### Using Cortex's Document AI / Parse Function on Stored Files

```sql
-- PARSE_DOCUMENT extracts text content directly from PDFs/images
-- stored in your stage — this is your OCR replacement for native PDFs!

SELECT 
    document_id,
    file_number,
    doc_type,
    SNOWFLAKE.CORTEX.PARSE_DOCUMENT(
        '@AAME_RAW_DOCS_STAGE', 
        relative_path,
        {'mode': 'LAYOUT'}  -- preserves table structure (critical for EOBs!)
    ) AS extracted_content
FROM RAW_DOCUMENT_REGISTRY
WHERE doc_type = 'EOB'
LIMIT 10;
```

**Output example (JSON structure preserving layout):**
```json
{
  "content": "Patient: John Smith | Claim #: CLM-2024-00129\n| CPT Code | Billed | Allowed | Paid | Denied |\n| 99213    | $450   | $320    | $256 | $130   |",
  "metadata": {
    "pageCount": 1,
    "ocrConfidence": 0.97
  }
}
```

> **This is a GAME CHANGER for your pipeline.** `PARSE_DOCUMENT` with `LAYOUT` mode means Snowflake Cortex can do your OCR + table extraction natively, potentially replacing the need for a separate Azure Document Intelligence step for many of your documents.

---

# PART 4: CORTEX SEARCH — COMPLETE DEEP DIVE

## 4.1 What Cortex Search Actually Is

Cortex Search is Snowflake's **fully managed RAG retrieval engine**. It handles chunking, embedding, indexing, and hybrid search — ALL automatically, without you managing a separate vector database (like Pinecone or Weaviate).

```
┌──────────────────────────────────────────────────────────────────────────┐
│               TRADITIONAL RAG (What you'd build manually)                │
├──────────────────────────────────────────────────────────────────────────┤
│  Your Code → Chunk Text → Call Embedding API → Store in Vector DB →      │
│  Query → Embed Question → Vector Search → Rerank → Return Results        │
│  (You manage EVERY step, EVERY piece of infrastructure)                  │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│               CORTEX SEARCH (What Snowflake does for you)                │
├──────────────────────────────────────────────────────────────────────────┤
│  You: "Here's my table of text + metadata"                               │
│  Snowflake: Automatically chunks, embeds, indexes, refreshes, and        │
│             serves hybrid search results via simple SQL or REST API      │
└──────────────────────────────────────────────────────────────────────────┘
```

## 4.2 The Complete Architecture of Cortex Search

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  STEP 1: SOURCE TABLE (your OCR'd/parsed text with metadata)               │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │ CHUNK_TEXT | FILE_NUMBER | DOC_TYPE | PATIENT_NAME | DRAWER | ...  │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                  │                                         │
│                                  ▼                                         │
│  STEP 2: CORTEX SEARCH SERVICE (created with one SQL command)              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │  a) Automatically embeds every row's text using a built-in model   │    │
│  │     (e.g., snowflake-arctic-embed-m)                                │    │
│  │  b) Builds a vector index (HNSW) for fast similarity search         │    │
│  │  c) ALSO builds a keyword/BM25 index for exact-match search         │    │
│  │  d) Keeps BOTH indexes in sync automatically as new rows are added  │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                  │                                         │
│                                  ▼                                         │
│  STEP 3: QUERY TIME (user asks a question)                                 │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │  User Query → Embedded automatically → Hybrid Search runs:         │    │
│  │     • Vector similarity (semantic meaning match)                   │    │
│  │     • Keyword/BM25 (exact term match, e.g., claim numbers)         │    │
│  │  → Results automatically RE-RANKED by relevance                    │    │
│  │  → Returns top-K chunks + full metadata columns                    │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 4.3 Step-by-Step: Building a Cortex Search Service for AAME's EOBs

### Step 1: Prepare Your Source Table (Post-Chunking)

```sql
-- This table holds your chunked, OCR'd document text + ALL metadata
-- that came from the ImageRight EAV extraction (Part covered earlier)

CREATE TABLE HEALTH_CLAIMS_CHUNKS (
    chunk_id          VARCHAR PRIMARY KEY,
    chunk_text         VARCHAR,          -- the actual text content (what gets embedded)
    document_id         VARCHAR,
    file_number          VARCHAR,          -- Claim Number (from ImageRight IR_File)
    policy_number        VARCHAR,
    patient_name         VARCHAR,
    doc_type              VARCHAR,          -- 'EOB', 'CMS-1500', etc.
    date_of_service       DATE,
    paid_amount           DECIMAL(10,2),
    denied_amount         DECIMAL(10,2),
    drawer_name           VARCHAR,          -- 'Bankers Fidelity - Health Claims'
    page_number           INT,
    source_blob_path      VARCHAR,
    chunk_sequence         INT               -- order within the document
);

-- Example rows after your parsing/chunking pipeline has run:
INSERT INTO HEALTH_CLAIMS_CHUNKS VALUES
('chk_001', 'Patient: John Smith, Claim CLM-2024-00129. CPT Code 99213 Office Visit. Billed $450.00, Allowed $320.00, Plan Paid $256.00, Patient Responsibility $64.00, Denied $130.00 due to CO-45 (charge exceeds fee schedule).',
 'doc_45002', 'CLM-2024-00129', 'POL-BFL-2024-88412', 'John Smith', 'EOB', '2024-03-15', 256.00, 130.00, 'Bankers Fidelity - Health Claims', 1, 'drawer=HealthClaims/.../page_001.pdf', 1);
```

### Step 2: Create the Cortex Search Service

```sql
CREATE OR REPLACE CORTEX SEARCH SERVICE AAME_HEALTH_CLAIMS_SEARCH
    ON chunk_text                          -- the column to embed/search
    ATTRIBUTES file_number, policy_number, patient_name, doc_type,
               date_of_service, paid_amount, denied_amount, drawer_name
    WAREHOUSE = RAG_COMPUTE_WH
    TARGET_LAG = '1 hour'                  -- how often it re-indexes new data
    AS (
        SELECT 
            chunk_id, chunk_text, file_number, policy_number,
            patient_name, doc_type, date_of_service, 
            paid_amount, denied_amount, drawer_name
        FROM HEALTH_CLAIMS_CHUNKS
    );
```

**What just happened under the hood:**
```
1. Snowflake read every row's "chunk_text" value
2. Called an internal embedding model (snowflake-arctic-embed) on each chunk
3. Stored the resulting vectors in an internal optimized index (HNSW graph)
4. ALSO built a keyword/BM25 inverted index on the same text
5. Attached ALL your other columns (file_number, patient_name, etc.) as 
   filterable "attributes" — searchable WITHOUT needing embeddings
6. Set up an automatic refresh job that checks for new/changed rows 
   every "TARGET_LAG" period (1 hour here) and re-indexes automatically
```

### Step 3: Querying the Cortex Search Service

#### Option A: Pure SQL Query (Simplest)

```sql
SELECT PARSE_JSON(
    SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
        'AAME_HEALTH_CLAIMS_SEARCH',
        '{
            "query": "What was denied on John Smith EOB for office visit?",
            "columns": ["chunk_text", "file_number", "patient_name", "paid_amount", "denied_amount"],
            "limit": 5
        }'
    )
) AS search_results;
```

#### Option B: With Metadata Filtering (Hybrid Precision)

```sql
SELECT PARSE_JSON(
    SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
        'AAME_HEALTH_CLAIMS_SEARCH',
        '{
            "query": "denied claim reason",
            "columns": ["chunk_text", "file_number", "doc_type", "denied_amount"],
            "filter": {
                "@and": [
                    {"@eq": {"patient_name": "John Smith"}},
                    {"@eq": {"doc_type": "EOB"}},
                    {"@gte": {"date_of_service": "2024-01-01"}}
                ]
            },
            "limit": 5
        }'
    )
) AS filtered_results;
```

**This is the MOST IMPORTANT concept in Cortex Search:** You combine **semantic vector search** (understanding meaning: "denied claim reason") with **hard metadata filters** (exact match: `patient_name = 'John Smith'`). This gives you both the flexibility of AI search AND the precision of a database query — critical for insurance where you cannot afford to mix up one patient's data with another's.

#### Option C: Python/REST API Query (For Your Application Layer)

```python
from snowflake.core import Root
from snowflake.snowpark import Session

session = Session.builder.configs(connection_params).create()
root = Root(session)

search_service = (root
    .databases["ATLANTIC_AMERICAN_DB"]
    .schemas["HEALTH_CLAIMS"]
    .cortex_search_services["AAME_HEALTH_CLAIMS_SEARCH"]
)

results = search_service.search(
    query="What was denied on John Smith's claim?",
    columns=["chunk_text", "file_number", "patient_name", "denied_amount"],
    filter={"@eq": {"patient_name": "John Smith"}},
    limit=5
)

for r in results.results:
    print(r["chunk_text"], r["denied_amount"])
```

## 4.4 How Chunking Should Work BEFORE It Reaches Cortex Search

Cortex Search does NOT chunk raw PDFs for you — **you must chunk the text yourself** before loading it into the source table. Here's the best-practice chunking pattern for insurance documents:

```python
from snowflake.snowpark.functions import col
from langchain.text_splitter import RecursiveCharacterTextSplitter

def chunk_eob_document(ocr_text: str, metadata: dict) -> list[dict]:
    """
    Chunking strategy specifically tuned for EOB/insurance documents.
    Key principle: NEVER split a table row across two chunks.
    """
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=800,          # smaller chunks = more precise retrieval
        chunk_overlap=100,       # preserve context across boundaries
        separators=["\n\n", "\n", ". ", " "]  # prefer natural breaks
    )
    
    raw_chunks = splitter.split_text(ocr_text)
    
    enriched_chunks = []
    for i, chunk in enumerate(raw_chunks):
        enriched_chunks.append({
            "chunk_id": f"{metadata['document_id']}_chunk_{i}",
            "chunk_text": chunk,
            "chunk_sequence": i,
            **metadata   # carry forward file_number, patient_name, doc_type, etc.
        })
    return enriched_chunks
```

> **Golden Rule for Insurance Chunking:** EVERY chunk must carry the FULL identity metadata (file_number, patient_name, policy_number, doc_type) as separate ATTRIBUTE columns — NOT just embedded in the text. This is what lets Cortex Search filter precisely and prevents the AI from confusing one patient's claim with another.

---

# PART 5: CORTEX ANALYST — COMPLETE DEEP DIVE

## 5.1 What Cortex Analyst Actually Is

While Cortex Search answers **"What does this document say?"**, Cortex Analyst answers **"What do the NUMBERS say?"** — it translates natural language into accurate SQL queries against your STRUCTURED tables (the EAV-pivoted claims data, billing data, etc.).

```
USER ASKS: "What is the total denied amount for Bankers Fidelity claims in Q1 2024?"
                                    │
                                    ▼
                    CORTEX ANALYST translates this into:
                                    │
                                    ▼
SELECT SUM(denied_amount) 
FROM claims_fact_table 
WHERE drawer_name = 'Bankers Fidelity - Health Claims' 
  AND date_of_service BETWEEN '2024-01-01' AND '2024-03-31';
                                    │
                                    ▼
                    Runs the SQL → Returns: $1,245,890.00
                                    │
                                    ▼
             Cortex Analyst formats a natural language answer:
    "The total denied amount for Bankers Fidelity Health Claims 
     in Q1 2024 was $1,245,890.00 across 3,204 claims."
```

**Why this is hard to do with a regular LLM alone:** A generic LLM might HALLUCINATE a SQL query with wrong table/column names, or calculate numbers incorrectly. Cortex Analyst solves this using a **Semantic Model** — essentially a strict rulebook that tells the AI EXACTLY what tables, columns, and calculations are allowed.

## 5.2 The Semantic Model — The Heart of Cortex Analyst

A Semantic Model is a **YAML file** that defines your business's "vocabulary" — it maps plain English terms to actual database columns and formulas.

```yaml
# semantic_model_health_claims.yaml

name: aame_health_claims_model
description: Semantic model for Bankers Fidelity Health Claims analysis

tables:
  - name: claims_fact
    base_table:
      database: ATLANTIC_AMERICAN_DB
      schema: HEALTH_CLAIMS
      table: CLAIMS_FACT_TABLE
    
    # DIMENSIONS = things you GROUP BY or FILTER ON (categorical data)
    dimensions:
      - name: patient_name
        expr: PATIENT_NAME
        data_type: VARCHAR
        description: "The name of the patient who received care"
        synonyms: ["patient", "member", "insured person"]

      - name: drawer_name
        expr: DRAWER_NAME
        data_type: VARCHAR
        description: "Which AAME business unit/subsidiary this claim belongs to"
        synonyms: ["business unit", "subsidiary", "company division"]

      - name: denial_reason_code
        expr: DENIAL_REASON_CODE
        data_type: VARCHAR
        description: "The standardized code explaining why a claim was denied"
        synonyms: ["denial code", "rejection reason"]

      - name: cpt_code
        expr: CPT_CODE
        data_type: VARCHAR
        description: "Current Procedural Terminology code for the medical service"

      - name: date_of_service
        expr: DATE_OF_SERVICE
        data_type: DATE
        description: "The date the medical service was provided"

    # TIME DIMENSIONS = special date/time fields for trend analysis
    time_dimensions:
      - name: service_date
        expr: DATE_OF_SERVICE
        data_type: DATE
        description: "Used for time-based trending (monthly, quarterly, yearly)"

    # MEASURES = numeric values you SUM, AVG, COUNT (the actual metrics)
    measures:
      - name: total_billed
        expr: SUM(BILLED_AMOUNT)
        data_type: NUMBER
        description: "Total dollar amount billed by providers"
        synonyms: ["total charges", "billed total"]

      - name: total_paid
        expr: SUM(PAID_AMOUNT)
        data_type: NUMBER
        description: "Total dollar amount actually paid out by AAME"
        synonyms: ["total payout", "amount paid"]

      - name: total_denied
        expr: SUM(DENIED_AMOUNT)
        data_type: NUMBER
        description: "Total dollar amount denied/rejected"
        synonyms: ["denied total", "rejected amount"]

      - name: claim_count
        expr: COUNT(DISTINCT CLAIM_NUMBER)
        data_type: NUMBER
        description: "Total number of distinct claims"

      - name: denial_rate
        expr: SUM(DENIED_AMOUNT) / NULLIF(SUM(BILLED_AMOUNT), 0)
        data_type: NUMBER
        description: "Percentage of billed amount that was denied"

    # VERIFIED QUERIES = pre-approved example Q&A pairs (improves accuracy!)
    verified_queries:
      - name: "total_denied_by_quarter"
        question: "What is the total denied amount by quarter?"
        sql: |
          SELECT 
            DATE_TRUNC('quarter', DATE_OF_SERVICE) AS quarter,
            SUM(DENIED_AMOUNT) AS total_denied
          FROM ATLANTIC_AMERICAN_DB.HEALTH_CLAIMS.CLAIMS_FACT_TABLE
          GROUP BY quarter
          ORDER BY quarter;
```

## 5.3 Deploying the Semantic Model & Creating Cortex Analyst

```sql
-- Upload the YAML file to a Snowflake Stage
PUT file:///local_path/semantic_model_health_claims.yaml 
    @ATLANTIC_AMERICAN_DB.HEALTH_CLAIMS.SEMANTIC_MODELS_STAGE;

-- Cortex Analyst is then invoked via the REST API using this semantic model
```

```python
import requests
import json

def ask_cortex_analyst(question: str):
    url = f"https://{account}.snowflakecomputing.com/api/v2/cortex/analyst/message"
    
    payload = {
        "messages": [
            {"role": "user", "content": [{"type": "text", "text": question}]}
        ],
        "semantic_model_file": "@ATLANTIC_AMERICAN_DB.HEALTH_CLAIMS.SEMANTIC_MODELS_STAGE/semantic_model_health_claims.yaml"
    }
    
    response = requests.post(url, headers=headers, json=payload)
    result = response.json()
    
    # Response includes: the generated SQL, the SQL results, AND a natural language summary
    return result

answer = ask_cortex_analyst(
    "What is the total denied amount for Bankers Fidelity claims in Q1 2024?"
)

print("Generated SQL:", answer["sql"])
print("Answer:", answer["text"])
```

**What the API returns:**
```json
{
  "sql": "SELECT SUM(DENIED_AMOUNT) AS total_denied FROM CLAIMS_FACT_TABLE WHERE DRAWER_NAME = 'Bankers Fidelity - Health Claims' AND DATE_OF_SERVICE BETWEEN '2024-01-01' AND '2024-03-31'",
  "text": "The total denied amount for Bankers Fidelity Health Claims in Q1 2024 was $1,245,890.00.",
  "confidence": "high",
  "suggestions": []
}
```

## 5.4 Why the Semantic Model Prevents Hallucination

```
WITHOUT a Semantic Model (raw LLM + raw schema):
┌────────────────────────────────────────────────────────────────┐
│ Risk: LLM guesses column names, might write:                   │
│   SELECT SUM(amount_denied) ... ← WRONG COLUMN NAME (doesn't    │
│   exist, actual column is DENIED_AMOUNT) → Query FAILS or       │
│   worse, silently returns WRONG data from a similar column      │
└────────────────────────────────────────────────────────────────┘

WITH a Semantic Model (Cortex Analyst):
┌────────────────────────────────────────────────────────────────┐
│ Safety: The model can ONLY use the exact measures/dimensions    │
│   explicitly defined in the YAML. "total_denied" is PRE-MAPPED  │
│   to SUM(DENIED_AMOUNT) — removing all ambiguity about which    │
│   column or calculation to use.                                 │
└────────────────────────────────────────────────────────────────┘
```

---

# PART 6: CORTEX AGENTS — COMPLETE DEEP DIVE

## 6.1 What Cortex Agents Actually Do

A **Cortex Agent** is an orchestrator that decides, FOR EACH user question, whether to:
1. Call **Cortex Search** (for unstructured document text)
2. Call **Cortex Analyst** (for structured numeric analysis)
3. Call BOTH and combine the results
4. Call a **custom tool/function** you've defined

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CORTEX AGENT ORCHESTRATION                       │
│                                                                         │
│  User: "How much was denied on John Smith's EOB, and does that align   │
│         with his policy's stated deductible?"                          │
│                          │                                              │
│                          ▼                                              │
│            ┌─────────────────────────────┐                             │
│            │   CORTEX AGENT (Planner)     │                             │
│            │   Breaks question into sub-   │                             │
│            │   tasks and decides tools     │                             │
│            └──────────┬──────────────────┘                             │
│                       │                                                 │
│         ┌─────────────┴─────────────┐                                 │
│         ▼                           ▼                                 │
│  ┌──────────────┐           ┌──────────────┐                          │
│  │ TOOL CALL 1:  │           │ TOOL CALL 2:  │                          │
│  │ Cortex Search │           │ Cortex Analyst│                          │
│  │ "Find John    │           │ "Get deductible│                         │
│  │  Smith's EOB  │           │  amount from   │                          │
│  │  text"        │           │  policy table" │                          │
│  └──────┬───────┘           └──────┬───────┘                          │
│         │                          │                                   │
│         └────────────┬─────────────┘                                  │
│                      ▼                                                 │
│            ┌─────────────────────────┐                                │
│            │  AGENT SYNTHESIZES BOTH  │                                │
│            │  RESULTS INTO ONE ANSWER │                                │
│            └─────────────────────────┘                                │
│                      │                                                 │
│                      ▼                                                 │
│  "John Smith's EOB shows $130 denied due to exceeding the fee          │
│   schedule (code CO-45). His policy's annual deductible is $500,        │
│   and he has met $370 so far this year — this denial is UNRELATED      │
│   to his deductible status."                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## 6.2 Creating a Cortex Agent

```sql
CREATE OR REPLACE CORTEX AGENT AAME_CLAIMS_ASSISTANT
    WAREHOUSE = RAG_COMPUTE_WH
    COMMENT = 'Unified assistant for Atlantic American claims, policy, and EOB questions';
```

```python
# Configuring the agent's available tools via the REST API / Python SDK

agent_config = {
    "models": {
        "orchestration": "llama3.1-70b"   # the "brain" that decides which tool to call
    },
    "instructions": {
        "response": "Always cite the specific claim number, policy number, and document type used in your answer. Never guess numbers — always retrieve them from the tools provided.",
        "orchestration": "If the question involves both document content AND financial totals, use BOTH Cortex Search and Cortex Analyst, then combine the results."
    },
    "tools": [
        {
            "tool_spec": {
                "type": "cortex_search",
                "name": "search_health_claims_docs",
                "search_service": "ATLANTIC_AMERICAN_DB.HEALTH_CLAIMS.AAME_HEALTH_CLAIMS_SEARCH",
                "description": "Searches EOB, CMS-1500, and claim correspondence text"
            }
        },
        {
            "tool_spec": {
                "type": "cortex_analyst_text_to_sql",
                "name": "analyze_claims_financials",
                "semantic_model": "@ATLANTIC_AMERICAN_DB.HEALTH_CLAIMS.SEMANTIC_MODELS_STAGE/semantic_model_health_claims.yaml",
                "description": "Answers financial/aggregate questions about claims (totals, averages, counts)"
            }
        },
        {
            "tool_spec": {
                "type": "generic",
                "name": "lookup_policy_terms",
                "description": "Custom function to fetch policy deductible/coverage limits from the policy database"
            }
        }
    ]
}
```

## 6.3 Calling the Agent (The User-Facing Chat Interface)

```python
def ask_agent(question: str, conversation_history: list = None):
    url = f"https://{account}.snowflakecomputing.com/api/v2/cortex/agent:run"
    
    payload = {
        "agent": "ATLANTIC_AMERICAN_DB.HEALTH_CLAIMS.AAME_CLAIMS_ASSISTANT",
        "messages": (conversation_history or []) + [
            {"role": "user", "content": [{"type": "text", "text": question}]}
        ]
    }
    
    response = requests.post(url, headers=headers, json=payload, stream=True)
    
    # Agent responses stream back showing: tool calls, intermediate results, final answer
    for chunk in response.iter_lines():
        event = json.loads(chunk)
        if event["type"] == "tool_use":
            print(f"🔧 Agent is calling: {event['tool_name']}")
        elif event["type"] == "text":
            print(f"💬 {event['text']}", end="")
    
    return response
```

## 6.4 Multi-Turn Conversational Memory

```python
# The agent maintains conversation context across turns

conversation = []

# Turn 1
conversation.append({"role": "user", "content": [{"type": "text", "text": "Show me John Smith's claims from 2024"}]})
response1 = ask_agent_with_history(conversation)
conversation.append({"role": "assistant", "content": response1})

# Turn 2 - the agent REMEMBERS "John Smith" and "2024" from context
conversation.append({"role": "user", "content": [{"type": "text", "text": "Which of those were denied?"}]})
response2 = ask_agent_with_history(conversation)
# Agent understands "those" = the claims from Turn 1
```

---

# PART 7: THE COMPLETE RAG ARCHITECTURE (All Pieces Together)

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                     COMPLETE AAME RAG ARCHITECTURE ON SNOWFLAKE                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

  AZURE BLOB STORAGE                    SNOWFLAKE                         USER INTERFACE
  (Raw ImageRight docs)                                                   
  
┌──────────────────┐         ┌───────────────────────────────┐      ┌──────────────────┐
│ raw-documents/    │         │ EXTERNAL STAGE                 │      │                  │
│  drawer=Health/   │────────►│ (points to Azure Blob)         │      │   Web App /      │
│  drawer=PC/       │         └──────────────┬──────────────────┘      │   Chat Interface │
│  drawer=Life/     │                        │                        │                  │
└──────────────────┘                        ▼                        └──────────┬───────┘
                                  ┌───────────────────────┐                      │
                                  │  PARSE_DOCUMENT()       │                      │
                                  │  (OCR + Layout extract) │                      │
                                  └──────────┬──────────────┘                      │
                                             ▼                                    │
                                  ┌───────────────────────┐                      │
                                  │  CHUNKING LOGIC         │                      │
                                  │  (Python UDF/Snowpark)  │                      │
                                  └──────────┬──────────────┘                      │
                                             ▼                                    │
                        ┌────────────────────┴────────────────────┐             │
                        ▼                                         ▼             │
            ┌────────────────────────┐                ┌────────────────────┐    │
            │  CHUNKS TABLE           │                │  STRUCTURED FACT    │    │
            │  (unstructured text +   │                │  TABLES (claims,    │    │
            │   metadata)             │                │  policies, billing) │    │
            └───────────┬─────────────┘                └──────────┬──────────┘    │
                        ▼                                         ▼             │
            ┌────────────────────────┐                ┌────────────────────┐    │
            │  CORTEX SEARCH SERVICE  │                │  SEMANTIC MODEL     │    │
            │  (auto embed + index)   │                │  (YAML definitions) │    │
            └───────────┬─────────────┘                └──────────┬──────────┘    │
                        │                                         │             │
                        ▼                                         ▼             │
            ┌────────────────────────┐                ┌────────────────────┐    │
            │  (Tool for Agent)       │                │  CORTEX ANALYST     │    │
            │                         │                │  (Tool for Agent)   │    │
            └───────────┬─────────────┘                └──────────┬──────────┘    │
                        │                                         │             │
                        └──────────────────┬──────────────────────┘             │
                                           ▼                                   │
                                ┌─────────────────────────┐                     │
                                │   CORTEX AGENT           │◄────────────────────┘
                                │   (orchestrates both,     │    User's question
                                │    decides which tool(s)  │    arrives here
                                │    to call, synthesizes   │
                                │    final answer)          │
                                └─────────────┬─────────────┘
                                             │
                                             ▼
                                   Natural language answer
                                   with citations, sent back
                                   to the user interface
```

---

# PART 8: END-TO-END WALKTHROUGH USING ATLANTIC AMERICAN DATA

Let's trace ONE complete question through the ENTIRE system, step by step, with zero gaps.

### The User's Question:
> *"What was the total amount denied for John Smith's claims in 2024, and can you show me the specific EOB line items that were denied?"*

```
═══════════════════════════════════════════════════════════════════════════
STEP 1: Question arrives at the Cortex Agent
═══════════════════════════════════════════════════════════════════════════
Agent's orchestration model (Llama 3.1 70B) analyzes the question and 
recognizes TWO distinct needs:
  (a) An aggregate NUMBER → "total amount denied" → needs Cortex Analyst
  (b) Specific DOCUMENT CONTENT → "specific EOB line items" → needs Cortex Search

═══════════════════════════════════════════════════════════════════════════
STEP 2: Agent calls Cortex Analyst (Tool Call #1)
═══════════════════════════════════════════════════════════════════════════
Agent generates this internal request to Cortex Analyst:
  "What is the total denied amount for patient John Smith in 2024?"

Cortex Analyst uses the Semantic Model to translate this into SQL:

  SELECT SUM(DENIED_AMOUNT) AS total_denied
  FROM ATLANTIC_AMERICAN_DB.HEALTH_CLAIMS.CLAIMS_FACT_TABLE
  WHERE PATIENT_NAME = 'John Smith'
    AND YEAR(DATE_OF_SERVICE) = 2024;

SQL Result: total_denied = $430.00

═══════════════════════════════════════════════════════════════════════════
STEP 3: Agent calls Cortex Search (Tool Call #2)
═══════════════════════════════════════════════════════════════════════════
Agent generates this internal request to Cortex Search:
  Query: "denied line items EOB"
  Filter: {"@and": [{"@eq": {"patient_name": "John Smith"}}, 
                     {"@eq": {"doc_type": "EOB"}},
                     {"@gte": {"date_of_service": "2024-01-01"}}]}

Cortex Search:
  1. Embeds the query "denied line items EOB" into a vector
  2. Runs hybrid search (vector + keyword) against the pre-built index
  3. Applies the metadata filters (patient_name, doc_type, date range)
  4. Returns top 5 most relevant chunks

Results returned:
  Chunk 1: "CPT 99213 Office Visit. Billed $450.00, Allowed $320.00, 
            Paid $256.00, Denied $130.00, Reason CO-45 (exceeds fee schedule)"
            [file_number: CLM-2024-00129]
            
  Chunk 2: "CPT 85025 Blood Panel. Billed $200.00, Allowed $150.00,
            Paid $0.00, Denied $150.00, Reason CO-50 (not medically necessary)"
            [file_number: CLM-2024-00144]
            
  Chunk 3: "CPT 71045 Chest X-Ray. Billed $300.00, Allowed $150.00,
            Paid $0.00, Denied $150.00, Reason CO-97 (bundled service)"
            [file_number: CLM-2024-00144]

═══════════════════════════════════════════════════════════════════════════
STEP 4: Agent Synthesizes the Final Answer
═══════════════════════════════════════════════════════════════════════════
The Agent combines BOTH tool results into one coherent, cited answer:

"John Smith's total denied amount for 2024 was $430.00 across 3 claim 
line items:

1. Claim CLM-2024-00129 (Office Visit, CPT 99213): $130.00 denied — 
   Reason: Charge exceeds fee schedule (CO-45)

2. Claim CLM-2024-00144 (Blood Panel, CPT 85025): $150.00 denied — 
   Reason: Not medically necessary (CO-50)

3. Claim CLM-2024-00144 (Chest X-Ray, CPT 71045): $150.00 denied — 
   Reason: Bundled service (CO-97)

Source documents: EOB dated 03/15/2024 and 06/02/2024, Bankers Fidelity 
Health Claims."

═══════════════════════════════════════════════════════════════════════════
STEP 5: Response delivered to user interface with citations
═══════════════════════════════════════════════════════════════════════════
```

---

# PART 9: QUERY LIFECYCLE — COMPLETE TECHNICAL FLOW

```
┌───────────────────────────────────────────────────────────────────────────┐
│ TIME │ COMPONENT              │ WHAT HAPPENS                              │
├──────┼────────────────────────┼────────────────────────────────────────────┤
│ T+0ms│ User Interface          │ User types question, hits "Send"          │
├──────┼────────────────────────┼────────────────────────────────────────────┤
│ T+10ms│ API Gateway            │ Request authenticated via Snowflake RBAC  │
│      │                        │ (user's role determines which Drawers'    │
│      │                        │ data they're allowed to see)              │
├──────┼────────────────────────┼────────────────────────────────────────────┤
│ T+50ms│ Cortex Agent           │ Orchestration LLM reads question, decides │
│      │ (Orchestration Model)  │ which tool(s) to invoke                   │
├──────┼────────────────────────┼────────────────────────────────────────────┤
│ T+100ms│ Tool Dispatch          │ Parallel calls fired to Cortex Search    │
│      │                        │ AND Cortex Analyst (if both needed)       │
├──────┼────────────────────────┼────────────────────────────────────────────┤
│ T+150ms│ Cortex Search          │ Query embedded (snowflake-arctic-embed)  │
│      │                        │ → Vector index searched (HNSW)            │
│      │                        │ → Keyword index searched (BM25)           │
│      │                        │ → Results merged + re-ranked              │
│      │                        │ → Metadata filters applied                │
├──────┼────────────────────────┼────────────────────────────────────────────┤
│ T+150ms│ Cortex Analyst         │ Question + Semantic Model → LLM generates│
│      │ (parallel)             │ SQL → SQL validated against schema →      │
│      │                        │ SQL executed on warehouse → Result set    │
├──────┼────────────────────────┼────────────────────────────────────────────┤
│ T+800ms│ Agent Synthesis        │ Both tool results fed back into the      │
│      │                        │ orchestration LLM, which writes the final │
│      │                        │ natural language answer with citations    │
├──────┼────────────────────────┼────────────────────────────────────────────┤
│ T+1.2s│ Response Streaming     │ Answer streamed token-by-token back to    │
│      │                        │ the user interface (like ChatGPT typing)  │
├──────┼────────────────────────┼────────────────────────────────────────────┤
│ T+1.5s│ Audit Logging          │ Full query, tools used, SQL executed, and │
│      │                        │ results logged to AUDIT_LOG table for     │
│      │                        │ HIPAA compliance tracking                 │
└──────┴────────────────────────┴────────────────────────────────────────────┘
```

---

# SUMMARY: THE COMPLETE MENTAL MODEL

```
╔═══════════════════════════════════════════════════════════════════════════╗
║  1. DOCUMENTS LIVE IN AZURE BLOB → Snowflake reads them via EXTERNAL STAGE ║
║                                                                           ║
║  2. SNOWFLAKE CORTEX PARSE_DOCUMENT() extracts text (OCR + table layout) ║
║                                                                           ║
║  3. YOUR CHUNKING LOGIC splits text into small pieces, tagging each      ║
║     with full identity metadata (patient, policy, claim, doc type)      ║
║                                                                           ║
║  4. UNSTRUCTURED TEXT → CORTEX SEARCH (auto-embeds, auto-indexes,        ║
║     serves hybrid vector+keyword search)                                 ║
║                                                                           ║
║  5. STRUCTURED NUMBERS → CORTEX ANALYST (uses a Semantic Model YAML      ║
║     to safely translate English questions into exact SQL)               ║
║                                                                           ║
║  6. CORTEX AGENT sits on top of BOTH, deciding which tool(s) to call    ║
║     for any given question, and synthesizing one final cited answer     ║
║                                                                           ║
║  7. EVERYTHING stays INSIDE Snowflake's governed, encrypted boundary —  ║
║     critical for HIPAA/PHI compliance with Atlantic American's data     ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

---

Does this fully close the gap on how Snowflake Cortex Search, Cortex Analyst, and Cortex Agents work together? 

Next, we could go deeper into:
1. **Writing the complete Snowpark Python pipeline** that takes raw ImageRight PDFs from Azure Blob → PARSE_DOCUMENT → chunking → loading into the Chunks table (the actual ETL code)
2. **Building the full Semantic Model YAML** for ALL of Atlantic American's business lines (Health, Life, P&C) with every measure/dimension mapped
3. **Setting up the Cortex Agent's security/RBAC** so a Health Claims user can't accidentally query P&C data

Let me know which you'd like to tackle next.
