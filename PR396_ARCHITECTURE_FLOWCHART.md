# PR #396 - AI Findings Extra Document Flow Architecture

## Overview

This PR implements the **"Pre-merge References Before Description"** approach for handling additional document uploads in AI Findings generation. When a user uploads additional documents to an existing RFX, the system merges references from both old and new documents BEFORE generating the description, allowing the LLM to see all context and create a unified description with inline conflict handling.

**Important**: This PR does NOT include special revision handling - revisions are treated the same as additional uploads (all references merged together).

---

## Flow Chart (Mermaid)

```mermaid
flowchart TD
    subgraph TRIGGER["🔄 TRIGGER"]
        A[User uploads additional document to existing RFX]
    end

    subgraph INIT["📦 STEP 1: Initialize"]
        B[AiFindingsOrchestration initialized]
        B --> C{doc_ids provided?}
        C -->|Yes| D[extra_doc = TRUE]
        C -->|No| E[extra_doc = FALSE]
    end

    subgraph STEP2["📋 STEP 2: Get Requirements"]
        F[Fetch requirement from DB with embeddings]
    end

    subgraph STEP3["🔍 STEP 3: Get Embeddings from Milvus"]
        G[Search Milvus for relevant chunks]
        G --> H[Get text embeddings]
        G --> I[Get vision embeddings]
        H --> J[Combine into DataFrame]
        I --> J
    end

    subgraph STEP4["🔗 STEP 4: Merge Back-to-Back"]
        K[Merge consecutive chunks from same page]
    end

    subgraph STEP4_5["⭐ STEP 4.5: Pre-Merge References NEW"]
        L{extra_doc?}
        L -->|No| M[Skip - proceed to Step 5]
        L -->|Yes| N[merge_references_before_description]
        
        N --> O[Query existing findings for this RFX]
        O --> P{Existing finding for same requirement?}
        
        P -->|No| Q[Keep only new references]
        
        P -->|Yes| R{Existing match_category?}
        R -->|NOT_MATCH| S[Mark for deletion, keep new refs only]
        R -->|DIRECT/RELATED| T[Fetch existing references from DB]
        
        T --> U[get_references_by_finding_id]
        U --> V[For each ref: fetch raw_text from Milvus]
        V --> W[Get document name from DB]
        W --> X[Add existing refs as rows to DataFrame]
        X --> Y[Mark existing finding for deletion]
        
        S --> Z[Combined DataFrame with all refs]
        Q --> Z
        Y --> Z
    end

    subgraph STEP5["🤖 STEP 5: LLM Description Generation"]
        AA[create_ai_name_and_description]
        AA --> AB[LLM sees ALL references combined]
        AB --> AC[Prompt includes conflict handling instruction]
        AC --> AD["Generate unified description with inline conflicts
        e.g., 'Max pressure: 85 dB from Doc A, 75 dB from Doc B'"]
    end

    subgraph STEP6["🗑️ STEP 6: Cleanup"]
        AE{extra_doc?}
        AE -->|No| AF[Skip]
        AE -->|Yes| AG[delete_not_match_findings]
        AG --> AH[Delete old findings marked in Step 4.5]
    end

    subgraph STEP7["💾 STEP 7: Save to Database"]
        AI{extra_doc?}
        AI -->|Yes| AJ[Filter out NOT_MATCH findings]
        AI -->|No| AK[Delete existing findings for requirement]
        AJ --> AL[Save new findings with all references]
        AK --> AL
    end

    subgraph RESULT["✅ RESULT"]
        AM[Single unified finding per requirement]
        AM --> AN["Description shows all document sources
        with inline conflict attribution"]
    end

    A --> B
    E --> F
    D --> F
    F --> G
    J --> K
    K --> L
    M --> AA
    Z --> AA
    AD --> AE
    AF --> AI
    AH --> AI
    AL --> AM
```

---

## Key Components Changed

### 1. `app/ai/service.py` - Main Orchestration Flow

```
Step 1: Initialize orchestration (sets extra_doc flag)
Step 2: Get requirements from DB
Step 3: Get embeddings from Milvus (text + vision)
Step 4: Merge back-to-back findings
Step 4.5: ⭐ NEW - Pre-merge references (if extra_doc)
Step 5: Create AI name and description (LLM)
Step 6: ⭐ CHANGED - Delete NOT_MATCH findings (if extra_doc)
Step 7: Save to database
```

### 2. `app/ai/ai_agents/ai_finding.py` - New Methods

#### `merge_references_before_description()` (NEW)
```python
# Purpose: Combine old + new references BEFORE LLM description generation
# 
# Logic:
# 1. Get existing findings for this RFX
# 2. For each new finding's requirement:
#    - If existing is NOT_MATCH → mark for deletion, use only new refs
#    - If existing has content → fetch old refs, add to DataFrame
# 3. Return combined DataFrame for LLM to process
```

#### `delete_not_match_findings()` (NEW)
```python
# Purpose: Clean up old findings after new ones are generated
#
# Logic:
# - Delete findings marked in merge_references_before_description
# - Prevents duplicate findings in database
```

### 3. `app/ai/prompts/ai_findings.py` - Conflict Handling Prompt

```
**HANDLING CONFLICTING VALUES FROM MULTIPLE DOCUMENTS**:
- When different documents provide different values for the same specification:
- DO NOT group by document
- Instead, integrate by specification item
- Format: "Maximum sound pressure level: 85 dB(A) from Document A, 75 dB(A) from Document B"
- NOT: "Document A specs: 85 dB... Document B specs: 75 dB..."
- Show conflicts inline for each spec item
```

### 4. `app/db/dao/ai_findings_embeddings_dao.py` - New DAO Method

#### `get_references_by_finding_id()` (NEW)
```python
# Purpose: Fetch all embedding references for a specific AI finding
# Used by: merge_references_before_description to get old refs
```

---

## Data Flow Example

### Scenario: User has Doc1, uploads Doc3 (additional)

```
BEFORE (Doc1 only):
┌─────────────────────────────────────────────────────────┐
│ Finding: "Acoustic Enclosure"                           │
│ Description: "Max pressure: 85 dB, Rating: IP54..."     │
│ References: [Doc1 chunk1, Doc1 chunk2]                  │
└─────────────────────────────────────────────────────────┘

AFTER (Doc1 + Doc3):
┌─────────────────────────────────────────────────────────┐
│ Finding: "Acoustic Enclosure"                           │
│ Description: "Max pressure: 85 dB from Doc1,            │
│              75 dB from Doc3. Rating: IP54 from Doc1,   │
│              IP65 from Doc3..."                         │
│ References: [Doc1 chunk1, Doc1 chunk2, Doc3 chunk1]     │
└─────────────────────────────────────────────────────────┘
```

---

## Why This Approach?

### Previous Approach (Post-merge)
- ❌ Generated description from new doc only
- ❌ Then tried to merge descriptions after
- ❌ Lost context, inconsistent formatting

### New Approach (Pre-merge) 
- ✅ LLM sees ALL references from ALL documents
- ✅ Single pass generates unified description
- ✅ Inline conflict attribution (spec-by-spec)
- ✅ Consistent formatting

---

## Files Changed Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `app/ai/service.py` | Modified | Added Step 4.5, changed Step 6 |
| `app/ai/ai_agents/ai_finding.py` | Added | `merge_references_before_description()`, `delete_not_match_findings()` |
| `app/ai/prompts/ai_findings.py` | Added | Conflict handling prompt instructions |
| `app/db/dao/ai_findings_embeddings_dao.py` | Added | `get_references_by_finding_id()` |

---

## Testing Scenarios

1. **Initial Upload**: Doc1 → Creates findings with Doc1 refs only
2. **Additional Upload**: Doc1 + Doc3 → Merges old refs + new refs, shows conflicts inline
3. **NOT_MATCH Replacement**: If existing was NOT_MATCH, delete old finding, create new with only new refs
4. **DIRECT/RELATED Merge**: If existing had content, merge all refs into new finding, delete old finding

**Note**: This PR treats revisions the same as additional uploads - it merges all references together. Special revision filtering (replacing old doc refs) is NOT implemented in this PR.

---

## What This PR Does NOT Include

| Feature | Status | Notes |
|---------|--------|-------|
| Revision filtering by `original_document_id` | ❌ Not included | Revisions merge all refs like additional uploads |
| UPDATE existing findings (preserve user edits) | ❌ Not included | Old findings are DELETED, new ones CREATED |
| Embedding ID fix for merged chunks | ❌ Not included | Was in older PR, not this one |
| Batch Milvus calls optimization | ❌ Not included | Individual `get_embedding` calls per ref |


