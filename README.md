# HVAC AI Receptionist — Vapi + n8n MCP

> **Reference project** for an AI receptionist serving a fictional residential HVAC company. The repository contains n8n workflow exports that connect an MCP tool router to customer, service-area, appointment, and call-log operations. **It is not a turnkey or production-validated deployment.**

Built to demonstrate how an AI voice agent can handle common HVAC phone inquiries while keeping customer records in Google Sheets and appointments in Google Calendar.

## Overview

An HVAC caller may ask whether the company services their ZIP code, request an AC/heating visit, check an appointment, reschedule or cancel, or ask for a staff member. **Vapi is the conversational interface; n8n exposes tools; Google Sheets and Google Calendar supply demo records and scheduling.**


### Architecture

```mermaid
flowchart TD
    Caller[Caller] <--> Vapi[Vapi voice assistant]
    Vapi <--> MCP[n8n MCP Server Trigger\nHVAC - MCP Router]
    MCP --> Customer[lookup_customer]
    MCP --> Area[check_service_area]
    MCP --> Availability[check_availability]
    MCP --> Booking[book_appointment]
    MCP --> Appointment[lookup_appointment]
    MCP --> Reschedule[reschedule_appointment]
    MCP --> Cancel[cancel_appointment]
    MCP --> Log[call_log]
    MCP -.-> Missing[create_customer\nReferenced in router; export missing]
    Customer --> Sheets[(Google Sheets)]
    Appointment --> Sheets
    Booking --> Sheets
    Reschedule --> Sheets
    Cancel --> Sheets
    Log --> Sheets
    Availability --> Calendar[(Google Calendar)]
    Booking --> Calendar
    Reschedule --> Calendar
    Cancel --> Calendar
```

## Repository contents

All paths below are at the repository root (including the spaces in filenames).

| File | What it does |
| --- | --- |
| [`HVAC - MCP Router.json`](HVAC%20-%20MCP%20Router.json) | MCP Server Trigger and connections to the tool workflows. |
| [`HVAC TOOL - lookup_customer.json`](HVAC%20TOOL%20-%20lookup_customer.json) | Validates an email, reads the `Clients` sheet and returns matching records. |
| [`HVAC TOOL - check_service_area.json`](HVAC%20TOOL%20-%20check_service_area.json) | Compares a caller's ZIP code with a hardcoded demo ZIP allowlist. |
| [`HVAC TOOL - check_availability.json`](HVAC%20TOOL%20-%20check_availability.json) | Reads Google Calendar events and returns busy time windows, not a complete bookable-slot engine. |
| [`HVAC TOOL - book_appointment.json`](HVAC%20TOOL%20-%20book_appointment.json) | Validates a request, creates a Calendar event and appends an appointment record in Sheets. |
| [`HVAC TOOL - lookup_appointment.json`](HVAC%20TOOL%20-%20lookup_appointment.json) | Finds appointment records associated with an email in the `Appointment Log` sheet. |
| [`HVAC TOOL - reschedule_appointment.json`](HVAC%20TOOL%20-%20reschedule_appointment.json) | Updates a Calendar event and attempts to synchronize the appointment log. |
| [`HVAC TOOL - cancel_appointment.json`](HVAC%20TOOL%20-%20cancel_appointment.json) | Deletes a Calendar event and attempts to synchronize the appointment log. |
| [`HVAC TOOL -  Call Log.json`](HVAC%20TOOL%20-%20%20Call%20Log.json) | Normalizes a call summary and appends a row to the `Call Log` sheet. Note the **two spaces** after the hyphen in the filename. |

The router exposes **nine tool connections**.

## Supported demo use cases

- Identify an existing customer by email.
- Check whether a ZIP code belongs to the example service area.
- View busy windows from a designated demo Google Calendar.
- Create a demo appointment for a service type and customer.
- Look up, reschedule, or cancel an existing appointment.
- Record a call summary in Google Sheets.
- Support HVAC-specific dialogue in **Vapi**, if a properly configured Vapi assistant prompt and escalation rules are supplied.

The JSON workflows **do not by themselves implement** an emergency-dispatch system, customer-facing transfer routing, diagnosis, pricing, warranty decisions, a complete FAQ knowledge base, or automatic Vapi end-of-call webhook ingestion. Those behaviors need separate Vapi configuration and/or additional workflows.

## Prerequisites

1. A running n8n instance with **MCP Server Trigger** and **Call n8n Workflow Tool** support.
2. A Vapi account with an AI voice assistant and a supported MCP tool connection.
3. Google account credentials authorized in n8n for **Google Sheets** and **Google Calendar**.
4. A dedicated **demo** Google Sheet and Calendar, separate from real business operations.
5. A reachable, HTTPS-enabled n8n MCP endpoint and the authentication method supported by your n8n deployment.

Vapi usage, hosting, phone numbers, and LLM/voice services may incur costs.

## Setup

### 1. Create the demo Google data

Create a spreadsheet with at least these worksheets (match headers to the imported node mappings):

**`Clients`**

```csv
Email,Name,Phone,Customer ID
jordan.test@example.com,Jordan Test,+16025550101,CUST-DEMO-001
```

**`Appointment Log`**

```csv
Appointment ID,Calendar Event ID,Customer ID,Email,Service Type,Start ISO,End ISO,Status,ZIP,Notes
```

**`Call Log`**

```csv
Call ID,Caller Phone,Customer ID,Summary
```

Use synthetic data only. The current booking export produces some keys such as `StartTime` and `EndTime`, whereas the Google Sheets mapping expects `Start ISO` and `End ISO`; **align these before a booking test**. Also inspect the appointment cancellation and rescheduling mappings to avoid incorrect row changes.

Create a separate Google Calendar for sample HVAC service appointments. Choose one demo company timezone (for example, `America/Phoenix`) and use explicit ISO 8601 timestamps with offsets in test requests.

### 2. Import the child workflows into n8n

In n8n, use **Import from File** for each `HVAC TOOL - ...json` file. Open the imported workflows and:

- Replace existing spreadsheet and calendar references with **your** demo resources.
- Select or create your own Google credentials in each Google node.
- Check Code-node input and output shapes against the tool payloads you will send.
- Verify Google Sheets column mappings and any auto-mapping behavior.
- Run every child workflow with synthetic inputs before exposing it through MCP.
- Create/export the missing **create_customer** child workflow if new customer onboarding is part of your demo.

**Imported workflow IDs are environment-specific.** They will not automatically correspond to the IDs referenced in the router export.

### 3. Import and relink the MCP router

Import `HVAC - MCP Router.json`, then open every **Call n8n Workflow Tool** node. Select the corresponding imported child workflow in your instance. Remove or disable the `create_customer` tool until its missing workflow is available.

Review descriptions and schemas exposed to the voice assistant. Configure the MCP trigger's production URL and **actual supported authentication**; do not assume that an n8n API key alone is the correct bearer token for your MCP endpoint.

### 4. Configure Vapi

1. Create a Vapi assistant for a fictional HVAC company.
2. Use HVAC-specific prompt.
3. Add the n8n MCP endpoint using Vapi's currently supported MCP transport and authentication settings.
4. Confirm the intended tool names and argument formats appear and are callable from the assistant.
5. Use a Vapi test call to exercise one read-only tool (customer lookup or service-area check) before enabling appointment mutations.

**Voice guidance:** The assistant must not invent availability, diagnose or instruct on dangerous repairs, promise emergency dispatch, or expose other customers' calendar information. Urgent health/safety situations should be directed to appropriate emergency assistance and/or a human according to the demo's explicit escalation policy.

### 5. Test booking and changes

Suggested sequence with a synthetic caller:

1. Check a known supported ZIP code.
2. Look up a synthetic customer.
3. Request a calendar time window and inspect busy periods.
4. Book an appointment only after validating a free slot under the demo rules.
5. Verify the Calendar event **and** matching Appointment Log row.
6. Find the record by its customer identity.
7. Reschedule to a separately checked available time and verify synchronization.
8. Cancel and verify that the Calendar and Sheet agree.
9. Log a call summary and inspect `Call Log`.

## References

- [n8n documentation](https://docs.n8n.io/)
- [Vapi documentation](https://docs.vapi.ai/)
- [Google Sheets API](https://developers.google.com/workspace/sheets/api)
- [Google Calendar API](https://developers.google.com/workspace/calendar/api)

## Author

Maintained by [NAIMNoamane](https://github.com/NAIMNoamane).

Repository: [AI-Receptionist-for-HVAC-companies](https://github.com/NAIMNoamane/AI-Receptionist-for-HVAC-companies)

---

*This project is a reusable reference implementation. Adapt the company's service area, policies, booking rules, voice prompt, and integrations before any client-specific deployment.*
