# Extracting ImageRight Structure & Metadata via API
## Discovery Phase: Drawers, Folders, Document Types, Attributes (NOT the actual files)

---

# WHY DO THIS FIRST (Before Touching the 7TB of Documents)

Before you migrate a single document, you need a **complete map of the system** — like getting blueprints of a building before moving furniture. This phase answers:

```
┌─────────────────────────────────────────────────────────────────────┐
│  QUESTIONS THIS DISCOVERY PHASE ANSWERS                              │
├─────────────────────────────────────────────────────────────────────┤
│  1. How many Drawers exist, and what are they called?                │
│  2. How many Folders live inside each Drawer?                        │
│  3. What Document Types exist system-wide (EOB, FNOL, CMS-1500...)?  │
│  4. What custom metadata fields (Attributes) exist per Drawer?       │
│  5. How many Files/Documents/Pages exist in each Drawer?             │
│  6. What's the TOTAL size and shape of the data before we move it?   │
└─────────────────────────────────────────────────────────────────────┘
```

**The output of this phase is NOT documents — it's a "Data Dictionary" or "Catalog"** — a set of small JSON/CSV files describing the SHAPE of the entire 7TB system, usually just a few megabytes in total.

---

# PART 1: UNDERSTANDING THE API YOU'LL USE

ImageRight exposes a REST API called **ImageRight Connect**. For structure/metadata discovery, you primarily use **read-only GET endpoints** — fast, lightweight, and safe to run against production without impacting performance.

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    IMAGERIGHT CONNECT API — DISCOVERY ENDPOINTS           │
├──────────┬──────────────────────────────────┬─────────────────────────────┤
│ METHOD   │ ENDPOINT                         │ RETURNS                     │
├──────────┼──────────────────────────────────┼─────────────────────────────┤
│ GET      │ /api/v1/drawers                  │ List of ALL Drawers          │
│ GET      │ /api/v1/drawers/{id}              │ Single Drawer details        │
│ GET      │ /api/v1/drawers/{id}/folders      │ Folders inside a Drawer      │
│ GET      │ /api/v1/doctypes                  │ ALL Document Type defs       │
│ GET      │ /api/v1/doctypes/{id}             │ Single Doc Type details      │
│ GET      │ /api/v1/drawers/{id}/attributes    │ Custom metadata fields       │
│          │                                    │ (EAV Attribute Definitions)  │
│ GET      │ /api/v1/drawers/{id}/files/count   │ Total File count in Drawer   │
│ GET      │ /api/v1/drawers/{id}/stats         │ Aggregate stats (if exposed) │
│ GET      │ /api/v1/users                      │ User/Security Group list     │
│ GET      │ /api/v1/workflow/steps             │ All Workflow Queue defs      │
└──────────┴──────────────────────────────────┴─────────────────────────────┘
```

> **Note:** Exact endpoint paths vary slightly by ImageRight version (7.x vs 8.x) and your Vertafore API license tier. Always confirm against your instance's Swagger/OpenAPI documentation, typically found at `https://{your-imageright-host}/api/swagger`.

---

# PART 2: AUTHENTICATION SETUP

```python
import requests

# ──────────────────────────────────────────────
# STEP 1: Authenticate and get a Bearer Token
# ──────────────────────────────────────────────

AUTH_URL = "https://imageright.atlam.com/api/oauth/token"
API_BASE = "https://imageright.atlam.com/api/v1"

auth_payload = {
    "grant_type": "client_credentials",
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "YOUR_CLIENT_SECRET"
}

response = requests.post(AUTH_URL, data=auth_payload)
token = response.json()["access_token"]

headers = {
    "Authorization": f"Bearer {token}",
    "Accept": "application/json"
}

print("✅ Authenticated successfully")
```

---

# PART 3: EXTRACTING THE DRAWER STRUCTURE

### 3.1 Get All Drawers

```python
import json
import pandas as pd

def get_all_drawers():
    """Fetch the complete list of Drawers (top-level categories)."""
    url = f"{API_BASE}/drawers"
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.json()["items"]  # adjust key based on actual API response shape

drawers = get_all_drawers()

print(f"Total Drawers found: {len(drawers)}")
for d in drawers:
    print(f"  [{d['drawerId']}] {d['drawerName']}  (Active: {d['isActive']})")

# Save raw output
with open("01_drawers.json", "w") as f:
    json.dump(drawers, f, indent=2)

# Convert to a clean DataFrame for easy viewing/export
drawers_df = pd.DataFrame(drawers)
drawers_df.to_csv("01_drawers.csv", index=False)
```

**Expected output (example for Atlantic American):**
```
Total Drawers found: 18
  [1]  Bankers Fidelity - Health Claims        (Active: True)
  [2]  Bankers Fidelity - Life Claims          (Active: True)
  [3]  Bankers Fidelity - Underwriting         (Active: True)
  [4]  Employee Benefits - Group Claims        (Active: True)
  [5]  Employee Benefits - Enrollment          (Active: True)
  [6]  American Southern - P&C Claims          (Active: True)
  [7]  American Southern - Underwriting        (Active: True)
  [8]  Corporate - Compliance & Legal          (Active: True)
  ...
```

---

### 3.2 Get All Folders Inside Each Drawer

```python
def get_folders_for_drawer(drawer_id):
    """Fetch all Folders nested inside a specific Drawer."""
    url = f"{API_BASE}/drawers/{drawer_id}/folders"
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.json()["items"]

all_folders = []

for d in drawers:
    folders = get_folders_for_drawer(d["drawerId"])
    for f in folders:
        f["drawerId"] = d["drawerId"]
        f["drawerName"] = d["drawerName"]
        all_folders.append(f)
    print(f"Drawer '{d['drawerName']}' has {len(folders)} folders")

folders_df = pd.DataFrame(all_folders)
folders_df.to_csv("02_folders.csv", index=False)

with open("02_folders.json", "w") as f:
    json.dump(all_folders, f, indent=2)
```

---

# PART 4: EXTRACTING DOCUMENT TYPES (System-Wide Dictionary)

Document Types are usually a **global dictionary** (not nested per-drawer), though some are drawer-specific. This tells you EXACTLY what kinds of documents exist across the whole 7TB (EOB, FNOL, CMS-1500, etc.).

```python
def get_all_doctypes():
    """Fetch the master Document Type dictionary."""
    url = f"{API_BASE}/doctypes"
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.json()["items"]

doctypes = get_all_doctypes()

print(f"Total Document Types found: {len(doctypes)}")
doctypes_df = pd.DataFrame(doctypes)
doctypes_df.to_csv("03_doctypes.csv", index=False)

with open("03_doctypes.json", "w") as f:
    json.dump(doctypes, f, indent=2)
```

**Expected output (example):**
```
Total Document Types found: 64
  [1]  EOB
  [2]  CMS-1500 Claim Form
  [3]  UB-04 Claim Form
  [4]  FNOL Report
  [5]  Adjuster Notes
  [6]  Damage Photos
  [7]  Death Certificate
  [8]  Beneficiary Designation
  [9]  Policy Contract
  [10] Declarations Page
  [11] Email Correspondence
  [12] Settlement Letter
  ... (etc.)
```

---

# PART 5: EXTRACTING CUSTOM METADATA FIELDS (EAV Attribute Definitions)

This is the MOST important part for your RAG/Cortex build — it tells you **exactly what business data fields exist** per Drawer (PatientName, PolicyNumber, CPT_Code, etc.) — the equivalent of discovering your database's column schema before querying data.

```python
def get_attributes_for_drawer(drawer_id):
    """Fetch all custom metadata field definitions for a Drawer."""
    url = f"{API_BASE}/drawers/{drawer_id}/attributes"
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.json()["items"]

all_attributes = []

for d in drawers:
    attrs = get_attributes_for_drawer(d["drawerId"])
    for a in attrs:
        a["drawerId"] = d["drawerId"]
        a["drawerName"] = d["drawerName"]
        all_attributes.append(a)
    print(f"Drawer '{d['drawerName']}' has {len(attrs)} custom attributes")

attributes_df = pd.DataFrame(all_attributes)
attributes_df.to_csv("04_attribute_definitions.csv", index=False)

with open("04_attribute_definitions.json", "w") as f:
    json.dump(all_attributes, f, indent=2)
```

**Expected output (example for "Health Claims" Drawer):**
```
Drawer 'Bankers Fidelity - Health Claims' has 22 custom attributes
  [42] PatientName       (STRING)   - File Level
  [43] PolicyNumber      (STRING)   - File Level
  [44] PatientDOB        (DATETIME) - File Level
  [45] ClaimNumber       (STRING)   - File Level
  [46] ProviderName      (STRING)   - Document Level
  [47] CPT_Code          (STRING)   - Document Level
  [48] PaidAmount        (DECIMAL)  - Document Level
  [49] DeniedAmount      (DECIMAL)  - Document Level
  [50] DenialReasonCode  (STRING)   - Document Level
  ...
```

---

# PART 6: GETTING RECORD COUNTS (File / Document / Page Counts) WITHOUT PULLING ACTUAL DOCUMENTS

This tells you the SIZE and SHAPE of the data in each Drawer — critical for planning your migration batches.

```python
def get_drawer_counts(drawer_id):
    """
    Get aggregate counts for a Drawer without pulling any documents.
    Many ImageRight APIs expose a lightweight 'count' or 'stats' endpoint.
    If not available directly, use a paginated search with page_size=1
    and read the 'totalCount' field from the response envelope.
    """
    url = f"{API_BASE}/drawers/{drawer_id}/files"
    params = {"pageSize": 1, "pageNumber": 1}  # minimal payload, just need total count
    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()
    data = response.json()
    return data.get("totalCount", 0)

drawer_stats = []

for d in drawers:
    file_count = get_drawer_counts(d["drawerId"])
    drawer_stats.append({
        "drawerId": d["drawerId"],
        "drawerName": d["drawerName"],
        "totalFiles": file_count
    })
    print(f"Drawer '{d['drawerName']}': {file_count:,} files")

stats_df = pd.DataFrame(drawer_stats)
stats_df.to_csv("05_drawer_file_counts.csv", index=False)
```

### Getting Document and Page counts per Drawer (drill down one more level)

```python
def get_document_count_for_file(file_id):
    url = f"{API_BASE}/files/{file_id}/documents"
    params = {"pageSize": 1, "pageNumber": 1}
    response = requests.get(url, headers=headers, params=params)
    return response.json().get("totalCount", 0)

# NOTE: Calling this once per File for millions of Files is too slow.
# Instead, use a dedicated aggregate/reporting endpoint if available:
def get_drawer_full_stats(drawer_id):
    """Some ImageRight Connect versions expose a direct stats rollup."""
    url = f"{API_BASE}/drawers/{drawer_id}/statistics"
    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        return response.json()
    else:
        return None  # fallback: estimate via SQL instead (see Part 9)
```

> **Practical tip:** The REST API is great for Drawers, Folders, DocTypes, and Attribute Definitions (small, fast, lightweight lists). But for **precise File/Document/Page COUNTS across millions of records**, a direct read-only SQL query (covered in Part 9) is dramatically faster than looping through API calls one by one.

---

# PART 7: HANDLING PAGINATION (Critical for Large Lists)

Most ImageRight Connect endpoints paginate results. You must loop through all pages to get the COMPLETE list (especially for Folders if a Drawer has hundreds).

```python
def get_all_pages(endpoint_url, params=None):
    """
    Generic pagination handler — loops through all pages
    of a paginated ImageRight Connect API response.
    """
    all_items = []
    page_number = 1
    page_size = 100  # typical max page size

    if params is None:
        params = {}

    while True:
        params.update({"pageNumber": page_number, "pageSize": page_size})
        response = requests.get(endpoint_url, headers=headers, params=params)
        response.raise_for_status()
        data = response.json()

        items = data.get("items", [])
        all_items.extend(items)

        total_count = data.get("totalCount", len(all_items))
        print(f"  Fetched page {page_number}: {len(items)} items "
              f"({len(all_items)}/{total_count} total)")

        if len(all_items) >= total_count or len(items) == 0:
            break

        page_number += 1

    return all_items

# Example usage: get ALL folders across a drawer with 500+ folders
folders = get_all_pages(f"{API_BASE}/drawers/5/folders")
```

---

# PART 8: PUTTING IT ALL TOGETHER — THE FULL DISCOVERY SCRIPT

```python
import requests
import pandas as pd
import json
import time
from datetime import datetime

API_BASE = "https://imageright.atlam.com/api/v1"
HEADERS = {"Authorization": f"Bearer {token}", "Accept": "application/json"}
OUTPUT_DIR = "imageright_catalog"

import os
os.makedirs(OUTPUT_DIR, exist_ok=True)


def api_get(url, params=None, retries=3):
    """Robust GET with retry logic and rate-limit handling."""
    for attempt in range(retries):
        response = requests.get(url, headers=HEADERS, params=params)
        if response.status_code == 429:  # Rate limited
            wait = int(response.headers.get("Retry-After", 5))
            print(f"  ⚠️  Rate limited. Waiting {wait}s...")
            time.sleep(wait)
            continue
        response.raise_for_status()
        return response.json()
    raise Exception(f"Failed after {retries} retries: {url}")


def paginate(url, params=None):
    all_items = []
    page = 1
    params = params or {}
    while True:
        params.update({"pageNumber": page, "pageSize": 100})
        data = api_get(url, params)
        items = data.get("items", [])
        all_items.extend(items)
        if len(items) == 0 or len(all_items) >= data.get("totalCount", 0):
            break
        page += 1
        time.sleep(0.1)  # gentle rate limiting
    return all_items


def main():
    report = {"startTime": datetime.now().isoformat()}

    # ── 1. DRAWERS ──────────────────────────────────────
    print("\n[1/5] Fetching Drawers...")
    drawers = paginate(f"{API_BASE}/drawers")
    pd.DataFrame(drawers).to_csv(f"{OUTPUT_DIR}/01_drawers.csv", index=False)
    print(f"  ✅ {len(drawers)} drawers saved")

    # ── 2. FOLDERS (per drawer) ─────────────────────────
    print("\n[2/5] Fetching Folders for each Drawer...")
    all_folders = []
    for d in drawers:
        folders = paginate(f"{API_BASE}/drawers/{d['drawerId']}/folders")
        for f in folders:
            f["drawerId"] = d["drawerId"]
            f["drawerName"] = d["drawerName"]
        all_folders.extend(folders)
        print(f"  {d['drawerName']}: {len(folders)} folders")
    pd.DataFrame(all_folders).to_csv(f"{OUTPUT_DIR}/02_folders.csv", index=False)

    # ── 3. DOCUMENT TYPES (global) ──────────────────────
    print("\n[3/5] Fetching Document Types...")
    doctypes = paginate(f"{API_BASE}/doctypes")
    pd.DataFrame(doctypes).to_csv(f"{OUTPUT_DIR}/03_doctypes.csv", index=False)
    print(f"  ✅ {len(doctypes)} document types saved")

    # ── 4. ATTRIBUTE DEFINITIONS (per drawer) ───────────
    print("\n[4/5] Fetching Attribute Definitions for each Drawer...")
    all_attrs = []
    for d in drawers:
        attrs = paginate(f"{API_BASE}/drawers/{d['drawerId']}/attributes")
        for a in attrs:
            a["drawerId"] = d["drawerId"]
            a["drawerName"] = d["drawerName"]
        all_attrs.extend(attrs)
        print(f"  {d['drawerName']}: {len(attrs)} attributes")
    pd.DataFrame(all_attrs).to_csv(f"{OUTPUT_DIR}/04_attributes.csv", index=False)

    # ── 5. FILE/DOCUMENT COUNTS (per drawer) ────────────
    print("\n[5/5] Fetching File Counts for each Drawer...")
    drawer_stats = []
    for d in drawers:
        data = api_get(f"{API_BASE}/drawers/{d['drawerId']}/files",
                        params={"pageSize": 1, "pageNumber": 1})
        count = data.get("totalCount", 0)
        drawer_stats.append({
            "drawerId": d["drawerId"],
            "drawerName": d["drawerName"],
            "totalFiles": count
        })
        print(f"  {d['drawerName']}: {count:,} files")
    pd.DataFrame(drawer_stats).to_csv(f"{OUTPUT_DIR}/05_file_counts.csv", index=False)

    # ── SUMMARY REPORT ───────────────────────────────────
    report.update({
        "endTime": datetime.now().isoformat(),
        "totalDrawers": len(drawers),
        "totalFolders": len(all_folders),
        "totalDocTypes": len(doctypes),
        "totalAttributes": len(all_attrs),
        "totalFilesAcrossAllDrawers": sum(s["totalFiles"] for s in drawer_stats)
    })
    with open(f"{OUTPUT_DIR}/00_summary_report.json", "w") as f:
        json.dump(report, f, indent=2)

    print("\n" + "="*60)
    print("DISCOVERY COMPLETE")
    print("="*60)
    for k, v in report.items():
        print(f"  {k}: {v}")


if __name__ == "__main__":
    main()
```

---

# PART 9: FASTER ALTERNATIVE — DIRECT SQL FOR STRUCTURE DISCOVERY

For counts and structural discovery across MILLIONS of records, going through the API one call at a time can be slow. If you have **read-only SQL access** to the ImageRight database (common for migration projects, with IT approval), this is 10-100x faster for this specific discovery phase:

```sql
-- ═══════════════════════════════════════════════════════════
-- ONE QUERY TO RULE THEM ALL: Complete Drawer-Level Summary
-- ═══════════════════════════════════════════════════════════
SELECT 
    d.DrawerId,
    d.DrawerName,
    COUNT(DISTINCT fol.FolderId)     AS TotalFolders,
    COUNT(DISTINCT f.FileId)         AS TotalFiles,
    COUNT(DISTINCT doc.DocumentId)   AS TotalDocuments,
    COUNT(DISTINCT p.PageId)         AS TotalPages,
    SUM(CAST(p.ByteSize AS BIGINT)) / 1024.0 / 1024.0 / 1024.0 AS TotalSizeGB,
    MIN(doc.DateReceived)            AS OldestDocument,
    MAX(doc.DateReceived)            AS NewestDocument
FROM IR_Drawer d
LEFT JOIN IR_Folder fol ON d.DrawerId = fol.DrawerId
LEFT JOIN IR_File f ON fol.FolderId = f.FolderId
LEFT JOIN IR_Document doc ON f.FileId = doc.FileId
LEFT JOIN IR_Page p ON doc.DocumentId = p.DocumentId
GROUP BY d.DrawerId, d.DrawerName
ORDER BY TotalSizeGB DESC;

-- ═══════════════════════════════════════════════════════════
-- Document Type breakdown PER Drawer (what kinds of docs exist)
-- ═══════════════════════════════════════════════════════════
SELECT 
    d.DrawerName,
    dt.DocTypeName,
    COUNT(doc.DocumentId) AS DocumentCount,
    COUNT(p.PageId)       AS PageCount,
    SUM(CAST(p.ByteSize AS BIGINT)) / 1024.0 / 1024.0 AS SizeMB
FROM IR_Document doc
JOIN IR_DocType dt ON doc.DocTypeId = dt.DocTypeId
JOIN IR_File f ON doc.FileId = f.FileId
JOIN IR_Folder fol ON f.FolderId = fol.FolderId
JOIN IR_Drawer d ON fol.DrawerId = d.DrawerId
LEFT JOIN IR_Page p ON doc.DocumentId = p.DocumentId
GROUP BY d.DrawerName, dt.DocTypeName
ORDER BY d.DrawerName, DocumentCount DESC;

-- ═══════════════════════════════════════════════════════════
-- Attribute Definitions (metadata schema) PER Drawer
-- ═══════════════════════════════════════════════════════════
SELECT 
    d.DrawerName,
    ad.AttributeName,
    ad.DataType,
    ad.IsRequired,
    COUNT(fav.ValueId) AS TimesUsed
FROM IR_AttributeDefinition ad
JOIN IR_Drawer d ON ad.DrawerId = d.DrawerId
LEFT JOIN IR_FileAttributeValue fav ON ad.AttributeDefId = fav.AttributeDefId
GROUP BY d.DrawerName, ad.AttributeName, ad.DataType, ad.IsRequired
ORDER BY d.DrawerName, TimesUsed DESC;
```

```python
# Pull these same results via SQL in Python for consistency with your catalog
import pyodbc
import pandas as pd

conn = pyodbc.connect(
    'DRIVER={ODBC Driver 18 for SQL Server};'
    'SERVER=ATL-SQL-IR01;DATABASE=ImageRight;'
    'Trusted_Connection=yes;'
)

drawer_summary_sql = """
SELECT d.DrawerId, d.DrawerName,
       COUNT(DISTINCT fol.FolderId) AS TotalFolders,
       COUNT(DISTINCT f.FileId) AS TotalFiles,
       COUNT(DISTINCT doc.DocumentId) AS TotalDocuments,
       COUNT(DISTINCT p.PageId) AS TotalPages,
       SUM(CAST(p.ByteSize AS BIGINT))/1024.0/1024.0/1024.0 AS TotalSizeGB
FROM IR_Drawer d
LEFT JOIN IR_Folder fol ON d.DrawerId = fol.DrawerId
LEFT JOIN IR_File f ON fol.FolderId = f.FolderId
LEFT JOIN IR_Document doc ON f.FileId = doc.FileId
LEFT JOIN IR_Page p ON doc.DocumentId = p.DocumentId
GROUP BY d.DrawerId, d.DrawerName
ORDER BY TotalSizeGB DESC
"""

drawer_summary_df = pd.read_sql(drawer_summary_sql, conn)
drawer_summary_df.to_csv("imageright_catalog/sql_drawer_summary.csv", index=False)
print(drawer_summary_df)
```

> **Best Practice:** Use the **API** for Drawer/Folder/DocType/Attribute **definitions** (clean, structured, governed by application logic). Use **direct SQL** for **counts and statistics** across millions of rows (much faster for aggregation). Cross-validate both sources to make sure numbers match.

---

# PART 10: THE FINAL OUTPUT — YOUR "IMAGERIGHT CATALOG"

After running this discovery phase, you'll have a small set of files that fully describe the STRUCTURE of the entire 7TB system:

```
imageright_catalog/
│
├── 00_summary_report.json          ← High-level totals (drawers, files, size)
├── 01_drawers.csv                  ← All 18 Drawers with names/IDs
├── 02_folders.csv                  ← All Folders nested in each Drawer
├── 03_doctypes.csv                 ← All 64 Document Types system-wide
├── 04_attributes.csv               ← All custom metadata fields per Drawer
├── 05_file_counts.csv              ← File/Document/Page counts per Drawer
└── sql_drawer_summary.csv          ← Cross-validated stats from SQL
```

### Example of what `00_summary_report.json` looks like:

```json
{
  "startTime": "2025-01-15T09:00:00",
  "endTime": "2025-01-15T09:47:32",
  "totalDrawers": 18,
  "totalFolders": 342,
  "totalDocTypes": 64,
  "totalAttributes": 287,
  "totalFilesAcrossAllDrawers": 1247893
}
```

### Example of what `05_file_counts.csv` looks like:

```
drawerId,drawerName,totalFiles,totalDocuments,totalPages,totalSizeGB
1,Bankers Fidelity - Health Claims,412000,3850000,22100000,2850.4
2,Bankers Fidelity - Life Claims,98000,620000,3200000,410.2
3,Bankers Fidelity - Underwriting,205000,980000,5400000,890.7
6,American Southern - P&C Claims,380000,4100000,28500000,2340.9
7,American Southern - Underwriting,152000,710000,3900000,510.3
...
```

**This single file now tells you EXACTLY how to plan your migration batches** — you can see immediately that "Health Claims" (2.85 TB) and "P&C Claims" (2.34 TB) are your two biggest, highest-priority migration targets.

---

# PART 11: WHY THIS MATTERS FOR YOUR NEXT STEPS (Azure Blob + Snowflake Cortex)

This catalog becomes the **blueprint** for everything downstream:

```
┌────────────────────────────────────────────────────────────────────────────┐
│  HOW THIS CATALOG FEEDS YOUR PIPELINE                                     │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  01_drawers.csv          →  Becomes your Azure Blob folder structure      │
│                              (drawer=HealthClaims/, drawer=PC_Claims/)    │
│                                                                            │
│  03_doctypes.csv         →  Tells you which OCR/parsing strategy to use   │
│                              per document (table-aware for EOB/CMS-1500)  │
│                                                                            │
│  04_attributes.csv       →  Becomes your Snowflake table SCHEMA           │
│                              (columns like PatientName, PolicyNumber,     │
│                               CPT_Code become actual Snowflake columns)   │
│                                                                            │
│  05_file_counts.csv      →  Tells you batch sizes and migration priority  │
│                              order (biggest/most-used drawers first)      │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

# SUMMARY: THE 5-STEP DISCOVERY CHECKLIST

```
☐ STEP 1: Authenticate to ImageRight Connect API (OAuth token)
☐ STEP 2: Pull all Drawers → understand business line structure
☐ STEP 3: Pull all Folders per Drawer → understand sub-categorization
☐ STEP 4: Pull all Document Types → understand what kinds of docs exist
☐ STEP 5: Pull all Attribute Definitions per Drawer → understand the 
          metadata schema (this becomes your future Snowflake columns)
☐ STEP 6: Pull File/Document/Page counts per Drawer → understand size
          and prioritize migration batches
☐ STEP 7: Cross-validate API results against direct SQL counts
☐ STEP 8: Save everything as a clean "Catalog" (CSV/JSON) — this is 
          your blueprint for the entire 7TB migration
```

---

Would you like me to go deeper into:
1. **How to turn `04_attributes.csv` into an actual Snowflake table DDL** (auto-generating `CREATE TABLE` statements from the discovered schema)?
2. **How to use this catalog to generate a prioritized, batched extraction plan** for the full document migration?
3. **How to build a simple dashboard/report** from this catalog to present to Atlantic American stakeholders before starting the full migration?
