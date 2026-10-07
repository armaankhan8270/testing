Yes. And for **your ImageRight migration use case**, I would teach this much more deeply than “Cortex Search does vector search, Analyst does SQL, Agent connects them.”

The real goal is that you can look at the architecture and answer:

> **Where is the document? Where is its text? Where is its metadata? What exactly gets indexed? Where are embeddings created? What happens when a user asks a question? How does Snowflake decide whether to search documents or query tables? How does the final answer get produced and cited?**

That is the level at which there should be no mystery.

One current change matters: as of 2026, Snowflake recommends using **Cortex Agents as the orchestration layer**. Cortex Analyst is still important, but it is now better thought of as the **structured-data tool inside the Agent**, while Cortex Search is the **unstructured-data retrieval tool**. ([docs.snowflake.com][1])

# 1. First, forget the product names

Start with the underlying problem.

You have two kinds of information.

```text
                         YOUR CLAIM DATA
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
        STRUCTURED DATA                   UNSTRUCTURED DATA
              │                                 │
              │                                 │
      claim_id = 84729                    Police report
      loss_date                            FNOL
      amount                               Adjuster notes
      state                                Repair estimate
      status                               Medical report
      policy_id                            Photos
```

These require different ways of answering questions.

### Structured question

> “How many claims were opened in Maharashtra in 2025?”

The computer needs to calculate.

```text
Question
   ↓
SQL
   ↓
Snowflake tables
   ↓
SUM / COUNT / GROUP BY / WHERE
   ↓
Answer
```

### Unstructured question

> “What did the police report say caused the accident?”

The computer needs to **find relevant text**.

```text
Question
   ↓
Search
   ↓
Relevant document passages
   ↓
LLM
   ↓
Answer
```

That distinction is the foundation of everything.

---

# 2. Now put Snowflake around this

Your simplified architecture becomes:

```text
                         IMAGERIGHT
                             │
                             ▼
                       AZURE BLOB
                      Raw documents
                             │
                             ▼
                      DOCUMENT PROCESSING
                             │
                  ┌──────────┴──────────┐
                  ▼                     ▼
             STRUCTURED            DOCUMENT TEXT
                DATA                    +
                  │                  METADATA
                  │                     │
                  ▼                     ▼
             SNOWFLAKE             SNOWFLAKE
                  │                     │
                  ▼                     ▼
        SEMANTIC VIEW            CORTEX SEARCH
                  │                     │
                  │                     ▼
                  │                 RAG
                  │                     │
                  └──────────┬──────────┘
                             ▼
                       CORTEX AGENT
                             │
                             ▼
                           USER
```

This is the architecture I want you to understand.

Not:

> “Cortex = AI magic.”

Instead:

> **Snowflake stores and governs the data. Cortex components give AI different abilities over that data.**

---

# 3. Where is the actual document stored?

This is one of the first things that causes confusion.

Suppose ImageRight contains:

```text
Claim 84729
 ├── FNOL.pdf
 ├── PoliceReport.pdf
 ├── Estimate.pdf
 ├── AdjusterNotes.pdf
 └── DamagePhoto01.jpg
```

After migration, you might have:

```text
Azure Blob Storage
│
└── ImageRight/
     └── claims/
          └── 84729/
               ├── FNOL.pdf
               ├── PoliceReport.pdf
               ├── Estimate.pdf
               ├── AdjusterNotes.pdf
               └── DamagePhoto01.jpg
```

Azure Blob is your **raw object/document storage**.

Snowflake can access cloud files through stages. Snowflake's unstructured-data model supports external or internal stages, and directory tables can expose file-level metadata about staged files. ([Snowflake Documentation][2])

So think:

```text
Azure Blob
     =
actual original file
```

while:

```text
Snowflake
     =
data + metadata + extracted content + search/analytics layer
```

You do **not** want to think of the PDF as becoming magically “a row in a Snowflake table.”

---

# 4. Then what does Snowflake actually know about that file?

You create an external stage pointing to the Azure location.

Conceptually:

```text
Azure Blob
     │
     ▼
Snowflake External Stage
     │
     ▼
Directory Metadata
```

The directory information can include things such as:

```text
relative_path
file_name
size
last_modified
file URL
```

Snowflake directory tables are specifically designed to expose metadata about files on stages. ([Snowflake Documentation][3])

So you might have something conceptually like:

```text
DOCUMENT_FILES

file_id
claim_id
document_id
file_name
file_path
document_type
last_modified
file_size
```

The important distinction is:

```text
DOCUMENT_FILES
       ↓
information ABOUT the file
```

versus:

```text
Azure Blob
       ↓
the actual PDF/image
```

---

# 5. Now we have to turn documents into something AI can understand

This is where your ImageRight use case becomes interesting.

A PDF may contain:

```text
PDF
├── selectable text
├── scanned text
├── tables
├── headers
├── paragraphs
├── images
├── signatures
└── mixed layouts
```

Snowflake currently provides `AI_PARSE_DOCUMENT` for document extraction. It supports OCR and layout-aware extraction, including tables, reading order, and optionally embedded images. Snowflake recommends LAYOUT for complex documents and OCR for straightforward scanned/text-heavy extraction. ([Snowflake Documentation][4])

For example:

```text
PoliceReport.pdf
        │
        ▼
AI_PARSE_DOCUMENT
        │
        ▼
{
   page 1:
      "The accident occurred..."
   
   page 2:
      "The driver stated..."
   
   page 3:
      table...
}
```

And importantly, `AI_PARSE_DOCUMENT` can return page-separated results and, in layout mode, preserve structural elements such as tables. ([Snowflake Documentation][4])

---

# 6. Why parsing matters so much

Suppose your PDF looks like this:

```text
Claim Estimate

Parts                  $5,000
Labor                  $3,000
Paint                  $1,500
------------------------------
Total                  $9,500
```

A weak extraction system might produce:

```text
Parts 5000 Labor 3000 Paint 1500 Total 9500
```

But a layout-aware parser can preserve the relationships more usefully.

The difference matters because later the system needs to understand:

```text
Total = 9500
```

rather than merely seeing four unrelated numbers.

That's why I'd teach **document processing before RAG**.

---

# 7. Now we create the document knowledge layer

After parsing, you should not throw everything into one giant text field.

Think in layers:

```text
CLAIM
   │
   ├── DOCUMENT
   │      │
   │      ├── PAGE
   │      │    │
   │      │    └── CONTENT
   │      │
   │      └── METADATA
   │
   └── OTHER CLAIM DATA
```

For your project I'd conceptually have something like:

```text
CLAIMS
DOCUMENTS
DOCUMENT_PAGES
DOCUMENT_CHUNKS
DOCUMENT_METADATA
```

For example:

```text
DOCUMENT_CHUNKS

chunk_id
claim_id
document_id
document_type
page_number
section
chunk_text
source_path
document_date
```

Now a chunk might be:

```text
chunk_id:
CH_90021

claim_id:
CLM_84729

document_id:
DOC_0021

document_type:
POLICE_REPORT

page_number:
4

chunk_text:
"The insured reported that the collision..."
```

This is where your document becomes **searchable knowledge**.

---

# 8. Now we need to understand embeddings

This is probably the most important RAG concept.

Suppose we have:

```text
"The vehicle collided with another car
at an intersection."
```

An embedding model converts that text into a numerical vector.

Very simplified:

```text
Text
 ↓
Embedding model
 ↓
[0.12, -0.82, 0.41, 0.09, ...]
```

The actual vector has many dimensions.

The important idea is not the numbers themselves.

The important idea is:

> **Texts with similar meaning tend to produce vectors that are closer together in vector space.**

So:

```text
"vehicle collided with another car"

and

"the insured was involved in a collision"
```

can be considered semantically related even though they don't use exactly the same words.

That is what makes vector search powerful.

---

# 9. But vector search alone is not enough

Suppose someone asks:

> “Find claim CLM-84729.”

Vector similarity isn't necessarily what you want.

You want an exact identifier.

Or:

> “Show document type POLICE_REPORT.”

Again, exact lexical matching can be more useful.

That's why Cortex Search uses **hybrid retrieval**.

Snowflake describes Cortex Search as combining:

```text
1. Vector search
2. Keyword search
3. Semantic reranking
```

The vector side finds semantic similarity, keyword search finds lexical matches, and reranking improves the ordering of the most relevant results. ([Snowflake Documentation][5])

This is a very important point:

> **Cortex Search is not simply “a vector database.”**

It is a managed search layer.

---

# 10. So what exactly is Cortex Search?

Think of a Cortex Search Service as:

> **A managed search engine built over a Snowflake source query.**

Conceptually:

```text
SNOWFLAKE TABLE / QUERY
        │
        ▼
CORTEX SEARCH SERVICE
        │
        ├── searchable text
        ├── metadata / attributes
        ├── indexes
        └── retrieval/ranking
```

When you create the service, you tell Snowflake what source data to search.

For example:

```text
SELECT
    chunk_id,
    chunk_text,
    claim_id,
    document_type,
    page_number
FROM document_chunks
```

Then you designate the searchable content and useful attributes.

Cortex Search supports attribute columns that can later be used for filters. ([Snowflake Documentation][6])

---

# 11. What happens when Cortex Search is created?

Conceptually:

```text
DOCUMENT_CHUNKS
      │
      ▼
Source query
      │
      ▼
Search indexing pipeline
      │
      ├── lexical index
      │
      ├── vector representation
      │
      └── ranking structures
```

Snowflake manages the underlying search infrastructure, embedding/indexing process, and refresh mechanisms for the standard managed setup. ([Snowflake Documentation][5])

You therefore don't have to manually build:

```text
OpenSearch
Pinecone
FAISS
Milvus
custom embedding workers
custom index refresh jobs
```

just to get the core retrieval layer.

That is one of the reasons Cortex Search fits your Snowflake-centered architecture.

---

# 12. What if a document changes?

Suppose:

```text
Estimate.pdf
```

changes.

Your underlying Snowflake source data changes.

Cortex Search has a refresh mechanism driven by the service's `TARGET_LAG`.

Conceptually:

```text
SOURCE DATA
    │
    │ changed
    ▼
CORTEX SEARCH REFRESH
    │
    ▼
INDEX UPDATED
```

`TARGET_LAG` defines the maximum desired lag between source changes and what the search service serves. Snowflake also has optimized refresh behavior for services with primary keys. ([Snowflake Documentation][6])

So:

```text
New claim document
      ↓
Parsed
      ↓
Chunk table updated
      ↓
Search service refreshes
      ↓
New document becomes searchable
```

That's the real lifecycle.

---

# 13. Now the most important distinction: Search vs RAG

People often say:

> “Cortex Search is RAG.”

Not exactly.

### Search

Search answers:

> **Which pieces of information are relevant?**

Example:

```text
Question:
"What caused claim 84729?"

        ↓

Search
        ↓

Top results:
1. Police report — page 4
2. FNOL — page 2
3. Adjuster note — page 1
```

That is retrieval.

### RAG

RAG adds generation:

```text
Question
    ↓
Retrieve relevant chunks
    ↓
Give those chunks to an LLM
    ↓
Generate answer
```

So:

```text
SEARCH
 =
retrieve evidence

RAG
 =
retrieve evidence
+
generate grounded answer
```

Snowflake explicitly describes Cortex Search with Cortex Agents as providing the retrieval layer, with retrieved context passed to the LLM for grounded response generation. ([Snowflake Documentation][7])

---

# 14. Let's follow one real question

Suppose the adjuster asks:

> **“What caused the accident in claim 84729?”**

This is what happens conceptually.

```text
USER
 │
 ▼
QUESTION
 │
 ▼
CORTEX AGENT
 │
 ▼
CORTEX SEARCH
 │
 ▼
Retrieve relevant chunks
 │
 ├── Police report page 4
 ├── FNOL page 2
 └── Adjuster note page 1
 │
 ▼
LLM
 │
 ▼
Grounded answer
 │
 ▼
Citation/source
```

The final response could conceptually be:

> The police report states that the accident occurred when the insured vehicle entered the intersection and collided with another vehicle.

And then:

```text
Source:
PoliceReport.pdf
Page 4
```

Cortex Agent responses can contain Cortex Search citation annotations that identify the document and text excerpt used as a citation. ([Snowflake Documentation][8])

---

# 15. Now we get to Cortex Analyst

This solves a completely different problem.

Suppose you ask:

> **“How many claims were opened in Maharashtra last year?”**

There may be nothing useful to retrieve from PDFs.

The answer is in structured tables.

```text
CLAIMS
--------------------------------
claim_id
state
opened_date
claim_amount
status
policy_id
```

You want:

```sql
SELECT COUNT(*)
FROM claims
WHERE state = 'Maharashtra'
AND opened_date ...
```

But the user doesn't want to write SQL.

So:

```text
Natural language
      ↓
Cortex Analyst
      ↓
SQL
      ↓
Snowflake
      ↓
Result
```

---

# 16. But how does Analyst know your business?

This is where **Semantic Views** become critical.

A semantic view is not another copy of your claim data.

Think of it as a **business dictionary + relationship map + calculation layer** over your physical data.

Snowflake semantic views can define:

```text
Logical tables
Dimensions
Facts
Metrics
Relationships
Descriptions
Synonyms
Verified queries
Instructions
```

([Snowflake Documentation][9])

---

# 17. Example semantic model for your claims

Suppose the physical table says:

```text
CLAIM_AMT
```

But the business calls it:

```text
Claim Amount
```

A semantic definition can tell the AI:

```text
claim_amount
=
physical_table.claim_amt
```

Suppose:

```text
LOSS_DT
```

means:

```text
date the insured loss occurred
```

The semantic layer can provide that business meaning.

Then:

```text
"claims from last quarter"
```

can map correctly to the relevant date concept.

---

# 18. Dimensions, facts and metrics

You need to deeply understand these.

## Dimension

A dimension answers:

> **By what category do I want to look at the data?**

Examples:

```text
State
Claim Type
Policy Type
Claim Status
Loss Date
Adjuster
```

---

## Fact

A fact is a row-level quantitative concept.

For example:

```text
claim_amount
repair_cost
reserve_amount
```

---

## Metric

A metric is a business calculation.

For example:

```text
Total Claim Amount
=
SUM(claim_amount)
```

or:

```text
Average Claim Amount
=
AVG(claim_amount)
```

Snowflake's semantic-view model explicitly distinguishes dimensions, facts and metrics, with metrics representing aggregated business KPIs. ([Snowflake Documentation][9])

---

# 19. Why the semantic layer is so important

Imagine the database has:

```text
premium_amount
claim_amount
reserve_amount
paid_amount
```

User asks:

> “What is the total claim cost?”

A generic LLM might guess which column you mean.

That is dangerous.

Semantic definitions can establish exactly what your organization means by a metric.

So instead of:

```text
LLM guessing database meaning
```

you get:

```text
Business definition
        ↓
Semantic View
        ↓
Cortex Analyst
        ↓
SQL
```

This is one of the biggest differences between an enterprise analytics system and “give an LLM database access.”

---

# 20. Analyst therefore works roughly like this

User:

> “Show me the number of automobile claims by state.”

Conceptually:

```text
Question
   ↓
Understand business concepts
   ↓
Find relevant semantic entities
   ↓
Determine dimensions/metrics
   ↓
Construct SQL
   ↓
Execute SQL
   ↓
Return result
```

The semantic view tells the system:

```text
Claim
State
Claim Type
Claim Count
```

and how those concepts map to your physical Snowflake data. ([Snowflake Documentation][9])

---

# 21. Verified Queries

This is another important piece.

Suppose your business team says:

> “When we say active claims, use this exact definition.”

You can have a verified query representing the intended question and SQL answer.

Snowflake's current semantic-view capabilities support verified queries specifically to improve Cortex Analyst accuracy and trustworthiness. ([Snowflake Documentation][10])

Think of them as:

```text
Business question
        +
approved SQL answer
        =
gold example
```

So over time:

```text
User questions
       ↓
identify recurring patterns
       ↓
verify important questions
       ↓
improve semantic layer
       ↓
better future answers
```

---

# 22. Now Cortex Agent

This is where everything comes together.

Cortex Agent is the **orchestrator**.

Snowflake describes an Agent as a managed agentic platform that can plan work, use tools, execute code where enabled, and generate a response while operating within Snowflake's governed environment. Its tools include Cortex Analyst and Cortex Search. ([Snowflake Documentation][11])

Think:

```text
                  USER QUESTION
                        │
                        ▼
                  CORTEX AGENT
                        │
             ┌──────────┼──────────┐
             │          │          │
             ▼          ▼          ▼
        SEARCH       ANALYST    CODE/OTHER
             │          │
             ▼          ▼
        Documents    SQL/Data
             │          │
             └────┬─────┘
                  ▼
               ANSWER
```

---

# 23. Why an Agent is necessary

Suppose the user asks:

> **“For claim 84729, what was the final repair cost and what does the adjuster say caused the delay?”**

Now you need both types of data.

First:

```text
"What was the final repair cost?"
```

Structured.

Potentially:

```text
Cortex Analyst
      ↓
SQL
      ↓
claim/payment tables
```

Second:

```text
"What does the adjuster say caused the delay?"
```

Unstructured.

```text
Cortex Search
      ↓
Adjuster Notes
      ↓
Relevant passages
```

Then the Agent combines those results.

This is exactly the sort of structured + unstructured workflow that Cortex Agents is designed to orchestrate. ([Snowflake Documentation][11])

---

# 24. The Agent's job is not to replace Search or Analyst

This distinction is crucial.

Think:

```text
Cortex Search
=
"I can find relevant text."

Cortex Analyst
=
"I can answer structured analytical questions."

Cortex Agent
=
"I can decide which capability is needed
and combine the results."
```

That is the clean mental model.

---

# 25. Now let's look at a much harder question

User asks:

> “Why did claim 84729 cost more than average claims in Maharashtra last year?”

Now the Agent might need:

```text
                  QUESTION
                      │
                      ▼
                 AGENT
                  /   \
                 /     \
                ▼       ▼
          ANALYST     SEARCH
             │            │
             ▼            ▼
        Calculate       Read claim
        benchmark       documents
             │            │
             └─────┬──────┘
                   ▼
                 AGENT
                   │
                   ▼
                ANSWER
```

Analyst could determine:

```text
Claim 84729:
₹15,00,000

Maharashtra average:
₹9,80,000
```

Search could find:

```text
Major structural damage
Additional repair discovered
Parts delay
Supplemental estimate
```

Then the answer becomes:

> Claim 84729 was approximately 53% above the Maharashtra average. The documents indicate that the higher cost was associated with additional structural damage discovered after the initial estimate and subsequent supplemental repair work.

That is far more useful than either Search or Analyst by itself.

---

# 26. Now understand the RAG pipeline properly

A real RAG pipeline is:

```text
DOCUMENT
   ↓
PARSE
   ↓
CLEAN
   ↓
CHUNK
   ↓
METADATA
   ↓
SEARCH INDEX
   ↓
USER QUESTION
   ↓
RETRIEVAL
   ↓
TOP RELEVANT CONTENT
   ↓
LLM
   ↓
GROUNDED RESPONSE
```

There are three fundamentally different systems here:

```text
DATA PREPARATION
        +
RETRIEVAL
        +
GENERATION
```

Do not mentally combine them.

---

# 27. Data preparation

This is your ImageRight side.

```text
ImageRight
   ↓
Azure Blob
   ↓
Parse
   ↓
Extract
   ↓
Normalize
   ↓
Chunk
   ↓
Metadata
```

This determines the quality of your knowledge base.

---

# 28. Retrieval

This is Cortex Search.

```text
Question
   ↓
Search
   ↓
Relevant passages
```

This determines whether the LLM receives the **right evidence**.

---

# 29. Generation

This is the LLM reasoning/generation stage.

```text
Question
+
Retrieved evidence
        ↓
       LLM
        ↓
Answer
```

This determines how well the answer is synthesized.

So if an answer is wrong:

```text
Could be ingestion
Could be parsing
Could be chunking
Could be retrieval
Could be filtering
Could be ranking
Could be generation
```

That is why production RAG requires evaluation at multiple levels.

---

# 30. The hidden-but-critical part: metadata

For your use case, metadata is almost as important as the actual text.

Imagine two chunks:

```text
Chunk A
"Final amount was $17,500"

Claim:
84729

Document:
Estimate

Date:
2025-09-10
```

and:

```text
Chunk B
"Final amount was $17,500"

Claim:
99214

Document:
Estimate

Date:
2024-01-10
```

Without metadata, the search system may find both.

But if the user asks:

> “For claim 84729…”

you want:

```text
claim_id = 84729
```

as a filter.

Cortex Search supports attribute-based filtering, including equality, containment, ranges and logical operators. ([Snowflake Documentation][12])

This is why I would strongly emphasize:

> **Metadata is not decoration. Metadata controls retrieval quality, security, scope, and relevance.**

---

# 31. Metadata could look like this

```text
claim_id
policy_id
document_id
document_type
document_date
source_system
page_number
customer_id
state
adjuster_id
language
processing_status
```

Then a search can conceptually become:

```text
Query:
"cause of accident"

Filter:
claim_id = 84729

Search:
Police Report
+
FNOL
+
Adjuster Notes
```

That is dramatically safer than:

```text
Search every document in the company.
```

---

# 32. Search does not mean "give the whole document to the LLM"

This is another major misconception.

Suppose you have:

```text
10 million documents
```

You don't send 10 million documents to the LLM.

You do:

```text
10M documents
      ↓
Search
      ↓
20 relevant passages
      ↓
Maybe rerank
      ↓
Top few passages
      ↓
LLM
```

This is what keeps RAG practical.

---

# 33. Why chunking matters

Imagine this 50-page document:

```text
Page 1     Claim details
Page 2     Incident summary
Page 3     Witness
...
Page 34    Repair estimate
...
Page 49    Correspondence
Page 50    Final settlement
```

User asks:

> “What was the final settlement?”

You don't want retrieval to return the entire 50-page document.

You want something like:

```text
Chunk 981:
Page 50
Final settlement information...
```

That makes the LLM's context much cleaner.

But you also don't want chunks so tiny that context gets destroyed.

So I'd teach you **semantic/structural chunking**, not “split every 500 tokens.”

---

# 34. Now understand the difference between document storage and search storage

This is extremely important.

You might have:

```text
AZURE BLOB
   ↓
Original PDF
```

Then:

```text
SNOWFLAKE
   ↓
Metadata
   ↓
Parsed content
   ↓
Chunks
```

Then:

```text
CORTEX SEARCH
   ↓
Search/index representation
```

These are three different things.

```text
             SOURCE
               │
               ▼
          Azure Blob
               │
               ▼
       Parsed Snowflake data
               │
               ▼
       Cortex Search index
```

Do not confuse:

> **where the original document lives**

with:

> **where the searchable representation lives.**

---

# 35. Can Snowflake work directly with Azure documents?

Yes.

Snowflake supports external stages for cloud storage, and `AI_PARSE_DOCUMENT` can process documents stored on stages. Snowflake also supports file objects and scoped URLs for controlled access to staged files. ([Snowflake Documentation][4])

That means your architecture does not necessarily need:

```text
Azure Blob
 ↓
download everything
 ↓
copy everything somewhere else
 ↓
parse
```

You can design a more direct flow around the stage.

For your architecture, I would evaluate:

```text
ImageRight
   ↓
Azure Blob
   ↓
Snowflake external stage
   ↓
AI_PARSE_DOCUMENT
   ↓
Snowflake
```

rather than automatically duplicating all raw documents again inside Snowflake.

---

# 36. Now the biggest mental model: three different "brains"

I want you to remember this:

### Cortex Search

**Brain for finding text.**

```text
"What information is relevant?"
```

### Cortex Analyst

**Brain for querying structured data.**

```text
"What numbers/calculations does the data say?"
```

### Cortex Agent

**Brain for coordinating capabilities.**

```text
"Which tool should I use,
in what order,
and how should I combine the results?"
```

This is the cleanest mental model.

---

# 37. And one more layer: the LLM

The LLM is not your database.

It is not your source of truth.

Think:

```text
Snowflake
   =
truth/data

Cortex Search
   =
retrieval

Cortex Analyst
   =
structured-data interpretation + SQL generation

Cortex Agent
   =
orchestration

LLM
   =
language reasoning/generation
```

That distinction will save you from a lot of bad RAG architecture.

---

# 38. What happens when the user asks a question?

Let's trace everything.

User:

> “Why was claim 84729's repair cost higher than normal?”

### Step 1

Agent receives request.

```text
USER
 ↓
AGENT
```

### Step 2

Agent determines it needs both structured and unstructured information.

```text
AGENT
 ├── Analyst
 └── Search
```

### Step 3

Analyst handles the quantitative side.

```text
"repair cost for 84729"
"normal cost"
```

It uses the semantic layer to generate/query SQL.

```text
Semantic View
     ↓
SQL
     ↓
Snowflake
```

### Step 4

Search handles the explanatory side.

```text
"why was the repair expensive?"
```

Cortex Search retrieves relevant claim documents.

### Step 5

Agent receives both.

```text
Structured evidence
+
Document evidence
```

### Step 6

The model produces the final answer.

### Step 7

Relevant document evidence can be cited.

That is the complete flow.

---

# 39. Now understand where each piece lives

This table is worth memorizing.

| Component                 | What it stores/does                                     |
| ------------------------- | ------------------------------------------------------- |
| **ImageRight**            | Original legacy document system                         |
| **Azure Blob**            | Raw migrated document files                             |
| **Snowflake stage**       | Secure reference/access layer for staged files          |
| **Snowflake tables**      | Metadata, parsed text, chunks, structured business data |
| **Semantic View**         | Business meaning and relationships                      |
| **Cortex Search Service** | Search/index representation over searchable source data |
| **Cortex Search**         | Retrieves relevant unstructured information             |
| **Cortex Analyst**        | Generates/executes SQL for structured questions         |
| **Cortex Agent**          | Orchestrates tools and combines results                 |
| **LLM**                   | Generates the final natural-language response           |

---

# 40. What I would specifically build for your ImageRight system

I would not make one gigantic table called:

```text
documents
```

and throw everything into it.

I'd design something closer to:

```text
CLAIMS
POLICIES
DOCUMENTS
DOCUMENT_PAGES
DOCUMENT_CHUNKS
DOCUMENT_METADATA
PROCESSING_RUNS
```

And potentially:

```text
DOCUMENT_IMAGES
TABLE_EXTRACTIONS
```

Then:

```text
                 DOCUMENTS
                     │
                     ▼
              DOCUMENT_PAGES
                     │
                     ▼
              DOCUMENT_CHUNKS
                     │
                     ▼
             CORTEX SEARCH
```

While:

```text
CLAIMS
POLICIES
PAYMENTS
RESERVES
LOSS_EVENTS
```

feed:

```text
SEMANTIC VIEW
      │
      ▼
CORTEX ANALYST
```

Then:

```text
Search + Analyst
       ↓
Cortex Agent
```

---

# 41. Why I would not start by building the Agent

This is important.

A beginner usually thinks:

```text
Let's build an Agent first.
```

I would do the opposite.

First:

```text
Source quality
 ↓
Document parsing
 ↓
Data model
 ↓
Search quality
 ↓
Semantic model
 ↓
Evaluation
 ↓
Agent
```

Because an Agent sitting on bad data simply makes bad results easier to access.

---

# 42. Your final architecture should look mentally like this

```text
                    ┌──────────────────┐
                    │    IMAGERIGHT    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    AZURE BLOB    │
                    │   RAW DOCUMENTS  │
                    └────────┬─────────┘
                             │
                             ▼
                ┌─────────────────────────┐
                │  DOCUMENT PROCESSING    │
                │                         │
                │  OCR / Layout / Images  │
                └────────────┬────────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    SNOWFLAKE     │
                    │                  │
                    │ structured data  │
                    │ metadata         │
                    │ parsed content   │
                    │ chunks           │
                    └────────┬─────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
      ┌──────────────────┐       ┌──────────────────┐
      │  SEMANTIC VIEW   │       │ CORTEX SEARCH    │
      │                  │       │                  │
      │ metrics          │       │ keyword search   │
      │ dimensions       │       │ vector search    │
      │ relationships    │       │ reranking        │
      │ business rules   │       │ filters          │
      └────────┬─────────┘       └────────┬─────────┘
               │                          │
               ▼                          ▼
      ┌──────────────────┐       ┌──────────────────┐
      │ CORTEX ANALYST   │       │       RAG        │
      │                  │       │                  │
      │ natural language │       │ retrieve context │
      │ → SQL            │       │ → LLM            │
      └────────┬─────────┘       └────────┬─────────┘
               │                          │
               └────────────┬─────────────┘
                            ▼
                  ┌───────────────────┐
                  │   CORTEX AGENT    │
                  │                   │
                  │ orchestration     │
                  │ tool selection    │
                  │ multi-step work   │
                  │ response          │
                  └─────────┬─────────┘
                            │
                            ▼
                          USER
```

---

# 43. The five questions you should always ask

Whenever somebody explains one of these technologies to you, ask:

### Search

> **What exactly is being indexed?**

Not “Snowflake data.”

Ask:

> Which column? Which rows? Which metadata? Which chunks?

---

### RAG

> **What exact text is being given to the LLM?**

Not “the documents.”

Ask:

> Which chunks? How many? From which documents/pages?

---

### Analyst

> **What physical tables does the semantic view map to?**

Not “the database.”

Ask:

> Which metric? Which dimensions? Which relationships? Which SQL?

---

### Agent

> **Which tool is being called for this question, and why?**

Not “the agent handles it.”

Ask:

> Search? Analyst? Both? In what sequence?

---

### Security

> **Whose data is actually searchable?**

This is one of the most important questions in enterprise RAG.

---

# 44. And I would teach you the failure modes too

Because knowing the happy path isn't enough.

### Wrong answer from RAG

Could be:

```text
Bad extraction
      OR
Bad chunking
      OR
Bad metadata
      OR
Bad retrieval
      OR
Bad filtering
      OR
Bad generation
```

### Wrong SQL answer

Could be:

```text
Bad semantic definition
      OR
Wrong relationship
      OR
Wrong metric
      OR
Ambiguous terminology
      OR
Missing verified query
```

### Wrong Agent behavior

Could be:

```text
Wrong tool selection
      OR
Poor tool descriptions
      OR
Weak orchestration instructions
      OR
Insufficient metadata
```

This is why Snowflake's current guidance puts significant emphasis on semantic-view quality, verified queries, modeling, and evaluation—not just turning the feature on. ([Snowflake Documentation][13])

---

# 45. One particularly important current capability

Cortex Search has also evolved beyond the old simplistic:

```text
one text column
+
one embedding
```

Current Cortex Search supports **multi-index search** and **custom/user-provided embeddings**, and those capabilities became generally available in 2026. ([Snowflake Documentation][14])

That means later, for your ImageRight system, we can consider designs where:

```text
DOCUMENT_TITLE
    → keyword index

CLAIM_ID
    → exact/indexed field

DOCUMENT_TEXT
    → semantic/vector index
```

rather than treating every field identically.

That's particularly useful for insurance data because:

```text
CLM-84729
```

needs different search behavior from:

```text
"The insured entered the intersection..."
```

---

# 46. Another advanced distinction: normal RAG vs corpus-wide analysis

Suppose you ask:

> “What did the adjuster say about claim 84729?”

Normal retrieval/RAG is perfect.

But now suppose you ask:

> “What are the most common reasons for claim delays across 500,000 claims?”

That's a different problem.

Retrieving five chunks is not enough.

Snowflake now has an **Analytical Search** capability in Cortex Agents aimed at large-document-collection analysis involving counts, aggregates, trends and broader corpus analysis. ([Snowflake Documentation][15])

So your architecture eventually has three kinds of questions:

```text
1. Structured analytics
   ↓
   Analyst

2. Find/answer from specific documents
   ↓
   Search + RAG

3. Analyze large document collections
   ↓
   Analytical Search / Agent
```

That distinction is very useful for your project.

---

# 47. What I want you to be able to explain after learning this

Someone should be able to ask you:

> “What is Cortex Search?”

And you should be able to say:

> Cortex Search is Snowflake's managed hybrid retrieval layer over Snowflake data. It combines semantic/vector retrieval, keyword retrieval and semantic reranking, supports metadata filtering, and automatically refreshes its searchable content from its source query. ([Snowflake Documentation][5])

Then:

> “What is Cortex Analyst?”

You should say:

> Cortex Analyst handles natural-language questions over structured data. A semantic view gives it business concepts such as dimensions, metrics, facts and relationships so it can generate SQL against the underlying Snowflake data. Snowflake is currently recommending Cortex Agents as the higher-level interface that uses these Analyst capabilities. ([Snowflake Documentation][9])

Then:

> “What is RAG?”

You should say:

> RAG retrieves relevant evidence from a knowledge base and passes that evidence to an LLM so the answer is grounded in the retrieved information rather than relying only on the model's pretraining.

Then:

> “What is Cortex Agent?”

You should say:

> Cortex Agent is the orchestration layer that can use tools such as Cortex Search and Cortex Analyst to answer multi-step questions over unstructured and structured data inside Snowflake's governed environment. ([Snowflake Documentation][11])

And most importantly:

> “Where is the original ImageRight document?”

You should answer:

```text
Azure Blob
```

> “Where is its searchable extracted representation?”

```text
Snowflake tables
      +
Cortex Search index
```

> “Where is the business meaning of claim amount, claim state, claim count, etc.?”

```text
Semantic View
```

> “Who combines all of that?”

```text
Cortex Agent
```

That is the mental model I want you to have.

---

## The way I would take you from here

I would now teach this as a **proper hands-on masterclass in sequence**, using your ImageRight claim example throughout:

**Lesson 1:** What exactly gets stored after ImageRight → Azure → Snowflake, with real table designs.

**Lesson 2:** `AI_PARSE_DOCUMENT` deeply — OCR vs LAYOUT, pages, tables, images, extraction JSON, and how we turn a PDF into usable records.

**Lesson 3:** Chunking deeply — what a chunk really is, how to choose chunk boundaries, overlap, page/section metadata, and how bad chunking ruins RAG.

**Lesson 4:** Embeddings from first principles — vectors, similarity, semantic search, keyword search, hybrid retrieval, reranking.

**Lesson 5:** Cortex Search internally — source query → indexing → attributes → filters → refresh → querying → result structure → citations.

**Lesson 6:** Cortex Analyst from first principles — physical tables → semantic view → dimensions/facts/metrics → relationships → SQL generation → verified queries.

**Lesson 7:** Cortex Agents — how Search and Analyst become tools, how orchestration works, how the Agent handles a mixed question, and where Agent instructions fit.

**Lesson 8:** Build the complete ImageRight RAG architecture and walk one claim through it from the original PDF all the way to the final cited answer.

That sequence is much better than trying to memorize Snowflake feature names, because by the end you will understand **the data flow, the AI flow, and the reason for every component**.

[1]: https://docs.snowflake.com/en/release-notes/2026/other/2026-08-28-cortex-analyst-transition-cortex-agents?utm_source=chatgpt.com "Aug 28, 2026: Snowflake recommends transitioning from Cortex Analyst to Cortex Agents | Snowflake Documentation"
[2]: https://docs.snowflake.com/en/user-guide/unstructured-intro?utm_source=chatgpt.com "Introduction to unstructured data | Snowflake Documentation"
[3]: https://docs.snowflake.com/en/en/user-guide/data-load-dirtables?utm_source=chatgpt.com "Directory tables | Snowflake Documentation"
[4]: https://docs.snowflake.com/en/sql-reference/functions/ai_parse_document?utm_source=chatgpt.com "AI_PARSE_DOCUMENT | Snowflake Documentation"
[5]: https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview?utm_source=chatgpt.com "Cortex Search | Snowflake Documentation"
[6]: https://docs.snowflake.com/en/sql-reference/sql/create-cortex-search?utm_source=chatgpt.com "CREATE CORTEX SEARCH SERVICE | Snowflake Documentation"
[7]: https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-agents?utm_source=chatgpt.com "Use Cortex Search with Cortex Agents | Snowflake Documentation"
[8]: https://docs.snowflake.com/en/en/user-guide/snowflake-cortex/cortex-agents-run?utm_source=chatgpt.com "Cortex Agents Run API | Snowflake Documentation"
[9]: https://docs.snowflake.com/en/user-guide/views-semantic/overview?lang=en&utm_source=chatgpt.com "Overview of semantic views | Snowflake Documentation"
[10]: https://docs.snowflake.com/en/release-notes/2026/other/2026-04-05-semantic-views-verified-queries?utm_source=chatgpt.com "Apr 05, 2026: Support for verified queries in semantic views | Snowflake Documentation"
[11]: https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents?utm_source=chatgpt.com "Cortex Agents | Snowflake Documentation"
[12]: https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/query-cortex-search-service?utm_source=chatgpt.com "Query a Cortex Search Service | Snowflake Documentation"
[13]: https://docs.snowflake.com/en/user-guide/views-semantic/best-practices?utm_source=chatgpt.com "Best practices for semantic views | Snowflake Documentation"
[14]: https://docs.snowflake.com/en/release-notes/2026/other/2026-03-12-recent-cortex-search?utm_source=chatgpt.com "Mar 12, 2026: Recent Cortex Search updates (*Generally Available*) | Snowflake Documentation"
[15]: https://docs.snowflake.com/en/release-notes/2026/other/2026-06-30-analytical-search-public-preview?utm_source=chatgpt.com "Jun 30, 2026: Analytical search (*Public Preview*) | Snowflake Documentation"
