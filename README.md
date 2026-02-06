<div align="center">
<img width="1200" height="475" alt="GHBanner" src="https://github.com/user-attachments/assets/0aa67016-6eaf-458a-adb2-6e31a0763ed6" />
</div>

# Run and deploy your AI Studio app

This contains everything you need to run your app locally.

View your app in AI Studio: https://ai.studio/apps/drive/1f9b3R5HfNhi9UlyscfLe25qnpEj5RhJR

## Run Locally

**Prerequisites:**  Node.js


1. Install dependencies:
   `npm install`
2. Set the `GEMINI_API_KEY` in [.env.local](.env.local) to your Gemini API key
3. Run the app:
   `npm run dev`



# Architecture Documentation: Musahama Discovery Engine (v2.18)

## 1. Executive Summary
The **Musahama Discovery Engine** is a proprietary financial intelligence platform engineered for **WorldBC.co**. It automates the discovery, vetting, and engagement of high-value targets (Investors, Partners, Suppliers) for large-scale capital raising and procurement mandates.

The system is designed as a **"Human-in-the-Loop" AI Agent**. It does not merely scrape data; it actively reasons, formulates strategies, iteratively "hardens" its search parameters based on user feedback, and generates deep psychometric dossiers for decision-makers.

---

## 2. Core Technology Stack

### Frontend Application
- **Framework:** React 19 (ESM Architecture via `esm.sh`)
- **Language:** TypeScript
- **Styling:** Tailwind CSS (Atomic Utility System)
- **State Management:** React Hooks (`useState`, `useEffect`) + LocalStorage Persistence.
- **Build System:** No-Build (Browser-native ES Modules) or Vite-compatible structure.

### Intelligence Layer
- **Primary AI SDK:** `@google/genai` (v1.33+)
- **Models:** 
  - `gemini-3-flash-preview`: High-speed search grounding, initial discovery, and rapid iteration.
  - `gemini-3-pro-preview`: Complex reasoning, deep dossier generation, and "Thinking" tasks (Budget: 8k-16k tokens).
- **Enrichment Services:**
  - **Unipile API:** LinkedIn data extraction and activity tracking.
  - **Google Knowledge Graph:** Official entity verification.
  - **Google Custom Search (CSE):** Real-time entity verification and public email detection.
    - **Lusha Prospecting API:** Configured for **Contact-First Enrichment** (Cost Optimized). Used primarily to reveal emails/mobiles, bypassing company firmographics to save credits.


### Backend & Integration
- **Serverless Backend:** Google Apps Script (GAS) deployed as Web Apps (`doPost` handlers).
- **Database:** Google Sheets / AppSheet (via GAS Dispatcher).
- **File Storage:** Google Drive (via GAS `DriveApp` for Workspace backups).

---

## 3. System Design Patterns

### 3.1 The Intelligence Pyramid
Data flows through the system in stages of increasing fidelity:
1.  **Draft (Stage A):** Raw Project/Mandate ingestion.
2.  **Decomposition (Stage B):** AI analyzes the mandate to generate distinct **Investor Categories** (Personas).
3.  **Discovery Strategy (Stage C):** Iterative generation of search queries. Strategies are **"Hardened"** by `RefinementRules` (e.g., "Exclude Retail Banks", "Must be Shariah Compliant").
4.  **Discovery & Analysis (Stage D):** Google Search + AI processing to find entities.
5.  **Target Selection (Stage E):** High-fidelity enrichment (Lusha/Unipile), Psychometric Profiling, and CRM Sync.

### 3.2 The "Circuit Breaker" AI Caller
Located in `services/utils.ts`, the `callAI` function is the critical stability layer:
- **Strict Validation:** Ensures valid API keys before requests.
- **Model Fallback:** If `gemini-3-pro` hits a 429 Quota limit, it automatically retries with `gemini-3-flash`.
- **Cool-down Logic:** Maintains a global cooldown timer for Pro models to prevent "Hard Stop" errors from Google.
- **Hallucination Safety:** The `ensureString` utility flattens complex AI JSON objects into readable text strings to prevent React rendering crashes ("Error #31").

### 3.3 Dynamic Schema Mapping (The "Sync Engine")
Located in `services/integration.ts`.
- **Concept:** Intelligent Module Resolution + Dependency-Aware Sorting.
- **Problem:** CRM tables have relational dependencies (e.g., a Contact MUST reference an existing Company).
- **Solution:** 
    1.  **Module Mapping:** The system scans configured `TableMappings` for column signatures (e.g., if a table has `lmc_fit.score`, it identifies as a "Match Table").
    2.  **Topology Sort:** It prioritizes Parent tables (Companies) before Child tables (Contacts/Matches).
    3.  **FK Injection:** The system generates deterministic IDs (`comp_leadId`) to satisfy constraints if a company profile hasn't been fully enriched before a contact sync.

### 3.4 Global Strategy Governance & Constraint Sync
A hierarchical constraint management system located in `services/discovery.ts` and `views/SourcePlanningView.tsx`.
- **Global Scope:** `RefinementRules` and `ExclusionLists` defined at the Project level apply automatically to *all* child Investor Categories.
- **Bi-Directional CRM Sync:**
    - **Fetch:** On Ingest, if a `LeadID` is selected, the system pulls existing global constraints from the CRM `ResearchRequestConstraints` table.
    - **Push:** When a user adds or removes a Global Rule in the app, it triggers an immediate sync (`Add` or `SoftDelete`) to the CRM, ensuring the strategy is recorded remotely.

### 3.5 Deterministic Identity Management
To prevent duplicate records in the CRM when a user syncs a lead multiple times (e.g., after re-enrichment), the system enforces deterministic ID generation:
- **Company ID:** `comp_{lead_id}`
- **LMC Match ID:** `lmc_{lead_id}`
- **Psych Profile ID:** `psy_{lead_id}`
This ensures 1:1 mapping between the React app's state and the CRM rows.

### 3.6 Autopilot Agent (Autonomous Discovery Loop)
Located in `views/CategoryWorkflow.tsx`.
- **State Machine:** Operates on a cycle of `SEARCHING` -> `ANALYZING` -> `HARDENING` -> `COOLDOWN`.
- **Goal Seeking:** Automatically continues the search loop until a user-defined number of **Shortlisted Candidates** (e.g., 20) is reached.
- **Self-Correction:** Automatically analyzes rejected leads from each batch to identify patterns, generating and auto-approving new `RefinementRules` to improve the next batch.

### 3.7 Manual Truth Injection
Located in `views/SelectedTargetsView.tsx`.
- **Logic:** Allows users to manually correct Company Names, Websites, and Contact details if AI discovery is ambiguous.
- **Signal:** Manually edited fields are flagged with `verification_status: 'VERIFIED'`.
- **Impact:** Downstream enrichment tasks (e.g., Lusha Contact Search) prioritize this manually verified data over API-guessed data, increasing hit rates.

---

## 4. Core Modules & Responsibilities

### 4.1 Discovery Service (`services/discovery.ts`)
- **`analyzeProject`**: Decomposes a raw text description into structured Personas.
- **`executeCategorySearch`**: The main loop. Uses Google Search Grounding to find entities. It respects **Global Exclusions** and **Hardening Rules**.
- **`analyzeBatchFeedback`**: A meta-cognitive loop. Analyzes *why* a user rejected a batch of leads and proposes new Constraints (Rules) to fix the strategy.
- **`findCoInvestors`**: Network Expansion logic. Given an anchor lead, finds who they co-invest with.

### 4.2 Diligence Service (`services/diligence.ts`)
- **`analyzeAndScoreLead`**: Calculates the fit score (0-100). Uses **Dual-Track Reasoning**:
    - *Track A (Corporate):* Does the entity's mandate match?
    - *Track B (Personal):* Does the individual (HNWI) have an angel track record?
- **`findCompanyInfo`**: Company Verification Workflow (Cost Optimized):
    1.  **Google Knowledge Graph (KG):** Primary Source of Truth for Official Name, Website, and Description (Free/Low Cost).
    2.  **Lusha Company Enrichment:** **ENABLED (Secondary)**. Used primarily for firmographic data (Employee count, Revenue) if KG lacks it. Configured with strict page size limits to prevent API errors.
    3.  **Unipile (LinkedIn):** Social context and "About" section (Fallback).
    4.  **AI Synthesis:** Gemini merges the above into a single `CompanyProfile`.
- **`findContactInfo`**: Contact Discovery Workflow (Direct Mode):
    1.  **Lusha (Prospecting API):** **Direct Search**. Do The pre-flight company check.
    2.  **Google CSE:** Searches for public email patterns on company pages.
    3.  **Unipile:** LinkedIn Profile scrape (Fallback).
    4.  **AI Inference:** Web pattern matching (Last Resort).
- **`findVideoIntent`**: Searches video platforms (YouTube, Bloomberg) for "Verbal Intent" (Direct quotes from the target confirming their investment strategy).

### 4.3 Intelligence Service (`services/intelligence.ts`)
- **`generatePsychProfile`**: 
    - Analyzes text footprint (LinkedIn, Emails, Articles).
    - **Constraint:** Strictly classifies into one of 12 specific DISC types (e.g., "DS (The Producer)").
    - Generates tactical "How to Sell" advice.
- **`generateLMCFit`**: Lead-Matched-Company logic. Generates the "Why Fit" narrative and identifies Gaps.

### 4.4 Backend Gateway (Google Apps Script)
The frontend communicates with a GAS deployment.
- **`CRM_SCRIPT_URL`**: Handles SQL-like operations (Add/Edit rows) to AppSheet.
- **`CLOUD_BACKUP_URL`**: Handles file operations.
    - `syncWorkspace`: Saves the entire React state (`ResearchSession[]`) to a JSON file in a specific Google Drive Folder.
    - `getWorkspace`: Retrieves the latest backup.

---

## 5. Quality Assurance & Testing

The system includes a built-in **Diagnostics Console** (`components/TestDashboard.tsx`) available via the Integrations menu.

### 5.1 Test Categories
1.  **Core Utilities**: Validates JSON sanitization (`cleanJson`) and React rendering safety (`ensureString`) to prevent frontend crashes from hallucinated AI output.
2.  **Discovery Logic**: Checks TypeScript interface integrity and deep object access paths.
3.  **Data Integrity**: Verifies CSV escape logic and Cloud Backup serialization/deserialization.
4.  **Integration Payloads**: Mocks the CRM Sync payload generation to ensure fields (like `lmc_fit.score`) are correctly mapped to flat JSON structures for AppSheet.

### 5.2 Execution Strategy
- Tests are run entirely client-side using mocks.
- `tests.ts` contains the logic for the "Dry Run" execution.
- The Dashboard visualizes the sequence of operations (e.g., Lead Generation -> Scoring -> CRM Sync).

---

## 6. Data Models (`types.ts`)

### `LeadCandidate`
The central object representing a target.
- `lead_id`: Unique system ID.
- `pipeline_status`: `DISCOVERED` -> `ANALYZED` -> `SELECTED`.
- `company_profile`: Enriched corporate data.
- `psych_profile`: Enriched personality data.
- `evidence`: Array of `EvidenceItem` (News, Video, Tweets).

### `ProjectProfile`
The root mandate object.
- `crm_lead_id`: The ID of the parent record in the CRM (ResearchRequest).
- `globalRules`: Array of `RefinementRule` applied to all strategies.
- `globalExclusions`: Array of strings (entity names) to be strictly avoided.

### `InvestorCategory`
Represents a "Persona" or "Strategy".
- `masterStrategy`: Contains the `basePrompt` and array of `RefinementRule`s (Local).
- `searchParams`: Geo/Language constraints.

### `IntegrationSettings`
- Stores API Keys (Lusha, Unipile, AppSheet).
- Stores `tableConfigs`: The mapping definitions for the CRM Sync.

---

## 7. Change Log Summary (Recent)
- **v2.18**: **Autopilot & Manual Truth**. Added Autonomous Discovery Mode (self-driving search loop) and Manual Verification Overrides for entity data correction.
- **v2.17**: **Verification Sync & Documentation Alignment**. Updated Logic description to reflect that Lusha Company Enrichment is active (Secondary) for firmographics, not disabled. Bumped all UI version indicators.
- **v2.16**: **Verified "Google-First" Architecture**. Refactored `diligence.ts` to strictly prioritize Google Knowledge Graph and Search for company verification to conserve credits. Lusha API calls fixed (page size 10) and re-routed to Contact Discovery only.
- **v2.15**: **Cost Optimization & Network Fixes**. Lusha integration refactored to disable Company Enrichment (saving 1 credit) and skip "Pre-flight" checks during Contact Discovery.
- **v2.13**: Enhanced UI Background with textured gradients and ambient orbs. Added "Untitled Draft" deletion capability for cleaner workspace management.
- **v2.12**: Enforced strict enrichment hierarchy (KG -> Lusha -> Unipile) and added robust error handling for Lusha API fetch failures.
- **v2.11**: Integrated Lusha Prospecting API (Search + Enrich) & Google Knowledge Graph.
- **v2.10**: Global Constraints CRM Sync & Editable Lead ID.
- **v2.9**: Added Global Constraints & Exclusion Lists. Project Data is now editable post-ingest.
- **v2.8**: Added System Diagnostics Dashboard for comprehensive unit testing and health checks.
- **v2.7**: Robust CRM Sync with Dependency Sorting & Module Auto-Resolution.
- **v2.6**: Implemented `Contact_Psy_Proflie` schema support with specific DISC logic. Added Schema Configuration Download.
- **v2.5**: Network Expansion (Co-Investor) module.
- **v2.4**: Circuit Breaker for AI Quota management.
- **v2.2**: High-Fidelity PDF Dossier Export (`printDossier`).

---

*Confidential Architecture - WorldBC.co Engineering Team*
