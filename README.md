# Agent-ClaimAlly 
### Problem Statement — the problem you're trying to solve, and why it matters

Patients in the U.S. lose billions of dollars each year to wrong or unfairly denied medical bills.

The core problems:

- Bills are full of CPT/ICD-10 codes, jargon, and fine print that most patients can’t interpret.  
- Insurance policies are long PDF documents with buried coverage rules.  
- Appeals require structured, legalistic letters that reference exact policy clauses and billing codes.  
- Patients are effectively fighting against automated denial systems without comparable tools, time, or expertise.

The Patient Advocate agent is designed to level the playing field: it helps a patient understand their bill, compare it to their own insurance policy, and generate a strong, policy-grounded appeal letter when a denial looks incorrect.

---

### Why agents? — why agents are the right solution

This problem is naturally an assembly line of expert tasks:

- Read and interpret a complex, multimodal input (bill image + policy PDF).  
- Cross-check the bill against policy rules and past context (deductibles, history).  
- Draft a formal, high-stakes appeal letter.

Agents fit because:

**Specialization:**

- An Auditor agent is optimized for analytical, fact-finding work.  
- An Advocate agent is optimized for writing clear, persuasive, legalistic appeals.  

- Sequential workflow: Using a SequentialAgent guarantees the policy is checked first, and only then is a letter generated.  
- Tool use: Agents can reliably call tools like `analyze_bill_image` and `policy_rag_lookup` instead of “hallucinating” coverage rules.  
- Long-term memory: A memory service lets the system remember deductibles, prior authorizations, and patient details across sessions, reducing user friction.  
- Safety: A guardrail plugin ensures the agent stays in its lane (billing advocacy) and does not drift into medical diagnosis.

---

### What you created — overall architecture

**Project Name:** ClaimAlly 

**High-level architecture:** Hierarchical & Sequential Multi-Agent System

#### Input

User uploads:

- Image of a medical bill (JPEG/PNG).  
- PDF of their insurance policy.  

#### Agent A – “The Forensic Auditor”

- **Type:** Agent  

**Responsibilities:**

- Call `analyze_bill_image(image_path)` to extract:
  - Service date  
  - CPT codes  
  - Billed amounts  
  - Denial reason  
- Call `policy_rag_lookup(query)` to search coverage clauses in the policy PDF.  
- Call `load_memory` to pull patient-specific context (deductible status, pre-authorizations).  
- Produce a structured Discrepancy Report explaining where the denial conflicts with policy language.  

- **Output key:** `audit_report`

#### Memory Service – “The Filing Cabinet”

- **Implementation:** `InMemoryMemoryService` (swappable later for `VertexAiMemoryBankService`).  

**Stores:**

- Patient identifiers (name, policy ID).  
- Deductible status & relevant history.  

Accessed via the `load_memory` tool.

#### Agent B – “The Advocate”

- **Type:** Agent  

**Responsibilities:**

- Read `{audit_report}` from Agent A.  
- Draft a formal Appeal Letter:
  - Cites specific policy sections from the RAG lookup.  
  - References CPT codes and dates of service.  
  - Uses a firm, professional, legalistic tone.  
  - Outputs text ready for PDF export.  

- **Output key:** `final_appeal_letter`

#### Orchestrator – “PatientAdvocateWorkflow”

- **Type:** `SequentialAgent`  
- `sub_agents = [auditor_agent, advocate_agent]`  

**Enforces the pipeline:**

- Run Auditor → produce `audit_report`.  
- Pass `audit_report` into Advocate → produce `final_appeal_letter`.  

#### Safety Plugin – “MedicalAdviceGuardrail”

- **Type:** `BasePlugin`  
- **Hook:** `before_agent_callback`  

**Behavior:**

- Checks user input for medical-diagnosis language (e.g., “diagnose”, “symptoms”, “lump”).  
- If triggered, raises an error and redirects the user away from medical advice toward consulting a physician.  

---

### Demo — how the solution works from a user’s perspective

#### Upload step

The user uploads a photo of their medical bill and the PDF of their insurance policy into the interface.

#### Audit phase

The Auditor agent:

- “Reads” the bill via `analyze_bill_image`.  
- Looks up coverage rules for those CPT codes via `policy_rag_lookup`.  
- Uses `load_memory` to check whether the deductible is already met.  
- Outputs a Discrepancy Report, e.g.:

> “CPT 99214 is listed as ‘Not Medically Necessary’, but Section 4.2 states CPT 99201–99215 are covered 100% after deductible for acute symptoms, which are documented in the visit notes.”

#### Advocacy phase

The Advocate agent:

- Consumes the `audit_report`.  
- Generates a polished appeal letter addressed to the insurance company:
  - Includes patient details, claim number, dates of service.  
  - Quotes the exact policy section returned by the RAG tool.  
  - Argues clearly why the denial should be overturned.  

Returns the `final_appeal_letter`, ready for the user to:

- Download as PDF or  
- Paste into their insurer’s appeal portal.  

#### Safety checks

If the user asks, for example, “Can you diagnose this pain?” the `MedicalAdviceGuardrail` intercepts the request and blocks the agent from answering with clinical advice.

---

### The Build — how you created it, tools & technologies used

#### Core stack

##### Agent framework

Google Agent Development Kit (ADK):

- `Agent`, `SequentialAgent` for orchestration.  
- `InMemoryMemoryService` for long-term context.  
- `BasePlugin` for safety guardrails and governance.  

##### Model

Gemini 2.5 Pro (via `Gemini(model="gemini-2.5-pro")`):

- Strong reasoning over long policy text.  
- Multimodal capabilities (bill image reading in a real deployment).  

##### Custom tools (Function Tool pattern)

- `analyze_bill_image(image_path: str) -> dict`  
  - Simulated multimodal extraction of:
    - Service date  
    - CPT code  
    - Billed amount  
    - Denial reason  

- `policy_rag_lookup(query: str) -> str`  
  - Simulated RAG lookup into the user’s insurance policy PDF, returning the exact clause text.  

- `load_memory`  
  - Reads from `InMemoryMemoryService` (e.g., whether the user’s deductible is met).  

#### Agents

- `auditor_agent`:
  - Uses all tools (`analyze_bill_image`, `policy_rag_lookup`, `load_memory`).  
  - Outputs structured `audit_report`.  

- `advocate_agent`:
  - Consumes `audit_report`.  
  - Produces `final_appeal_letter`.  

#### Orchestration

- `patient_advocate_system = SequentialAgent(...)`  
- Ensures deterministic order: **Audit → Advocacy**.  

#### Safety & governance

- `MedicalAdviceGuardrail` plugin:
  - Uses a keyword check (e.g., “diagnose”, “cancer”, “symptoms”) to prevent medical advice.  
  - Raises an exception and stops unsafe runs.  

---

### If I had more time, this is what I’d do

If I extend The Patient Advocate beyond this blueprint, I’d:

- **Replace simulated tools with production integrations**
  - Connect `analyze_bill_image` to real Gemini Vision or OCR for robust CPT/ICD-10 extraction.  

- **Build a true RAG pipeline:**
  - Chunk and embed the user’s policy PDF.  
  - Query a vector database (e.g., Vertex AI, Pinecone, or similar) for precise clause retrieval.  

- **Upgrade memory from in-memory to persistent**
  - Swap `InMemoryMemoryService` for a persistent memory layer (e.g., `VertexAiMemoryBankService` or a database) so:
    - Users can return over months and keep a longitudinal history of claims, deductibles, and appeals.  
    - The system can detect patterns of repeated denials.  

- **Add a “Claim Tracker” layer**
  - Track status: Submitted → Under Review → Approved/Denied.  
  - Generate follow-up letters or escalation templates when deadlines approach.  

- **Multi-language support**
  - Support appeals in Spanish, French, etc., while keeping policy citations and codes accurate.  

- **User interface + integrations**
  - Simple web UI where users drag-and-drop bills and policies.  
  - Optional integration with insurer portals (where allowed) to pre-fill appeal forms.  

That roadmap would turn The Patient Advocate from a working capstone prototype into a powerful, real-world tool for helping patients fight unfair medical bills.
