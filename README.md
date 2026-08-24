# RevenueOS v1: AI Lead Qualification & Routing

## The Business Problem

At **Lolia**, a Series B AI-powered Revenue Intelligence SaaS company, inbound lead volume had grown faster than the Sales team could manually qualify and route every prospect.

This created two operational problems:

1. **Sales reps were spending valuable selling time** researching and qualifying poor-fit leads.
2. **High-value prospects were sitting in a pending queue**, delaying assignment and follow-up and slowing pipeline velocity.

The underlying problem was not simply lead volume—it was the lack of an automated process for turning raw inbound CRM data into an actionable sales decision.

---

## The Solution

**RevenueOS v1** automates the inbound lead qualification and routing process inside HubSpot.

Instead of requiring a salesperson to manually review every inbound lead, the workflow:

1. Retrieves pending lead data from HubSpot.
2. Normalizes the relevant CRM information.
3. Evaluates the prospect against Lolia's Ideal Customer Profile (ICP) using an AI qualification engine.
4. Produces an **AI-generated qualification score, qualification status, ICP match, and reasoning**.
5. Routes qualified prospects into the appropriate sales segment:

   * Commercial
   * Mid-Market
   * Enterprise
6. Assigns the appropriate sales representative.
7. Writes the qualification results and AI reasoning back to HubSpot.
8. Creates a follow-up task for the assigned sales representative.
9. Disqualifies poor-fit leads in HubSpot so they do not continue through the qualified-lead workflow.

### Workflow

**HubSpot → Data Normalization → AI Qualification → ICP Decision → Sales Segment Routing → Rep Assignment → HubSpot Update → Follow-up Task**

This turns lead qualification from a manual queue-management process into an automated GTM decision system.

## Business Impact

RevenueOS v1 is designed to improve the operational efficiency of the inbound sales process by:

* Reducing manual lead research and qualification.
* Prioritizing qualified prospects for sales follow-up.
* Improving lead-to-rep routing consistency.
* Preventing poor-fit leads from consuming sales capacity.
* Ensuring qualification decisions and reasoning are recorded directly in the CRM.
* Reducing the time between inbound lead submission and sales action.

The key outcome is **faster movement from inbound lead → qualified opportunity**, while allowing sales representatives to spend more time selling rather than performing repetitive qualification work.

---

## System Architecture & Logic

RevenueOS v1 is an event-driven GTM automation workflow orchestrated through **n8n**. It integrates **HubSpot** as the CRM, **DeepSeek** as the AI qualification engine, and **PostgreSQL** for operational error logging.

The workflow follows a controlled processing pipeline:

**HubSpot Intake → Data Normalization → AI Qualification → Deterministic Routing → CRM State Update**

The architecture intentionally separates **AI-based evaluation** from **deterministic business logic**. The LLM evaluates the lead against Lolia's ICP, while explicit workflow rules control segmentation, ownership, and CRM actions.

### 1. Data Intake & Normalization

HubSpot provides the source lead and company data. Because CRM properties can contain inconsistent formats, the workflow normalizes key fields before they reach the AI layer.

A custom JavaScript node:

* Converts revenue strings such as `"$50M"` or `"50 million"` into numeric values such as `50000000`.
* Parses employee ranges such as `"100-500"` into `employee_min: 100` and `employee_max: 500`.
* Standardizes the input structure used by downstream qualification and routing logic.

This creates a consistent data contract between the CRM and the qualification engine.

### 2. AI Qualification Engine

The normalized lead data is passed to **DeepSeek** together with Lolia's defined Ideal Customer Profile (ICP).

The model is constrained to return a structured JSON response containing four fields:

* `lead_score` — qualification score from 0–100
* `qualification_status` — `Qualified` or `Unqualified`
* `icp_match` — Boolean ICP match
* `reasoning` — concise explanation of the decision

Example:

```json
{
  "lead_score": 85,
  "qualification_status": "Qualified",
  "icp_match": true,
  "reasoning": "VP of Sales at a B2B SaaS company with $50M revenue and 100-500 employees, matching ICP criteria."
}
```

The AI layer is responsible for **evaluating the prospect and providing contextual reasoning**. It does not independently control CRM actions.

### 3. Deterministic Sales Routing

If the AI classifies the lead as `Qualified`, the workflow applies explicit business rules to determine the appropriate sales segment.

The routing hierarchy is:

* **Enterprise:** revenue ≥ $100M **OR** employees ≥ 1,000
* **Mid-Market:** revenue ≥ $25M **OR** employees ≥ 100
* **Commercial:** all other qualified leads

The resulting segment is mapped to a predefined HubSpot Owner ID, allowing the workflow to automatically assign the lead to the appropriate sales representative.

This creates a deliberate separation of responsibilities:

**AI → qualification intelligence**

**Workflow logic → deterministic routing and operational decisions**

This reduces reliance on the LLM for decisions that should remain predictable, auditable, and reproducible.

### 4. CRM State Management

Once processing is complete, RevenueOS writes the resulting state back to HubSpot through the `ai_qualification_status` property.

This provides a persistent processing state and prevents already-processed leads from remaining in the pending queue.

#### Qualified Path

For qualified leads, the workflow:

* Writes the qualification score, ICP match, and AI reasoning to HubSpot.
* Determines the appropriate sales segment.
* Assigns the corresponding HubSpot owner.
* Changes the qualification status to `Qualified`.
* Creates a next-day follow-up task for the assigned sales representative.

#### Disqualified Path

For unqualified leads, the workflow:

* Writes the qualification results and reasoning to HubSpot.
* Changes the qualification status to `Disqualified`.
* Removes the lead from the pending qualification queue.
* Stops the lead from generating unnecessary sales activity.

The result is a CRM that reflects the current state of every processed lead rather than relying on a separate external tracking system.

### 5. Error Handling & Operational Logging

External APIs and LLM services can fail, return malformed responses, or behave unexpectedly. RevenueOS therefore includes an explicit error-handling path around the AI qualification stage.

When the DeepSeek request fails or returns invalid JSON, the workflow prevents the invalid response from continuing into the CRM update process.

Instead, the workflow captures the:

* Lead email
* Workflow stage
* Error message

and writes the information to a PostgreSQL `workflow_errors` table.

This creates an operational error log that can be used for monitoring, debugging, investigation, and future retry workflows.

### Architecture Principles

RevenueOS v1 is built around four core engineering principles:

1. **Normalize before inference** — inconsistent CRM data is cleaned before reaching the LLM.
2. **Constrain AI output** — the model returns a predictable machine-readable schema.
3. **Keep business rules deterministic** — segmentation and ownership decisions are controlled by explicit workflow logic rather than delegated entirely to the LLM.
4. **Preserve operational state** — successful and failed processing paths leave an observable record in the CRM or error log.

The resulting architecture combines **AI reasoning with deterministic automation**, creating a system that is both intelligent and operationally controllable.

---

## Why This Project Matters

RevenueOS v1 demonstrates more than an AI integration.

It combines:

**CRM data + business rules + AI decision-making + workflow automation + sales routing**

into a single operational system.

The goal is not to replace the sales team with AI. The goal is to **remove repetitive operational work from the sales process so human sellers can focus on higher-value activities**.

This is the type of problem Revenue Operations and GTM Engineering teams solve: turning fragmented GTM processes into reliable, automated systems that directly improve revenue operations.

---

## Project Status

**Status: Complete**

RevenueOS v1 has been implemented and tested end-to-end.

The workflow successfully:

* Retrieves lead data from HubSpot
* Normalizes CRM data
* Evaluates leads using AI
* Generates structured qualification results
* Determines qualification status
* Routes qualified leads by sales segment
* Assigns the appropriate representative
* Updates HubSpot with qualification results
* Creates sales follow-up tasks
* Disqualifies poor-fit leads

---

## Next Version

RevenueOS v1 establishes the inbound qualification layer of the broader RevenueOS platform.

Future versions can extend the system into additional GTM workflows such as:

* Sales prospecting
* Quote-to-cash automation
* Customer onboarding
* Customer health monitoring
* Revenue intelligence
* Unified revenue operations

**RevenueOS v1 is the foundation: turning inbound demand into an automated, sales-ready process.**