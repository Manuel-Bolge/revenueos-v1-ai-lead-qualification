# Error Logging & Resilience Schema

## Overview

RevenueOS v1 includes an error-handling path designed to prevent AI or API failures from silently dropping leads or incorrectly updating the CRM.

If the AI/API request fails or the AI response cannot be parsed into the expected JSON structure, the workflow routes the failure to a dedicated error-handling path before the lead reaches the CRM update stage.

The workflow extracts the available lead information and records the failure in PostgreSQL for operational investigation and future reprocessing.

## Database Schema

**Table:** `workflow_errors`

| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | Primary Key | Auto-incrementing record ID |
| `timestamp` | DateTime | Time the failure was recorded |
| `workflow` | String | Workflow identifier (`RevenueOS v1`) |
| `stage` | String | Workflow stage where the failure occurred |
| `error_message` | String | Error returned by the API, parser, or n8n |
| `lead_email` | String | Email address of the lead being processed |
| `retry_count` | Integer | Number of retry attempts; initially `0` |

## Failure Flow

```text
Lead
  ↓
AI Qualification
  ↓
AI/API or Parsing Failure
  ↓
Error Handling
  ↓
Extract Lead + Error Data
  ↓
PostgreSQL
  ↓
workflow_errors