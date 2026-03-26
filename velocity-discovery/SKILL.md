---
name: velocity-discovery
description: Master coordination and recording controller for the Discovery Optimization framework. Manages the 7-day sprint by sequencing nested skills and maintaining the interview state.
---

# Discovery Orchestrator (The Cortex Engine)

## 1. Skill Registry & Global Locations
To execute the discovery sprint, this orchestrator references the following local skill files and global manifests:

* **Agnostic Foundation & Recorder:** `./industry-agnostic/SKILLS.md`
* **Industry Deep-Dive & Prober:** `./Industry-specialist/SKILLS.md`
* **Strategic Alignment & PPT Gen:** `./Strategist/SKILL.md`
* **Data Readiness & Pre-EDA:** `./Data-Readiness/SKILL.md`
* **Global Agent Manifest:** `../../../AGENTS.md` (Workspace root)

## 2. Meeting Transcript Database (Persistent Knowledge Source)

The framework uses an **unstructured** Meeting Transcript Database in Snowflake. Transcripts are loaded as-is (raw text, JSON, or any format) into a single VARIANT column. All intelligence extraction (client name, sector, topics, action items) happens at query time via Snowflake's semi-structured functions and Cortex LLM functions.

### Fixed Location

```
Database:  VELOCITY_DISCOVERY
Schema:    DISCOVERY
Table:     MEETING_TRANSCRIPTS
```

### Table Schema

```sql
VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS
├── MEETING_ID      VARCHAR    (DEFAULT UUID_STRING()) -- Auto-generated unique ID
├── TRANSCRIPT_RAW  VARIANT    -- Raw transcript loaded as-is (text, JSON, any format)
└── LOADED_AT       TIMESTAMP_NTZ (DEFAULT CURRENT_TIMESTAMP()) -- When the record was inserted
```

The `TRANSCRIPT_RAW` column holds the entire raw meeting content. It may contain:
- Plain text transcripts (stored as a VARIANT string)
- JSON objects with structured fields (speaker, timestamp, text)
- Session state snapshots (persisted at end of conversation)
- Any other format — the skill must parse whatever it finds

### Transcript Query Protocol

At the start of every phase, the active skill MUST:
1. Query the transcript DB and scan `TRANSCRIPT_RAW` for content matching the current client/engagement.
2. Use Snowflake string functions or Cortex LLM functions to extract relevant context from the raw content (client name, sector, pain points, decisions, KPIs, etc.).
3. Pre-populate the phase's question framework with extracted answers — mark them as `[FROM TRANSCRIPT]`.
4. Only ask the client questions whose answers are NOT found in the transcript DB.
5. If the transcript contains conflicting information across meetings, flag the conflict and ask the client to clarify.

```sql
-- Load all transcripts (scan raw content for client references)
SELECT MEETING_ID, TRANSCRIPT_RAW, LOADED_AT
FROM VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS
ORDER BY LOADED_AT DESC;

-- Search for a specific client within raw transcript content
SELECT MEETING_ID, TRANSCRIPT_RAW, LOADED_AT
FROM VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS
WHERE TRANSCRIPT_RAW::VARCHAR ILIKE '%<CLIENT_NAME>%'
ORDER BY LOADED_AT DESC;

-- Insert a new raw transcript (loaded as-is)
INSERT INTO VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS (TRANSCRIPT_RAW)
SELECT PARSE_JSON($$<RAW_TRANSCRIPT_CONTENT>$$);

-- Insert plain text transcript
INSERT INTO VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS (TRANSCRIPT_RAW)
SELECT TO_VARIANT('<RAW_TEXT_TRANSCRIPT>');
```

## 3. Persistent Session State (Cross-Phase Memory)

All information captured during the discovery sprint MUST persist across phases and conversations. The orchestrator maintains a structured session state that every sub-skill reads from and writes to. Session state is persisted back to the same `MEETING_TRANSCRIPTS` table as a VARIANT JSON object.

### State Architecture

```
SESSION_STATE (in-memory during conversation, persisted to TRANSCRIPT_RAW at end)
├── _type: "DISCOVERY_SESSION_STATE"   -- Identifies this record as session state (not a raw transcript)
├── engagement_id           -- Unique engagement identifier
├── client_profile          -- From Phase 0 Step A (company, role, size, geography)
├── sector_context          -- From Phase 0 Step B (industry, sub-sector, regulatory)
├── use_case_brief          -- From Phase 0 Step C (use case, trigger, success metric)
├── foundation_capture      -- From Phase 1 (operational, scaling, budget findings)
├── industry_deep_dive      -- From Phase 2 (sector-specific gaps, compliance, benchmarks)
├── strategic_alignment     -- From Phase 3 (4-pillar loop answers, open loops)
├── data_readiness          -- From Phase 4 (gate results, schema profiles, quality signals)
├── transcript_references   -- List of MEETING_IDs consumed during this sprint
├── open_loops              -- Unanswered questions aggregated across all phases
└── last_updated            -- Timestamp of last state mutation
```

### Persistence Rules
- **Within a conversation:** All phases share the same session state object. Data captured in Phase 0 is immediately available to Phase 1, 2, 3, and 4 without re-asking.
- **Across conversations:** At the end of each conversation, the orchestrator inserts the session state as a new row in the transcript DB (with `_type: "DISCOVERY_SESSION_STATE"` in the VARIANT). At the start of a new conversation, the orchestrator queries for the latest session state and resumes.
- **No data loss:** If a conversation is interrupted, the state captured up to that point is preserved. The next conversation picks up from the last known state.
- **Deduplication:** Before asking any question, every sub-skill MUST check both the session state AND the transcript DB. If the answer exists in either, skip the question and note `[ALREADY CAPTURED]`.

### State Read & Write Queries

```sql
-- Read: Get the latest session state for a client
SELECT TRANSCRIPT_RAW, LOADED_AT
FROM VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS
WHERE TRANSCRIPT_RAW:"_type"::VARCHAR = 'DISCOVERY_SESSION_STATE'
  AND TRANSCRIPT_RAW:"client_profile":"company_name"::VARCHAR ILIKE '%<CLIENT_NAME>%'
ORDER BY LOADED_AT DESC
LIMIT 1;

-- Write: Persist session state at end of conversation
INSERT INTO VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS (TRANSCRIPT_RAW)
SELECT PARSE_JSON($${
  "_type": "DISCOVERY_SESSION_STATE",
  "engagement_id": "<ID>",
  "client_profile": { ... },
  "sector_context": { ... },
  "use_case_brief": { ... },
  "foundation_capture": { ... },
  "industry_deep_dive": { ... },
  "strategic_alignment": { ... },
  "data_readiness": { ... },
  "transcript_references": [ ... ],
  "open_loops": [ ... ],
  "last_updated": "<TIMESTAMP>"
}$$);

-- Read: Get all raw transcripts (excluding session state records)
SELECT MEETING_ID, TRANSCRIPT_RAW, LOADED_AT
FROM VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS
WHERE TRANSCRIPT_RAW:"_type" IS NULL
   OR TRANSCRIPT_RAW:"_type"::VARCHAR != 'DISCOVERY_SESSION_STATE'
ORDER BY LOADED_AT DESC;
```

## 4. Lifecycle & Interview Management (The 7-Day Sprint)

- **Phase 0: Intake (Conversation Start — MANDATORY FIRST STEP):**
    * *State:* **INTAKE**.
    * **Step 0 — Transcript Pre-Scan:** Before asking the client anything, query `VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS` for any prior meetings or session state matching the client name or engagement context. Parse `TRANSCRIPT_RAW` to extract known data. Pre-populate the session state.
    * **Step A — Customer Profile (ask ONLY if not found in transcripts):**
        - Company/Organization name
        - Primary contact name & role
        - Company size (employees, customers, revenue band)
        - Geographic footprint (regions/countries of operation)
    * **Step B — Sector Context (ask ONLY if not found in transcripts):**
        - Industry sector (e.g., Banking, Insurance, Fintech, Wealth Management)
        - Sub-sector or line of business (e.g., Retail Banking, Commercial Lending, P&C Insurance)
        - Regulatory environment summary (e.g., OCC-regulated, state-chartered, EU-licensed)
    * **Step C — Use Case Brief (ask ONLY if not found in transcripts):**
        - Primary use case in one sentence (e.g., "Predict customer churn and automate retention offers")
        - What triggered this initiative now? (e.g., board mandate, competitive pressure, regulatory change)
        - Desired outcome / success metric (e.g., "Reduce net churn by 15% by Q4")
    * **Transition Rule:** Summarize ALL captured data (from transcripts + client responses) back to the client for confirmation. Do NOT proceed to Phase 1 until confirmed. Write confirmed intake to session state.

- **Phase 1: Capture (Day 1-2):** Trigger `industry-agnostic/SKILLS.md`.
    * *State:* **RECORDING START**. Read session state for pre-populated answers. Focus on baseline operational, scaling, and budgetary metrics. Skip questions already answered in transcripts.
- **Phase 2: Synthesis (Day 3-4):** Trigger `Industry-specialist/SKILLS.md`.
    * *State:* **GAP ANALYSIS**. Read session state + transcript DB. Use "most-asked" industry data to probe for missing regulatory/technical requirements. Cross-reference transcript history for previously discussed compliance topics.
- **Phase 3: Delivery (Day 5-7):** Trigger `Strategist/SKILL.md`.
    * *State:* **FINAL ALIGNMENT**. Read full session state. Close all recorded open loops and execute the automated PowerPoint generation.
- **Phase 4: Data Gate (Day 7):** Trigger `Data-Readiness/SKILL.md`.
    * *State:* **DATA GATE & PRE-EDA**. Read session state for discussed tables and use cases. Validate that discussed data actually exists in Snowflake, profile schemas, map data flow, assess quality signals, and produce a Pre-EDA Readiness Report. Also scan the transcript DB itself as a potential data source if meeting content is relevant to the use case.

## 5. Data Flow & Recording Integration
- **Transcript Pre-Load:** At sprint start, query `VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS` and load all relevant meeting history into the session state. This is the FIRST action before any human interaction.
- **Intelligent Capture:** Orchestrate the raw session ingestion. Ensure the **Architect** is logging foundational data into `./logs/foundation_capture.json` and updating the session state.
- **Contextual Synthesis:** Pass the Architect's log + session state to the **Sector Analyst**. Record and append vertical-specific anomalies to the session record.
- **Final Delivery:** Consolidate all recorded logs into a single `final_synthesis.json` and pass it to the **Strategist** for PPT creation.
- **Data Validation:** Pass `final_synthesis.json` to the **Data Readiness** engine. It reads the synthesis + session state to identify which tables and use cases were discussed, then validates against the live Snowflake environment. Output: `pre_eda_report.json`.
- **State Persistence:** At conversation end, insert the complete session state as a new VARIANT row into `VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS`.

## 6. File Delivery Protocol (MANDATORY)

When generating any file deliverable (PDF, PPTX, CSV, etc.) from a notebook, the **ONLY** supported delivery method is the **Base64 Table Approach**. Presigned URLs and scoped URLs do NOT produce downloadable files reliably in this environment.

### Required Steps

1. **Generate the file** in the notebook and write it to `/tmp/`.
2. **Base64-encode** the file contents in Python.
3. **Store the base64 string** in a Snowflake table:
   ```sql
   CREATE OR REPLACE TABLE <DATABASE>.<SCHEMA>.FILE_DOWNLOADS (
       FILENAME VARCHAR,
       CONTENT VARCHAR
   );
   INSERT INTO FILE_DOWNLOADS VALUES ('<filename>', '<base64_string>');
   ```
4. **Instruct the user** to run the SELECT query and decode locally:
   ```sql
   SELECT CONTENT FROM <DATABASE>.<SCHEMA>.FILE_DOWNLOADS WHERE FILENAME = '<filename>';
   ```
   Then decode locally with Python:
   ```python
   import base64
   b64_string = "..."  # paste query result
   with open("<filename>", "wb") as f:
       f.write(base64.b64decode(b64_string))
   ```

### Prohibited Methods (DO NOT USE)
- `GET_PRESIGNED_URL()` — downloads corrupt files in this environment
- `BUILD_SCOPED_FILE_URL()` — same issue
- `GET @stage` — not supported in Snowsight SQL worksheets
- Direct file write to workspace — workspace filesystem is read-only from notebook kernels

### Reference Implementation (Python notebook cell)
```python
import base64
from snowflake.snowpark.context import get_active_session

with open('/tmp/<filename>', 'rb') as f:
    data = f.read()
b64 = base64.b64encode(data).decode('ascii')

session = get_active_session()
session.sql("CREATE OR REPLACE TABLE <DB>.<SCHEMA>.FILE_DOWNLOADS (FILENAME VARCHAR, CONTENT VARCHAR)").collect()
session.sql(f"INSERT INTO FILE_DOWNLOADS VALUES ('<filename>', '{b64}')").collect()
print("File stored. Run: SELECT CONTENT FROM <DB>.<SCHEMA>.FILE_DOWNLOADS;")
print("Then decode locally with base64.")
```

## 7. Mandatory Quality Gates
- **Two-Week Drag Prevention:** Flag as **Critical** if any phase (Capture, Synthesis, or Delivery) exceeds a 24-hour processing window.
- **Probing Check:** If the **Specialist** flags a missing "most-asked" requirement, the Orchestrator MUST block the "Delivery" phase until a recorded answer is obtained.
- **Deliverable Integrity:** Validate that the `./output/discovery_report.pptx` is generated successfully via the workspace root script.
- **Data Gate Enforcement:** The **Data Readiness** phase MUST pass all three gates (database existence, schema existence, table viability) before the engagement can proceed to EDA or modeling. If the gate fails, the Orchestrator blocks downstream work and surfaces the Gap Report to stakeholders.
- **Transcript Integrity:** Before asking any question, verify it hasn't already been answered in the transcript DB or session state. Redundant questioning is flagged as a **Quality Violation**.
- **State Persistence Gate:** The orchestrator MUST NOT close a conversation without writing the session state back to `VELOCITY_DISCOVERY.DISCOVERY.MEETING_TRANSCRIPTS`. If the write fails, flag as **Critical** and retry.

## 8. Global Agent Callback
If a conflict arises during the recording (e.g., a Budget/Scale anomaly), invoke the agent definitions in `../../../AGENTS.md` to re-assign the "Cortex Prime" persona for immediate stakeholder intervention.
