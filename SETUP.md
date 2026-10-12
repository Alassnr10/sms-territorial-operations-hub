# SMS Territorial Operations Hub V2 — Setup Guide

## Purpose and current state

The SMS Territorial Operations Hub is a template for local governments and municipal departments. It is designed to help field agents report public works and departmental activities, report incidents, and request a personal status update or brief by SMS.

The current public export is **workflows/sms-territorial-operations-hub-combined.public-final.json**, one combined n8n canvas. It contains the Main workflow section and the Send SMS sub-workflow section, with their Sticky Notes. Importing this JSON creates one inactive workflow; it does not automatically create two linked workflows.

The combined draft has not been imported into a clean n8n instance or tested end to end. Earlier simulated tests of a separate project workflow do not validate this combined public draft. Treat installation and compatibility as unverified until you configure and test your own copy.

### What the current workflow represents

- **UPDATE** stores an agent's report in UPDATES and prepares a simulated confirmation.
- **ALERT** stores an incident in INCIDENTS and reads active ALERT rules from ROUTING_RULES. Rules identify departments, not individual phone numbers. No real SMS notification is sent.
- **STATUS** reads UPDATES and INCIDENTS for the identified agent's ID and prepares a response.
- **BRIEF** prepares a summary from the agent's recent records.
- Outbound replies are split into SMS-sized segments and logged in MESSAGES as simulation results.

### Limits visible in this version

- No SMS provider is configured. The send node is a simulation placeholder; no SMS is sent or delivered.
- **Verify Gateway** is a placeholder condition that always passes. It does not authenticate a gateway.
- **Check Interpretation and Scope** is also a placeholder condition that always passes. It does not enforce an allowed subject or category scope.
- The workflow identifies an agent by the incoming phone value. Without gateway authentication, a caller could supply another agent's phone number. This design is not suitable for production or for handling real personal data.
- The ROUTING_RULES table has fields for UPDATE rules, but the current workflow reads active rules only for ALERT. UPDATE recipient routing is not implemented.
- The subject_key columns are part of the table schemas, but the current Register UPDATE and Register ALERT Incident nodes write null to them. The current workflow does not use subject_key as a search key.
- The imported workflow must remain inactive while you configure it. Use a Test URL only when you are ready to run a controlled test in your own instance.

## Prerequisites

- An n8n instance with the built-in Data Table, Webhook, and Execute Sub-workflow nodes used by this template.
- Permission to create Data Tables and workflows in the same n8n project.
- Your own OpenAI credential for the Information Extractor.
- Synthetic agent and message data for any test.

The JSON selects the model gpt-6-luna, but contains no OpenAI credential reference. After import, select your own OpenAI credential in OpenAI Chat Model and confirm that your account and n8n version support the selected model and structured extraction settings.

## Create the five Data Tables

Create the tables in the same n8n project as the workflows. In the project, open **Data tables**, choose **Create Data table**, select **From scratch**, and add each business column below with its exact name and type.

n8n adds system columns such as id, createdAt, and updatedAt. Do not add those as business columns. Names such as created_at and updated_at are separate project fields and must be created as listed.

The table schemas define column names and types. They do not define required, unique, or nullable constraints. The value lists below describe workflow conventions; they are not database-enforced enums. Optional values may be blank or null depending on the node and n8n version.

### AGENTS

Stores the agents whose incoming phone number can be matched by Main.

| Column | n8n type | Expected value or format |
|---|---|---|
| agent_id | string | Stable business identifier for the agent. |
| phone | string | Phone number stored as text. The workflow trims leading and trailing spaces only and does not add a country code. |
| name | string | Agent name or label. Use synthetic data in demonstrations. |
| department | string | Department name; free text. |
| area | string | Area name; free text. |
| access_level | string | department, area_coordinator, or registry_admin. |
| active | boolean | true for an enabled agent; false to disable the record. |
| created_at | date | ISO 8601 date or date-time. |

### UPDATES

Stores each accepted field report as its own row. incident_id can link a report to an existing incident.

| Column | n8n type | Expected value or format |
|---|---|---|
| update_id | string | Business identifier generated for the report. |
| source_message_id | string | Identifier of the inbound message recorded in MESSAGES. |
| department | string | Department of the matched agent. |
| area | string | Agent area or an area explicitly retained by the workflow. |
| subject | string | Report subject; no fixed list is enforced by the table. |
| subject_key | string | Included in the schema for compatibility. The current workflow writes null and does not use this field for searches. |
| status | string | unknown, planned, ongoing, blocked, or completed. |
| progress | number | A value from 0 to 100, or empty when not stated. |
| message | string | Original SMS text. |
| reported_event_at | date | ISO 8601 date or date-time when explicitly available; otherwise empty. The workflow must not invent a date. |
| created_at | date | ISO 8601 date-time when the row is created. |
| agent_id | string | Business identifier of the matched agent. |
| incident_id | string | Existing incident reference, or empty when not applicable. |
| share_requested | boolean | true only when sharing was explicitly requested; otherwise false. |
| previous_incident_status | string | Empty when not applicable; otherwise the prior incident status. |
| requested_incident_status | string | Empty when not applicable; an explicit request can be acknowledged or resolved. |
| application_status | string | pending, applied, or failed. |

### INCIDENTS

Stores incident reports. The initial description is retained here; later incident updates belong in UPDATES and can reference incident_id.

| Column | n8n type | Expected value or format |
|---|---|---|
| incident_id | string | Business identifier generated from the source message record. |
| source_message_id | string | Identifier of the inbound message recorded in MESSAGES. |
| category | string | Extracted category; free text, with no fixed list enforced by the table. |
| department | string | Department of the matched agent. |
| area | string | Area only when the message clearly locates the event there. |
| subject | string | Incident subject. |
| subject_key | string | Included in the schema for compatibility. The current workflow writes null and does not use this field for searches. |
| severity | string | unknown, low, medium, high, or critical. |
| description | string | Original incident report text. |
| status | string | reported, acknowledged, or resolved. |
| reported_event_at | date | ISO 8601 date or date-time when explicitly available; otherwise empty. |
| created_at | date | ISO 8601 date-time when the incident row is created. |
| updated_at | date | ISO 8601 date-time when the incident row is updated. |
| agent_id | string | Business identifier of the agent who reported the incident. |

### ROUTING_RULES

Stores deterministic routing criteria. In this version, Main reads active ALERT rules only; UPDATE routing is not implemented.

| Column | n8n type | Expected value or format |
|---|---|---|
| rule_id | string | Stable identifier for the rule. |
| message_type | string | UPDATE or ALERT. The current workflow applies ALERT rules only. |
| source_department | string | all or the exact source department. |
| update_trigger | string | For a future UPDATE rule: explicit_share or incident_status_changed. Not used by the current workflow. |
| category | string | all or an exact category. |
| area | string | all or an exact area. |
| department_to_notify | string | Exact department name. This field does not supply a recipient phone number. |
| minimum_severity | string | Empty for no threshold, or low, medium, high, or critical in ascending order. |
| include_unknown_severity | boolean | Whether an ALERT rule can match an incident with unknown severity. |
| active | boolean | true to include the rule in matching; false to ignore it. |

For ALERT rules, an absent optional filter is non-restrictive. A severity below minimum_severity does not match. Unknown severity is included only when include_unknown_severity is true. A matched department is not an individually addressable SMS recipient.

### MESSAGES

Provides traceability and deduplication for inbound messages and simulated outbound segments.

| Column | n8n type | Expected value or format |
|---|---|---|
| message_id | string | Business identifier for an inbound message or outbound segment. |
| direction | string | inbound or outbound. |
| provider | string | simulation in this template; no SMS provider is configured. |
| provider_message_id | string | Empty when no provider has supplied an identifier. |
| idempotency_key | string | Stable deduplication key. Inbound keys use the simulation: prefix; each outbound segment has its own key. |
| correlation_id | string | Links outbound replies to the originating inbound message. |
| phone_from | string | Source number stored as text. |
| phone_to | string | Intended destination number stored as text. |
| text | string | Original inbound SMS or simulated outbound segment. |
| intent | string | UPDATE, ALERT, STATUS, or BRIEF when known; otherwise empty. |
| processing_status | string | Inbound messages use received. A successful simulated outbound result uses simulated_not_sent. |
| delivery_status | string | A successful simulated outbound result uses not_sent_simulation. This does not mean delivered. |
| attempt_count | number | Attempt count; the simulation starts at 0. |
| error_code | string | Error code when present; otherwise empty. |
| created_at | date | ISO 8601 date-time when the row is created. |
| updated_at | date | ISO 8601 date-time when the row is updated. |

## Synthetic demonstration data

These values are fictional examples for a controlled demonstration. Do not replace them with real personal data in a public copy.

### TEST agent

Add this row to AGENTS:

| agent_id | phone | name | department | area | access_level | active | created_at |
|---|---|---|---|---|---|---|---|
| TEST-AGENT-001 | +999000000001 | Test Agent | hydraulique | TEST-AREA | department | true | 2026-01-01T00:00:00.000Z |

The phone value is a synthetic test value. It is not a real contact number.

### ALERT routing example

Add this row to ROUTING_RULES only for a demonstration:

| rule_id | message_type | source_department | update_trigger | category | area | department_to_notify | minimum_severity | include_unknown_severity | active |
|---|---|---|---|---|---|---|---|---|---|
| DEMO-ALERT-HIGH-ALL-001 | ALERT | all | (empty) | all | all | hydraulique | high | false | true |

This example matches high or critical ALERT incidents from any source department, category, or area. It identifies the hydraulique department but supplies no phone number, so no department SMS can be sent or simulated as delivered.

## Import the combined canvas and split Send SMS into its own workflow

The public JSON is a single combined canvas. It contains both workflow sections on one canvas, with the Send SMS section positioned below Main. Import it once using the workflow menu and **Import from File**. n8n currently documents selecting nodes and copying them between editor canvases with Cmd+C/Cmd+V on macOS or Ctrl+C/Ctrl+V on Windows/Linux.

1. Keep the imported combined workflow inactive.
2. In the combined canvas, select the full Send SMS section, including its Execute Sub-workflow Trigger, all processing nodes, and its four Sticky Notes. Copy the selected nodes.
3. Create a new workflow, paste the nodes, and save it as **SMS Territorial Operations Hub - Send SMS**. Confirm the trigger, connections, and notes are present. Keep this workflow inactive.
4. Return to the combined workflow. Remove the copied Send SMS section from that canvas and rename the remaining workflow **SMS Territorial Operations Hub - Main**.
5. In Main, open each Execute Workflow node and select the new Send SMS workflow:
   - Execute UPDATE SMS Reply
   - Execute ALERT SMS Reply
   - Execute STATUS SMS Reply
   - Execute BRIEF SMS Reply
6. Assign the Data Tables to the nodes listed below and select your OpenAI credential in OpenAI Chat Model.
7. Save both workflows and leave them inactive.

The combined JSON has empty Data Table selectors and empty sub-workflow selectors. Its OpenAI credential reference and instance webhookId are absent. Internal node IDs remain in the JSON to preserve graph connections. These references must be configured in your own n8n instance.

### Assign the Data Tables

After creating the tables, open each Data Table node and choose the corresponding table in your project.

| Table | Main workflow nodes |
|---|---|
| AGENTS | Find Agent |
| UPDATES | Register UPDATE, Get Agent UPDATES, Get BRIEF UPDATES |
| INCIDENTS | Register ALERT Incident, Get Agent INCIDENTS, Get BRIEF INCIDENTS |
| ROUTING_RULES | Get Active ALERT Routing Rules |
| MESSAGES | Find Existing Message, Register Inbound Message |

| Table | Send SMS workflow nodes |
|---|---|
| MESSAGES | Check Existing Outbound, Register Outbound, Save Send Result |

Do not rename columns. The listed nodes use the business fields above; n8n maintains its own system columns.

## Configure the OpenAI credential

Open **OpenAI Chat Model** in Main and select an OpenAI credential already created in your n8n instance. The public JSON contains no credential reference or secret. The model selection is gpt-6-luna; availability and permission depend on your OpenAI account and n8n version. Never paste an API key into a node parameter, Sticky Note, workflow export, or repository.

## Use the Webhook Test URL

The entry node is **Webhook — Receive SMS**. It accepts HTTP POST and is configured to respond through the Acknowledge Gateway node. The public JSON does not contain an instance-specific webhookId or Test URL.

For a later manual test, keep Main inactive, open the Webhook node, choose **Test URL**, and click **Listen for Test Event**. Send a synthetic JSON body such as:

    {
      "from": "+999000000001",
      "to": "+999000000099",
      "text": "STATUS",
      "idempotency_key": "DEMO-SETUP-STATUS-001"
    }

Use the Test URL displayed by your own n8n instance only in your local terminal. Do not copy it into this repository, a public issue, or a screenshot. Use a new idempotency_key for each distinct test message. Do not activate or publish the workflow to use this setup guide.

## Security and release limits

- This template has no real SMS provider or SMS credential. Simulated SMS Send does not send a message; successful simulated outbound segments are logged with processing_status simulated_not_sent and delivery_status not_sent_simulation.
- A department selected by ALERT routing is not a verified person or phone number. Department notifications remain unresolved when there is no specific recipient number.
- Verify Gateway does not authenticate requests. The current phone-number lookup is not proof of identity.
- Check Interpretation and Scope currently passes through a placeholder condition and does not enforce an approved V1 scope.
- The workflow does not provide production-grade authentication, authorization, abuse protection, or privacy safeguards. Do not use it with real public traffic or personal data.
- Keep both separated workflows inactive during setup. Do not publish them until the missing production security and operational controls have been designed and reviewed.
- The combined public JSON has not been imported or executed in a clean environment. Node compatibility, table mapping, credential access, child-workflow links, and end-to-end behavior remain to be verified on the target n8n instance.

## Official n8n references

- [Data Tables](https://docs.n8n.io/data/data-tables/)
- [Export and import workflows](https://docs.n8n.io/workflows/export-import/)
- [Webhook node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)
- [OpenAI credentials](https://docs.n8n.io/integrations/builtin/credentials/openai/)
- [Creator Hub submission and review information](https://support.n8n.io/article/becoming-an-n8n-template-creator-and-getting-help)
- [n8n Creator Dashboard](https://creators.n8n.io/)
