Yes. This is the layer I would teach **before Cortex Search / Analyst / Agent**, because this is where the actual insurance information is captured, moved, interpreted, and prepared for Snowflake.

For your project, the complete learning path should really be:

```text
INSURANCE BUSINESS
      ↓
IMAGERIGHT
      ↓
EXTRACTION / MIGRATION
      ↓
AZURE NETWORK + IDENTITY
      ↓
AZURE BLOB / ADLS
      ↓
INGESTION / ORCHESTRATION
      ↓
PARSING / OCR / DOCUMENT INTELLIGENCE
      ↓
VALIDATION / NORMALIZATION
      ↓
SNOWFLAKE
      ↓
CORTEX SEARCH / ANALYST
      ↓
CORTEX AGENT / RAG
```

The important thing is that **VM, Blob, API, OCR, parsing, Event Grid, ADF, Managed Identity, etc. are not random Azure services**. Each exists because a specific problem has to be solved.

---

# 1. First: understand what the insurance data actually is

Before Azure, understand the thing we're migrating.

ImageRight is an insurance document/content-management and workflow system used across claims, underwriting and distribution. Vertafore describes it as managing correspondence, forms, attachments and metadata across policy and claims lifecycles, and its current platform exposes an open architecture and APIs for integrations. ([Vertafore][1])

So don't imagine ImageRight as:

```text
Folder
  └── PDF
```

Think:

```text
                    INSURANCE ACCOUNT
                           │
             ┌─────────────┴─────────────┐
             │                           │
          POLICY                       CLAIM
             │                           │
      ┌──────┼──────┐             ┌─────┼─────┐
      │      │      │             │     │     │
    Forms  Emails  Docs          FNOL  Reports Notes
                                  │
                                  ├── Estimates
                                  ├── Photos
                                  ├── Correspondence
                                  ├── Invoices
                                  └── Other attachments
```

And there is metadata around those documents.

For example:

```text
claim_id
policy_id
file_id
document_id
document_type
document_date
created_date
modified_date
folder
department
owner
workflow
status
```

That metadata is extremely important later.

---

# 2. Insurance terms you need to know

You don't need to become an insurance expert, but you need enough domain understanding that a data model makes sense.

## Policy

The insurance contract.

```text
Policy
 ├── Policy number
 ├── Insured
 ├── Coverage
 ├── Effective date
 ├── Expiry date
 └── Endorsements
```

## Claim

A reported loss under a policy.

```text
Claim
 ├── Claim number
 ├── Policy number
 ├── Loss date
 ├── Loss location
 ├── Claim type
 ├── Claim status
 ├── Reserve
 ├── Payments
 └── Documents
```

## FNOL

**First Notice of Loss.**

This is commonly the initial report of an incident.

A claim might therefore have:

```text
CLM-84729
    │
    └── FNOL
          ├── Date
          ├── Incident description
          ├── Parties
          └── Initial loss details
```

## Adjuster

The person handling/evaluating the claim.

## Reserve

The amount set aside as an expected claim obligation.

## Payment

Money actually paid.

## Estimate

Expected repair/replacement cost.

## Supplement

Additional estimate/cost discovered later.

For example:

```text
Initial estimate      $10,000
Supplement             $3,500
Final cost            $13,500
```

## Correspondence

Emails, letters, messages and other communications.

## Endorsement

A change/addition to a policy.

## Underwriting

Evaluating risk before or during insurance placement.

## Submission

The package of information presented for underwriting.

These distinctions eventually become dimensions, facts, metrics and document types in Snowflake.

---

# 3. Now ImageRight

The first technical question isn't:

> “How do I copy PDFs?”

It's:

> **“How do I reliably extract the complete ImageRight information model?”**

That can include:

```text
Business object
       +
metadata
       +
document relationship
       +
document binary/content
       +
version/history
       +
workflow information
```

Vertafore publicly states that ImageRight has RESTful APIs/open architecture, but the exact endpoints, permissions and export capabilities depend on the customer's ImageRight environment/version/contract, so I would **not invent a specific endpoint** until the customer's ImageRight API documentation is available. ([Vertafore][1])

That is an important real-world point.

---

# 4. The ImageRight extraction layer

There are three broad possibilities.

### A. API-based extraction

Best when the API exposes everything you need.

```text
ImageRight
    ↓
REST API
    ↓
Documents + metadata
```

### B. Export-based extraction

If ImageRight provides a supported export mechanism:

```text
ImageRight
    ↓
Export
    ↓
Files + metadata
```

### C. Connector/application-based extraction

Sometimes a legacy environment or integration requires a Windows application/connector.

Then:

```text
ImageRight
    ↓
Connector / application
    ↓
Azure
```

This is where an Azure VM can potentially become relevant.

---

# 5. What an API actually means

Suppose ImageRight exposes something conceptually like:

```text
GET /claims/84729
```

You send a request.

The system returns structured information:

```json
{
  "claimId": "84729",
  "status": "Open",
  "documents": [...]
}
```

That's **metadata**.

Then you may have a document retrieval operation:

```text
GET /documents/12345/content
```

which gives you the actual file bytes.

So:

```text
API call #1
      ↓
metadata

API call #2
      ↓
actual document
```

Don't confuse those.

---

# 6. The migration service needs a manifest

This is a very important concept.

You don't want your migration process to blindly copy files.

Create a **manifest**.

Think:

```text
MIGRATION_MANIFEST

source_system
source_file_id
source_document_id
claim_id
policy_id
source_path
target_blob_path
file_name
file_size
last_modified
checksum
migration_status
migration_run_id
error_message
```

For example:

```text
DOC-7821
CLAIM-84729
PoliceReport.pdf
12.8 MB
SHA256 = abc123...
status = COPIED
```

Now your migration is trackable.

---

# 7. Why checksum matters

Imagine the pipeline runs twice.

First run:

```text
PoliceReport.pdf
       ↓
Blob
```

Second run:

```text
PoliceReport.pdf
       ↓
Blob
```

How do you know whether it's the same content?

Use a checksum/hash.

```text
file
 ↓
SHA-256
 ↓
ABC123...
```

If the same source content produces the same hash:

```text
same hash
=
likely same bytes
```

So you can detect duplicates or changed content.

This is part of what makes the migration **idempotent**.

---

# 8. Idempotency

Very important term.

An operation is idempotent when running it again doesn't incorrectly create another copy/change.

Bad:

```text
Run 1 → DOC123
Run 2 → DOC123
Run 3 → DOC123
```

Good:

```text
Run 1 → DOC123 created
Run 2 → DOC123 unchanged → skip
Run 3 → DOC123 unchanged → skip
```

Or:

```text
source version changed
       ↓
new version processed
```

---

# 9. Now Azure

Think of Azure as several different things:

```text
Azure
│
├── Storage
├── Compute
├── Networking
├── Identity
├── Security
├── Integration
├── Monitoring
└── AI / Document Processing
```

Your project doesn't need every Azure service.

We select services according to the actual workload.

---

# 10. Azure Subscription

A **subscription** is the billing/resource boundary in Azure.

Think:

```text
Azure Tenant
   │
   ├── Subscription A
   │      ├── Storage
   │      ├── VMs
   │      └── AI services
   │
   └── Subscription B
```

---

# 11. Azure Resource Group

A Resource Group is a logical grouping of resources.

For example:

```text
rg-imageright-migration-prod
    │
    ├── Storage Account
    ├── VM / VMSS
    ├── Document Intelligence
    ├── Key Vault
    ├── Event Grid
    └── Monitoring resources
```

This makes lifecycle and management easier.

---

# 12. Azure Region

A region is a geographic Azure location.

For your project, region selection matters because of:

```text
Data residency
Latency
Service availability
Compliance
Disaster recovery
Cost
```

For insurance data, don't randomly pick a region.

The target region should be determined by the organization's residency/compliance requirements.

---

# 13. Now Azure Storage

This is where I would spend serious time.

Azure Blob Storage is object storage designed for large volumes of unstructured data such as documents, images and binary content. Azure Data Lake Storage Gen2 builds on Blob Storage and adds a hierarchical namespace and more data-lake-oriented access semantics. ([Microsoft Learn][2])

---

# 14. Storage Account

Think:

```text
Storage Account
     │
     ├── Blob Storage
     ├── containers
     └── objects/files
```

Example:

```text
stimagerightprod
```

---

# 15. Container

A container is a logical grouping of blobs.

For your project, I would not dump everything into:

```text
/documents/
```

I'd separate the lifecycle.

For example:

```text
imageright-raw
imageright-quarantine
imageright-parsed
imageright-rejected
imageright-manifests
```

Or organize by prefixes:

```text
raw/
parsed/
rejected/
manifests/
```

---

# 16. Blob

A blob is the actual stored object.

For example:

```text
raw/claims/84729/police-report.pdf
```

That's the actual PDF.

---

# 17. Data Lake Gen2

For a serious enterprise data platform, I'd strongly consider enabling **hierarchical namespace** so the storage account provides data-lake-style directory/file semantics. Microsoft describes ADLS Gen2 as Blob Storage plus hierarchical namespace, large-scale analytics support, and finer-grained ACL capabilities. ([Microsoft Learn][3])

Then:

```text
raw/
  claims/
    84729/
      documents/
        police-report.pdf
        estimate.pdf
```

behaves more like a data lake filesystem.

The critical point:

> **ADLS Gen2 is not a completely separate storage product from Blob Storage. It is Blob Storage with Data Lake capabilities enabled through hierarchical namespace.** ([Microsoft Learn][4])

---

# 18. Your Azure folder design

I'd use something like:

```text
/raw/
    /imageright/
        /claims/
            /84729/
                /documents/
                    DOC-001/
                        source.pdf

                /images/
                    IMG-001.jpg

        /policies/
            /POL-9281/
                ...

/quarantine/
    ...

/parsed/
    /claims/
        /84729/
            DOC-001/
                extraction.json
                content.md

/rejected/
    ...

/manifests/
    migration_run_2026_10_07.json
```

The important thing is **source identity and lineage**, not the exact folder names.

---

# 19. Why raw should stay raw

Don't modify:

```text
source.pdf
```

and overwrite it with:

```text
processed.pdf
```

Instead:

```text
RAW
  ↓
original.pdf

PARSED
  ↓
output.json
output.md
```

This gives you:

```text
original
    ↓
reprocess
    ↓
better parser
    ↓
new extraction
```

without going back to ImageRight.

---

# 20. Versioning, soft delete and immutability

For enterprise source data, you should consider:

```text
Blob versioning
Soft delete
Possibly immutability/WORM
Retention policies
Lifecycle policies
```

Blob soft delete allows recovery from accidental deletion/overwrite for the configured retention period. Azure also offers immutable storage for workloads that require WORM-style protection. ([Microsoft Learn][5])

But don't automatically turn on every feature.

Retention/immutability must align with the organization's legal, regulatory and records-management requirements.

---

# 21. Lifecycle management

Some documents are accessed constantly.

Others are old.

You can use storage lifecycle policies to move older data to lower-cost tiers where appropriate.

Conceptually:

```text
Hot
 ↓
Cool
 ↓
Archive
```

But be careful:

> Archive storage is cheap to store but not necessarily appropriate for content that needs frequent AI retrieval.

So RAG access patterns need to influence storage-tier decisions.

---

# 22. Now the big question: what is a VM?

An Azure VM is basically:

> **A computer you rent from Azure.**

You control its operating system and installed software.

Microsoft describes VMs as IaaS compute where you still own responsibilities such as configuring, patching and maintaining the OS/software. ([Microsoft Learn][6])

Think:

```text
Physical server
      ↓
Azure virtualization
      ↓
Your VM
      ↓
Windows/Linux
      ↓
Your migration software
```

---

# 23. Do we actually need a VM?

**Not necessarily.**

This is important.

People often say:

> “We'll use an Azure VM to migrate ImageRight.”

That's not automatically the best design.

Use a VM when you genuinely need:

```text
Custom installed software
Persistent process
Legacy connector
Windows-specific application
On-premises connectivity
Special libraries
Long-running custom extraction
```

If all you need is:

```text
API
 ↓
download file
 ↓
Blob
```

you may be better with:

```text
Azure Function
Azure Container Apps
Data Factory
or another managed compute service
```

depending on the workload.

---

# 24. When VM becomes useful in your ImageRight case

Suppose the customer's ImageRight integration requires:

```text
Windows library
+
special client
+
private corporate network
```

Then:

```text
Azure VM
   │
   ├── ImageRight connector
   ├── migration application
   └── required runtime
```

could make sense.

If ImageRight exposes a clean supported REST API, you may not need that VM at all.

Vertafore currently advertises RESTful APIs/open architecture, so I'd first investigate the supported API route before committing to VM-based extraction. ([Vertafore][1])

---

# 25. VM vocabulary you need

If we use a VM, you need to understand these terms.

### VM image

The starting operating-system template.

```text
Windows Server image
Linux image
```

### VM size

Determines:

```text
CPU
RAM
network bandwidth
```

### Managed disk

The VM's persistent storage.

Azure recommends managed disks, which Azure manages as a scalable disk resource. ([Microsoft Learn][7])

### NIC

Network Interface Card.

It connects the VM to the virtual network.

### Private IP

Used inside the private network.

### Public IP

Internet-facing address, when one is assigned.

For your migration VM, you generally want to avoid unnecessary public exposure.

### NSG

Network Security Group.

Controls network traffic allowed to/from the VM.

Azure documents NSGs as the mechanism used to manage VM network traffic. ([Microsoft Learn][6])

---

# 26. VM Scale Set

Suppose you have:

```text
100,000 documents
```

and one VM isn't enough.

You can have:

```text
VM 1 → documents 1–10k
VM 2 → documents 10k–20k
VM 3 → documents 20k–30k
...
```

A VM Scale Set manages a group of VMs and can scale the number of instances. ([Microsoft Learn][8])

But again:

> Don't introduce VMSS until actual throughput requires it.

---

# 27. Managed Identity

This is one of the most important Azure security concepts.

Bad architecture:

```text
VM
 ↓
storage_account_key = "secret..."
```

Better:

```text
VM
 ↓
Managed Identity
 ↓
Microsoft Entra ID
 ↓
Azure Storage
```

Managed identities allow Azure resources to authenticate to supported services without putting credentials in application code. ([Microsoft Learn][9])

That's exactly what I'd want for your migration workers.

---

# 28. System-assigned vs user-assigned identity

### System-assigned

Identity belongs to the resource.

```text
VM
 └── identity
```

Delete VM → identity goes away.

### User-assigned

Identity is a separate Azure resource.

```text
Managed Identity
      │
      ├── VM A
      ├── VM B
      └── Function
```

Useful when multiple resources should share the same identity/permissions.

---

# 29. Azure RBAC

RBAC = Role-Based Access Control.

Instead of:

> “This application has everything.”

You give it only what it needs.

For example:

```text
Migration identity
        ↓
Blob Storage
        ↓
Only required container/path
        ↓
Read / Write
```

That's least privilege.

---

# 30. Private Endpoint

This is another term you'll see constantly.

Normally:

```text
VM
 ↓
Internet/Public endpoint
 ↓
Storage
```

With Private Link:

```text
VM
 ↓
VNet
 ↓
Private Endpoint
 ↓
Azure Storage
```

The storage account can be accessed via a private IP inside the VNet, and public access can be restricted/disabled. Microsoft specifically recommends private endpoints plus restricted/public network access for sensitive storage where appropriate. ([Microsoft Learn][10])

For insurance documents, this becomes very important.

---

# 31. VNet

VNet = Virtual Network.

Think:

> **Your private Azure network.**

```text
VNet
│
├── subnet-app
├── subnet-processing
└── subnet-private-endpoints
```

---

# 32. Subnet

A subnet is a smaller network segment inside the VNet.

Example:

```text
VNet
 ├── ingestion-subnet
 ├── processing-subnet
 └── private-endpoint-subnet
```

---

# 33. NSG vs Firewall

Easy distinction:

```text
NSG
=
network traffic rules around subnets/NICs

Storage Firewall
=
who can reach the storage service
```

They're complementary, not interchangeable.

---

# 34. Key Vault

Never put:

```text
API key
password
client secret
certificate
```

inside:

```text
Python code
VM config
GitHub
```

Use:

```text
Azure Key Vault
```

and ideally managed identities to access it.

For extremely sensitive environments, customer-managed storage encryption keys can also be stored in Key Vault or Managed HSM. Azure Storage is encrypted at rest by default, with Microsoft-managed keys unless customer-managed keys are configured. ([Microsoft Learn][11])

---

# 35. Now ingestion/orchestration

You need something coordinating:

```text
Get ImageRight data
        ↓
Download
        ↓
Upload Blob
        ↓
Record manifest
        ↓
Trigger parsing
        ↓
Validate
        ↓
Load Snowflake
```

This is where services such as:

```text
Azure Data Factory
Azure Functions
Azure Container Apps
Event Grid
Service Bus
VM / VMSS
```

may appear.

They have different jobs.

---

# 36. Azure Data Factory

Think:

> **Data movement + pipeline orchestration.**

Example:

```text
ADF Pipeline
   │
   ├── Read source
   ├── Copy data
   ├── Check status
   ├── Call processing
   └── Load destination
```

ADF supports incremental-loading patterns such as loading only files newer than a `LastModifiedDate` watermark. ([Microsoft Learn][12])

Important:

> ADF does not magically understand ImageRight.

If ImageRight has a supported connector/API, you configure it. Otherwise, you build a custom API/HTTP/extraction mechanism.

---

# 37. Azure Function

Think:

> **Small piece of event-driven code.**

For example:

```text
BlobCreated
     ↓
Function
     ↓
create parsing job
```

Good for short, focused operations.

---

# 38. Event Grid

Event Grid tells you:

> **Something happened.**

For example:

```text
BlobCreated
BlobDeleted
BlobRenamed/changed events depending on operation
```

Azure Blob Storage can emit BlobCreated events through Event Grid. ([Microsoft Learn][13])

So:

```text
New document enters Blob
        ↓
Event Grid
        ↓
Function / Service Bus
        ↓
Parsing
```

That allows an event-driven pipeline instead of constant polling.

---

# 39. Service Bus

Think:

> **A reliable queue between steps.**

Example:

```text
ImageRight ingestion
       ↓
Service Bus Queue
       ↓
Parsing workers
```

Why?

Because maybe:

```text
100,000 documents
```

arrive.

You don't want everything trying to parse simultaneously.

The queue allows controlled processing.

---

# 40. The difference

Memorize this:

```text
ADF
=
"Run this workflow."

Event Grid
=
"Something happened."

Service Bus
=
"Here is work you need to process."

Function
=
"Run this code."

VM
=
"Give me a computer."

Blob
=
"Store this file."
```

That makes Azure architecture much easier.

---

# 41. Now the most important part: Parsing

This is where your ImageRight migration becomes a **document intelligence** problem.

Suppose we receive:

```text
PoliceReport.pdf
```

It could be:

```text
Born-digital PDF
Scanned PDF
Image-only PDF
Mixed PDF
```

---

# 42. OCR

OCR = Optical Character Recognition.

It converts:

```text
image pixels
```

into:

```text
machine-readable text
```

For example:

```text
[scanned page]
     ↓
OCR
     ↓
"Vehicle collided..."
```

But OCR alone is not enough.

---

# 43. Layout analysis

You also want to know:

```text
Where is the text?
Which text belongs together?
Where is the table?
Which page?
Which heading?
```

Azure Document Intelligence's layout model can extract pages, paragraphs, text/words, tables, selection marks, figures and sections. ([Microsoft Learn][14])

So instead of:

```text
random text
```

you get something more like:

```text
Page 4
   │
   ├── Heading
   ├── Paragraph
   ├── Table
   │    ├── Row
   │    ├── Cell
   │    └── Cell
   └── Figure
```

---

# 44. Reading order

Imagine a document with two columns:

```text
Column A       Column B
----------     ----------
Paragraph 1    Paragraph 4
Paragraph 2    Paragraph 5
Paragraph 3    Paragraph 6
```

Naive extraction might produce:

```text
1 4 2 5 3 6
```

You want logical reading order:

```text
1 2 3 4 5 6
```

Document layout extraction exists partly to preserve document structure and reading order. ([Microsoft Learn][14])

---

# 45. Tables

This is extremely important for insurance.

Example:

```text
Repair Item       Cost

Windshield        $800
Labor             $500
Paint             $300
-----------------------
Total             $1600
```

You don't want only:

```text
Windshield 800 Labor 500 Paint 300 Total 1600
```

You want structural table information.

Document Intelligence layout extraction provides table structure including rows, columns, spans and cell information. ([Microsoft Learn][15])

---

# 46. Key-value pairs

Forms often look like:

```text
Claim Number: 84729

Loss Date: 2026-08-21

Policy Number: POL-4421
```

You can transform that into:

```json
{
  "claim_number": "84729",
  "loss_date": "2026-08-21",
  "policy_number": "POL-4421"
}
```

This is different from just extracting the text.

It is **field extraction**.

---

# 47. Document classification

This is another important concept.

Imagine ImageRight contains:

```text
100,000 documents
```

You first need to determine:

```text
FNOL
POLICE_REPORT
ESTIMATE
INVOICE
MEDICAL_REPORT
CORRESPONDENCE
POLICY_FORM
OTHER
```

That's classification.

Azure Document Intelligence supports custom classification models that can identify document type before a relevant extraction model is invoked. ([Microsoft Learn][16])

So:

```text
Document
   ↓
Classifier
   ↓
POLICE_REPORT
   ↓
Police report extraction model
```

---

# 48. Extraction

Classification answers:

> **What kind of document is this?**

Extraction answers:

> **What information is inside it?**

Example:

```text
POLICE_REPORT
      ↓
extract:
 ├── incident_date
 ├── incident_location
 ├── parties
 ├── narrative
 └── report_number
```

---

# 49. Custom model

Suppose your company has a very specific form:

```text
Company Accident Form v7
```

and standard models aren't enough.

You can train a custom extraction model against representative labeled documents.

Azure Document Intelligence supports custom extraction and custom classification models, including custom neural models for structured, semi-structured and unstructured documents. ([Microsoft Learn][16])

---

# 50. Template model vs neural model

Simplified:

### Template

Good when:

```text
same layout
same fields
same structure
```

### Neural/custom document model

Better when:

```text
layout varies
documents vary
information appears in different places
```

Microsoft currently recommends starting with custom neural models when the document/language scenario supports them. ([Microsoft Learn][17])

---

# 51. Confidence score

A document extraction system may tell you:

```text
claim_number = 84729
confidence = 0.99
```

but perhaps:

```text
policy_number = POL-44?1
confidence = 0.52
```

Now you have a decision:

```text
high confidence
      ↓
accept

low confidence
      ↓
review / retry / alternate processing
```

This is incredibly important in insurance.

Don't pretend every OCR/extraction result is equally trustworthy.

---

# 52. Human-in-the-loop

For sensitive fields:

```text
Extraction
   ↓
Confidence check
   ↓
Low confidence?
   ├── No → continue
   └── Yes
          ↓
      Human review
```

This gives you an enterprise-safe pattern.

---

# 53. OCR vs parsing vs extraction

These are not the same.

```text
OCR
=
read pixels → characters

Parsing
=
understand document structure

Extraction
=
pull specific information

Classification
=
determine document type
```

Example:

```text
PDF
 ↓
OCR
 ↓
Text + layout
 ↓
Classification
 ↓
POLICE_REPORT
 ↓
Extraction
 ↓
claim number / incident date / narrative
```

---

# 54. Markdown is useful for RAG

For RAG, you often want a clean representation like:

```markdown
# Police Report

## Incident Details

The accident occurred...

## Parties

...

## Witness Statement

...
```

Why?

Because later:

```text
markdown
 ↓
chunking
 ↓
Cortex Search
```

can preserve semantic structure better than one giant raw extraction blob.

Azure's newer document-analysis capabilities explicitly describe text, paragraphs, tables and hierarchical sections as useful foundations for retrieval-augmented generation. ([Microsoft Learn][18])

---

# 55. Images need a separate mental model

Suppose:

```text
DamagePhoto01.jpg
```

There is no traditional text.

You have:

```text
pixels
```

You may need:

```text
image analysis
 ↓
description / labels / OCR / metadata
```

But be careful:

> **Don't automatically replace the original image with an AI-generated description.**

Keep:

```text
original image
+
derived description
+
provenance
```

For example:

```text
IMAGE
 ├── blob_path
 ├── width
 ├── height
 ├── mime_type
 └── checksum

IMAGE_ANALYSIS
 ├── description
 ├── detected_objects
 ├── extracted_text
 └── model_version
```

---

# 56. Parsing should therefore produce more than text

Your processing output should conceptually be:

```text
DOCUMENT
     │
     ├── metadata
     │
     ├── pages
     │
     ├── text
     │
     ├── sections
     │
     ├── tables
     │
     ├── images
     │
     ├── fields
     │
     ├── classification
     │
     └── confidence / processing metadata
```

---

# 57. Processing status

Every document needs a lifecycle.

Something like:

```text
DISCOVERED
   ↓
COPIED
   ↓
VALIDATED
   ↓
CLASSIFIED
   ↓
PARSED
   ↓
NORMALIZED
   ↓
LOADED_TO_SNOWFLAKE
   ↓
INDEXED
```

Failures:

```text
FAILED
QUARANTINED
NEEDS_REVIEW
```

Don't simply have:

```text
processed = true
```

That's too weak for enterprise migration.

---

# 58. Why quarantine exists

Suppose:

```text
password-protected PDF
corrupt PDF
unsupported format
malformed image
OCR failure
API returned incomplete data
```

Don't silently discard it.

Put it into:

```text
/quarantine/
```

and capture:

```text
error_code
error_message
document_id
pipeline_run_id
timestamp
retry_count
```

---

# 59. Retry

A temporary error:

```text
Document Intelligence timeout
```

should not necessarily become:

```text
permanent failure
```

Pipeline:

```text
Attempt 1 → timeout
Attempt 2 → timeout
Attempt 3 → success
```

Use exponential backoff rather than:

```text
retry 100 times immediately
```

---

# 60. Dead-letter concept

If a message repeatedly fails:

```text
Normal queue
     ↓
retry
     ↓
retry
     ↓
retry
     ↓
Dead-letter queue
```

This prevents one bad document from blocking the whole pipeline.

---

# 61. Event-driven vs batch

Your migration actually has **two different workloads**.

### Initial migration

Millions of historical documents:

```text
Batch
```

### Ongoing changes

New/changed documents:

```text
Event-driven / incremental
```

So your architecture can be:

```text
             INITIAL
ImageRight ───────────→ Batch → Azure

             ONGOING
ImageRight ───────────→ Incremental → Azure
                                  ↑
                              events/API
```

Azure Data Factory supports incremental file-loading patterns based on last modified time, while Blob Storage can emit creation events through Event Grid. ([Microsoft Learn][12])

---

# 62. Now Snowflake connection

Once the files are in Azure:

```text
Azure Blob
     ↓
Snowflake External Stage
```

Snowflake supports Azure external stages and recommends storage integrations rather than embedding long-lived cloud credentials into stage definitions. Storage integrations use an Azure identity/service principal and constrain allowed storage locations. ([Snowflake Documentation][19])

Conceptually:

```text
Azure Storage
      ↑
      │ authorized access
      │
Snowflake Storage Integration
      │
      ▼
External Stage
```

---

# 63. Storage Integration

This is the Snowflake/Azure trust relationship.

Think:

```text
Snowflake
   │
   │ "I need to read this Azure storage"
   ▼
Azure identity
   │
   ▼
RBAC permissions
   │
   ▼
Blob container
```

So you don't want:

```text
Snowflake SQL
   ↓
hard-coded Azure secret
```

Snowflake's storage integration is specifically intended to avoid repeatedly supplying cloud credentials/SAS tokens. ([Snowflake Documentation][19])

---

# 64. External Stage

A stage is basically:

> **Snowflake's named doorway to external files.**

For example:

```text
@IMAGERIGHT_RAW_STAGE
```

behind it:

```text
azure://storageaccount/container/raw/
```

Then Snowflake can work with the files there.

---

# 65. Directory table

A directory table provides metadata about files in a stage.

Conceptually:

```text
FILE_PATH
FILE_SIZE
LAST_MODIFIED
...
```

That becomes useful for:

```text
file discovery
incremental processing
monitoring
lineage
```

Snowflake supports automatic directory refresh mechanisms for Azure event notifications as well. ([Snowflake Documentation][20])

---

# 66. Do we parse in Azure or Snowflake?

This is a major architecture decision.

You actually have two strong patterns.

## Pattern A — Azure parses

```text
ImageRight
   ↓
Azure Blob
   ↓
Azure Document Intelligence
   ↓
JSON / Markdown
   ↓
Snowflake
```

Advantages:

```text
Strong document processing ecosystem
Custom classification/extraction
Easy to inspect raw extraction
```

---

## Pattern B — Snowflake parses

```text
ImageRight
   ↓
Azure Blob
   ↓
Snowflake External Stage
   ↓
AI_PARSE_DOCUMENT
   ↓
Snowflake
```

Snowflake provides `AI_PARSE_DOCUMENT` for extracting text/layout from staged documents, making it possible to keep more of the document-processing flow inside Snowflake. ([Snowflake Documentation][19])

---

# 67. Which would I choose for your project?

I would **not decide blindly**.

I'd run a representative benchmark.

Take something like:

```text
50–200 real documents
```

including:

```text
simple PDF
scanned PDF
multi-page claim file
table-heavy estimate
email
poor scan
photo
mixed document
```

Then compare:

```text
Azure Document Intelligence
vs
Snowflake AI_PARSE_DOCUMENT
```

on:

```text
Extraction accuracy
Table accuracy
OCR quality
Layout preservation
Processing time
Cost
Operational complexity
Image handling
Custom extraction requirements
```

That is much more defensible than saying one is automatically better.

---

# 68. For your use case, I'd likely use this architecture

My starting architecture would be:

```text
                         IMAGERIGHT
                             │
                    REST/API/export
                             │
                             ▼
                  INGESTION SERVICE
                  │
          ┌───────┴────────┐
          │                │
       API calls        metadata
          │                │
          └───────┬────────┘
                  ▼
             AZURE BLOB
             / ADLS GEN2
                  │
       ┌──────────┼──────────┐
       │          │          │
      RAW      QUARANTINE   MANIFEST
       │
       ▼
     EVENT GRID
       │
       ▼
    QUEUE / ORCHESTRATOR
       │
       ▼
 DOCUMENT PROCESSING
       │
       ├── classification
       ├── OCR
       ├── layout
       ├── tables
       ├── fields
       └── validation
       │
       ▼
    PARSED OUTPUT
       │
       ▼
     SNOWFLAKE
       │
       ├── documents
       ├── pages
       ├── chunks
       ├── metadata
       └── structured claim data
       │
       ├───────────────┐
       ▼               ▼
CORTEX SEARCH    SEMANTIC VIEW
                       │
                       ▼
                CORTEX ANALYST
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
      SEARCH                      ANALYST
         \                           /
          \                         /
           └────── CORTEX AGENT ───┘
```

---

# 69. Where does the VM sit?

This is important.

It is **not necessarily here**:

```text
ImageRight
 ↓
VM
 ↓
Blob
```

It is:

```text
             Do we need custom/legacy compute?
                         │
                  ┌──────┴──────┐
                  │             │
                 YES            NO
                  │             │
                  ▼             ▼
             Azure VM      managed service
```

If ImageRight requires a Windows connector or private software environment:

```text
ImageRight
   ↓
Azure VM
   ↓
Blob
```

If clean REST/API integration exists:

```text
ImageRight
   ↓
Function / Container / ADF
   ↓
Blob
```

The VM is a **compute choice**, not a mandatory part of the data architecture.

---

# 70. Insurance-specific document taxonomy

For your project, I'd build a document taxonomy before parsing everything.

Something like:

```text
CLAIMS
├── FNOL
├── POLICE_REPORT
├── ADJUSTER_NOTE
├── CLAIM_CORRESPONDENCE
├── REPAIR_ESTIMATE
├── REPAIR_INVOICE
├── PHOTOGRAPH
├── MEDICAL_DOCUMENT
├── WITNESS_STATEMENT
├── SETTLEMENT
├── SUBROGATION
└── OTHER

UNDERWRITING
├── SUBMISSION
├── APPLICATION
├── QUOTE
├── BINDER
├── POLICY_FORM
├── ENDORSEMENT
├── INSPECTION_REPORT
└── CORRESPONDENCE
```

This is not just organizational.

It affects:

```text
classification
extraction
metadata
search filtering
chunking
retrieval
security
```

---

# 71. A claim document should ultimately have lineage

For example:

```text
claim_id
    ↓
document_id
    ↓
source_system
    ↓
source_document_id
    ↓
source_version
    ↓
blob_path
    ↓
parse_run_id
    ↓
parser/model version
    ↓
extraction
    ↓
chunk_id
    ↓
search result
    ↓
final answer
```

That gives you **data lineage**.

Then if an adjuster asks:

> “Where did the AI get this answer?”

you can trace it back.

That's exactly the kind of provenance you want in an insurance RAG system.

---

# 72. Data lineage vs metadata vs provenance

These terms are related but different.

### Metadata

Information describing something.

```text
document_type = POLICE_REPORT
```

### Lineage

Where data came from and how it transformed.

```text
ImageRight
 → Blob
 → parser
 → Snowflake
 → chunk
```

### Provenance

Evidence/source supporting a particular result.

```text
Answer
 ↓
Police Report
 ↓
Page 4
 ↓
specific extracted passage
```

For RAG, provenance is particularly important.

---

# 73. The master ingestion state machine

I would actually teach your project using this:

```text
DISCOVERED
    ↓
EXTRACTED_FROM_IMAGERIGHT
    ↓
COPIED_TO_BLOB
    ↓
CHECKSUM_VERIFIED
    ↓
CLASSIFIED
    ↓
PARSED
    ↓
VALIDATED
    ↓
NORMALIZED
    ↓
LOADED_TO_SNOWFLAKE
    ↓
SEARCHABLE
```

Failure can occur at any point:

```text
         ┌──────────── FAILED ────────────┐
         │                                │
         ▼                                │
      RETRY ─────────────────────────────┘
         │
         ▼
   QUARANTINE / REVIEW
```

That's much closer to a real production architecture.

---

# 74. The terms you should now know

| Term                     | Simple meaning                                                  |
| ------------------------ | --------------------------------------------------------------- |
| **Tenant**               | Your organization's Microsoft identity boundary                 |
| **Subscription**         | Azure billing/resource boundary                                 |
| **Resource Group**       | Logical group of Azure resources                                |
| **Region**               | Azure geographic location                                       |
| **VNet**                 | Private Azure network                                           |
| **Subnet**               | Smaller network segment                                         |
| **NSG**                  | Network traffic rules                                           |
| **Private Endpoint**     | Private network connection to an Azure service                  |
| **Storage Account**      | Azure storage resource                                          |
| **Container**            | Logical Blob grouping                                           |
| **Blob**                 | Individual stored object/file                                   |
| **ADLS Gen2**            | Blob Storage with hierarchical namespace/data-lake capabilities |
| **VM**                   | Virtual computer                                                |
| **VM Size**              | CPU/RAM/network capacity                                        |
| **Managed Disk**         | Persistent VM disk                                              |
| **VMSS**                 | Group of scalable VMs                                           |
| **Managed Identity**     | Azure-managed identity without hard-coded credentials           |
| **RBAC**                 | Role-based permissions                                          |
| **Key Vault**            | Secure secret/key/certificate store                             |
| **ADF**                  | Data movement/orchestration service                             |
| **Function**             | Small event-driven compute                                      |
| **Event Grid**           | Event notification system                                       |
| **Service Bus**          | Reliable message/queue system                                   |
| **API**                  | Programmatic interface to another system                        |
| **REST**                 | Common HTTP API style                                           |
| **Pagination**           | Retrieving API results in multiple pages                        |
| **Watermark**            | State used for incremental processing                           |
| **Checkpoint**           | Remembered processing progress                                  |
| **Idempotency**          | Re-running doesn't create bad duplicates                        |
| **Checksum**             | Hash used to identify content                                   |
| **Manifest**             | Record of files/items and migration state                       |
| **Quarantine**           | Isolated area for problematic data                              |
| **OCR**                  | Image → text                                                    |
| **Layout analysis**      | Understand document structure                                   |
| **Classification**       | Determine document type                                         |
| **Extraction**           | Pull specific fields/information                                |
| **Table extraction**     | Recover rows/columns/cells                                      |
| **Bounding box/polygon** | Physical location of content on a page                          |
| **Confidence**           | How confident the extraction model is                           |
| **Custom model**         | Model specialized for your documents                            |
| **Normalization**        | Convert extracted data into consistent structure                |
| **Lineage**              | Track data's journey                                            |
| **Provenance**           | Trace an answer/result to supporting source                     |

---

# 75. The entire thing in one simple example

Let's take:

```text
Claim 84729
```

ImageRight contains:

```text
PoliceReport.pdf
RepairEstimate.pdf
DamagePhoto.jpg
AdjusterNotes.pdf
```

### Step 1 — Extract

```text
ImageRight
   ↓
API
```

### Step 2 — Raw storage

```text
Azure Blob

/raw/claims/84729/
    PoliceReport.pdf
    RepairEstimate.pdf
    DamagePhoto.jpg
    AdjusterNotes.pdf
```

### Step 3 — Manifest

```text
DOC001 → copied
DOC002 → copied
DOC003 → copied
DOC004 → copied
```

### Step 4 — Classification

```text
PoliceReport.pdf
 → POLICE_REPORT

RepairEstimate.pdf
 → REPAIR_ESTIMATE

DamagePhoto.jpg
 → PHOTOGRAPH

AdjusterNotes.pdf
 → ADJUSTER_NOTE
```

### Step 5 — Parsing

```text
PoliceReport
 ├── pages
 ├── paragraphs
 └── tables

Estimate
 ├── pages
 └── tables

AdjusterNotes
 └── paragraphs
```

### Step 6 — Normalize

```text
DOCUMENTS
DOCUMENT_PAGES
DOCUMENT_CHUNKS
```

### Step 7 — Snowflake

```text
CLAIMS
DOCUMENTS
DOCUMENT_PAGES
DOCUMENT_CHUNKS
PAYMENTS
RESERVES
```

### Step 8 — Search

Cortex Search indexes relevant document content.

### Step 9 — Analyst

Semantic view describes:

```text
Claim
Claim Amount
Reserve
Payment
State
Claim Type
```

### Step 10 — User asks:

> “Why was claim 84729 more expensive than average?”

Agent can combine:

```text
Analyst:
claim cost = $15,000
average = $9,800

Search:
additional damage
supplementary estimate
higher labor cost
```

### Step 11 — Agent produces grounded answer

That is the complete system.

---

# 76. The single biggest lesson

Don't think:

```text
ImageRight → Azure → Snowflake → AI
```

Think:

```text
SOURCE
 ↓
CAPTURE
 ↓
RAW STORAGE
 ↓
IDENTITY / SECURITY
 ↓
PROCESSING
 ↓
DOCUMENT UNDERSTANDING
 ↓
VALIDATION
 ↓
NORMALIZATION
 ↓
DATA MODEL
 ↓
SEARCH / ANALYTICS
 ↓
AI
```

Each layer has a job.

And **VM is only one possible compute mechanism inside the processing/integration layer**.

For your use case, the most important technical foundation is actually:

```text
ImageRight
   ↓
correct extraction
   ↓
correct metadata/relationships
   ↓
safe immutable raw storage
   ↓
high-quality parsing
   ↓
correct document classification
   ↓
page/table/field preservation
   ↓
strong lineage
   ↓
Snowflake
```

If that foundation is correct, the Cortex layer we discussed earlier becomes much easier to understand and much more reliable.

The next level I would teach is the **actual implementation of this architecture**, starting with a real `Claim → Document → Page → Chunk` data model and then walking through **ImageRight API → Azure Blob → VM/managed compute → Document Intelligence → parsed JSON/Markdown → Snowflake external stage**, including the actual request/response shapes, folder structure, manifest schema, retry logic, and sample Python/Azure/Snowflake code.

[1]: https://www.vertafore.com/products/insurance-document-management-system/imageright?utm_source=chatgpt.com "Insurance Document Management & Workflow Software | ImageRight"
[2]: https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-overview?utm_source=chatgpt.com "About Blob (object) storage - Azure Storage | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-namespace?utm_source=chatgpt.com "Azure Data Lake Storage hierarchical namespace - Azure Storage | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction?utm_source=chatgpt.com "Azure Data Lake Storage overview - Azure Storage | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-manage?utm_source=chatgpt.com "Manage and restore soft-deleted blobs - Azure Storage | Microsoft Learn"
[6]: https://learn.microsoft.com/en-us/azure/virtual-machines/overview?utm_source=chatgpt.com "Overview of virtual machines in Azure - Azure Virtual Machines | Microsoft Learn"
[7]: https://learn.microsoft.com/en-us/azure/virtual-machines/managed-disks-overview?utm_source=chatgpt.com "Overview of Azure Disk Storage - Azure Virtual Machines | Microsoft Learn"
[8]: https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview?wt.mc_id=AZ-MVP-5005255&utm_source=chatgpt.com "Azure Virtual Machine Scale Sets overview - Azure Virtual Machine Scale Sets | Microsoft Learn"
[9]: https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-managed-identities-work-vm?utm_source=chatgpt.com "How managed identities for Azure resources work with Azure virtual machines - Managed identities for Azure resources | Microsoft Learn"
[10]: https://learn.microsoft.com/azure/storage/common/storage-private-endpoints?utm_source=chatgpt.com "Use private endpoints - Azure Storage | Microsoft Learn"
[11]: https://learn.microsoft.com/en-us/azure/storage/common/secure-storage?utm_source=chatgpt.com "Secure your Azure Storage account | Microsoft Learn"
[12]: https://learn.microsoft.com/en-us/azure/data-factory/tutorial-incremental-copy-overview?utm_source=chatgpt.com "Incrementally copy data from a source data store to a destination data store - Azure Data Factory | Microsoft Learn"
[13]: https://learn.microsoft.com/en-us/azure/event-grid/event-schema-blob-storage?utm_source=chatgpt.com "Azure Blob Storage as Event Grid source - Azure Event Grid | Microsoft Learn"
[14]: https://learn.microsoft.com/azure/ai-services/document-intelligence/prebuilt/layout?utm_source=chatgpt.com "Document layout analysis - Document Intelligence - Foundry Tools | Microsoft Learn"
[15]: https://learn.microsoft.com/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0&utm_source=chatgpt.com "Document layout analysis - Document Intelligence - Foundry Tools | Microsoft Learn"
[16]: https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?source=recommendations&view=doc-intel-3.1.0&utm_source=chatgpt.com "Custom document models - Document Intelligence - Foundry Tools | Microsoft Learn"
[17]: https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0&utm_source=chatgpt.com "Custom document models - Document Intelligence - Foundry Tools | Microsoft Learn"
[18]: https://learn.microsoft.com/en-us/azure/ai-services/content-understanding/document/overview?utm_source=chatgpt.com "Azure Content Understanding in Foundry Tools Document Overview - Foundry Tools | Microsoft Learn"
[19]: https://docs.snowflake.com/en/en/user-guide/data-load-azure-config?utm_source=chatgpt.com "Configure an Azure container for loading data | Snowflake Documentation"
[20]: https://docs.snowflake.com/en/en/user-guide/data-load-azure-create-stage?utm_source=chatgpt.com "Create an Azure stage | Snowflake Documentation"
